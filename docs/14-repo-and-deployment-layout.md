# 14 — Repo & Deployment Layout

> Skeleton. Proposed tree — nothing here exists yet beyond `docs/`.

## 1. Monorepo

One repo through Phase 3. The services are small and change together; the
coordination cost of splitting exceeds the benefit until teams outgrow it.

```
open-agent/
├── PLAN.md                     # master plan
├── docs/                       # this document set
│
├── services/
│   ├── gateway/                # Go — edge, auth, routing, SSE
│   ├── accounts/               # Go — orgs, users, keys
│   ├── plans/                  # Go — entitlements
│   ├── slots/                  # Go — THE SLOT BROKER
│   ├── scheduler/              # Go — placement, fair share
│   ├── registry/               # Go — models, deployments
│   ├── billing/                # Go — subscriptions
│   ├── toolproxy/              # Go — sandboxed tool egress
│   └── agent-runtime/          # Python — the agent loop
│       ├── runtime/
│       │   ├── loop.py         # doc 05 §6
│       │   ├── context.py
│       │   └── heartbeat.py
│       └── optimizations/      # ← doc 10 lives here
│           ├── base.py         # Optimization protocol, StageInput/Output
│           ├── registry.py     # stage ordering, per-tenant enablement
│           ├── admit/          # Stage 1  (stubs)
│           ├── assemble/       # Stage 2  (stubs)
│           ├── route/          # Stage 3  (stubs)
│           ├── decode/         # Stage 4  (stubs)
│           └── settle/         # Stage 5  (stubs)
│
├── libs/
│   ├── go/                     # shared Go: authn, telemetry, errors, config
│   └── python/                 # shared Python: clients, tokenizers, types
│
├── proto/                      # gRPC contracts (source of truth for internal APIs)
│   ├── slots/v1/
│   ├── plans/v1/
│   └── scheduler/v1/
│
├── deploy/
│   ├── terraform/              # VPC, nodes, GPU pools, storage
│   ├── helm/                   # per-service charts
│   ├── k8s/                    # base manifests, GPU device plugin
│   └── engines/                # vLLM/SGLang configs per model class
│
├── db/
│   ├── migrations/             # Postgres (doc 07)
│   └── clickhouse/
│
├── bench/                      # ⚠️ Phase 0 — produces every number in doc 13
│   ├── throughput/             # tok/s per node per class
│   ├── context-curve/          # throughput vs. context length
│   ├── prefix-cache/           # hit rate on agent traffic
│   └── replay/                 # recorded traffic for doc 10 offline eval
│
├── evals/                      # quality gates (doc 11 §7)
│   ├── capability/
│   ├── agent-behavior/
│   ├── regression/
│   └── safety/
│
└── tools/                      # dev scripts, load gen, local stack
```

Two directories deserve their prominence: **`bench/`** gates every economic
claim in doc 13, and **`optimizations/`** is where your systems land — laid out
as five empty stage packages so the shape is set before the first one arrives.

## 2. Local development

```bash
make dev-up        # Postgres, Redis, NATS, ClickHouse, MinIO (docker compose)
make dev-mock      # mock inference — no GPU required
make dev-gpu       # single small model on a local GPU
make test          # unit + integration
make bench         # against the bench env
```

**A GPU must not be required to work on the platform.** Mock inference returns
deterministic token streams at a configurable rate, which exercises the slot,
rate-limit, and streaming paths — the parts most likely to break — without
hardware. Most of this codebase is orchestration, and orchestration is testable
without accelerators.

## 3. Environments

| Env | Deploy trigger | Data | GPU |
|---|---|---|---|
| `dev` | Local | Synthetic | Mock or 1 small |
| `staging` | Merge to `main` | Synthetic tenants | 1×S, 1×M |
| `bench` | Manual | Recorded traffic | 1 per pool class |
| `prod` | Tagged release + approval | Real | Full fleet |

## 4. CI/CD

```
PR  ──► lint ─► unit ─► integration ─► build images ─► deploy staging ─► smoke
                                                                          │
tag ──► evals ─► approval ─► canary (1 replica) ─► ramp ─► prod ─► monitor
                                                                    │
                                                          auto-rollback on
                                                          SLO/eval regression
```

Gates that block a release:

| Gate | Blocks on |
|---|---|
| Unit + integration | Any failure |
| **Slot invariant tests** | Any failure — I-1..I-7 (doc 02 §3) |
| Eval suite | Quality regression vs. current active |
| Canary | SLO breach within 15 min |

The slot invariant suite is called out separately because a bug there breaks the
product's central promise silently — over-admission does not throw an error, it
just quietly gives away capacity. Concurrency tests hammering `acquire` from
many goroutines belong in CI from the first commit.

## 5. Deployment order

Dependency order for a cold start:

```
1. Platform    Postgres, Redis, NATS, ClickHouse, S3
2. Control     accounts → plans → slots → registry → scheduler → billing
3. Inference   pull weights, warm replicas, mark ready
4. Data        agent-runtime, toolproxy
5. Edge        gateway (last — opens the door only when everything behind it is up)
```

Rolling updates run in reverse: drain edge first, so no request arrives at a
service that is mid-restart.

## 6. Configuration

| Layer | Mechanism |
|---|---|
| Static | YAML per env, in repo |
| Secrets | Vault, injected at runtime |
| Dynamic | Feature flags service, hot-reload |
| Per-tenant | Entitlements + overrides (doc 01 §6) |

**Rate ceilings and optimization toggles are dynamic, never static.** Both need
to change without a deploy — the kill switch in doc 10 §4 is worthless if
flipping it requires a release.

## 7. Deferred

- Multi-repo split
- Blue/green vs. canary for GPU pools
- GitOps (ArgoCD/Flux)
- Preview environments per PR

---
*Next: [15 — Roadmap](15-roadmap.md)*
