# 20 — Free Tier & Public Agent

> Skeleton, and explicitly exploratory — you flagged this as still in progress.
> Structure and decision points, not conclusions.

## 1. The idea

A free, public agent as a funnel: anyone can try it, no signup. Contributors to
the compute program (doc 19) get more of it without paying.

## 2. This is the one place the slot model does not protect us

Every paid plan is safe because a subscription bounds the damage: slots × rate
ceiling × price. A free public agent has **no subscription and no identity**, so
both terms of that protection are gone.

> **The free tier is the only part of the platform that must have a hard token
> cap.** Everywhere else, caps would contradict the product. Here, a cap is the
> product's only defense.

## 3. Containment

| Control | Placeholder | Purpose |
|---|---|---|
| Model class | **S only** | Cheapest per token |
| Priority | **P2** | Never competes with paying work |
| Pool | **Pool-B** (spot) | Cheapest capacity we have |
| Rate ceiling | 5 tok/s | Slow but usable |
| Session cap | 20 steps / 10 min | Bounds a single visit |
| Daily token cap | Hard, per identity | The actual defense |
| Concurrency | 1 | No parallel farming |
| Context | 32k | Bounds KV pressure |
| Ultra-think | ✗ | Cost multiplier (doc 18) |
| Tools | Read-only, allowlisted | Bounds abuse and egress |

### Identity without signup

The hardest part: rate limiting anonymous users. Layered, weakest first:

| Layer | Strength | Cost to user |
|---|---|---|
| IP address | Weak (CGNAT, VPN) | None |
| Browser fingerprint | Moderate | None |
| Proof-of-work challenge | Moderate | Slight delay |
| Email signup | Strong | Friction |
| Payment card on file | Strongest | High friction |

Suggested ladder: **anonymous** gets a small daily allowance; **email signup**
gets more; **contributor** (doc 19) gets more again. Each step trades friction
for allowance, and each step also makes the user more valuable.

## 4. Why the public agent can be unusually cheap

A public agent sees enormous question overlap — "what can you do", "write me a
poem", the same trending topics all day. That makes it the **best possible
candidate for aggressive caching**:

| Optimization | Effect on a public agent |
|---|---|
| Exact response cache (Stage 1) | High hit rate |
| Semantic cache (Stage 1) | **Very high** — many phrasings, few intents |
| Shared system prompt prefix | ~100% prefix cache hit, one prompt for everyone |
| Router (Stage 3) | Nearly everything routes to S |

A public agent with a strong semantic cache may serve a large fraction of
traffic without inference at all. **It is also the ideal proving ground for the
doc 10 pipeline**: high volume, low risk, no paying customer to disappoint, and
a clean signal on cache hit rates. Consider shipping optimizations here first.

One caveat that must be designed in: the public agent's cache is shared across
anonymous users by definition, which is safe **only** because there is no tenant
data in it. Enforce it structurally — the public agent runs under a dedicated
synthetic tenant, and its cache namespace is never reachable from a paying
tenant's path (invariant I-6).

## 5. Contributor benefits (doc 19 §7 D-09)

Options for what a non-paying contributor earns:

| Benefit | Cost to us | Draw |
|---|---|---|
| Higher daily token cap | Low | Moderate |
| Class M access | Medium | High |
| Priority P1 instead of P2 | Low | Moderate |
| More concurrent sessions | Medium | Moderate |
| Credit toward a future subscription | Deferred revenue | **Highest** |

The last is the most interesting: it converts contribution into a subscription
on-ramp rather than a permanent free ride, which is the behavior we actually
want from the program.

## 6. Funnel

```
public agent (anonymous)
   → email signup           (more allowance)
   → contributor            (more again, doc 19)
   → Trial                  (full features, 1 slot, token cap)
   → Solo / Pro             (unlimited tokens, real slots)
```

The conversion pitch writes itself and is honest: the free tier is capped in
exactly the way paid plans are not.

> *"You've hit today's limit. Paid plans don't have one."*

That is the single clearest statement of the product, and the free tier's main
job is to make a visitor feel it.

## 7. Cost model

```
free_tier_monthly_cost
  = daily_active_users × sessions_per_user × tokens_per_session
  × (1 − cache_hit_rate) × cost_per_token_S
```

The `(1 − cache_hit_rate)` term dominates. At an 80% semantic cache hit rate the
free tier costs a fifth of its naive price, which is why the caching work in §4
is not an optimization here — it is the business case.

Set an explicit **monthly budget** for the free tier and degrade on breach:
lower caps, longer queues, or a waitlist. Never let it borrow paid capacity.
Budget breach should trip automatically, not require a human.

## 8. Open questions

| ID | Question |
|---|---|
| D-14 | Anonymous access at all, or email required from the start? |
| D-15 | Free-tier monthly budget ceiling |
| D-16 | Is the public agent a *product* (a named assistant) or a *demo*? |
| D-17 | Do contributor benefits stack with a paid plan, or only replace it? |
| D-18 | Public agent content moderation posture — anonymous input is abuse-prone |

D-18 needs attention before launch, not after: an anonymous, free, public LLM is
a magnet for abuse, and the moderation classifier (doc 12 §8) is on Pool-E,
which is the capacity the free tier is already competing for.

## 9. Deferred

- Public agent branding and UX
- Embeddable widget
- Shareable conversation links
- Referral mechanics

---
*Back to [PLAN.md](../PLAN.md)*
