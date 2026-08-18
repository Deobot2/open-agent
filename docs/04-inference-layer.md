# 04 — Inference Layer

> Skeleton. Engine choice (D-03) and flagship model (D-02) are open decisions.
> We commit to the *interface*, not yet the implementation behind it.

## 1. Position

The inference layer is the only place GPUs are touched. Everything above it
speaks one OpenAI-compatible HTTP interface, which is what makes the engine and
the model swappable without touching the agent runtime.

```
oa-agent-runtime ──HTTP──► [engine replica] ──► GPU
                  (OpenAI-compatible: /v1/chat/completions, /v1/completions)
```

**Rule: no service other than the agent runtime calls an engine directly.**
Anything else (evals, benchmarks, internal tools) goes through the same path so
metering, rate limits, and Stage 1–5 apply uniformly. A bypass is a metering
hole and a fairness hole at the same time.

## 2. Engine

Candidates, to be benchmarked in `bench` (D-03):

| Engine | Strength | Watch out for |
|---|---|---|
| **vLLM** | Mature, broad model support, PagedAttention, strong community | Prefix caching good, not best-in-class |
| **SGLang** | RadixAttention prefix cache, excellent for repeated prompt structure | Smaller model coverage |
| TensorRT-LLM | Peak throughput | Build complexity, model support lag |

**Bias toward SGLang for the agent workload.** Agent traffic is pathologically
repetitive — the same system prompt, tool schemas, and conversation prefix are
re-sent on every step of every loop. A prefix cache that turns a 50k-token
re-read into a cache hit is worth more to us than raw FLOPs, and it directly
raises the ceiling on how many slots a node can carry. Benchmark decides, but
this is the hypothesis to test.

The abstraction is thin by design: we configure and operate engines, we do not
fork them.

## 3. Model classes → deployments

The registry maps a stable class to a concrete deployment
(see [06 — Control Plane](06-control-plane.md)):

```jsonc
{
  "class": "M",
  "model_id": "oa-m-2026-01",
  "weights_uri": "s3://oa-models/<family>/<version>/",
  "engine": "sglang",
  "engine_args": {
    "tensor_parallel_size": 2,
    "max_model_len": 262144,
    "quantization": "fp8",
    "enable_prefix_caching": true,
    "max_running_requests": 256
  },
  "pool": "pool-m",
  "capabilities": ["tools", "json_schema", "reasoning"],
  "status": "active"           // active | canary | draining | retired
}
```

Classes are the contract. Swapping the model behind class M is a registry
change plus a canary, not a code change — that is the property to protect.

## 4. Quantization

| Precision | Memory | Quality | Use |
|---|---|---|---|
| BF16 | 1.0× | Reference | Eval baseline only |
| **FP8** | ~0.5× | Near-parity on most tasks | **Default for S/M/L** |
| INT4 / AWQ | ~0.25× | Task-dependent loss | Pool-B, drafts, non-critical |

FP8 as default is a big lever: roughly double the batch and KV cache in the
same HBM, at near-parity quality on current hardware. Any quantization change
must pass the eval gate in [11 — Observability](11-observability.md) before it
reaches `active` — quality regressions are the one cost saving we will not take
silently.

## 5. Batching and the KV cache

Continuous batching is the default and the throughput story: requests join and
leave the running batch per-step rather than waiting for a full batch.

Per replica, tune three numbers and accept the trade:

| Knob | Raises | Costs |
|---|---|---|
| `max_running_requests` | Aggregate throughput | Per-request latency |
| `max_model_len` | Context ceiling | KV memory per request |
| KV cache fraction | Concurrency | Weights/activation headroom |

### The KV cache is the real constraint

```
kv_bytes ≈ 2 × layers × kv_heads × head_dim × dtype_bytes × tokens
```

Long-context agents are KV-bound long before they are compute-bound. A handful
of 256k-token sessions can exhaust an 80 GB card's cache budget while the SMs
sit idle. Three consequences that shape the platform:

1. **Context length is a plan feature** (doc 01) because it is a real resource.
2. **Context compaction is the highest-leverage optimization** we can add — it
   attacks the binding constraint, not the visible one (doc 10, Stage 2).
3. **Prefix cache hit rate is a first-class SLO metric**, not a nice-to-have.

### Prefix caching

Agent loops re-send a near-identical prefix every step. With prefix caching,
step *N*'s prompt is largely a cache hit and only the delta is prefilled.

- Cache key **must be namespaced by tenant** (invariant I-6). A cross-tenant
  prefix hit is a data leak, full stop.
- Target hit rate: **>70%** on steady-state agent traffic. Alert below 50%.
- Eviction: LRU, with pinning for shared system prompts.

## 6. Scheduling and priority

`oa-scheduler` places work; the engine schedules within a replica.

| Class | Plans | Behavior under contention |
|---|---|---|
| **P0** | Team, Business, Enterprise | Never delayed; first to claim batch slots |
| **P1** | Solo, Pro | Queued behind P0, target < 2 s TTFT |
| **P2** | Trial, background, batch | Delayed freely; may be moved to Pool-B |

Placement inputs: model class, priority, estimated prompt tokens, **prefix
affinity** (prefer the replica that already holds this session's prefix), and
current replica load.

**Prefix affinity is sticky session routing and it matters a lot.** Sending
step *N+1* to a different replica than step *N* throws away the prefix cache and
turns a cheap step into a full prefill. Session → replica stickiness is
therefore a correctness-adjacent concern for cost, with fallback on drain.

## 7. Replica lifecycle

```
  provision → pull weights → warm → HEALTHY ──► serving
                                       │
                                  drain (finish in-flight, no new)
                                       │
                                   terminate
```

- **Warm** = load weights, run synthetic prompts to trigger CUDA graph capture
  and allocator settling. A cold replica's first request is dramatically slower;
  never expose one to a customer.
- Health: `/health` (liveness), `/health/ready` (post-warm only).
- Rolling updates: one replica at a time per pool, canary first, auto-rollback
  on eval or latency regression.
- Drain before terminate, always — including on spot eviction notice in Pool-B.

## 8. Failure handling

| Failure | Response |
|---|---|
| Replica OOM | Kill, restart with lower `max_running_requests`, alert |
| CUDA fault / ECC error | Cordon node, drain, page on-call |
| Spot eviction (Pool-B) | Drain on notice, requeue P2 work |
| Timeout mid-generation | Retry once on another replica (steps are idempotent) |
| Whole pool down | Degrade class with explicit tenant notice; never silent |

Retry safety comes from step idempotency: an agent step is keyed
`(session_id, step_id)`, so a retried step cannot double-bill or double-apply.

## 9. Interface we depend on

Keeping the surface small is what keeps engines swappable:

```
POST /v1/chat/completions     # streaming + non-streaming, tools, json_schema
POST /v1/completions          # raw, for drafts and internal use
GET  /health, /health/ready
GET  /metrics                 # Prometheus: queue depth, batch size,
                              # cache hit rate, tok/s, KV utilization
```

Anything an engine offers beyond this is a bonus we do not build on without a
fallback.

## 10. Deferred

- Speculative decoding (Stage 4 hook exists — doc 10)
- Disaggregated prefill/decode
- Multi-LoRA serving for per-tenant adapters
- CPU offload for cold sessions
- Custom kernels

---
*Next: [05 — Agent Runtime & Slots](05-agent-runtime-and-slots.md)*
