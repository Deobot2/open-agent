# 13 — Capacity & Economics

> **Every number here is a placeholder** until measured in the `bench`
> environment (doc 03 §7). The *model* is the deliverable — plug real numbers in
> and it still works.

## 1. The cost chain

```
node $/hour ──► node tok/s ──► $/token ──► $/slot-month ──► plan price
   (known)      (MEASURE)      (derived)     (derived)       (decided)
```

Exactly one input needs a benchmark: **aggregate sustained output tok/s per
node, at production batch sizes and context lengths.** Everything downstream is
arithmetic. That is why `bench` is Phase 0 work — without it, every business
decision below is a guess.

## 2. Worked example (placeholders)

### Inputs

| Input | Placeholder | Source |
|---|---|---|
| Node cost | **$20.00 / hour** | 8×80GB GPU node, rented |
| Node throughput (class M) | **4,000 output tok/s** | ⚠️ MEASURE ME |
| Seconds per month | 2,592,000 | 30 × 86,400 |

### Derived cost per token

```
$20/hr ÷ 3600 s          = $0.005556 per second
$0.005556 ÷ 4,000 tok/s  = $0.00000139 per token
                         = $1.39 per 1M output tokens
```

### Cost of one slot, running flat out forever

```
25 tok/s × 2,592,000 s   = 64,800,000 tokens / month
64.8M × $1.39/M          = $90.00 per slot-month
```

**A single slot at a 25 tok/s ceiling, generating 24/7 for a month, costs $90.**

That number is the foundation. It is also, immediately, a problem worth stating
plainly rather than designing around.

## 3. The honest problem

A 5-slot Pro plan, with all 5 slots generating flat out all month:

```
5 slots × $90 = $450 / month in GPU cost
```

No consumer-priced plan survives that. **You cannot price unlimited tokens
against a true 100% duty cycle at a generous rate ceiling.** Any business model
claiming otherwise is either rate-limiting far harder than it admits or losing
money on its heaviest users.

So the model is explicitly two-layer, and both layers must be understood:

| Layer | Type | Provides |
|---|---|---|
| **Rate ceiling** | Hard, structural | A finite, known worst case ($90/slot). Bounds the tail. |
| **Utilization reality** | Statistical | Actual cost ~5% of worst case. Provides the margin. |

The rate ceiling does not make the business profitable — it makes the worst case
*finite and knowable*. Profitability comes from the second layer. Confusing the
two is the mistake that sinks this business model.

## 4. The utilization model

Real cost is the worst case multiplied by three factors, all < 1:

```
actual_cost = max_slots
            × occupancy      # fraction of purchased slots actually held
            × duty_cycle     # fraction of held time actually generating
            × rate_ceiling × 2,592,000 × cost_per_token
```

| Factor | Expected | Why below 1 |
|---|---|---|
| **Occupancy** | 0.20 – 0.40 | Customers buy headroom; rarely run every slot at once |
| **Duty cycle** | 0.10 – 0.20 | Agents wait on tools, networks, and humans far more than they generate |
| **Combined** | **~0.03 – 0.08** | The two multiply |

Both are measured from day one — `slot_ledger` gives occupancy, and
`active_seconds / slot_seconds` gives duty cycle (doc 07 §2, doc 09 §3). Until
they are measured, they are assumptions, and they are the assumptions the whole
business rests on.

## 5. Break-even utilization — the number per plan

Rearranged, each plan has one threshold: the combined utilization at which it
stops making money.

```
capacity_cost   = max_slots × sustained_tps × 2,592,000 × cost_per_token
break_even_util = price ÷ capacity_cost
```

| Plan | Price | Slots | tok/s | Capacity cost (100%) | **Break-even util** | Expected util | Headroom |
|---|---|---|---|---|---|---|---|
| Solo | $29 | 2 | 15 | $108 | **26.9%** | ~5% | **5.4×** |
| Pro | $99 | 5 | 25 | $450 | **22.0%** | ~5% | **4.4×** |
| Team | $399 | 20 | 25 | $1,800 | **22.2%** | ~5% | **4.4×** |
| Business | $1,299 | 60 | 30 | $6,480 | **20.0%** | ~5% | **4.0×** |

Read a row as: *"Pro is profitable as long as its customers keep combined
occupancy × duty cycle below 22%. We expect ~5%, so there is roughly 4.4× of
headroom."*

This single number replaces vague confidence with something monitorable. It goes
on dashboard D3 (doc 11), per plan, every day. **If measured utilization ever
approaches break-even, the plan is wrong** — and the fix is the rate ceiling or
the price for *new* subscribers, never a quiet throttle of existing ones
(doc 09 §7).

## 6. The tail risk, stated

A customer at 100% utilization on Pro costs $450 against $99 of revenue — a 4.5×
loss. That customer is rare, but they exist, and they are exactly the person a
"limitless tokens" offer attracts.

Four mitigations, in order of how much they actually help:

1. **The rate ceiling caps the damage at a known number.** Without it there is
   no bound at all. This is the one that must never be removed.
2. **Portfolio effect.** One 100% customer is covered by ~10 typical ones. The
   business needs enough customers for the average to hold — small customer
   counts have high variance and this model is fragile at low N.
3. **The margin alarm** (doc 09 §7) flags any tenant above 80% cost-to-revenue
   for human review.
4. **Optimizations** (doc 10) lower `cost_per_token`, raising break-even for
   everyone including the heavy user.

## 7. What the optimizations are worth

An optimization that cuts effective cost per token by *X* raises break-even
utilization by the same factor. Illustrative, on Pro:

| Optimization applied | Effective $/1M | Capacity cost | Break-even util | Headroom |
|---|---|---|---|---|
| None (baseline) | $1.39 | $450 | 22.0% | 4.4× |
| +30% (prefix cache) | $0.97 | $315 | 31.4% | 6.3× |
| +50% (cache + compaction) | $0.70 | $225 | 44.0% | 8.8× |
| +70% (cache + compaction + routing) | $0.42 | $135 | 73.3% | 14.7× |

At 70% savings, even a customer running **73% utilization** is profitable on Pro.
That is the arc: optimizations do not just improve margin, they progressively
eliminate the tail risk that makes unlimited-token pricing dangerous in the
first place.

Percentages above are illustrative placeholders. Each becomes real only when
measured end-to-end through the doc 10 harness — and **savings do not add**
(doc 10 §7), so the stack must be measured together, not summed.

[17 — Optimization Catalog §6](17-optimization-catalog.md) maps the round-one
systems onto this table. Two findings from that analysis change what to expect
here:

- The **router** supplies most of the movement (~88% of every downrouted step).
  If only one optimization ships, it should be that one.
- The **prompt reducer** contributes ~nothing directly — it is a 2.4× net loss
  as a token saver, and earns its place by making the router accurate. Do not
  book savings for it in this model.

## 7a. Two features that push cost the other way

Not everything on the roadmap reduces cost. Both of these are contained by
design rather than by hope:

| Feature | Effect on cost | Containment |
|---|---|---|
| **Ultra-thinking** (doc 18) | Would be 3–10× per slot if sub-agents got their own rate ceilings | Sub-agents share the parent's bucket → **cost identical to a normal slot** |
| **Free public agent** (doc 20) | Unbounded — no subscription bounds it | Hard token caps + class S + Pool-B + heavy caching + a monthly budget that trips automatically |

Ultra-thinking is worth stating plainly because it is counter-intuitive: it
raises **duty cycle** (more of the held time is generating) without raising
**cost per slot-hour**. If both rise together, the rate bucket is leaking —
that is invariant I-8 failing, and it is the alert to build.

## 7b. Contributed compute (doc 19)

A cost *offset*, not a cost reduction, and a modest one:

| Quantity | Value |
|---|---|
| Contributor electricity | $0.052 / hr |
| Our cost for equivalent compute | $0.625 / hr |
| Viable credit band | $0.05 – $0.62 / hr |

There is genuine margin, but uncapped 24/7 contribution earns $146–219/month —
more than the plan it discounts. Hence the three caps in doc 19 §4. Model it as
a **retention feature with a capacity bonus**, and do not book it as capacity in
the fleet sizing below.

## 8. Fleet sizing

```
required_tok_s = Σ(plan_subscribers × max_slots × occupancy × duty_cycle × sustained_tps)
required_nodes = required_tok_s ÷ node_tok_s × (1 + headroom)
```

Placeholder at 1,000 Pro-equivalent subscribers:

```
1,000 × 5 slots × 0.30 occupancy × 0.15 duty × 25 tok/s = 5,625 tok/s
5,625 ÷ 4,000 tok/s per node                             = 1.4 nodes
1.4 × 1.5 headroom                                       = 2.1 → 3 nodes (N+1)
```

Peak-to-average matters more than the average: size for **p95 concurrent
demand**, not the mean, and let P2 work absorb the difference. Headroom of 1.5×
covers peaks plus one node failing.

## 9. Build vs. rent (D-01)

| | Rent | Own |
|---|---|---|
| $/GPU-hour | $2.00–3.00 | ~$1.00–1.40 amortized (3 yr + power + colo) |
| Capital | None | $200k+ per node |
| Scaling | Minutes | Weeks |
| Break-even | — | ~55–65% sustained utilization |

**Rent through Phase 3.** Owning only wins at high, *predictable* utilization,
and until duty cycle and occupancy are measured, utilization is not predictable.
Revisit when sustained fleet utilization holds above 60% for a quarter.

## 10. Sensitivity

Which inputs actually move the outcome:

| Input | ±50% change → break-even util | Sensitivity |
|---|---|---|
| Node throughput (tok/s) | ±50% | **Highest — measure it well** |
| Node cost ($/hr) | ∓33% | High |
| Rate ceiling (tok/s) | ∓33% | High — and it is ours to choose |
| Duty cycle | no change to break-even; ±50% to actual cost | High |
| Occupancy | same as duty cycle | Medium |
| Context length | indirect, via KV pressure on throughput | Medium |

Two of the top three — **rate ceiling** and **throughput** — are under our
direct control. That is the encouraging part: the levers that matter most are
levers we hold, which is precisely what doc 10 is built to pull.

## 11. Phase 0 benchmark checklist

Nothing in this document is trustworthy until these are measured:

- [ ] Aggregate output tok/s per node, per model class, at production batch sizes
- [ ] Throughput vs. context length curve (where does KV pressure bite?)
- [ ] Prefix cache hit rate on realistic agent traffic
- [ ] Actual node cost including storage, network, egress
- [ ] Cold start time per pool (drives autoscaling feasibility)
- [ ] Runtime RAM per active slot (drives Tier-2 node count)
- [ ] Real occupancy and duty cycle from the first cohort of users

---
*Next: [14 — Repo & Deployment Layout](14-repo-and-deployment-layout.md)*
