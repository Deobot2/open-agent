# 19 — Contributed Compute & Credits

> Skeleton, and the most technically fraught item on the list. The idea is good;
> the obvious implementation does not work. This doc explains why and proposes
> the version that does.

## 1. The idea

Users contribute their own hardware to process work for the platform. Compute
contributed earns credit against their subscription. Free users contribute for
platform benefits instead of money.

Genuinely attractive: it converts idle consumer GPUs into capacity, aligns
customers with our cost structure, and creates a reason to stay.

## 2. What does not work: sharding live inference

The intuitive version — put some model layers on user hardware, some on ours —
fails on three independent grounds, any one of which is disqualifying.

### Latency

Decode is **sequential**: every output token traverses the full layer stack. A
network hop mid-stack is paid per token, not per request.

```
50 ms round trip × 800 output tokens = 40 seconds of pure network
```

That is before compute. Interactive agents cannot work this way. (Petals-style
distributed inference is real and works — for batch, at latencies no interactive
product can accept.)

### Privacy — the disqualifying one

Sharding another tenant's prompt onto a stranger's GPU means **customer A's data
is processed in customer B's RAM**. Activations are not plaintext, but they are
derived from it and are increasingly invertible. This violates invariant I-6 and
every reasonable customer expectation. No credit scheme is worth it.

### Trust

Volunteer nodes can return subtly wrong results. Detecting a hostile participant
who returns *plausible* garbage requires redundant execution — which costs more
than the contribution is worth.

**Conclusion: do not shard live inference across user hardware.**

## 3. What does work: two viable models

### Model A — Self-serve compute (recommended first)

> Your hardware runs *your own* account's auxiliary work.

The privacy problem disappears entirely, because no data crosses tenants.

| Runs locally | Latency-tolerant? | Cost saved |
|---|---|---|
| Embeddings for skill/tool matching (doc 16 §4) | Yes | Pool-E |
| Context summarization / compaction | Yes | Pool-E |
| Prompt reduction, non-LLM tier (doc 17 §3) | Yes | ~free anyway |
| Semantic cache index maintenance | Yes | Pool-E |
| Draft model for speculative decoding | **No** — inline | Not viable remotely |
| Small-class agent steps (class S) | Borderline | Pool-S |

Simple, private, immediately valuable. Offloads Pool-E, which is exactly the
capacity every optimization in doc 17 competes for.

### Model B — Volunteer batch pool (later, harder)

> Your hardware runs *platform* batch work that contains no customer data.

| Candidate work | Contains customer data? |
|---|---|
| Model eval suite runs (doc 11 §7) | No |
| Synthetic data generation | No |
| Public/free agent traffic (doc 20), with consent | Disclosed |
| Benchmarking | No |
| Anything touching a paying tenant's prompts | **Yes — excluded** |

Smaller and slower to build, but it is real capacity and it does not require
trusting participants with anyone's data.

**Recommendation: ship Model A first.** It delivers most of the benefit with a
fraction of the risk, and Model B can follow once verification is proven.

## 4. Credit economics

Measured against the doc 13 cost model:

| Quantity | Value |
|---|---|
| Contributor's electricity (350 W @ $0.15/kWh) | **$0.052 / hr** |
| Our cost for equivalent compute | **$0.625 / hr** |
| Viable credit band | **$0.05 – $0.62 / hr** |

There is real margin. But the band has a trap at the top:

| Credit rate | Contributor earns/mo | Our margin | Problem |
|---|---|---|---|
| $0.10/hr | $73 | +$0.53/hr | Fine |
| $0.20/hr | $146 | +$0.43/hr | **Exceeds a $99 Pro plan** |
| $0.30/hr | $219 | +$0.33/hr | **Free subscription + cash** |

Uncapped 24/7 contribution can out-earn the subscription it is meant to
discount. Three rules follow:

| # | Rule | Why |
|---|---|---|
| C-1 | **Credit never exceeds a % of plan price** (suggest 50% cap) | It is a discount, not a wage — no payouts, no negative invoices |
| C-2 | **Credit accrues only for work we actually dispatched** | Natural demand cap; we never pay for capacity we did not need |
| C-3 | **Credit rate ≤ 40% of our marginal cost saved** | Preserves margin on every contributed hour |

C-2 is the important structural one: it makes the program self-limiting. If
there is no batch work queued, no credit accrues, so the scheme cannot outrun
demand.

## 5. Fraud — the hardest part

The credit turns compute into money, which creates an incentive to fake it.

| Attack | Guard |
|---|---|
| Report work never done | Verifiable output: results must hash-match a spec |
| Return plausible garbage | **Spot-check** — re-run k% (5–10%) on trusted hardware |
| Sybil (many fake nodes) | Account binding, payment identity, per-account caps |
| GPU spoofing | Timing attestation — real work has a measurable latency floor |
| Result replay | Nonce per work unit |
| Collusion between nodes | Randomized assignment; never same unit to related accounts |

**Spot-checking is the backbone.** Full redundant execution costs more than the
contribution; sampling 5–10% with a hard ban on mismatch makes cheating
negative-expected-value while keeping overhead low.

Reputation: new nodes start untrusted (high spot-check rate, low-value work) and
earn trust over time. Trust is lost instantly and permanently on a mismatch.

## 6. Architecture sketch

```
oa-contrib (new control service)
  ├── node registry      enrollment, capability probe, attestation
  ├── work dispatcher    units → eligible nodes (privacy class enforced)
  ├── verifier           spot-checks, hash matching, reputation
  └── ledger             contributed units → credits → oa-billing
                                                          │
                                            invoice discount, capped by C-1

oa-contrib-client (user-installed)
  ├── sandboxed runner   pinned engine build, no host access
  ├── resource governor  GPU %, thermal, schedule ("only when idle")
  └── reporting          signed results + timing attestation
```

Every work unit carries a **privacy class**, and the dispatcher treats it as a
hard filter:

| Class | Contains | May run on |
|---|---|---|
| `own` | The contributor's own account data | That contributor's node only |
| `public` | No customer data | Any enrolled node |
| `tenant` | Another tenant's data | **Platform hardware only — never dispatched** |

That table is the whole privacy design, and the `tenant` row must be enforced in
the dispatcher, not by convention.

## 7. Open questions

| ID | Question |
|---|---|
| D-08 | Model A only, or A then B? |
| D-09 | Credit as invoice discount only, or a free-tier currency too? |
| D-10 | Credit rate, and the C-1 cap percentage |
| D-11 | Client distribution — desktop app, CLI, container? |
| D-12 | Do we support CPU-only contribution, or GPU only? |
| D-13 | Tax/regulatory treatment of credits (get advice — discounts and payments differ) |

D-13 is worth resolving early: a credit that can only reduce an invoice is
commercially and legally simpler than anything resembling earnings, which is a
further argument for the C-1 cap.

## 8. Honest assessment

| | |
|---|---|
| **Strongest as** | A retention and community feature that happens to offload Pool-E |
| **Weakest as** | A serious source of interactive inference capacity |
| **Biggest risk** | Fraud economics — credits are money, and money attracts attackers |
| **Biggest constraint** | Privacy. It rules out the most capacity-rich version of the idea |
| **Suggested scope** | Model A, Pool-E work only, credit capped at 50% of plan price |

Build it for the reason it is actually good — alignment and stickiness — and let
the capacity be a bonus. Sized that way it is a strong feature; sized as a
capacity strategy it will disappoint.

---
*Next: [20 — Free Tier & Public Agent](20-free-tier-and-public-agent.md)*
