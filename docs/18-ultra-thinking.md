# 18 — Ultra-Thinking Mode

> Skeleton. The one item on the list that *increases* cost — so the design
> question is containment, not savings.

## 1. What it is

A premium mode that trades latency for quality: the agent decomposes a task,
runs sub-agents in parallel or sequence, and synthesizes their results. Slower,
deeper, better on hard problems.

Everything else in doc 17 reduces cost. This one multiplies it, which makes it a
**product feature with a cost-control problem**, not an optimization.

## 2. It breaks the slot model — unless sub-agents share the budget

The slot economics (doc 13) hold because one slot consumes at most
`sustained_tps` tokens per second. Sub-agents put that directly at risk:

| Design | Cost of one slot at 100% duty | Verdict |
|---|---|---|
| Normal slot @ 25 tok/s | **$90 / month** | Baseline |
| 3 sub-agents, each at full 25 tok/s | **$270 / month** | 3× blowout |
| 5 sub-agents, each at full 25 tok/s | **$450 / month** | 5× blowout |
| 10 sub-agents, each at full 25 tok/s | **$901 / month** | 10× blowout |
| **N sub-agents sharing the parent's 25 tok/s** | **$90 / month** | ✅ **Unchanged** |

If sub-agents each get their own rate ceiling, a Pro customer can turn a 5-slot
plan into 50 agents' worth of consumption without buying anything — and every
break-even number in doc 13 becomes fiction.

## 3. The resolution: fork the token bucket, don't create new ones

> **A slot's rate ceiling is shared across the parent and all of its sub-agents.**

```
parent slot: 25 tok/s sustained
   ├── sub-agent A  ─┐
   ├── sub-agent B  ─┼── all draw from the SAME 25 tok/s bucket
   ├── sub-agent C  ─┘
   └── synthesis
```

With N active sub-agents, each effectively runs at ~`25/N` tok/s. The mode
therefore produces *more thinking at the same cost*, spread over more wall-clock
time.

This lands exactly where you already wanted it: **"it will slow things down."**
That slowness is not a limitation bolted on — it is the direct, honest
consequence of doing more work within a fixed budget. The design and the desired
behavior agree, which is the sign it is the right one.

### Consequences

| Property | Result |
|---|---|
| Cost to us | **Identical** to a normal slot. Fully bounded. |
| Slots consumed | **One.** Sub-agents are not separate slots. |
| Wall-clock time | Longer, roughly ∝ total work ÷ budget |
| Customer experience | "Deeper, slower" — a clear, explicable trade |
| Economic model | **Unchanged.** Doc 13 still holds exactly. |

## 4. Sub-agent model

```
ultra_think(task):
    plan      = decompose(task)              # 1 call, class M/L
    results   = parallel([ sub_agent(s) for s in plan.subtasks ])   # class S/M
    synthesis = combine(results)             # 1 call, class L
```

- Sub-agents are **ephemeral** — no slot, no lease, no separate session record.
- They inherit the parent's tenant, entitlements, rate bucket, and priority.
- Depth is capped (default 2). Sub-agents spawning sub-agents is where recursive
  cost blowups live, and a depth cap is the only reliable guard.
- Fan-out is capped (default 5 concurrent).
- Sub-agents routinely run on **cheaper classes** than the parent — decomposed
  subtasks are usually narrower and easier, so ultra-think and the router (doc 17
  §4) work together rather than against each other.

## 5. Plan gating

| Plan | Ultra-think | Max fan-out | Max depth |
|---|---|---|---|
| Trial | ✗ | — | — |
| Solo | ✓ | 3 | 1 |
| Pro | ✓ | 5 | 2 |
| Team | ✓ | 5 | 2 |
| Business | ✓ | 8 | 2 |
| Enterprise | ✓ | Custom | Custom |

Gated by plan not because it costs us more — it does not, given §3 — but because
it consumes the slot for far longer, and slot-time is what customers are actually
buying. Higher tiers get wider fan-out because they hold more slots.

## 6. Metering

Sub-agent work is attributed to the parent session so cost attribution stays
intact:

```jsonc
{ "session_id": "…", "step_id": 7,
  "ultra_think": { "depth": 1, "fan_out": 4, "sub_agent_id": "sub_2" },
  "completion_tokens": 1840 }
```

New dashboard rows (doc 11 D3): ultra-think adoption rate, mean fan-out, and
**tokens per completed task** with vs. without — the only measure of whether the
mode is worth its wall-clock cost to the customer.

Expect ultra-think sessions to raise **duty cycle** (more of the held time is
spent generating) while leaving **cost per slot-hour** flat. That is the signature
of a correctly-implemented budget fork, and it is worth alerting on the inverse:
if cost per slot-hour rises with ultra-think adoption, the bucket is leaking.

## 7. Risks

| Risk | Guard |
|---|---|
| **Sub-agents bypass the rate bucket** | Single shared limiter object; invariant test in CI |
| Recursive spawning | Hard depth cap enforced in the runtime, not the prompt |
| Slot held for hours | Idle reaping does not apply (it is working) — cap total steps |
| Sub-agent failure cascades | Partial results synthesize; never fail the whole run |
| Customer expects it to be *faster* | Name and document it as the slow mode |

The first row deserves a dedicated invariant, since it is invisible when broken:

> **I-8 — All sub-agents of a session draw from that session's single rate
> bucket.** No code path may construct a second limiter for a sub-agent.

## 8. Deferred

- Sub-agent specialization (researcher / critic / synthesizer roles)
- Adaptive fan-out based on task difficulty
- Letting customers see and steer the decomposition
- Ultra-think as a paid per-invocation add-on

---
*Next: [19 — Contributed Compute & Credits](19-contributed-compute.md)*
