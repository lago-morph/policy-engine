# G8 — Rego-Authoring & Human Factors — PLAN

**Component:** G8 · **Domain:** G — Operational / NFR · **Status:** DRAFT v1 · **Date:** 2026-05-30
**Reads with:** `SPEC.md` (this dir), B1, A2, B3, E1, E2, E3, D2, D3, G3.

This plan is built to exploit parallelism: the four workstreams (linting/test framework, generator,
template library, onboarding/measurement) are largely independent and converge on the inner-loop contract.
The critical path runs through the **lint ruleset** (everything gates on it) and the **mandatory-simulation
guardrail** (the highest-value safety item, blocked only on E1).

---

## 1. Workstreams

| WS | Name | Owns (SPEC §) | Can start when |
|---|---|---|---|
| **WS-A** | **Linting & inner loop** | Regal platform ruleset + custom rules (§5.1), LSP setup (§5.4), pre-commit (§5.5), local eval harness (§5.3), the inner-loop=CI contract (§5, F8) | B1 metadata/decision contract (B1 §4–5) frozen; Regal available (market §2) |
| **WS-B** | **Test framework** | unit-test ergonomics + scaffold helpers (§5.2), coverage floor wiring (§5.2), audit-derived fixture workflow (§5.2/§5.3) | B1-R31 coverage contract; E1 §17.5 fixture export (for the audit-derived part) |
| **WS-C** | **Generator quality bar** | §26.1 full-vs-template gate from the HF side (§4.1/4.2), fail-closed-TODO defaults (§4.2), typed `template_todo` slots lint can verify (§4.3, F2) | A2 `:generate` / `GenerationDecision` entity exists; A1 catalog stub; E3 catalog (for "is there a decision point?") |
| **WS-D** | **Template & cookbook library** | template catalog keyed by (archetype × product) (§4.3), built on E3 entries; cookbook patterns (§9); each lint-clean+test-green | E3 per-product libraries (E3 §5); WS-A lint ruleset (templates must pass it) |
| **WS-E** | **Guardrails — expert footguns** | mandatory differential sim for enforce (§6.1), R-LINT-FAILOPEN/OVERBROAD (§6.1), portability lint + conformance gate (§6.2), replay-completeness lint (§6.3), metadata single-source (§6.4) | WS-A (lint host); E1 (differential sim); B1-R30 conformance suite; G3 (circuit-breaker contract) |
| **WS-F** | **Novice NS-author guardrails** | template-and-fill primary path (§7), most-restrictive-wins / can-only-tighten lint (§7), permissiveness check (§7), mandatory-review wiring (§7) | WS-D (templates), WS-A (lint), D2 (NS scope), E1 (permissiveness sim), E2 (NS view) |
| **WS-G** | **Review & approval** | CODEOWNERS wiring + DT-71 (§8), code-review-distinct-from-promotion gate (§8), machine-assisted review evidence on PR (§8) | WS-A/WS-E (the evidence to surface); A2/D3 (promotion approval to be distinct from) |
| **WS-H** | **Onboarding, docs & measurement** | golden onboarding path (§9), error-message contract (§10), author docs/cookbook (§10), the M1–M9 metrics + instrumentation (§11) | WS-A..G feed metrics; E2/C5 for dashboards; mostly authorable in parallel as a doc/instrumentation track |

---

## 2. Dependency DAG

```
            B1 §4–5 (metadata, canonical decision, purity/replay contract)   ── frozen first
                 │
                 ▼
   ┌──────────  WS-A  Linting & inner loop  ───────────────────────────────┐   ← CRITICAL PATH ROOT
   │  (Regal platform ruleset + LSP + pre-commit + local-eval harness)      │
   │       │                │                  │                            │
   │       ▼                ▼                  ▼                            ▼
   │     WS-B            WS-C  Generator     WS-D  Templates           WS-E  Expert guardrails
   │   Test fw        (full/template gate)  (built on E3 + WS-A)      (lint rules + MANDATORY SIM)
   │     │                  │ (needs A2,E3,A1)    │ (needs E3)             │ (needs E1, B1-R30, G3)
   │     │                  └─────────┬──────────┘                        │
   │     │                            ▼                                    │
   │     │                     WS-F  Novice NS guardrails  ◄───────────────┘
   │     │                 (templates + lint + permissiveness sim + D2 + E2)
   │     │                            │
   │     ▼                            ▼
   │   WS-G  Review & approval (CODEOWNERS/DT-71; code-review ≠ promotion; A2/D3)
   │     │
   └─────┴──────────────────────────► WS-H  Onboarding + docs + M1–M9 measurement (E2/C5)
```

**External blockers (contracts G8 does not own but requires):**
- **E1** differential simulation must be callable as a gate (WS-E §6.1, WS-F §7). *Highest-value external dep.*
- **B1-R30** cross-engine conformance suite must run in CI (WS-E §6.2 portability guard).
- **G3** must provide staged rollout + rollback + deny-rate circuit breaker (WS-E §6.1, SPEC G8-MUST-032).
- **E3** per-product libraries must exist for the template catalog (WS-D).
- **A2** `:generate` + `GenerationDecision` + the G-AUTHOR/G-DIFF gates G8 feeds.
- **D2** NS write-scope boundary; **E2** NS Authoring + Rego Explorer views render the constraints.

---

## 3. Critical path

```
B1 §4–5 frozen
  → WS-A lint ruleset (R-LINT-META/CTRLID/PKG/DECISION/PURE/CLAIMS/PORT/FAILOPEN) + inner-loop=CI contract
    → WS-E mandatory-differential-sim guardrail for `enforce`  [blocked on E1]
      → WS-F novice NS guardrails (most-restrictive-wins + permissiveness check)  [blocked on E1, D2, E2]
        → WS-G review gates  → WS-H measurement (M3/M5/M6/M7/M9 depend on guardrails firing)
```

The lint ruleset (WS-A) is the spine: WS-D templates must pass it, WS-E/WS-F guardrails are mostly lint rules,
WS-G reviews surface its output, WS-H measures its catches. **De-risk by landing WS-A's R-LINT-META + the
local-eval harness first** (smallest correct inner loop), then layering the safety rules.

The **single highest-value, highest-risk item** is **WS-E §6.1 mandatory differential simulation for enforce**
— it is the guardrail that prevents the fleet-wide mass-deny (the #1 OPA field incident) and the only one
fully blocked on an external component (E1). Pull E1's gate interface forward; if E1 slips, WS-E ships the
*lint* halves (FAILOPEN/OVERBROAD/PORT/CLAIMS) which are E1-independent, and the sim gate lands when E1 does.

---

## 4. What can be built concurrently / what blocks what

**Embarrassingly parallel once WS-A's lint ruleset interface is fixed:**
- WS-B (test framework) — only needs B1-R31 + the test-shape helper; the audit-derived-fixture piece waits on E1.
- WS-C (generator gate) — needs A2/A1/E3 stubs, not the full lint; develop against the lint *contract*.
- WS-D (templates) — one template per (archetype × product) is **embarrassingly parallel** across products
  (mirrors E3's per-product parallelism, E3 OQ-1); each template is independently authorable + testable.
- WS-H docs/onboarding — authorable in parallel as a documentation track; only the *metrics instrumentation*
  half waits on the guardrails to have something to measure.

**Hard serialization (cannot parallelize away):**
- WS-E mandatory-sim gate ⟶ blocked on E1 differential engine.
- WS-F permissiveness check ⟶ blocked on E1 + D2 + E2 NS view.
- The conformance gate (WS-E §6.2) ⟶ blocked on B1-R30 suite existing.
- WS-G four-eyes-on-code composing with promotion SoD ⟶ needs A2/D3 promotion-approval semantics fixed.

**Anti-bottleneck:** WS-A must *not* wait for the full guardrail set; ship the minimal correct inner loop
(R-LINT-META + local eval + pre-commit) early so authors get value and the team gets feedback on lint
ergonomics before the heavy rules land.

---

## 5. Milestones

| M | Deliverable | Exit criteria | Scenarios |
|---|---|---|---|
| **M0** | Inner loop MVP | Regal platform ruleset (R-LINT-META/CTRLID/PKG/DECISION) + LSP + pre-commit + local-eval harness; identical in editor/pre-commit/CI | DT-11 (metadata), DT-40 (coverage) |
| **M1** | Test + coverage | `opa test --coverage` floor wired into G-AUTHOR; scaffold helpers; audit-derived fixture workflow | DT-40, DT-25 |
| **M2** | Generator quality bar + templates | full-vs-template gate honors §4.1; fail-closed TODO defaults; typed-slot lint; template catalog ≥ 80% POC controls, each lint-clean+test-green | DT-09 |
| **M3** | Expert guardrails | R-LINT-FAILOPEN/OVERBROAD/PORT/CLAIMS live; **mandatory differential sim** non-bypassable for enforce; B1-R30 conformance gate in CI; G3 circuit-breaker contract wired | DT-05, HL-02, HL-17 |
| **M4** | Novice NS guardrails | template-and-fill primary path; most-restrictive-wins lint; permissiveness check routes `permissive_ns_policy` to central review; NS promotion needs sim+review | DT-43 |
| **M5** | Review & onboarding | CODEOWNERS/DT-71; code-review distinct from promotion; four-eyes; golden onboarding path hits time targets; actionable-error contract live | DT-71, HL-04 |
| **M6** | Measurement | M1–M9 metrics instrumented + on E2/C5; M9 author-escape-rate measured over a real corpus and below threshold (the gated acceptance criterion) | §11 |

---

## 6. Test strategy

- **Toolchain dogfooding:** the §9 tutorial corpus and §9.1 cookbook patterns are **integration tests for the
  DX** — if a tutorial or cookbook pattern stops being lint-clean/test-green, the toolchain regressed (CI runs them).
- **Lint-rule tests:** every custom Regal rule (§5.1) ships positive + negative fixtures (a package that should
  pass, one that should fail) so the rules themselves can't silently rot.
- **Guardrail red-team tests:** explicit adversarial fixtures — an overbroad mass-deny policy (must be blocked
  by R-LINT-OVERBROAD + caught by mandatory sim), a too-permissive NS policy (must be flagged
  `permissive_ns_policy` + routed to review), a replay-incomplete policy (must fail R-LINT-CLAIMS), a
  non-portable construct (must fail B1-R30 conformance). These prove the guardrails *fire*, not just exist.
- **Inner-loop = CI parity test:** a harness that runs the identical ruleset/invocation in both contexts and
  asserts identical results on a corpus (guards F8 "works on my machine").
- **Generator regression:** a corpus of controls with known full/template expectations; the gate must not drift
  toward emitting incomplete `full` (guards F1).
- **Metric instrumentation tests:** M1–M9 computed from synthetic event streams with known outcomes.

---

## 7. Sequencing recommendation (MVP-first)

Given the corpus's MVP scoping (F3) and that G8 is an NFR layer, the **highest-ROI MVP cut** is:

1. **M0 + M3's mandatory-sim guardrail** — the inner loop plus the one guardrail that prevents the
   reputation-ending mass-deny. This alone moves the platform from "expert-only, footgun-prone" to "an expert
   cannot trivially nuke a fleet."
2. **M2 templates for the POC's actual controls** — so new controls start paved-road, not blank.
3. **M4 novice NS guardrails** — gated behind the NS-authoring feature actually shipping (E2 V5 + D2); if NS
   authoring is deferred (per the sequencing memos' "single-tenant first"), M4 defers with it, but its
   *requirements* stay on record so NS authoring is never enabled without them.
4. **M5/M6** — review formalization + measurement; M6's M9 gate is what lets the platform *claim* the median
   engineer succeeds, so it is a GA gate, not an MVP gate.

> Defer-with-the-feature rule: **novice NS-authoring guardrails (WS-F/M4) MUST NOT lag behind the NS-authoring
> feature.** Shipping E2's Namespace Authoring view (D2 write scope) without G8's NS guardrails ships the
> security hole the whole component exists to close. They release together or not at all.
