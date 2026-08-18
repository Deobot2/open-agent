# 06 — Control Plane

> Skeleton. One section per service: what it owns, its interface, its failure
> mode. Implementation is stubs.

## Shared rules

1. Each service owns its tables. Cross-service reads go through gRPC, never a
   foreign SELECT — this is what lets any of them be split out later.
2. All are stateless processes; state lives in Postgres/Redis.
3. All expose `/health`, `/health/ready`, `/metrics`.
4. All decisions that affect a customer emit an audit event.
5. **No service hardcodes a plan tier.** Tier behavior comes from entitlements
   (doc 01 §6). A grep for `"pro"` in service code is a bug.

---

## `oa-accounts`

**Owns:** organizations, users, seats, API keys, sessions, OAuth.

```
POST   /internal/v1/auth/verify        # token -> {tenant_id, user_id, scopes}
GET    /internal/v1/tenants/{id}
POST   /v1/orgs, /v1/orgs/{id}/members, /v1/api-keys
```

- API keys stored as Argon2id hashes; prefix (`oa_live_…`) kept plaintext for lookup.
- Key verification is cached in Redis for 60 s — it is on every request's hot path.
- Seats ≠ slots. Seats are people; slots are concurrency. Never conflate them.

**Down:** gateway serves cached auth for 60 s, then 503.

---

## `oa-plans`

**Owns:** plan definitions, entitlement resolution, feature flags, overrides.

```
GET    /internal/v1/entitlements/{tenant_id}   # -> the doc 01 §6 object
POST   /internal/v1/plans                      # admin
POST   /internal/v1/overrides/{tenant_id}      # per-tenant tweak
```

- Entitlements are **versioned and immutable**; a change writes a new version.
  This makes "what were they entitled to at the time?" answerable during a
  billing dispute.
- Cached in Redis, 15 min TTL, explicit invalidation on change.
- Overrides layer on top of the plan (enterprise deals, temporary raises,
  optimization A/B assignment).

**Down:** last-known entitlements served from cache; no plan changes take effect.

---

## `oa-slots` — the important one

**Owns:** slot counters, leases, the durable slot ledger, admission decisions.

```
POST   /internal/v1/slots/acquire      # -> Lease | Rejection
POST   /internal/v1/slots/renew
POST   /internal/v1/slots/release
GET    /internal/v1/slots/active/{tenant_id}
```

Full mechanism in [05](05-agent-runtime-and-slots.md). What matters here:

- **Atomic acquire via Redis Lua.** Non-negotiable (doc 05 §3).
- **Ledger in Postgres** is the billing and rebuild source of truth.
- **Reaper** sweeps expired leases every 10 s.
- **p99 < 5 ms** — it is in front of every session start.

This service enforces the headline product limit. It gets the tightest SLO, the
most tests, and the loudest alerts of anything in the control plane.

**Down:** no new sessions (fail closed); running sessions continue on cached
leases with self-extension (fail open). See invariants I-3/I-4.

---

## `oa-scheduler`

**Owns:** placement of inference work onto replicas, priority queues, fair share.

```
POST   /internal/v1/schedule           # -> {replica_url, queue_position, eta}
GET    /internal/v1/pools              # live pool state
```

Placement inputs, in priority order:

1. Model class (hard filter)
2. Priority class — P0 never queues behind P1/P2
3. **Prefix affinity** — prefer the replica holding this session's prefix (doc 04 §6)
4. Current load / queue depth
5. Pool health

Fair share: per-tenant weighted, so one tenant's 60 slots cannot starve a
tenant's 2. Weight ∝ plan priority, capped so no single tenant exceeds a
configured share of a pool.

**Down:** runtime falls back to direct round-robin over healthy replicas. Loses
prefix affinity and fairness — degraded but serving, which is the right trade.

---

## `oa-registry`

**Owns:** model classes, model versions, engine configs, replica inventory.

```
GET    /internal/v1/classes/{class}/active     # -> deployment config
POST   /internal/v1/models                     # register a version
POST   /internal/v1/models/{id}/promote        # canary -> active
POST   /internal/v1/models/{id}/drain
```

- Model lifecycle: `registered → canary → active → draining → retired`.
- Promotion requires passing the eval gate (doc 11).
- Keeps the previous `active` version warm for instant rollback.
- Class → model is the indirection that makes model swaps a config change.

**Down:** cached deployment config; no promotions or rollouts.

---

## `oa-billing`

**Owns:** subscriptions, payment provider, invoices, dunning.

```
POST   /internal/v1/subscriptions
POST   /webhooks/payments               # provider callbacks
GET    /v1/billing/invoices
```

- Payment provider is the source of truth for payment state; we mirror it.
- Subscription state change → `oa-plans` entitlement update → Redis invalidation.
- Because tokens are unlimited, **invoicing is flat and boring** — no metered
  overage, no usage-based line items. Usage data is for *our* cost attribution
  (doc 09), not for customer charges.

**Down:** no signups or plan changes. Existing service unaffected — never gate
running agents on billing availability.

---

## Control plane deployment

| Property | Value |
|---|---|
| Nodes | 3, one per AZ |
| Model | All services on all nodes, stateless |
| LB | Internal, health-checked |
| Deploy | Rolling, one node at a time |
| Rollback | Previous image, < 2 min |

Small enough to co-locate every service on every control node in v0.1. Split
only when one service's scaling or blast radius demands it.

---
*Next: [07 — Data Model](07-data-model.md)*
