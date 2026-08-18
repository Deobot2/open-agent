# 03 — Infrastructure Layout

> Skeleton. Node counts and hardware are **placeholders** sized for a Phase 2
> launch (~500–1,000 paying slots). Re-derive from doc 13 before ordering.

## 1. Topology

Single region, three availability zones, one logical cluster.

```
                        ┌─────────────────────────┐
   Internet ──────────► │  Anycast / CDN / WAF    │
                        └───────────┬─────────────┘
                                    │
                        ┌───────────▼─────────────┐
                        │  TIER 0 — Edge          │  2–4 × CPU nodes
                        │  oa-gateway, LB, TLS    │
                        └───────────┬─────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
┌───────▼────────┐        ┌─────────▼────────┐        ┌─────────▼────────┐
│ TIER 1         │        │ TIER 2           │        │ TIER 4           │
│ Control        │        │ Agent Runtime    │        │ Data / Platform  │
│ 3 × CPU        │        │ 6–20 × CPU       │        │ 5–8 × CPU+NVMe   │
│ (quorum)       │        │ (horizontal)     │        │ PG/Redis/CH/NATS │
└────────────────┘        └─────────┬────────┘        └──────────────────┘
                                    │
                        ┌───────────▼─────────────────────────────┐
                        │ TIER 3 — Inference (GPU)                │
                        │  Pool-S │ Pool-M │ Pool-L │ Pool-B │ Pool-E │
                        └─────────────────────────────────────────┘
                                    │
                        ┌───────────▼─────────────┐
                        │ TIER 5 — Observability  │  2 × CPU+disk
                        └─────────────────────────┘
```

## 2. Node classes

| Tier | Class | Shape (placeholder) | Count | Scales on |
|---|---|---|---|---|
| 0 | Edge | 16 vCPU / 32 GB / 10 GbE | 2–4 | Concurrent connections |
| 1 | Control | 16 vCPU / 64 GB | 3 | Fixed (quorum) |
| 2 | Agent | 32–64 vCPU / 128 GB | 6–20 | **Active slots** |
| 3 | GPU | see §3 | see §3 | Sustained tok/s demand |
| 4 | Data | 32 vCPU / 256 GB / NVMe | 5–8 | Storage + IOPS |
| 5 | Observ. | 16 vCPU / 64 GB / big disk | 2 | Retention window |

**Tier 2 is CPU-only and cheap.** Agent runtimes orchestrate; they do not do
math. One agent session is a coroutine holding a lease, a context buffer, and
some HTTP connections — budget ~200–500 MB per active slot and pack them
densely. This tier is what scales with the product's headline number, so
keeping it off GPUs is a deliberate cost decision.

## 3. GPU pools

Pools are the scheduling unit. Each is homogeneous so the scheduler can treat
replicas as interchangeable.

| Pool | Serves | GPUs / replica | Parallelism | Phase 2 replicas | Purpose |
|---|---|---|---|---|---|
| **Pool-S** | Class S (7–14B) | 1 | none | 4–8 | High-QPS small work, drafts, routing |
| **Pool-M** | Class M (27–70B) | 2 | TP2 | 4–8 | Daily driver |
| **Pool-L** | Class L (MoE) | 8 | TP8 / EP | 1–3 | Flagship reasoning |
| **Pool-B** | Batch / P2 | 1–2 | any | 0–6 (spot) | Off-peak, preemptible, background |
| **Pool-E** | Embeddings, rerank, guards, draft models | 1 | none | 2–4 | Cheap accelerators; feeds doc 10 |

### Node shapes

| Pool | Node (placeholder) | Notes |
|---|---|---|
| S / E | 4–8 × mid-tier GPU (24–48 GB) | Density over peak FLOPs |
| M | 8 × 80 GB HBM, NVLink | Two TP2 replicas per half-node |
| L | 8 × 80 GB HBM, NVLink, 3.2 Tbps IB | One replica = one node |
| B | Spot/preemptible anything | Must tolerate eviction |

**Pool-E is small and strategically important.** Almost every optimization in
doc 10 needs a cheap model: routers, draft models for speculative decoding,
embedding models for semantic cache, summarizers for compaction, safety
classifiers. Reserving that capacity now means those systems land without a
hardware conversation.

## 4. Placement rules

| Rule | Reason |
|---|---|
| One inference replica never spans nodes | Cross-node TP needs IB and dies to network jitter |
| Pool-L replicas each get a dedicated node | MoE + long context fills HBM; no co-tenancy |
| Agent runtimes are AZ-spread | An AZ loss should cost ≤ 1/3 of slots |
| Control quorum is AZ-spread, one per AZ | Survive one AZ |
| Postgres primary + sync replica in different AZs | RPO ≈ 0 |
| Pool-B is spot-only, never serves P0/P1 | Evictions must be invisible to paying interactive work |

## 5. Network

| Link | Bandwidth | Notes |
|---|---|---|
| Internet → Edge | 10–25 Gbps | Streaming tokens is small; TLS is the cost |
| Edge ↔ Agent | 10 GbE | Prompt bodies, moderate |
| Agent ↔ GPU | 25 GbE | **Long contexts make this real.** 256k-token prompts are MB-scale |
| Intra-node GPU | NVLink | TP traffic |
| Inter-node GPU | 3.2 Tbps IB | Only if a replica ever spans nodes (avoid; see §4) |
| Node → Storage | 25 GbE | Weight loading: tens to hundreds of GB per cold start |

### Egress policy

Agent tool calls reach the internet. That is a security boundary, not a
networking detail — all tool egress is forced through `oa-toolproxy` with
per-tenant allow/deny, and no GPU or control node has a default route out.
See [12 — Security & Tenancy](12-security-and-tenancy.md).

## 6. Storage

| Store | Tech | Size (placeholder) | Retention |
|---|---|---|---|
| Model weights | S3 + per-node NVMe cache | 5–20 TB | Permanent |
| System of record | PostgreSQL | 500 GB | Permanent |
| Hot state | Redis (AOF + replica) | 64 GB | Ephemeral, rebuildable |
| Usage warehouse | ClickHouse | 2–10 TB | 25 months (billing disputes) |
| Transcripts | S3, tenant-encrypted | Grows with usage | Tenant-configurable, default 30 d |
| KV cache spill | Local NVMe | 2–4 TB / GPU node | Ephemeral |
| Logs / metrics / traces | Loki / Prom / Tempo | 2–5 TB | 30 d / 90 d / 7 d |

**Weight distribution matters more than it looks.** A cold Pool-L replica pulls
hundreds of GB. Pull from S3 once per node, cache on NVMe, and pre-warm on
deploy — otherwise a scale-up event takes 20+ minutes and the autoscaler is
useless. Budget the NVMe for two full model versions so a rollback is instant.

## 7. Environments

| Env | Purpose | GPU |
|---|---|---|
| `dev` | Local / per-engineer | Mocked inference or 1 small GPU |
| `staging` | Full stack, prod-shaped, synthetic tenants | 1 × S, 1 × M |
| `prod` | Customers | Full fleet |
| `bench` | **Isolated, for doc 13 + doc 10 measurement** | 1 node per pool class |

`bench` is not optional. Every number in this document set is a placeholder
until it produces the real one, and every optimization needs a clean room to
prove its savings. Standing it up is Phase 0 work.

## 8. Capacity levers

When demand rises, in cost-efficiency order:

1. Raise batch size / concurrency per replica (free until latency SLO bends)
2. Enable/raise prefix cache hit rate (doc 10, Stage 2)
3. Shift P2 work to Pool-B / off-peak
4. Route more traffic down a class (L → M → S) via Stage 3
5. Add replicas to the saturated pool
6. Add nodes
7. Add a region

Steps 1–4 are software and are exactly where the optimization systems live.
Steps 5–7 cost money. **The whole point of doc 10 is to stay in steps 1–4 for
as long as possible.**

## 9. Deferred

- Multi-region, geo-routing, data residency
- Owned hardware / colo (see D-01)
- Bare-metal vs. Kubernetes for GPU nodes (leaning K8s + device plugin)
- Disaster recovery drills, RTO/RPO targets
- Dedicated single-tenant pools for Enterprise

---
*Next: [04 — Inference Layer](04-inference-layer.md)*
