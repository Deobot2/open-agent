# 10 — Optimization Framework

> **This is the doc that exists to be filled in.** Everything else describes a
> platform; this describes the *seams* your token-reduction systems mount to.
> No optimization is implemented in v0.1 — every stage ships as a pass-through.

## 1. Why a framework instead of optimizations

Optimizations that reduce token usage are the difference between a viable
unlimited-token business and an unviable one. But they share failure modes:
they change output quality, they interact with each other, they are hard to
attribute savings to, and they are very hard to remove once tangled into a hot
path.

So the platform commits up front to four rules, and the framework is what makes
them enforceable:

| # | Rule | Why |
|---|---|---|
| R-1 | Every optimization is a **stage plugin**, never inline code | Removable, testable, orderable |
| R-2 | Every optimization is **individually toggleable per tenant** | A/B against real traffic; instant kill switch |
| R-3 | Every optimization **reports its own savings** in a common unit | Prove it earns its keep |
| R-4 | Every optimization declares a **quality risk level** | Gate the risky ones behind evals |

R-2 is the one that matters most in practice. An optimization you cannot turn
off for one tenant is an optimization you cannot safely ship, because the first
quality complaint becomes a platform-wide rollback instead of a one-tenant flag
flip.

## 2. The five stages

The agent loop (doc 05 §6) calls exactly five hooks per step:

```
   ┌──────────────────────────── AGENT STEP ────────────────────────────┐
   │                                                                     │
   │  1. ADMIT      Should this inference call happen at all?            │
   │       ↓        Cheapest possible token: the one never generated     │
   │  2. ASSEMBLE   What exactly goes into the prompt?                   │
   │       ↓        Attacks prefill cost + the KV cache constraint       │
   │  3. ROUTE      Which model / pool / engine serves it?               │
   │       ↓        Attacks cost-per-token by using a cheaper model      │
   │  4. DECODE     How are the tokens physically produced?              │
   │       ↓        Attacks tokens/second and tokens emitted             │
   │  5. SETTLE     What do we keep for next time?                       │
   │                Feeds stages 1–3 on future steps                     │
   └─────────────────────────────────────────────────────────────────────┘
```

They are ordered by leverage. **Stage 1 saves 100% of a call's tokens; Stage 4
saves a percentage.** When an optimization could live in two stages, put it in
the earlier one.

### Stage 1 — ADMIT

*Question: can we skip inference entirely?*

Highest leverage, because the saving is total. Candidate systems: exact/semantic
response cache, deterministic short-circuits, loop detection (agent repeating
itself), no-op step detection, precondition checks.

```python
@dataclass
class AdmitVerdict:
    skip: bool = False            # skip inference
    halt: bool = False            # end the session
    response: str | None = None   # substitute answer if skipping
    reason: str = ""
    tokens_saved_est: int = 0
```

### Stage 2 — ASSEMBLE

*Question: what is the smallest correct prompt?*

Attacks prefill and the KV cache, which doc 04 §5 identifies as the real binding
constraint. Candidates: context compaction/summarization, sliding windows,
tool-result truncation and dedup, retrieval instead of full context, prompt
dedup, message pruning, prefix-stability rewriting (ordering the prompt so the
cacheable prefix stays byte-identical across steps).

```python
@dataclass
class AssembledPrompt:
    messages: list[dict]
    prefix_hint: str | None = None    # stable prefix for cache affinity
    tokens_before: int = 0
    tokens_after: int = 0
    lossy: bool = False               # did we drop information?
```

`lossy` is required, not decorative: it drives the eval gate in §6. An
optimization that drops information without declaring it is the one that
silently degrades quality.

**Prefix stability is a quiet multiplier.** A compaction that rewrites the top
of the prompt destroys the prefix cache hit it was meant to help, and can cost
more than it saves. Stage 2 plugins must preserve prefix stability or explicitly
declare they break it.

### Stage 3 — ROUTE

*Question: what is the cheapest model that can do this step?*

Candidates: difficulty classification → class S/M/L, cascade (try small, escalate
on low confidence), tool-call-only steps routed to a small model, pool selection
by priority, off-peak deferral to Pool-B.

```python
@dataclass
class RouteDecision:
    model_class: str              # S | M | L
    pool: str
    replica_url: str | None       # None => let scheduler pick
    escalation_allowed: bool = True
    reason: str = ""
```

Routing is where the biggest *cost* wins live — class S can be an order of
magnitude cheaper per token than class L — and also where the biggest quality
risk lives. Cascades must count escalations: a cascade that escalates 80% of the
time costs more than never cascading, since it pays for both calls.

### Stage 4 — DECODE

*Question: how do we produce these tokens cheaply?*

Candidates: speculative decoding (draft model from Pool-E), constrained/grammar
decoding, adaptive `max_tokens`, early stopping, stop-sequence tuning, sampling
parameter tuning, quantization selection per request.

```python
@dataclass
class DecodeParams:
    max_tokens: int
    temperature: float
    stop: list[str]
    grammar: str | None = None
    speculative: SpecConfig | None = None
```

### Stage 5 — SETTLE

*Question: what do we keep so future steps are cheaper?*

Not a saving itself — it is what makes stages 1–3 effective later. Candidates:
response cache writes, summary/memory writes, embedding index updates, prefix
pinning, telemetry and savings attribution.

```python
async def settle(ctx: Context, step: int, result: StepResult) -> SettleReport:
    """Persist artifacts; emit the usage event carrying savings attribution."""
```

## 3. Plugin contract

Every optimization implements one interface, in one stage:

```python
class Optimization(Protocol):
    name: str                      # "semantic-cache"
    stage: Stage                   # ADMIT | ASSEMBLE | ROUTE | DECODE | SETTLE
    version: str
    quality_risk: Risk             # NONE | LOW | MEDIUM | HIGH   (R-4)
    default_enabled: bool

    async def apply(self, ctx: Context, input: StageInput) -> StageOutput: ...
    def report(self) -> SavingsReport: ...          # R-3
    async def health(self) -> bool: ...
```

```python
@dataclass
class SavingsReport:
    tokens_saved: int              # common unit across all stages
    latency_delta_ms: int          # negative = faster
    cost_saved_micros: int
    applied: bool
    fallback_reason: str | None = None
```

### Non-negotiables

| Rule | Reason |
|---|---|
| **Fail open.** A plugin error is logged, skipped, step proceeds unoptimized | An optimization must never be able to take down a session |
| **Hard time budget** per stage (default 50 ms) | Optimization latency must not exceed the savings |
| **No cross-tenant state** without namespacing (invariant I-6) | A shared cache is a data leak waiting to happen |
| **Deterministic under replay** | Debuggability; step idempotency |
| **Declare quality risk honestly** | Drives the eval gate |

The fail-open rule is the one to defend hardest. A plugin that can throw and
kill a customer's 40-minute agent run is strictly worse than no optimization,
regardless of how many tokens it saves.

## 4. Registry and per-tenant control

```python
PIPELINE = OptimizationRegistry(
    admit    = [ExactCache(), SemanticCache(), LoopDetector()],
    assemble = [ToolResultTruncator(), ContextCompactor(), RetrievalInjector()],
    route    = [DifficultyRouter(), CascadeRouter()],
    decode   = [AdaptiveMaxTokens(), SpeculativeDecoder()],
    settle   = [CacheWriter(), MemoryWriter(), SavingsReporter()],
)
```

Within a stage, plugins run **in order**, each seeing the previous one's output.
Order is configuration, not code.

Resolution per request (satisfying R-2):

```
enabled(plugin, tenant) =
    plugin.default_enabled
      ⊕ plan.features.optimizations          # plan-level
      ⊕ org_overrides.features_patch         # tenant-level
      ⊕ experiment_assignment(tenant, plugin)# A/B bucket
      ⊕ global_kill_switch(plugin)           # ops override, wins over all
```

A global kill switch that overrides everything is required. When an optimization
misbehaves in production at 3am, the fix must be one flag, no deploy.

## 5. Measurement — the harness that decides what ships

R-3 means no optimization ships on plausibility. Each must show, on real traffic:

| Metric | Question |
|---|---|
| `tokens_saved` / total tokens | How much did it actually save? |
| `latency_delta_ms` | Did it make things slower? |
| `quality_delta` (eval suite) | Did answers get worse? |
| `apply_rate` | How often does it fire at all? |
| `fallback_rate` | How often does it bail? |
| `cost_saved − cost_incurred` | **Net win?** Router/draft models cost GPU too |

The last row is the one that kills optimizations that look good in isolation. A
semantic cache needs an embedding call; a cascade sometimes pays twice; a
speculative decoder burns Pool-E capacity. Savings are net or they are not
savings.

### Shipping gate

```
1. Offline:  replay recorded traffic, measure savings + quality
2. Shadow:   run in production, compute but do not apply, compare
3. Canary:   1% of tenants, quality risk ≤ LOW; MEDIUM/HIGH need eval sign-off
4. Ramp:     10% → 50% → 100%, halt on quality or latency regression
5. Default:  flip default_enabled, keep the kill switch forever
```

Shadow mode is the highest-value step and needs the plugin API to support
compute-without-apply from day one — it gives a real-traffic estimate at zero
customer risk.

## 6. Quality gates

| Risk | Examples | Gate |
|---|---|---|
| **NONE** | Prefix stabilization, telemetry | Ship |
| **LOW** | Exact cache, tool-result dedup | Canary + monitor |
| **MEDIUM** | Semantic cache, adaptive max_tokens | Eval suite, no regression |
| **HIGH** | Context compaction, model downrouting | Eval + human review + slow ramp |

Anything with `lossy = True` (Stage 2) or a model downgrade (Stage 3) is
**automatically at least MEDIUM**. Those two are where real capability
regressions come from, and they are also the two biggest savings — so they get
the most scrutiny rather than the least.

## 7. Interaction effects

Optimizations are not additive. Known interactions to watch:

| A | B | Effect |
|---|---|---|
| Context compaction | Prefix caching | **Negative** — rewriting the prefix voids the cache |
| Semantic cache | Cascade routing | Overlap — cache hits reduce cascade's measured value |
| Speculative decoding | Small-model routing | Compete for Pool-E capacity |
| Adaptive max_tokens | Early stopping | Redundant; keep one |
| Retrieval injection | Context compaction | Can fight: one adds, one removes |

Because savings do not add up, **the pipeline's total is measured end-to-end**,
not summed from individual plugins. Per-plugin numbers are for deciding whether
to keep a plugin; the end-to-end number is the one that goes in the cost model.

## 8. Where savings show up

Every stage writes into the doc 07 schema that already exists:

```sql
tokens_saved     UInt32                      -- summed savings for the step
stages_applied   Array(LowCardinality(String))  -- ["semantic-cache","compactor"]
est_cost_micros  UInt64                      -- post-optimization actual cost
```

Those columns are in the baseline schema on purpose (doc 07 §4): the day the
first optimization ships, before/after comparison against real historical
traffic is already possible.

## 9. Slots reserved for your systems

**Round one is placed — see [17 — Optimization Catalog](17-optimization-catalog.md)
for each system's mechanism, net-savings analysis, and risk class.** The mapping
table below stays as the intake rule for everything added later.

Drop each new system into the stage that matches its mechanism:

| If the system… | It is Stage | Notes for integration |
|---|---|---|
| Avoids a call entirely | **1 ADMIT** | Return `skip=True` + a substitute response |
| Shrinks or restructures the prompt | **2 ASSEMBLE** | Must declare `lossy`; preserve prefix stability |
| Picks a cheaper model/pool | **3 ROUTE** | Count escalations; measure net |
| Changes how tokens are produced | **4 DECODE** | Pool-E capacity is reserved for draft/aux models |
| Builds state for future steps | **5 SETTLE** | Namespace by tenant (I-6) |
| Doesn't fit any of these | — | **Flag it — it may need a sixth stage or a scheduler-level hook** |

Two systems from round one did *not* fit the five stages, which is the intake
rule working as intended:

| System | Where it went | Why |
|---|---|---|
| **Auto-skill maker** (doc 16 §5) | Offline / batch | Generates artifacts between sessions, not during a step |
| **Ultra-thinking** (doc 18) | Runtime orchestration | Increases cost; contained by the rate bucket, not a stage |
| **Contributed compute** (doc 19) | Capacity layer | Changes *where* work runs, not what work happens |

Two seams beyond the five stages already exist for systems that are not
per-step: **the scheduler** (doc 06) for cross-session batching, off-peak
deferral, and fair-share; and **the registry** (doc 04) for anything that swaps
model versions or engine configs per workload.

## 10. Deferred until your rundown

- Concrete plugin implementations (all of them)
- Offline replay corpus and eval suite contents
- Experiment assignment / bucketing service
- Cross-session and cross-tenant optimization (needs isolation review first)
- Learned/adaptive routing policies

Round-one systems are specified in doc 17 but not implemented; the framework
still ships before any of them (doc 15, Phase 4).

---
*Next: [11 — Observability](11-observability.md)*
