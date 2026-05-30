# F4 — AI / Agent Governance Extension — ADVERSARIAL REVIEW

**Reviewer persona:** Red-team AI-safety + agent-security engineer + skeptical buyer. Mandate: attack the "no architecture change needed" thesis.

---

## 1. Headline finding

The reframe's central claim — *"very little of the original architecture has to change"* — is **rhetorically elegant and partially true, but it smuggles in the hardest unsolved problem in the platform as if it were a field addition.** The behavioral-evaluation tier (DELTA-C) is described as "just policies that consume model outputs," but evaluators that are themselves **non-deterministic, latency-heavy, expensive model calls placed on the real-time enforcement path** are categorically different from a Rego rule over a Kubernetes object. Calling them "PDP plugins" hides that the platform's core guarantees (deterministic replay, sub-millisecond admission, reproducible decisions) **do not hold** for this tier. **DEFECT-1 (critical).**

## 2. Defect list (prioritized)

**DEFECT-1 (critical) — Behavioral evaluators break the platform's defining guarantees.** The base platform's value is *deterministic, replayable, traceable* decisions. A judge-model evaluator is non-deterministic, can cost more and take longer than the action it gates, and cannot be exactly replayed. F4 acknowledges this in R-F4-AUD-2/EVAL-4 but then still places these evaluators inline via `require_async_check`. The contradiction: you cannot have a "replay-capable governance platform" whose key enforcement tier is non-replayable. Resolution must be explicit: behavioral evaluators are a **distinct, clearly-labeled best-effort tier**, never conflated with the deterministic tier in reports/evidence. (The ALT doc's async-evaluator architecture exists precisely because of this.)

**DEFECT-2 (critical) — "Same audit schema, just more fields" understates a privacy/scope explosion.** DELTA-D puts the *entire assembled context* (system prompt, full conversation, retrieved docs, tool args) into `request_object`. That is the most sensitive payload the platform has ever stored, it lands in the same store §26.1 defers, and it flows to every plugin/export-adapter (F2 DEFECT-3) and every `/audit/events` reader (F1 DEFECT-2). Field-level scope + classification-aware redaction is now **mandatory, not optional**, and retention of raw prompts may itself violate the regulations F4 claims to satisfy (EU AI Act / GDPR). The schema delta is small; the data-governance delta is enormous.

**DEFECT-3 (high) — The trust-gradient auto-relaxation is an attack surface.** DELTA-E promotes relaxations based on "the agent's past traces under a relaxed policy." An adversary (or a compromised/poisoned agent) can **behave benignly during the observation window to earn relaxation, then exploit it** — classic train-then-defect. "Replay shows it would have been fine" is exactly the wrong safety argument for an adaptive adversary. Auto-relaxation needs: adversarial-window detection, human approval for any privilege increase (not just "optionally"), and rate-limited, reversible grants. R-F4-TRUST-3 says "one capability at a time" but R-F4-TRUST-2's reviewer tagging is advisory.

**DEFECT-4 (high) — Capability-token intersection assumes honest runtimes.** R-F4-SUBJ-1 (child ≤ parent) and R-F4-APPR-2 (suspend/resume) **depend on the agent runtime cooperating**. A self-hosted or compromised runtime can ignore the capability token, skip the suspend, or forge the delegation chain. The MCP gateway (R4) is the only *enforced* choke point; everything that relies on the runtime self-reporting (step budgets, loop detection, delegation) is advisory unless the gateway independently observes it. The spec must mark which controls are **enforced at a choke point** vs **trusted to the runtime**.

**DEFECT-5 (high) — Replay-completeness caveat is a sales/compliance landmine.** R-F4-AUD-2 correctly says exact-output replay is best-effort. But the platform's pitch (and Wedge-7) is "audit-replay matters MORE for agents." Buyers will hear "we can replay what the agent did" and assume exactness. If a regulator asks "reproduce this decision," the honest answer for outputs is "we can't, only the policy decision." This gap between the marketing and the guarantee must be stated in the product, or it's misrepresentation under EU AI Act traceability obligations.

**DEFECT-6 (medium) — Evaluator latency budgets vs agent UX.** `require_async_check` gating an output on a judge-model call adds seconds of latency per output; for interactive agents this is fatal to UX, so teams will set permissive budgets or skip gating — defeating the control. The spec needs a tiered strategy (cheap deterministic pre-filters inline, expensive judge-model checks async/sampled), which is closer to the ALT async-evaluator design than the inline framing.

**DEFECT-7 (medium) — "Standards align, don't reinvent" pins to immature, fast-moving specs.** MCP, OTel-GenAI, FINOS-AIGF v2.0, and the NIST AI Agent Standards Initiative (announced Feb 2026, i.e. brand new) are all moving targets. Building enforcement on MCP as "the" tool-call layer is reasonable today but is a single-standard bet; non-MCP tool calls (direct function calling, proprietary runtimes) bypass the highest-leverage hook entirely. Need a non-MCP fallback interception story.

**DEFECT-8 (medium) — Sub-agent spawn + delegation chain has no depth/fan-out bound.** Agent A spawns B spawns C…; the delegation chain can explode. Step budgets are per-session, but a fan-out of sub-agents multiplies cost/risk. Need a chain-depth and total-fan-out ceiling enforced at spawn (which is itself a runtime-trusted control — see DEFECT-4).

**DEFECT-9 (low) — "Behavioral tier sits between Runtime and Detective" is a taxonomy that hides timing.** Some behavioral checks are pre-commit (output groundedness), some are periodic (drift), some are post-hoc (detective). Lumping them in one "tier" obscures which ones actually block. The catalog should classify each evaluator by *when it acts* (pre-commit / async-gating / periodic / detective).

**DEFECT-10 (low) — Trust grade as a single scalar over a tuple is lossy.** `trust_grade` collapses (agent, model, tools, context) into one value, but a tuple can be trusted for reads and untrusted for writes. Trust should be per-capability, not a scalar — the spec implies this ("one capability at a time") but the subject field is a single grade.

## 3. Inconsistencies vs other components

- **vs C2/F1:** DELTA-D depends on C2 being explicitly extensible (F1 DEFECT-8) AND field-level scope (F1 DEFECT-2) — both currently unresolved.
- **vs F2:** agent events are the worst case for plugin data-exfil (F2 DEFECT-3).
- **vs E1:** DELTA-E claims the differential engine "just works" on agent traces, but E1's determinism assumptions don't hold for outputs (DEFECT-1/5).
- **vs the positioning memo:** memo calls Wedge-7 "the optional long bet, watch 6 months"; F4 SPEC reads as committed — the ALT doc addresses this tension.

## 4. "Won't survive contact because…"

…the elegant "no architecture change" thesis is true for the *plumbing* and false for the *guarantees*: the moment a non-deterministic, expensive, runtime-trusted, privacy-explosive evaluator tier is placed on the enforcement path, the platform's deterministic-replay promise no longer covers the part buyers care most about — and the trust-gradient auto-relaxation rewards an adversary for patience.

## 5. Top fixes to merge into SPEC

1. Make the behavioral tier an **explicitly separate, best-effort, non-replayable** tier — never conflated with deterministic decisions in evidence (DEFECT-1, 5).
2. Mandatory field-level scope + classification-aware redaction + retention limits on agent `request_object` (DEFECT-2).
3. Human approval (not optional) for any trust-gradient privilege increase; adversarial-window hardening (DEFECT-3).
4. Label every control **enforced-at-choke-point vs runtime-trusted**; MCP gateway is the real enforcement, the rest is advisory (DEFECT-4, 8).
5. Tiered evaluator strategy (cheap inline / expensive async-sampled) + per-capability trust, not a scalar (DEFECT-6, 10). Treat standards as optional compatibility targets with non-MCP fallback (DEFECT-7).
