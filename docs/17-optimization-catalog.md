# 17 — Optimization Catalog

> Your named systems, placed into the doc 10 stages, with a net-savings check on
> each. Skeleton — contracts and analysis, no implementations.

## 1. Placement summary

| System | Stage | Attacks | Net value | Risk |
|---|---|---|---|---|
| **Caveman** (compression) | 2 ASSEMBLE | Prefill + KV pressure | **High if amortized, negative if per-request** | MEDIUM |
| **Prompt Reducer** | 2 ASSEMBLE (pre) | Clarity, router quality | **Not a token saver — see §3** | LOW |
| **Router** | 3 ROUTE | Cost per token | **Highest of the five** | HIGH |
| **State Saver** | 5 SETTLE → 1/2 | Prefill, warm start | **High, low risk** | LOW |
| **Skills / Tools** | 2 ASSEMBLE | Steps avoided | High if conditionally loaded (doc 16) | MEDIUM |

Reading it in one line: **the router is where the money is, the state saver is
the cheapest win, and the two compression systems need care to avoid costing
more than they save.**

---

## 2. Caveman — telegraphic compression

> **Assumption to confirm:** "Caveman" = compressing text to telegraphic style
> (drop articles, filler, politeness; keep content words). If you meant
> something else, this section moves but the analysis pattern holds.

**Stage 2 ASSEMBLE.** Rewrites text to fewer tokens while preserving meaning.

### The rule that decides everything: compress once, reuse many times

Compression costs an LLM call. That call is only worth it if the compressed
output is used repeatedly.

| Target | Reuse count | Verdict |
|---|---|---|
| **System prompts** | Every step, forever | ✅ **Compress at authoring time**, store compressed |
| **Skill bodies** (doc 16) | Every load | ✅ **Compress once at generation** |
| **Tool schemas** | Every step | ✅ **Compress once at registration** |
| Long stable documents in context | Many steps | ✅ Compress, cache the result |
| The user's message | Once | ❌ Never worth an LLM call |
| Freshly-fetched tool results | Once | ⚠️ Only with a non-LLM truncator |

Break-even measured at **~2.4 reuses** (§3 math). Anything reused more than 3
times is a clear win; anything used once is a clear loss.

### The prefix-cache interaction — the thing that makes this subtle

Caveman must run **before** a prompt becomes a cached prefix, never after.

```
✅ author → compress → store compressed → serve (stable bytes, cache hits)
❌ author → store → compress at request time (prefix changes → cache MISS)
```

Compressing a stable prefix at request time destroys the prefix cache hit it was
meant to help — and prefix caching is already saving more than compression can.
This is exactly the negative interaction flagged in doc 10 §7, and it is the
single easiest way to make this optimization a net loss.

### Risk

Telegraphic prompts can degrade instruction-following: dropped function words
sometimes carry real constraint ("do not" → "not" is fine; "if X then Y" losing
its structure is not). Classified **MEDIUM**, `lossy = True`, eval gate required
(doc 10 §6).

---

## 3. Prompt Reducer — typo and intent normalization

**Stage 2 ASSEMBLE, pre-pass.** Same prompt, cleaned: typos fixed, intent
explicit, fewer tokens.

### The honest finding: this is not a token saver

Measured against the doc 13 cost model:

| Scope | Saved | Cost | Net |
|---|---|---|---|
| Full 10k context, 30% reduction | $0.000625 | $0.001477 | **−2.4×** |
| Full 50k context, 30% reduction | $0.003128 | $0.007384 | **−2.4×** |
| User message only (120 tok) | $0.000008 | $0.000018 | **−2.4×** |

The ratio is scale-invariant, and the reason is structural:

> Reduction removes **prefill** tokens (≈15% of decode cost per token), but the
> reducer must **decode** the reduced text to produce it. It pays full decode
> price to save discounted prefill price. It cannot win on tokens alone.

### Which does not mean drop it — it means justify it correctly

The reducer earns its place three other ways:

1. **Router input.** A cleaned, intent-explicit prompt makes classification
   markedly more accurate — and the router (§4) is worth 88% of a step. The
   reducer's real ROI is *making the router work*, not saving prefill.
2. **Quality.** Typos and ambiguity cost *steps* — the agent asks a clarifying
   question, or goes down a wrong path and backtracks. One avoided step dwarfs
   any prefill saving.
3. **Cached reuse.** If the reduction is stored with the session (§5), later
   steps reuse it free and it passes break-even at ~2.4 reuses.

### Design consequences

| Do | Don't |
|---|---|
| Scope it to the **user's new message** only | Run it over the full assembled context |
| Use **non-LLM** normalization first (spellcheck, unicode, whitespace, casing) | Reach for an LLM on a 40-token message |
| Escalate to a small LLM only above a length threshold | Run it on every message unconditionally |
| Cache the reduced form on the session | Re-reduce the same text each step |
| Measure it on **steps avoided**, not tokens removed | Report prefill savings as the win |

Classical normalization is essentially free and captures most of the typo
benefit. The LLM pass is for genuine ambiguity, and should be the exception.

---

## 4. Router — small agent vs. large agent

**Stage 3 ROUTE.** The highest-value optimization on the list.

```
800 output tokens on class L = $0.001112
800 output tokens on class S = $0.000139
saved per downrouted step    = $0.000973   (88% of the step)
```

Every step successfully sent to S instead of L saves ~88% of that step. Nothing
else in the catalog is close.

### The router itself must be nearly free

An LLM call to decide which model to call is a tax on every single step. Use a
cascade of increasingly expensive classifiers, stopping as early as possible:

| Tier | Method | Cost | Handles |
|---|---|---|---|
| 0 | Heuristics (length, tool-call-only step, continuation) | ~0 | Large fraction |
| 1 | Embedding + trained classifier (Pool-E) | ~free | Most of the rest |
| 2 | Small LLM judge | Real | Genuinely ambiguous only |
| 3 | Default to L | — | Unclassifiable |

Tier 0 catches more than it sounds: a step whose only job is emitting a tool call
does not need a frontier model, and those are common in agent loops.

### Escalation is what makes or breaks it

If a cheap answer is wrong and must be redone on L, that step cost **S + L**,
not L. The break-even escalation rate is roughly:

```
max_escalation_rate ≈ 1 − (price_S / price_L) ≈ 88%
```

Comfortable headroom — but the failure mode is not cost, it is **quality**. A
router that downroutes hard problems produces worse answers without producing a
visible error, which is the most expensive kind of failure a platform can have.

Classified **HIGH** risk: eval gate, human review, slow ramp (doc 10 §6). Track
`downroute_rate`, `escalation_rate`, and a quality delta per route decision.

### Interaction

Overlaps with the state saver and semantic cache: a cache hit means no routing
decision at all. Measure the stack end-to-end (doc 10 §7), never by summing.

---

## 5. State Saver — warm start

**Stage 5 SETTLE, consumed by Stages 1 and 2.** The cheapest good idea in the
catalog: remember enough about a session that resuming it is nearly free.

### What gets saved

| Artifact | Enables | Where |
|---|---|---|
| Normalized/reduced prompt (§3) | Skip re-reduction | Session state |
| Assembled prefix + its hash | Prefix cache hit | Redis + engine cache |
| Compacted context summary | Skip re-summarization | Session state |
| Route decision + difficulty score | Skip re-classification | Session state |
| Matched skills/tools (doc 16) | Skip re-matching, **stable prefix** | Session state |
| Replica affinity | Land on the warm replica | `affinity:{session_id}` |

### Why it compounds

The state saver is what converts every other optimization from a per-step cost
into a once-per-session cost. The reducer's 2.4× loss becomes a win at 3 reuses
*because* the state saver holds the result. It is infrastructure for the rest of
the catalog more than a standalone saving.

### Design notes

- **Prefix stability is the real product here.** Keep the saved prefix
  byte-identical across steps; anything that reorders it voids the cache.
- Tie the TTL to the slot's idle reap window (doc 01) — state should outlive a
  brief pause, not a dead session.
- Cheap to store: a few KB per session in Redis, spill to Postgres on IDLE.
- Namespace by tenant (invariant I-6).

Classified **LOW** risk. This is the first one to build.

---

## 6. Combined effect

Savings do not add (doc 10 §7) — the router's win partly overlaps the cache's,
skills change step counts which changes everything downstream. Rough placeholder
expectation for the stack, pending measurement:

| Stack | Effective $/1M | Break-even util (Pro) | Headroom |
|---|---|---|---|
| Baseline | $1.39 | 22.0% | 4.4× |
| + state saver / prefix cache | ~$0.97 | ~31% | ~6.3× |
| + skills (conditional loading) | ~$0.83 | ~37% | ~7.4× |
| + router | ~$0.50 | ~61% | ~12× |
| + caveman (amortized) | ~$0.45 | ~68% | ~13.6× |

⚠️ Illustrative only. Every row must come from the doc 10 measurement harness on
real traffic before it informs pricing.

## 7. Build order

Cheapest and safest first; the risky, high-value one after the infrastructure
that makes it measurable exists:

1. **State saver** — LOW risk, enables everything else
2. **Skills with progressive disclosure** (doc 16) — big, structural
3. **Reducer, non-LLM tier only** — near-free, improves routing accuracy
4. **Caveman on system prompts and skill bodies** — authoring-time, amortized
5. **Router, tiers 0–1** — the big win, behind an eval gate
6. Router tier 2, LLM reducer escalation — only if measurement justifies them

---
*Next: [18 — Ultra-Thinking Mode](18-ultra-thinking.md)*
