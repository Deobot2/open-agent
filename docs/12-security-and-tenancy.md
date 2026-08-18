# 12 — Security & Tenancy

> Skeleton. Establishes boundaries and non-negotiables; controls are stubs.

## 1. Tenancy model

**Shared infrastructure, isolated data.** One tenant = one org. Enterprise may
buy dedicated pools; everyone else shares GPUs.

| Layer | Isolation |
|---|---|
| Data (Postgres) | `org_id` on every row + row-level security |
| Cache (Redis) | Key namespacing by `org_id` |
| **Prefix cache (GPU)** | **Namespaced by tenant — invariant I-6** |
| Agent runtime | Process-level; one session per coroutine |
| Tool execution | Sandboxed container per call |
| Inference | Shared batch, isolated KV entries |
| Storage (S3) | Per-tenant prefix + per-tenant encryption key |

### The prefix cache is the sharpest edge

A prefix cache keyed only by content hash will happily serve tenant B a cache
entry created by tenant A. If A's system prompt contains proprietary
instructions and B sends a colliding prefix, that is a cross-tenant data leak
through a *performance* feature.

```
cache_key = H(org_id || model_id || prompt_prefix)
```

The `org_id` in that hash is mandatory and must be enforced in the engine
configuration, not just in our code — it costs a small amount of cache hit rate
and prevents an entire class of breach. Any optimization in doc 10 that touches
caching inherits this requirement.

## 2. Authentication & authorization

| Surface | Mechanism |
|---|---|
| Customer API | Bearer API keys (`oa_live_…`), Argon2id hashed |
| Dashboard | OAuth/OIDC + session cookies, MFA available |
| Internal (service→service) | mTLS + SPIFFE-style identity |
| Admin | SSO + MFA required, all actions audited |

Roles: `owner` > `admin` > `member`. Scoped API keys deferred.

## 3. Network

```
Internet ──► WAF ──► Edge (only public tier)
                       │
              ┌────────┴────────┐
              │ private subnets │   no default route to internet
              │ control · agent │
              │ GPU · data      │
              └────────┬────────┘
                       │
                oa-toolproxy ──► internet (the ONLY egress path)
```

- GPU and control nodes have **no outbound internet route**. Weights come from
  S3 via a VPC endpoint.
- All agent tool egress goes through `oa-toolproxy`: per-tenant allow/deny
  lists, DNS pinning, private-range blocking (SSRF), rate limits, full logging.

Forcing every agent-initiated request through one proxy is what makes egress
auditable at all. An agent that can reach the internet directly from a GPU node
is an exfiltration path with no chokepoint.

## 4. Tool sandboxing

Agent tools run untrusted-ish code on behalf of customers.

| Control | Setting |
|---|---|
| Runtime | gVisor / Firecracker microVM |
| Lifetime | Ephemeral, one per call |
| Filesystem | Read-only base + scratch tmpfs |
| Network | Through toolproxy only |
| Limits | CPU, memory, wall-clock (default 30 s) |
| Secrets | Injected per call, never persisted |

## 5. Prompt & data handling

| Principle | Implementation |
|---|---|
| Customer prompts are customer data | Encrypted at rest, per-tenant keys |
| No training on customer data | Contractual + no pipeline exists to do it |
| Transcripts opt-out available | Per-org setting; ephemeral mode keeps nothing |
| Retention | 30 d default, tenant-configurable, hard-deleted |
| PII | Not extracted or indexed by default |

**Prompt injection** is treated as a customer-facing risk we help with, not one
we claim to solve: tool results are clearly delimited in the prompt, the
toolproxy constrains what a hijacked agent can reach, and destructive tools are
gated. The honest position is that the sandbox and egress policy bound the
damage rather than preventing the injection.

## 6. Secrets

- Vault (or cloud KMS) for all service credentials
- No secrets in env vars in prod; injected at runtime
- Rotation: 90 d service creds, 30 d internal certs
- Customer tool secrets encrypted per-tenant, decrypted only inside a sandbox

## 7. Compliance posture (aspirational)

| Standard | Status |
|---|---|
| SOC 2 Type II | Target Phase 5 |
| GDPR | Design for it now: deletion, export, DPA |
| HIPAA | Not in scope v1 |
| Data residency | Deferred with multi-region |

Designing for GDPR deletion from the start is cheap; retrofitting per-tenant
hard delete across ClickHouse, S3, and backups is not.

## 8. Abuse prevention

| Vector | Control |
|---|---|
| Slot farming across trial accounts | Payment/identity verification, device fingerprinting |
| Inference resale | ToS + anomaly detection on traffic patterns |
| Credential stuffing | Rate limits, MFA, breach-password checks |
| Resource exhaustion | Rate ceilings (doc 05 §5) — already structural |
| Prohibited content | Classifier on Pool-E, tiered response |

Note that the rate ceiling doubles as the primary abuse control: an attacker who
compromises a key still cannot consume more than that key's slot ceiling. The
economic design and the security design reinforce each other here.

## 9. Incident response

1. Detect (alerts, doc 11)
2. Triage — severity by customer impact
3. Contain — kill switches: per-tenant suspend, per-plugin disable, pool drain
4. Communicate — status page; direct contact for affected tenants
5. Post-mortem — blameless, within 5 business days

## 10. Deferred

- Full threat model
- Pen test cadence
- Bug bounty
- Customer-managed encryption keys
- Audit log export for enterprise

---
*Next: [13 — Capacity & Economics](13-capacity-and-economics.md)*
