# 01 — Product & Plans

> Skeleton. All prices and limits are **placeholders** pending the benchmark in
> [13 — Capacity & Economics](13-capacity-and-economics.md).

## 1. The offer

**Unlimited tokens. Limited concurrent agents.**

The customer buys *N simultaneous agents*. Each agent may run continuously for
the entire billing period. There is no monthly token allowance, no top-up, no
overage invoice.

What is actually constrained, and stated plainly in the plan:

| Constrained | Not constrained |
|---|---|
| How many agents run **at once** | How many tokens per month |
| How fast one agent generates (**tok/s ceiling**) | How long an agent runs |
| Which model classes the plan may reach | How many agents you create over time |
| Context window ceiling | Number of conversations/projects |

This is the honest framing of "unlimited": there is no meter running against a
balance. There is a **speed limit and a lane count**.

## 2. The unit: an *active agent slot*

> An **active agent** is a session that currently holds a slot lease.

- Creating an agent is free and unlimited. Agents are just configuration.
- **Running** one consumes a slot for as long as it is active.
- Slots are released on completion, on explicit stop, or by idle reaping.
- Slot count is enforced at admission — see [05 — Agent Runtime & Slots](05-agent-runtime-and-slots.md).

A customer on a 5-slot plan may own 500 agents and run any 5 of them at a time.

## 3. Plan tiers — placeholder table

| | **Trial** | **Solo** | **Pro** | **Team** | **Business** | **Enterprise** |
|---|---|---|---|---|---|---|
| Price / mo | $0 | $29* | $99* | $399* | $1,299* | Custom |
| **Active agents** | 1 | 2 | 5 | 20 | 60 | Custom |
| Monthly tokens | 5M cap† | Unlimited | Unlimited | Unlimited | Unlimited | Unlimited |
| Model classes | S | S, M | S, M, L | S, M, L | S, M, L | + dedicated |
| Sustained rate / slot | 5 tok/s | 15 tok/s | 25 tok/s | 25 tok/s | 30 tok/s | Negotiated |
| Burst rate / slot | 20 tok/s | 60 tok/s | 100 tok/s | 100 tok/s | 120 tok/s | Negotiated |
| Max context | 32k | 128k | 256k | 256k | 256k | Model max |
| Priority class | P2 | P1 | P1 | P0 | P0 | P0 + reserved |
| Idle reap | 5 min | 30 min | 60 min | 60 min | 120 min | Configurable |
| Tool calls / slot | 2 | 4 | 8 | 8 | 16 | Custom |
| Seats | 1 | 1 | 3 | 25 | 100 | Custom |
| Support | Community | Email | Email | Priority | Priority + SLA | TAM |

\* Prices are **modeling placeholders** carried from
[13 — Capacity & Economics §5](13-capacity-and-economics.md), where each is
checked against its break-even utilization. They are not committed pricing.

† Trial is the one tier with a token cap — it is an evaluation tier, not a
subscription. Every paid tier is genuinely uncapped.

### Model classes

| Class | Size shape | Serves | Example families |
|---|---|---|---|
| **S** | 7–14B dense | Fast tools, routing, drafting, classification | Small open instruct models |
| **M** | 27–70B dense | The daily driver for most agent work | Mid-size open instruct models |
| **L** | Large MoE, ~200–700B total / 20–40B active | Hard reasoning, long-horizon planning | Frontier-class open MoE |

Model *identities* are deliberately not fixed here — see the Model Registry in
[06 — Control Plane](06-control-plane.md). Classes are the stable contract;
the model behind a class is swappable.

## 4. The rate ceiling is the product

The **sustained tok/s per slot** is the single number that makes unlimited
tokens safe. It is a plan feature, not a hidden throttle, and it belongs in
public docs.

- **Sustained** — the refill rate of the slot's token bucket. Guaranteed floor.
- **Burst** — the bucket depth. Absorbs the spiky reality of agent work: a long
  generation after a quiet stretch of tool calls runs at full speed.

Because agent duty cycle is low, a well-sized bucket means **customers
essentially never feel the ceiling**, while the ceiling still caps our worst
case exactly. Burst is where the customer experience lives; sustained is where
the unit economics live.

```
bucket capacity  = burst_rate × burst_window_seconds
refill rate      = sustained_rate
```

## 5. Fair use, stated honestly

Published policy, no fine-print traps:

1. Unlimited tokens, subject to the plan's per-slot rate ceiling.
2. Slots are for **your** agents. No reselling raw inference.
3. No sharing one seat's slots across an organization to dodge tier pricing.
4. Abuse (scraping-at-scale, crypto-mining-style resale, prohibited content)
   ends the account, and is the only thing that does.
5. We may shift **P2/batch** work to off-peak windows. P0/P1 are never delayed
   for capacity reasons.

## 6. Entitlements — the machine-readable form

Every limit above resolves to one object, fetched at admission and cached.
This is the contract the whole platform reads; nothing else hardcodes a tier.

```jsonc
{
  "plan_id": "pro",
  "version": 3,
  "limits": {
    "max_active_agents": 5,
    "max_context_tokens": 262144,
    "model_classes": ["S", "M", "L"],
    "rate": { "sustained_tps": 25, "burst_tps": 100, "burst_window_s": 60 },
    "tool_concurrency_per_slot": 8,
    "idle_reap_seconds": 3600,
    "priority_class": "P1",
    "monthly_token_cap": null        // null = unlimited; set only on trial
  },
  "features": {
    "byo_tools": true,
    "private_networking": false,
    "dedicated_pool": false,
    "optimizations_opt_out": false   // see doc 10 — per-tenant A/B control
  }
}
```

Adding a tier = adding a row. Adding a limit = adding a key here **and** an
enforcement point that reads it. No tier logic in service code.

## 7. Deferred

- Annual billing, discounts, promo codes
- Seat-based vs. slot-based hybrid pricing
- Slot bursting / on-demand extra slots (obvious future revenue line)
- Regional pricing
- Usage-based enterprise contracts

---
*Next: [02 — Architecture Overview](02-architecture-overview.md)*
