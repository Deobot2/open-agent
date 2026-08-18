# 07 — Data Model

> Skeleton. Illustrative DDL — enough to show shape and relationships, not
> migration-ready. Indexes and constraints are indicative.

## 1. Store map

| Store | Holds | Why there |
|---|---|---|
| **PostgreSQL** | Accounts, plans, agents, sessions, slot ledger, billing | Relational truth, transactions |
| **Redis** | Slot counters, leases, rate buckets, entitlement cache | Speed on the hot path |
| **ClickHouse** | Usage events, cost attribution, analytics | Append-only, huge, aggregate-heavy |
| **S3** | Transcripts, artifacts, model weights | Large blobs |

Rule of thumb: **if losing it breaks billing, it is in Postgres.** If losing it
costs a rebuild, Redis. If it is append-only and enormous, ClickHouse.

## 2. PostgreSQL

### Identity

```sql
CREATE TABLE orgs (
    id              UUID PRIMARY KEY,
    name            TEXT NOT NULL,
    plan_id         TEXT NOT NULL REFERENCES plans(id),
    status          TEXT NOT NULL DEFAULT 'active',   -- active|suspended|closed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL REFERENCES orgs(id),
    email           CITEXT NOT NULL UNIQUE,
    role            TEXT NOT NULL DEFAULT 'member',   -- owner|admin|member
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_keys (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL REFERENCES orgs(id),
    prefix          TEXT NOT NULL UNIQUE,             -- oa_live_xxxx, plaintext
    key_hash        TEXT NOT NULL,                    -- Argon2id
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    last_used_at    TIMESTAMPTZ,
    revoked_at      TIMESTAMPTZ
);
```

### Plans & entitlements

```sql
CREATE TABLE plans (
    id              TEXT PRIMARY KEY,                 -- trial|solo|pro|team|...
    display_name    TEXT NOT NULL,
    price_cents     INTEGER NOT NULL,
    is_public       BOOLEAN NOT NULL DEFAULT true
);

-- Immutable, versioned. Never UPDATE; always INSERT a new version.
CREATE TABLE entitlement_versions (
    id              UUID PRIMARY KEY,
    plan_id         TEXT NOT NULL REFERENCES plans(id),
    version         INTEGER NOT NULL,
    limits          JSONB NOT NULL,                   -- doc 01 §6 shape
    features        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (plan_id, version)
);

CREATE TABLE org_overrides (
    org_id          UUID PRIMARY KEY REFERENCES orgs(id),
    limits_patch    JSONB NOT NULL DEFAULT '{}',      -- merged over plan limits
    features_patch  JSONB NOT NULL DEFAULT '{}',
    expires_at      TIMESTAMPTZ
);
```

Immutability here is what makes historical entitlement questions answerable —
sessions reference the exact version they ran under.

### Agents & sessions

```sql
CREATE TABLE agents (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL REFERENCES orgs(id),
    name            TEXT NOT NULL,
    model_class     TEXT NOT NULL,                    -- S|M|L
    system_prompt   TEXT,
    tools           JSONB NOT NULL DEFAULT '[]',
    config          JSONB NOT NULL DEFAULT '{}',      -- max_steps, temperature...
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    archived_at     TIMESTAMPTZ
);
CREATE INDEX ON agents (org_id) WHERE archived_at IS NULL;

CREATE TABLE sessions (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL REFERENCES orgs(id),
    agent_id        UUID NOT NULL REFERENCES agents(id),
    slot_id         UUID,
    state           TEXT NOT NULL,                    -- see doc 05 state machine
    priority        TEXT NOT NULL,                    -- P0|P1|P2
    entitlement_id  UUID REFERENCES entitlement_versions(id),  -- what it ran under
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at        TIMESTAMPTZ,
    end_reason      TEXT,                             -- completed|stopped|reaped|error
    step_count      INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX ON sessions (org_id, started_at DESC);
CREATE INDEX ON sessions (state) WHERE ended_at IS NULL;
```

### Slot ledger — billing truth

```sql
-- Append-only. The durable record behind Redis counters.
CREATE TABLE slot_ledger (
    id              BIGSERIAL PRIMARY KEY,
    slot_id         UUID NOT NULL,
    org_id          UUID NOT NULL REFERENCES orgs(id),
    session_id      UUID REFERENCES sessions(id),
    event           TEXT NOT NULL,                    -- acquire|renew|release|reap|reject
    at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    meta            JSONB NOT NULL DEFAULT '{}'
);
CREATE INDEX ON slot_ledger (org_id, at DESC);
CREATE INDEX ON slot_ledger (slot_id);

-- Rolled up hourly for cost attribution and overcommit tuning.
CREATE TABLE slot_seconds_hourly (
    org_id          UUID NOT NULL,
    hour            TIMESTAMPTZ NOT NULL,
    slot_seconds    BIGINT NOT NULL,
    active_seconds  BIGINT NOT NULL,                  -- actually generating
    idle_seconds    BIGINT NOT NULL,
    PRIMARY KEY (org_id, hour)
);
```

`active_seconds / slot_seconds` **is the duty cycle** — the number the whole
economic model in doc 13 turns on, and the input to the overcommit ratio (D-06).
It is worth capturing from day one even before anything consumes it.

### Models

```sql
CREATE TABLE models (
    id              TEXT PRIMARY KEY,                 -- oa-m-2026-01
    model_class     TEXT NOT NULL,                    -- S|M|L
    weights_uri     TEXT NOT NULL,
    engine          TEXT NOT NULL,                    -- vllm|sglang
    engine_args     JSONB NOT NULL DEFAULT '{}',
    pool            TEXT NOT NULL,
    status          TEXT NOT NULL,                    -- registered|canary|active|draining|retired
    eval_results    JSONB,
    promoted_at     TIMESTAMPTZ
);
CREATE UNIQUE INDEX ON models (model_class) WHERE status = 'active';
```

That partial unique index enforces exactly one active model per class — a cheap
constraint that prevents an entire class of routing ambiguity.

## 3. Redis keyspace

| Key | Type | TTL | Purpose |
|---|---|---|---|
| `slots:active:{org_id}` | counter | — | **Live slot count.** Invariant I-1 |
| `lease:{slot_id}` | hash | 60 s | Lease record + expiry |
| `bucket:{slot_id}` | hash | session | Token bucket (tokens, last_refill) |
| `ent:{org_id}` | string | 15 min | Cached entitlements |
| `auth:{key_prefix}` | string | 60 s | Cached key verification |
| `affinity:{session_id}` | string | 1 h | Replica for prefix affinity |
| `queue:{pool}:{priority}` | list | — | Pending scheduling requests |

Persistence: AOF `everysec` + replica. Redis is rebuildable from Postgres, but
a rebuild means an admissions pause — so it is treated as important, just not
authoritative.

## 4. ClickHouse

```sql
CREATE TABLE usage_events (
    ts              DateTime64(3),
    org_id          UUID,
    session_id      UUID,
    step_id         UInt32,
    model_id        LowCardinality(String),
    model_class     LowCardinality(String),
    pool            LowCardinality(String),
    priority        LowCardinality(String),
    prompt_tokens       UInt32,
    cached_tokens       UInt32,   -- prefix cache hits
    completion_tokens   UInt32,
    ttft_ms             UInt32,
    duration_ms         UInt32,
    -- Optimization attribution (doc 10) — present from day one, zero until used
    tokens_saved        UInt32 DEFAULT 0,
    stages_applied      Array(LowCardinality(String)) DEFAULT [],
    est_cost_micros     UInt64 DEFAULT 0
) ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)
ORDER BY (org_id, ts, session_id);
```

`tokens_saved` and `stages_applied` are in the schema from the start
deliberately: when the optimization systems land, their measured savings must
be queryable against the same table as the baseline. Adding these columns later
means no before/after comparison. Zero-cost now, essential later.

## 5. S3 layout

```
s3://oa-models/{family}/{version}/            # weights
s3://oa-transcripts/{org_id}/{session_id}/    # SSE-CMK per tenant
s3://oa-artifacts/{org_id}/{session_id}/      # tool outputs, files
s3://oa-backups/postgres/{date}/
```

Transcripts encrypted with a per-tenant key; deletion is per-tenant and honored
by lifecycle policy (doc 12).

## 6. Retention

| Data | Retention | Driver |
|---|---|---|
| Slot ledger | 25 months | Billing disputes |
| Usage events | 25 months | Same |
| Sessions | 13 months | Support |
| Transcripts | 30 d default, tenant-configurable | Privacy |
| Logs / traces | 30 d / 7 d | Cost |

## 7. Deferred

- Partitioning strategy for `slot_ledger` at scale
- Read replicas and routing
- Per-tenant encryption key rotation
- Soft-delete vs. hard-delete semantics
- Backup/restore runbook, PITR

---
*Next: [08 — API Surface](08-api-surface.md)*
