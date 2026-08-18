# 11 — Observability

> Skeleton. Metric names and thresholds are placeholders; the structure is the
> deliverable.

## 1. Stack

| Signal | Tool | Retention |
|---|---|---|
| Metrics | Prometheus → Grafana | 90 d |
| Logs | Loki (structured JSON) | 30 d |
| Traces | Tempo (OpenTelemetry) | 7 d |
| Usage/analytics | ClickHouse | 25 months |
| Alerts | Alertmanager → PagerDuty/Slack | — |

## 2. The four dashboards

Deliberately few. Each answers one question.

### D1 — Product health *(is the promise being kept?)*
- Active slots vs. sold slots, by plan
- Slot acquire p99 latency (**SLO < 5 ms**)
- 409 `slot_limit_reached` rate — spikes mean a tier is mis-sized
- Session success rate; reap rate by reason
- TTFT p50/p95/p99 by priority class

### D2 — Fleet efficiency *(are we using the GPUs well?)*
- Aggregate tok/s by pool
- GPU utilization, KV cache utilization
- **Prefix cache hit rate** (target > 70%, alert < 50%)
- Batch size, queue depth by pool
- Cost per 1M tokens by pool

### D3 — Economics *(are we making money?)*
- **Fleet duty cycle** — the headline number
- Duty cycle distribution across tenants
- Margin by plan, margin by tenant
- Overcommit ratio vs. P0 latency headroom
- Cost per active slot per month vs. plan price

### D4 — Optimization *(is doc 10 working?)*
- `tokens_saved` total and by plugin
- Apply rate / fallback rate per plugin
- Quality delta vs. baseline
- Net cost saved (savings − optimizer overhead)

D3 is the one to look at daily. It is where the business shows up before the
invoice does.

## 3. Key metrics

```
# Slots
oa_slots_active{org,plan}
oa_slots_acquire_duration_seconds{result}
oa_slots_rejected_total{reason}
oa_slots_reaped_total{reason}

# Inference
oa_inference_tokens_total{pool,model,kind="prompt|completion|cached"}
oa_inference_ttft_seconds{pool,priority}
oa_inference_batch_size{replica}
oa_inference_kv_utilization{replica}
oa_prefix_cache_hit_ratio{pool}

# Economics
oa_duty_cycle{org,plan}
oa_cost_micros_total{pool,org}
oa_slot_seconds_total{org,state="active|idle"}

# Optimization
oa_opt_tokens_saved_total{plugin,stage}
oa_opt_apply_total{plugin,result="applied|skipped|failed"}
oa_opt_duration_seconds{plugin}
```

## 4. SLOs

| SLO | Target | Window |
|---|---|---|
| Slot acquire success (capacity available) | 99.9% | 30 d |
| Slot acquire latency p99 | < 5 ms | 30 d |
| TTFT p95, P0 | < 2 s | 30 d |
| TTFT p95, P1 | < 5 s | 30 d |
| Session completion (no platform error) | 99.5% | 30 d |
| API availability | 99.9% | 30 d |
| Stream continuity (no drop mid-run) | 99.5% | 30 d |

P2 has no latency SLO by design — that is the flexibility that lets batch work
absorb capacity pressure.

## 5. Alerts

### Page
- Slot acquire error rate > 1% for 5 min
- Redis unavailable (**admissions stop** — doc 02 §5)
- Postgres primary down
- Any pool with zero healthy replicas
- P0 TTFT p95 > 2× SLO for 10 min

### Ticket
- Prefix cache hit rate < 50% for 1 h
- Fleet duty cycle up > 20% week-over-week
- Any tenant cost > 80% of revenue
- Optimization plugin fallback rate > 10%
- Pool utilization > 85% sustained (capacity lead time)

The duty-cycle alert is unusual and intentional: it is a *business* alert on an
engineering dashboard, and it fires weeks before the cost problem it predicts.

## 6. Tracing

One trace per agent step, spanning the whole path:

```
run.step
├── opt.admit          [plugin spans]
├── opt.assemble       [plugin spans]
├── opt.route
├── scheduler.place
├── inference.generate ├── ttft
│                      └── decode
├── tool.execute       [per tool]
└── opt.settle
```

Trace ID propagates from the gateway through to the engine request, so a single
slow customer step is diagnosable end to end. Sample 1% of steps, 100% of errors
and of any step over the latency SLO.

## 7. Eval gate

Model promotions (doc 06) and MEDIUM/HIGH-risk optimizations (doc 10 §6) must
pass an automated suite before reaching `active`:

| Suite | Checks |
|---|---|
| Capability | Reasoning, coding, tool use, long-context recall |
| Agent behavior | Multi-step task completion, loop avoidance |
| Regression | Frozen prompt set, output diffed vs. current active |
| Safety | Refusal behavior, injection resistance |

Fails the gate → cannot promote. No manual override without written sign-off;
this is the only thing standing between a cost optimization and a silent
capability regression.

## 8. Deferred

- Per-customer status page
- Anomaly detection on duty cycle
- Profiling of the agent runtime hot path
- Log-based billing reconciliation

---
*Next: [12 — Security & Tenancy](12-security-and-tenancy.md)*
