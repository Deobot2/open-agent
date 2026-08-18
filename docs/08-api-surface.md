# 08 — API Surface

> Skeleton. Shapes and semantics, not a full spec. OpenAPI generation is a
> Phase 1 task.

## 1. Conventions

| | |
|---|---|
| Base | `https://api.<domain>/v1` |
| Auth | `Authorization: Bearer oa_live_…` |
| Format | JSON; SSE for streams |
| Errors | RFC 9457 problem details |
| Idempotency | `Idempotency-Key` header on all POSTs |
| Versioning | URL major (`/v1`) + `OA-Version: 2026-08-18` date header for minors |
| Pagination | Cursor: `?limit=50&after=<cursor>` |

## 2. Agents — configuration, free and unlimited

```http
POST   /v1/agents                  # create
GET    /v1/agents?limit=50
GET    /v1/agents/{id}
PATCH  /v1/agents/{id}
DELETE /v1/agents/{id}             # archive
```

```jsonc
// POST /v1/agents
{
  "name": "research-assistant",
  "model_class": "M",              // S | M | L (plan-gated)
  "system_prompt": "…",
  "tools": [ { "type": "http", "name": "search", "schema": {} } ],
  "config": { "max_steps": 50, "temperature": 0.7, "max_context_tokens": 128000 }
}
```

## 3. Runs — this is where slots are consumed

```http
POST   /v1/agents/{id}/runs        # start; ACQUIRES A SLOT
GET    /v1/runs?state=active
GET    /v1/runs/{id}
POST   /v1/runs/{id}/stop          # releases slot
POST   /v1/runs/{id}/messages      # send input to a running agent
GET    /v1/runs/{id}/events        # SSE stream (resumable)
```

### Starting a run

```jsonc
// POST /v1/agents/{id}/runs
{
  "input": "Summarize the Q3 filings.",
  "stream": true,
  "queue_if_full": false,          // true => wait for a slot instead of 409
  "metadata": { "trace": "user-1234" }
}
```

**Success — 201:**
```jsonc
{
  "run_id": "run_…",
  "slot_id": "slot_…",
  "state": "active",
  "stream_url": "/v1/runs/run_…/events",
  "slots": { "used": 3, "limit": 5 }        // always returned; drives the UI
}
```

**At limit — 409:** the most product-critical error we return.
```jsonc
{
  "type": "https://docs.<domain>/errors/slot-limit-reached",
  "title": "All agent slots are in use",
  "status": 409,
  "detail": "Your plan allows 5 concurrent agents. Stop one, or upgrade.",
  "slots": { "used": 5, "limit": 5 },
  "active_runs": [
    { "run_id": "run_a", "agent_name": "researcher", "started_at": "…", "state": "active" },
    { "run_id": "run_b", "agent_name": "monitor",    "started_at": "…", "state": "idle" }
  ],
  "options": { "queue": true, "upgrade_url": "https://…/billing" }
}
```

This response is a product surface, not an error string. It lists what is
running (flagging IDLE runs, which are the likely thing to stop), and offers
queue and upgrade paths. Getting this right is most of the difference between
"limited agents" feeling like a clear plan and feeling like a wall.

### Event stream

```
event: step.started      data: {"step":3}
event: token             data: {"text":"The "}
event: tool.called       data: {"name":"search","args":{}}
event: tool.result       data: {"name":"search","ok":true}
event: throttled         data: {"reason":"rate_ceiling","resume_in_ms":420}
event: step.completed    data: {"step":3,"tokens":847}
event: run.completed     data: {"reason":"completed","steps":4}
```

Resumable via `Last-Event-ID`. The explicit `throttled` event matters: when the
token bucket empties, the client should *show* a slower stream, not appear
hung. Honesty about the rate ceiling is part of the product (doc 01 §4).

## 4. Slots — make the limit visible

```http
GET    /v1/slots                   # current usage
GET    /v1/slots/history?days=30
```

```jsonc
// GET /v1/slots
{
  "limit": 5,
  "used": 3,
  "available": 2,
  "active": [
    { "run_id": "run_a", "agent_name": "researcher", "state": "active",
      "started_at": "…", "tokens_generated": 148203 },
    { "run_id": "run_b", "agent_name": "monitor", "state": "idle",
      "idle_since": "…", "reap_at": "…" }
  ],
  "plan": { "id": "pro", "max_active_agents": 5 }
}
```

Surfacing `reap_at` on idle runs is deliberate — the reaper should never look
like an unexplained disappearance.

## 5. Usage — transparency without a meter

```http
GET    /v1/usage?from=…&to=…&granularity=day
```

```jsonc
{
  "period": { "from": "…", "to": "…" },
  "tokens": { "prompt": 41000000, "completion": 12000000, "total": 53000000 },
  "unlimited": true,                        // there is no cap to display
  "slot_seconds": { "total": 3182400, "active": 412000, "idle": 2770400 },
  "duty_cycle": 0.129,
  "by_agent": [ { "agent_id": "…", "name": "researcher", "slot_seconds": 1200000 } ]
}
```

Tokens are shown for the customer's insight, never as a balance. `unlimited:
true` is explicit so no client renders a progress bar against a cap.

## 6. Account

```http
GET    /v1/me
GET    /v1/plan
GET    /v1/billing/invoices
POST   /v1/api-keys
DELETE /v1/api-keys/{id}
```

## 7. Errors

| Status | Code | Meaning |
|---|---|---|
| 401 | `unauthenticated` | Missing/invalid key |
| 403 | `model_class_not_in_plan` | Class L on a Solo plan |
| 403 | `seat_limit_reached` | Too many users |
| **409** | **`slot_limit_reached`** | **All agents in use — see §3** |
| 413 | `context_too_large` | Over plan's `max_context_tokens` |
| 429 | `rate_ceiling` | Only for *request* floods; token throttling is not an error |
| 503 | `capacity_unavailable` | Pool down / degraded |

**429 vs. throttling:** exceeding the token bucket during generation is *never*
an error — it slows down (doc 05 §5). 429 is reserved for abusive request
rates. Conflating them would break the unlimited-tokens promise.

## 8. Internal APIs

gRPC, mTLS, not public. Sketched in [06](06-control-plane.md).

```
oa-accounts   Verify, GetTenant
oa-plans      GetEntitlements, SetOverride
oa-slots      Acquire, Renew, Release, ListActive
oa-scheduler  Schedule, GetPools
oa-registry   GetActiveDeployment, PromoteModel
```

## 9. Compatibility shim (deferred, likely valuable)

An OpenAI-compatible `/v1/chat/completions` that transparently acquires a slot
for the duration of a request would let existing tooling point at us with a
base-URL change. Cheap adoption path; needs a mapping from stateless requests
to slot semantics. Phase 3.

## 10. Deferred

- Webhooks for run lifecycle
- WebSocket bidirectional transport
- Batch API for P2 work
- SDKs (Python, TS)
- Fine-grained scoped keys

---
*Next: [09 — Metering & Billing](09-metering-and-billing.md)*
