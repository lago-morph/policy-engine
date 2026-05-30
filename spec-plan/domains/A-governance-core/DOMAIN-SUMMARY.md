# Domain A — Governance Core — SUMMARY

**Scope:** A1 Governance Model (§6) + A2 Policy Lifecycle (§7). Domain A is the **root of the platform DAG**:
it mints the `control_id` everything else joins on, and it governs the safe transition of intent into running,
blocking enforcement.

---

## 1. The shared data model (the two components in one picture)

```
A1 (intent, system-of-record)                         A2 (lifecycle, orchestrator)
────────────────────────────                          ──────────────────────────────
Domain ─┬─ Objective ── Control ──┐                    PolicyLifecycle(control_id)
        │                         │                       └─ PolicyImplementation(s)
        │   ┌─ EnforcementReq ────┤   control_id            ├─ engine / mode / policy_version
        │   ├─ EvaluationReq      ├──────────────────────►  ├─ generated_from (full|template)
        │   ├─ EvidenceReq (⊇§13.3)│   (the join key)       └─ PromotionEvents (audited, ordered)
        │   └─ ExceptionReq ──────┘                       Gates: AUTHOR / SIM-BASELINE / DIFF /
        │                                                        SOAK / APPROVE / CLASS
        ├─ FrameworkRef ◄─satisfied_by─► Control          Reconciler: A1-status × A2-mode legality
        └─ LineageEdge (temporal, no graph DB)
```

**The single most important fact in Domain A:** `control_id` is the **universal join key for the entire
platform** — it appears in A1 controls, A2 implementations, Rego `__control_id__` (§8.3), Gatekeeper
constraints (§9.3), Conftest evidence (§10.3), audit events (§13.3), exception CRDs (§17C.6), and report
filters (§17E). **A1 is the sole allocator and uniqueness authority** (A1 D3). Get this wrong and every
downstream trace is built on sand.

**The second most important fact:** there are **two distinct lifecycles**, deliberately separated (A1 D7):
- **Governance status** (A1): `draft → in_review → active → deprecated → retired`.
- **Enforcement mode** (A2): `draft → dry-run → warn → enforce` (+ rollback/deprecated).

A control can be governance-`active` while enforcement is still `dry-run`. The two are linked by **events**,
not a shared field, and **A2 owns the reconciler** that keeps them legal (A2 D5).

---

## 2. Internal dependencies (A1 ↔ A2)

| From → To | Contract |
|---|---|
| A1 → A2 | `ControlActivated` event lets A2 create a lifecycle; A1 supplies `control_id`, `enforcement_class`, EnforcementReq (`mode_intent`, `deterministic`), EvidenceReq (replay window) |
| A1 → A2 | `ControlDeprecated` event drives A2 enforcement wind-down (DT-04) |
| A2 → A1 | `current_policy_version` (A2 is the authority; A1 mirrors read-only — OQ-4) |
| A2 → A1 | enforcement-points inventory answers A1's `deprecated→retired` guard (DT-04, closes A1-DEF-07) |
| A2 → A1 | reconciler findings (`governed_not_enforced`, `zombie_enforcement`, `mode_intent_conflict`) |

**Clean seam:** the two components never share a mutable field; everything crosses as events. This lets the A1
and A2 teams build in parallel against contract stubs.

---

## 3. The 5 hardest decisions in Domain A

1. **`control_id` as a single global immutable namespace, A1 the sole authority (A1 D3).** Chosen for
   referential integrity across the whole platform. Cost: no renames/reuse → the adversarial review demands an
   **alias mechanism** (A1-DEF-04) for typo-corrections/merges. *Resolution:* add `aliases[]` (accepted into the
   open-questions backlog; does not change the single-authority decision).

2. **Two lifecycles, event-linked, A2 owns the reconciler (A1 D7 + A2 D5).** Chosen so governance edits and
   enforcement tuning don't block each other. Cost: a seam where drift hides. Both adversarial reviews
   converge here (A1-DEF-03 + A2-DEF-04): the reconciler **must prevent the critical illegal states
   (`zombie_enforcement`, `premature_enforcement`) at the transition, not detect them on a timer.**
   *Resolution:* upgrade the reconciler from periodic-detection to transition-time-prevention for critical states.

3. **A1 is control-plane, off the enforcement hot path; engines cache what they need (A1 D9).** Chosen so Pod
   admission never depends on A1 uptime. Cost: cached **exceptions can fail *open*** under A1 partition
   (A1-DEF-02). *Resolution:* exception projection fails **most-restrictive** on staleness (deny new
   exceptions), not last-known.

4. **Demotion is ungated and instant; promotion is gated (A2 D4).** Chosen so rollback (DT-06) hits deny-rate-zero
   in <2 min without an approval queue causing an outage. Cost: a **one-click security-control kill switch**
   (A2-DEF-03). *Resolution:* keep ungated, but require **post-hoc** justification/review SLA.

5. **Relational + temporal lineage edges, no graph DB (A1 D6, per §26.1).** Chosen for POC simplicity and
   GitOps-reviewable Gemara YAML. The ALT proposes an **event-sourced governance ledger** for intrinsic
   tamper-evidence and free as-of-date queries; the **recommended synthesis** is a hybrid: CRUD authoring +
   a sealed, hash-chained governance-change *ledger* for the audit guarantee (best of both).

---

## 4. Consolidated open questions (decided defaults; revisit at GA)

| # | Question | Default | Owner |
|---|---|---|---|
| A-OQ-1 | `control_id` aliasing for renames/merges | add `aliases[]` (from A1-DEF-04) | A1 |
| A-OQ-2 | Graph DB vs relational lineage | relational + temporal CTE; event-ledger hybrid for tamper-evidence | A1 |
| A-OQ-3 | Coverage badge: management assertion vs certified | label as assertion; require operating-effectiveness signal to render green (A1-DEF-06) | A1 |
| A-OQ-4 | Reconciler: prevent vs detect | **prevent at transition** for critical states (A1-DEF-03/A2-DEF-04) | A2 |
| A-OQ-5 | Gates over incomplete audit history | assert `replay_completeness`; low ⇒ explicit ack (A2-DEF-01) | A2 |
| A-OQ-6 | `current_policy_version` under multi-selector rollout | selector→version *set*, not scalar; seal all live pairs (A2-DEF-06) | A2/A1 |
| A-OQ-7 | SoD when approver pool < 2 (POC team) | defined break-glass single-admin + heightened audit (A2-DEF-07) | A2 |
| A-OQ-8 | Gemara schema version pinning | pin version; scope signed export+verification to it (A1-DEF-05) | A1 |
| A-OQ-9 | Tightening vs loosening conflict on mode_intent | don't let "strictest wins" silently override deliberate relaxations (A2-DEF-05) | A2 |
| A-OQ-10 | Reusable requirement objects | templates that clone, not shared instances (A1 D1) | A1 |

---

## 5. What Domain A hands to the rest of the platform

- **To B1/B2/B3/B4:** `GovernanceProjection` cache (control_id, class, targets, required claims, ExceptionReq,
  current_policy_version); abstract `enforce` mode for B4 to map to concrete engine actions.
- **To C2:** the EvidenceRequirement (which §13.3 fields each control demands); governance-change + PromotionEvents.
- **To C5/§17E:** the coverage matrix + signed governance export (evidence packages); reconciliation findings.
- **To D3/§17C.6:** the ExceptionRequirement the `PolicyException` validator enforces.
- **To E1/§17:** the controls/implementations to simulate; A2 consumes E1's results as promotion gates.
- **To E2/§16.3:** control API + lineage for the Governance Graph View, framework panel, Rego Explorer.

Domain A is buildable **first and largely independently** (its only true upstream — the §13.3 field list — is
vendored as a constant), making it the natural Phase-0 of the master plan.
