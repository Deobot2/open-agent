# 15 — Roadmap

> Skeleton. Phases are dependency-ordered, not date-estimated.

## Phase 0 — Foundations & measurement

**Goal: know the real numbers before committing to anything.**

- [ ] Repo skeleton (doc 14), CI, local dev with mock inference
- [ ] Terraform for one GPU node + platform services
- [ ] **`bench` environment standing up**
- [ ] **Benchmark: tok/s per node per class, context curve, prefix cache rate**
- [ ] Fill in doc 13 with measured numbers; re-derive break-even per plan
- [ ] Decide D-02 (flagship model), D-03 (engine)

**Exit criterion:** doc 13's placeholders are replaced by measurements, and the
break-even table survives contact with reality.

Nothing after this phase is safe to commit to without it. If measured
throughput is half the placeholder, every plan's break-even halves too — and
that is a pricing decision, not an engineering one.

## Phase 1 — Vertical slice

**Goal: one agent, end to end, for one hardcoded tenant.**

- [ ] `oa-gateway`: auth, routing, SSE
- [ ] `oa-slots`: acquire/renew/release, Redis Lua, ledger
- [ ] `oa-agent-runtime`: the loop, with all five stages as **pass-throughs**
- [ ] One engine replica, class M, OpenAI-compatible
- [ ] `POST /v1/agents/{id}/runs` streams tokens
- [ ] Usage events → ClickHouse
- [ ] **Slot invariant tests in CI** (I-1, I-3)

**Exit:** an agent runs, streams, holds exactly one slot, and releases it —
including when its runtime is killed mid-run.

## Phase 2 — Multi-tenancy & the product

**Goal: real customers can subscribe and be correctly limited.**

- [ ] `oa-accounts`, `oa-plans` with versioned entitlements
- [ ] `oa-billing` + payment provider
- [ ] Plan tiers, enforced from entitlements only
- [ ] **Per-slot token buckets** (I-2) — the profitability enforcement point
- [ ] Idle reaping, the 409 response with active-run list (doc 08 §3)
- [ ] Dashboard: slots in use, usage view
- [ ] Duty cycle + occupancy measurement live (dashboard D3)
- [ ] Decide D-05 (rate ceiling) from Phase 0 + early usage data

**Exit:** a paying customer on a 5-slot plan can start 5 agents and not a 6th,
and we can see what they cost us.

## Phase 3 — Scale-out

**Goal: more than one node, more than one class, survive failures.**

- [ ] `oa-scheduler`: priority queues, fair share, **prefix affinity**
- [ ] `oa-registry`: model lifecycle, canary, rollback
- [ ] Pools S / M / L / E provisioned
- [ ] Pool-B on spot for P2
- [ ] Autoscaling within floor/ceiling
- [ ] AZ spread, failure drills
- [ ] Decide D-06 (overcommit) from measured duty cycle

**Exit:** a node can be lost without a customer noticing.

## Phase 4 — Optimization layer ← *your systems*

**Goal: drive down cost per token; widen the break-even headroom.**

- [ ] Plugin framework live: registry, per-tenant toggles, kill switches
- [ ] Shadow mode (compute without apply)
- [ ] Offline replay corpus + eval suite
- [ ] Savings attribution end-to-end (`tokens_saved`, `stages_applied`)
- [ ] Re-derive doc 13 with measured savings

Then the round-one systems, in the build order from doc 17 §7 — cheapest and
safest first, the high-value risky one once the harness can measure it:

- [ ] **State saver** (doc 17 §5) — LOW risk, enables everything else
- [ ] **Skills with progressive disclosure** (doc 16) — L0/L1/L2 loading
- [ ] **Prompt reducer, non-LLM tier** (doc 17 §3) — near-free, feeds the router
- [ ] **Caveman on system prompts + skill bodies** (doc 17 §2) — authoring-time
- [ ] **Router tiers 0–1** (doc 17 §4) — the 88% win, behind an eval gate
- [ ] **Auto-skill maker** (doc 16 §5) — offline, shadow-then-promote
- [ ] **Ultra-thinking** (doc 18) — shared rate bucket, invariant I-8 tested first

**Exit:** measured, net cost-per-token reduction with no quality regression, and
a break-even utilization materially above the baseline.

The framework lands before any optimization does. That ordering is the point:
the first optimization built without a harness is the one nobody can ever prove
is working, and the one nobody dares remove.

## Phase 5 — Hardening

- [ ] SOC 2 readiness, pen test
- [ ] Full SLO monitoring, error budgets
- [ ] DR: backups, restore drills, RTO/RPO
- [ ] Enterprise: dedicated pools, SSO, audit export
- [ ] Multi-region (if demanded)

## Phase 6 — Growth: free tier & contributed compute

**Goal: a funnel, and an aligned way for users to lower our costs.**

- [ ] Free public agent on Pool-B, class S, hard daily caps (doc 20)
- [ ] **Aggressive caching proved on public traffic first** — high volume, no
      paying customer at risk, clean signal on hit rates
- [ ] Anonymous → email → contributor allowance ladder
- [ ] `oa-contrib`: node registry, dispatcher, verifier, credit ledger (doc 19)
- [ ] Contributor client, sandboxed, self-serve (Model A) only
- [ ] Spot-check verification + reputation before any credit is issued
- [ ] Credit caps enforced: C-1 (% of plan), C-2 (dispatched work only), C-3 (rate)
- [ ] Decide D-08..D-18

**Exit:** the free tier runs inside a hard monthly budget, and contributed
compute measurably offloads Pool-E without a fraud problem.

Deliberately last. It depends on the optimization layer (the free tier is only
affordable with a high cache hit rate) and on metering being trustworthy
(credits are money). Building it earlier means paying for it twice.

## Sequencing rules

Three orderings are not negotiable, each for the same reason — measurement
before commitment:

| Rule | Why |
|---|---|
| **Phase 0 before pricing** | Pricing without measured throughput is a guess with a payment page attached |
| **Framework before optimizations** | Unmeasurable savings are indistinguishable from none |
| **Metering before scale** | Scaling an unprofitable unit economics makes it worse, faster |
| **Caching before the free tier** | An uncached public agent is an unbounded bill |
| **Verification before credits** | A credit is money; unverified work is free money |

## Parallelizable

Independent of the critical path — good candidates for a second workstream:

- Dashboard / web app
- SDKs and docs
- Eval suite content
- Terraform / IaC
- Marketing site

---
*Back to [PLAN.md](../PLAN.md)*
