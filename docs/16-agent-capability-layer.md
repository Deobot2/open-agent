# 16 — Agent Capability Layer

> Skeleton. Skills, tools, and plugins as a subsystem — plus the auto-skill
> maker that writes into it.

## 1. The three capability types

| Type | Is | Lives | Costs tokens |
|---|---|---|---|
| **Tool** | A callable function the agent may invoke | Registry, executed by `oa-toolproxy` | Schema in context + result tokens |
| **Skill** | A written procedure the agent follows | Skill store, injected as text | Body in context when loaded |
| **Plugin** | Platform code that changes behavior | `oa-agent-runtime` | None (runs outside the prompt) |

The distinction matters for cost: **plugins are free, tools are cheap, skills
are expensive** — because a skill is text that occupies context on every step it
is loaded for.

## 2. Skills reduce tokens only if loaded conditionally

A skill saves tokens by replacing rediscovery — the agent reads a known
procedure instead of reasoning one out across several steps. But the body has to
be in context to be read, and that is a recurring per-step cost.

The arithmetic decides the whole design:

```
50 skills × 500 tokens = 25,000 tokens injected on EVERY step
```

At 25k tokens of prefill per step across every session, a skill library becomes
one of the largest line items on the platform — and most of it is never read.
So the rule is:

> **Progressive disclosure is mandatory. Never load all skill bodies.**

| Level | What is in context | Size |
|---|---|---|
| **L0 — always** | Skill *names* + one-line descriptions | ~20 tokens each |
| **L1 — on match** | The matched skill's body | ~500 tokens |
| **L2 — on demand** | Referenced files the skill points to | Variable |

50 skills at L0 is ~1,000 tokens, not 25,000. The body loads only when the
router or the agent selects it. This single decision is the difference between
skills being a net token *saver* and a net token *disaster*.

## 3. Skill lifecycle

```
authored / generated → validated → indexed → matched → loaded → measured
                                                                   │
                                         ┌─────────────────────────┘
                                         ▼
                              promoted · demoted · pruned
```

Every skill carries usage telemetry. A skill that is never matched is dead
weight in the L0 index; a skill that is matched but does not improve outcomes is
worse than dead weight. Both get pruned automatically.

```jsonc
{
  "id": "skill_...",
  "org_id": "...",                    // or null = platform-wide
  "name": "extract-invoice-fields",
  "description": "Pull totals, dates, line items from an invoice",   // L0
  "body": "…",                                                       // L1
  "trigger": { "keywords": [...], "embedding": [...] },
  "origin": "authored | generated",
  "stats": { "matched": 412, "loaded": 380, "helped": 341, "tokens_saved_est": 1840000 },
  "status": "active | shadow | deprecated"
}
```

## 4. Skill matching

Matching must be cheaper than the thing it saves, so it is not an LLM call:

1. Embed the user message (Pool-E, ~free) → cosine match against skill embeddings
2. Keyword/regex prefilter for exact triggers
3. Threshold; load top-k bodies (k default 1–2, hard cap 3)

**The cap is load-bearing.** Without it, a vague prompt matches eight skills, and
4,000 tokens of skill bodies land in context to answer a one-line question.

## 5. Auto-skill maker

Watches completed sessions, finds repeated multi-step procedures, and writes
them into reusable skills so the procedure is read next time instead of
re-derived.

```
session transcripts
   → cluster similar successful task traces (embedding, offline/batch)
   → candidate: ≥ N occurrences, consistent shape, good outcome
   → draft skill (LLM, batch, off-peak — Pool-B)
   → validate: dedup against existing, lint, size cap
   → SHADOW: matched and measured, not loaded
   → promote if it demonstrably helps
```

Runs **offline on batch capacity**, never on the hot path. Skill generation is
latency-insensitive, so it belongs in the cheapest tier available.

### Failure modes this must be built against

Auto-generation is a library that grows without a librarian, and every failure
mode is a token cost:

| Risk | Guard |
|---|---|
| **Unbounded growth** → L0 index bloats | Hard cap per org (e.g. 200); prune by value |
| **Near-duplicates** | Embedding dedup at ≥0.92 similarity; merge, don't add |
| **Wrong skill** — confidently bad procedure | Shadow first; promote only on measured improvement |
| **Overfitting** to one session | Require N ≥ 5 distinct sessions |
| **Drift** — skill outlives its context | Decay score; re-validate quarterly |
| **Cross-tenant leakage** | Generated skills are org-scoped by default (invariant I-6) |

That last one is absolute. A skill generated from org A's sessions can contain
org A's data, names, and internal procedures. Promotion to platform-wide
requires explicit review and scrubbing — never automatic.

## 6. Measuring whether a skill earns its place

A skill's value is the tokens it saves minus the tokens it occupies:

```
net = (steps_avoided × avg_step_tokens × price_decode)
    − (times_loaded × body_tokens × price_prefill)
```

Because it replaces *decode* (expensive) with *prefill* (cheap), a skill that
reliably removes even one step is strongly positive. That is the whole reason
skills are worth the complexity — and the reason a skill that removes zero steps
is pure cost.

Reported through the same `SavingsReport` as every other optimization (doc 10 §3).

## 7. Where this sits in the pipeline

| Function | Stage (doc 10) |
|---|---|
| Skill matching + L0 index | **2 ASSEMBLE** |
| Skill body loading (L1/L2) | **2 ASSEMBLE** |
| Tool schema pruning (only relevant tools) | **2 ASSEMBLE** |
| Auto-skill generation | **offline** — not on the hot path |
| Skill usage telemetry | **5 SETTLE** |

### Tool schema pruning is the same problem again

Tool JSON schemas are large and are re-sent every step. An agent with 30 tools
may carry 6,000+ tokens of schema it will never call on this step. Same fix:
match a small relevant subset, and keep the selection **stable within a session**
so the prefix cache still hits (doc 10 §7).

## 8. Deferred

- Skill marketplace / sharing between orgs
- Versioning and rollback of generated skills
- Skill composition (skills that call skills)
- Customer-authored skill UI
- Cross-org skill promotion review workflow

---
*Next: [17 — Optimization Catalog](17-optimization-catalog.md)*
