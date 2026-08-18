# Glossary

Terms used precisely throughout this document set. Where a word has a loose
industry meaning and a specific meaning here, the specific one wins.

| Term | Definition |
|---|---|
| **Agent** | A saved configuration (model class, prompt, tools). Free and unlimited to create. |
| **Session** / **Run** | One execution of an agent. Holds a slot from start to finish. |
| **Slot** | The right to have one session active. **The unit we sell, meter, schedule, and price.** |
| **Active agent** | A session currently holding a slot. The customer-facing name for a slot in use. |
| **Lease** | A time-bounded grant of a slot (60 s TTL), renewed by heartbeat. |
| **Step** | One iteration of the agent loop: assemble → infer → act. |
| **Slot-second** | One slot held for one second. The base unit of internal cost accounting. |
| **Occupancy** | Fraction of a plan's purchased slots actually held. Expected 0.2–0.4. |
| **Duty cycle** | `active_seconds / slot_seconds`. Fraction of held time actually generating. Expected 0.1–0.2. |
| **Utilization (combined)** | `occupancy × duty_cycle`. Expected ~0.05. Compared against break-even. |
| **Break-even utilization** | The combined utilization at which a plan stops being profitable. One number per plan; see doc 13 §5. |
| **Rate ceiling** | Per-slot sustained tokens/second. Makes the worst case finite and knowable. |
| **Burst** | Token bucket depth. Absorbs spiky agent work so the ceiling is rarely felt. |
| **Overcommit ratio** | Sold slots ÷ simultaneously-servable slots. Justified by utilization < 1. |
| **Model class** | S / M / L. The stable contract; the model behind a class is swappable. |
| **Pool** | A homogeneous group of GPU replicas serving one class. The scheduling unit. |
| **Replica** | One engine process serving one model on one or more GPUs. |
| **Prefix cache** | Engine-level reuse of a shared prompt prefix. **Keyed per tenant** (invariant I-6). |
| **Prefix affinity** | Routing a session's steps to the replica already holding its prefix. |
| **Priority class** | P0 (never delayed) / P1 (standard) / P2 (deferrable). |
| **Stage** | One of the five optimization hooks: ADMIT, ASSEMBLE, ROUTE, DECODE, SETTLE. |
| **Plugin** | One optimization, mounted in exactly one stage. Toggleable per tenant. |
| **Shadow mode** | Running a plugin to compute savings without applying it. |
| **Quality risk** | NONE / LOW / MEDIUM / HIGH. Determines the shipping gate. |
| **Lossy** | A Stage 2 plugin that drops information. Automatically ≥ MEDIUM risk. |
| **Eval gate** | Automated quality suite blocking model promotions and risky optimizations. |
| **Reaping** | Reclaiming a slot from an idle or heartbeat-dead session. |
| **Fail open / fail closed** | Lease *renewals* fail open (keep running); *acquisitions* fail closed (refuse). |
| **Entitlements** | The machine-readable limit object. The only source of tier behavior. |
| **Invariant (I-1..I-7)** | Properties every change must preserve. Listed in doc 02 §3. |
| **Decision (D-01..D-07)** | Open architectural decisions tracked in PLAN.md §7. |

## Words we avoid

| Avoid | Because |
|---|---|
| "Unlimited" without qualification | Tokens are unlimited; concurrency and rate are not. Say which. |
| "Quota" | Implies a depleting balance. There isn't one. |
| "Throttle" (customer-facing) | Sounds punitive. The rate ceiling is a published plan feature. |
| "Concurrent requests" | Ambiguous. Say **slots** (sold) or **batch size** (engine). |
| "Token limit" | Reserved for context length, never for a monthly allowance. |
