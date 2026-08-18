# 09 — Metering & Billing

> Skeleton. The key inversion: **we meter heavily, we bill flatly.**

## 1. The inversion

| | Typical LLM API | Open Agent |
|---|---|---|
| Customer sees | Token meter → variable bill | Flat price, unlimited tokens |
| We measure | Tokens, to charge for | Tokens, to **understand our own cost** |
| Risk | Customer bill shock | **Our** margin erosion |

Because the customer's invoice is flat, all metering exists for internal
decisions: cost attribution, unit economics, capacity planning, and measuring
whether an optimization actually worked. It is an engineering instrument, not
a billing instrument.

**Consequence:** metering must never be on the critical path of a token
(doc 02 §4). If ClickHouse is down, agents keep running and we lose analytics —
never the reverse.

## 2. What we measure

### Per step (→ ClickHouse `usage_events`)

| Field | Use |
|---|---|
| `prompt_tokens`, `cached_tokens` | Prefill cost; cache effectiveness |
| `completion_tokens` | Decode cost (dominant) |
| `ttft_ms`, `duration_ms` | SLO, latency budget |
| `model_id`, `pool` | Cost attribution |
| `tokens_saved`, `stages_applied` | **Optimization attribution** (doc 10) |
| `est_cost_micros` | Computed at write time from pool cost rate |

### Per slot (→ Postgres `slot_ledger` → hourly rollup)

| Field | Use |
|---|---|
| `slot_seconds` | Total held |
| `active_seconds` | Actually generating |
| `idle_seconds` | Held but quiet |
| **`duty_cycle`** | **`active/total` — the number the business runs on** |

## 3. Duty cycle is the metric that matters

Everything about profitability compresses into one ratio:

```
duty_cycle = active_seconds / slot_seconds
```

| Duty cycle | Meaning |
|---|---|
| 1.00 | Worst case. Agent generating 24/7. Priced for in doc 13. |
| 0.10–0.20 | Expected steady state — tool calls, waits, human turns |
| < 0.05 | Mostly-idle slots; reap timers may be too generous |

Watched three ways:

1. **Fleet-wide** — sets the overcommit ratio (D-06)
2. **Per tenant** — finds the outliers priced for but rare
3. **Per plan** — is any tier structurally unprofitable?

**Alert on fleet duty cycle rising**, not just on cost. Rising duty cycle is the
earliest signal that the average is drifting toward the worst case, and it
arrives well before the invoice does.

## 4. Cost attribution

Push GPU cost down to the session, so "which customers cost us money" is
answerable:

```
pool_cost_per_second   = node_hourly_cost × node_count / 3600
pool_tokens_per_second = measured aggregate output tok/s
cost_per_token         = pool_cost_per_second / pool_tokens_per_second

step_cost = completion_tokens × cost_per_token
          + (prompt_tokens - cached_tokens) × cost_per_token × PREFILL_RATIO
```

`PREFILL_RATIO` (placeholder ~0.15) reflects that prefill is far cheaper per
token than decode — it is parallel, not sequential. Cached prefix tokens cost
approximately nothing, which is exactly why prefix cache hit rate shows up in
the cost model and not just in a latency dashboard.

Rolled up nightly into `org_cost_daily` for margin-per-tenant.

## 5. Billing

Genuinely simple, because the offer is:

```
subscription → payment provider → webhook → oa-billing
                                              │
                                     oa-plans entitlement update
                                              │
                                      Redis cache invalidation
```

- Flat monthly/annual per plan. No overage, no metered lines, no top-ups.
- Upgrade: prorate, entitlements effective immediately (slots available at once).
- Downgrade: effective next period. **Sessions over the new limit are never
  killed mid-run** — they drain naturally and the lower limit applies to new
  starts.
- Failed payment: dunning → grace (7 d) → suspend new sessions → running
  sessions drain. Never a hard kill.

The "never kill running work" rule appears in every transition on purpose. Work
in flight is the customer's, not ours to interrupt over a billing state change.

## 6. Internal reporting

| Report | Cadence | Question |
|---|---|---|
| Margin per plan | Daily | Is any tier losing money? |
| Margin per tenant | Daily | Who are the outliers? |
| Duty cycle distribution | Daily | Where is the average vs. the worst case? |
| Cost per 1M tokens by pool | Hourly | Is the fleet efficient? |
| **Optimization savings** | Daily | Is each doc-10 stage earning its keep? |
| Slot utilization vs. sold | Hourly | Overcommit headroom |

## 7. The margin alarm

Automated guard, because margin erosion is quiet:

```
IF   tenant_cost_30d > tenant_revenue_30d × ALARM_RATIO      # e.g. 0.8
THEN flag for review
```

Escalation, in order: verify legitimate use → check optimizations are applied →
review the plan's rate ceiling → contact the customer about a fitting tier.

**Never silently throttle below the published ceiling.** The published rate is a
promise; if a plan is structurally unprofitable, the plan is wrong and gets
changed openly for future subscribers — the fix is never a quiet degradation of
a customer already paying for it.

## 8. Deferred

- Revenue recognition, tax, invoicing detail
- Annual/multi-year, discounts, promos
- Reseller / partner billing
- Slot-bursting as a paid add-on
- Per-tenant cost dashboards exposed to customers

---
*Next: [10 — Optimization Framework](10-optimization-framework.md)*
