# Open Agent

A hosting platform for advanced open-source LLMs, sold as **agents** rather than
as tokens.

> **Unlimited tokens every month. A fixed number of agents that can run at once.**

Customers buy concurrency — how many agents may work simultaneously — not a
token allowance. There is no meter, no top-up, and no overage bill.

---

## Status

**Planning / skeleton — v0.1.** No implementation yet. This repository currently
holds the architectural blueprint the build will follow.

Every quantitative figure in these documents is a **placeholder** pending the
Phase 0 benchmark.

## Start here

📄 **[PLAN.md](PLAN.md)** — the master plan.

### Document set

| # | Document | Answers |
|---|---|---|
| 01 | [Product & Plans](docs/01-product-and-plans.md) | What we sell; the tier table |
| 02 | [Architecture Overview](docs/02-architecture-overview.md) | Services, lifecycle, invariants |
| 03 | [Infrastructure Layout](docs/03-infrastructure-layout.md) | Nodes, GPU pools, network, storage |
| 04 | [Inference Layer](docs/04-inference-layer.md) | Engines, batching, the KV cache constraint |
| 05 | [Agent Runtime & Slots](docs/05-agent-runtime-and-slots.md) | Slot lifecycle, admission, the agent loop |
| 06 | [Control Plane](docs/06-control-plane.md) | Each control service and its contract |
| 07 | [Data Model](docs/07-data-model.md) | Postgres / Redis / ClickHouse schema |
| 08 | [API Surface](docs/08-api-surface.md) | Public and internal endpoints |
| 09 | [Metering & Billing](docs/09-metering-and-billing.md) | Meter heavily, bill flatly |
| 10 | [**Optimization Framework**](docs/10-optimization-framework.md) | **Where token-reduction systems plug in** |
| 11 | [Observability](docs/11-observability.md) | Metrics, SLOs, the eval gate |
| 12 | [Security & Tenancy](docs/12-security-and-tenancy.md) | Isolation, sandboxing, egress |
| 13 | [Capacity & Economics](docs/13-capacity-and-economics.md) | The profitability math |
| 14 | [Repo & Deployment Layout](docs/14-repo-and-deployment-layout.md) | Directory tree, CI/CD |
| 15 | [Roadmap](docs/15-roadmap.md) | Build phases |
| 16 | [Agent Capability Layer](docs/16-agent-capability-layer.md) | Skills, tools, plugins, auto-skill maker |
| 17 | [Optimization Catalog](docs/17-optimization-catalog.md) | The named optimizations, placed and costed |
| 18 | [Ultra-Thinking Mode](docs/18-ultra-thinking.md) | Sub-agents on a shared token budget |
| 19 | [Contributed Compute & Credits](docs/19-contributed-compute.md) | User hardware for subscription discounts |
| 20 | [Free Tier & Public Agent](docs/20-free-tier-and-public-agent.md) | Growth funnel and its cost containment |
| — | [Glossary](docs/GLOSSARY.md) | Terms used precisely |

## The core idea

Selling unlimited tokens is only safe if one subscription can never cost more
than a known ceiling. That ceiling comes from making the **concurrency slot**
the unit of sale, and giving every slot a hard tokens-per-second limit.

```
worst_case_slot_cost = rate_ceiling × seconds_per_month × cost_per_token
```

Price above that and the plan cannot lose money. Then, because real agents idle
far more than they generate:

> **Price for the worst case. Profit from the average.**

Every optimization added later pushes the average further below the ceiling
already priced for. See [docs/13](docs/13-capacity-and-economics.md) for the
worked model and [docs/10](docs/10-optimization-framework.md) for the seams
those optimizations mount to.
