# C4 — Retrospective Audit Detection — SPEC

**Component ID:** C4 · **Domain:** C — Evidence, Audit & Analytics
**Spec sources:** §19 (Retrospective Audit Detection Scenario: §19.1 Objective, §19.2 Example + Analytics Flow), with inputs from §14.2 (bypass), §13 (C2 schema), §17.2/§17.3 (historical replay), §17E.3 (audit-derived violation report), §23 (evidence integrity).
**Status:** SPEC (depends on C2 frozen schema v1.0; consumes C3 detectors; requests E1 replay).
**Scenarios exercised:** HL-06 (retrospective bypass), HL-12 (outage retro uncovers silent regression), DT-30/DT-42 (bypass), DT-46/DT-78 (historical replay → violation report), HL-18 (auditor independent replay).

---

## 1. What C4 is (and how it differs from C3)

C4 answers one question over a **whole audit window**: *"Did anything ever bypass enforcement — and what would the verdict have been?"* (§19.1).

- **C3** runs **continuous interval detection** on the live stream (≤15 min, "is something being bypassed *right now*?").
- **C4** runs an **on-demand retrospective sweep** over an arbitrary historical window (a quarter, an audit period, an incident window), **reconstructs** the policy input for any unenforced action, **drives a replay** (via E1) against the policy that *should* have applied, and produces the **Audit-Derived Violation Report** population (§17E.3) — the auditor's evidence that the platform can prove, after the fact, what slipped through.

**Critical ownership rule (resolving C3 adversarial D7):** C3 and C4 **share the same detector library** (the bypass/coverage/correlation logic), so they cannot diverge. C4 = the library run over an *operator-chosen window* with *reconstruction + replay attached*; C3 = the library run on a *rolling interval* for live alerting. C4 is where the reconstruction and replay live (C3 only flags).

---

## 2. The retrospective sweep

### 2.1 Objective (§19.1)
Detect workloads/actions that bypassed enforcement across a window, classify them, reconstruct their inputs, replay them against the policy that should have applied, and emit audit-grade violation evidence.

### 2.2 The §19.2 canonical flow, generalized to a window
The spec's example (admin disables Gatekeeper → privileged deployment created → no deny event) is the unit case. C4 generalizes it to *every* action in the window:
```
1. Enumerate observed actions      (K8s API audit + runtime inventory)
2. Find unenforced actions         (no paired Gatekeeper/OPA decision — the §14.2 bypass condition)
3. Confirm via independent source  (runtime scanner finds the violating workload still present)
4. Reconstruct the policy input    (from requestObject + JWT claims at request time)
5. Replay against the policy that should have applied   (E1)
6. Emit: enforcement-bypass alert + missing-evidence + governance-noncompliance + violation report
```

### 2.3 The three observation sources (defense against C3 adversarial D1 — "audit source itself disabled")
C4 reconciles **three independent views** of what exists, so a bypass is caught even if one source was tampered:
1. **Decision view** — C2 `policy.decision` events (what was *evaluated*).
2. **Audit view** — C2 `resource.change` from the K8s API audit log (what the *API server saw*).
3. **Inventory view** — the *currently/historically running* fleet from runtime scanners (§19.2 step 4: "runtime scan identifies violating workload"), independent of the API audit path.

```
                  ┌── Decision view (C2 policy.decision) ──┐
 reconcile  ──────┼── Audit view (C2 resource.change) ─────┼──▶  set differences
                  └── Inventory view (runtime scanner) ─────┘
```
**Bypass classification by which views disagree:**
- In Audit ∧ in Inventory ∧ **not** in Decision → **classic bypass** (created, exists, never evaluated — §19.2, HL-06).
- In Inventory ∧ **not** in Audit ∧ **not** in Decision → **deep bypass** (exists but the API audit *also* missed it — the adversary disabled auditing, C3-A1's hole). C4 catches this precisely because the inventory view is independent.
- In Audit ∧ in Decision ∧ **not** in Inventory → evaluated+denied+absent → consistent (enforcement worked).

### 2.4 Reconstruction (C4 owns this)
For each bypassed action, C4 reconstructs a **C2 `replay.synthetic` event** (C2 SPEC §3.9):
- `request_object` from the K8s audit `requestObject` (or, for deep bypass, from the inventory object's current spec — lower confidence).
- `jwt_claims` from the subject's token-issuance (Keycloak) event joined by `sub`+time → `jwt_claims_completeness=reconstructed`.
- `external_data_refs`: the value the policy *would* have consulted, re-resolved at the action's timestamp where possible; if unavailable → reason code + lower fidelity.
- `policy_version`: the bundle that was deployed to that scope at the action's `timestamp` (inferred from the drift/deployment record).
- **Invariant N-C4-1 (= C2 N-C2-SYNTH):** the reconstructed event is **at most `best_effort`** and carries `confidence_level ∈ {high, medium, low}` (DT-30 step 4: `replay_completeness=partial`; never `complete`).

### 2.5 Replay (delegated to E1)
C4 hands the reconstructed event to **E1** to evaluate against the inferred `policy_version`. C4 does **not** embed an evaluator (it requests one — same boundary as C3 N-C3-5). E1 returns the verdict (`decision=deny/allow`) and a trace. DT-30: replay of the reconstructed input against `bundle:v12` returns `deny`, `replay_completeness=partial`.

### 2.6 Confidence scoring
The violation's `confidence_level` (→ §17E.3) is a function of reconstruction fidelity:
- **high:** `request_object` from audit log, `policy_version` known, all `external_data_refs` re-resolvable (DT-78 class (a)).
- **medium:** some inputs defaulted conservatively (e.g. `external_data_refs` re-resolved with possible drift; `missing_fields=["image.digest"]`) (DT-78 class (b)).
- **low:** replay must disclaim a verdict; recommends operator review; **not auto-counted in violation totals** without confirmation (DT-78 class (c), success criterion). **N-C4-2: `confidence=low` violations require operator confirmation before counting.**

---

## 3. Outputs

### 3.1 The four emitted artifacts (§19.2 step 6)
1. **Enforcement-bypass alert** — per bypassed action (mirrors C3's `enforcement_bypass`, but produced for the historical window).
2. **Missing-evaluation-evidence** record — the explicit "no decision existed" fact, tied to `correlation_id` with `missing_evaluation_evidence=true`.
3. **Governance-noncompliance event** — control-level: this control was bypassed in this window (feeds C1 verdict → `not_satisfied`).
4. **Audit-Derived Violation Report** rows (§17E.3) — the auditor-facing population (handed to C5 for rendering).

### 3.2 Audit-Derived Violation Report contract (§17E.3, the fields C4 must populate)
Per detected violation, C4 produces all 9 §17E.3 fields:
`violation_timestamp` (from source log) · `discovery_timestamp` (when the sweep/replay produced `deny`) · `source_audit_log` (dereferenceable id, round-trips to the raw row — DT-78 SC) · `reconstructed_policy_input` (the synthetic event's `request_object`+`jwt_claims`) · `policy_version` (replay bundle) · `confidence_level` (high|medium|low) · `missing_fields` (list, possibly empty) · `matched_control_id` · `recommended_remediation` (string).
**N-C4-3:** every row carries all 9 fields; `missing_fields` is a list (empty allowed); `confidence=low` rows carry a non-empty `recommended_remediation` and are excluded from auto-totals (DT-78 SC).

### 3.3 Re-execution for auditor independence (DT-78 step 5, HL-18)
C4's outputs are backed by the reconstructed synthetic C2 events and the inferred bundle (addressable by digest). An auditor with read-only scope can **independently re-execute** each reconstructed input against the same bundle from their own session (via E1) and must tie out to C4's stored verdict for ≥95% of sampled rows; divergence is flagged (DT-78 SC, §17.4). This is the auditor-independence guarantee, inherited from C2's deterministic replay.

### 3.4 Materialized retrospective dataset
A sweep over a window produces a **materialized dataset** (C2 §8.5) — immutable, scope-tagged, digest-addressable — reusable by engineering and auditors without re-running the sweep (DT-46/DT-78 reuse pattern). The 30-day historical replay (DT-46) is a C4 sweep whose dataset Daniel reuses two weeks later.

---

## 4. The silent-regression case (HL-12) — retrospective differential

§19 is "did it bypass?"; HL-12 is the sibling: "did a policy *change* silently start *allowing* what it used to deny?" C4 supports this as a **retrospective differential sweep**:
```
for a window straddling a bundle promotion v_old → v_new:
   for each admission event evaluated under v_new:
      replay the same input under v_old (E1, differential — §17.4)
      if v_old=deny and v_new=allow → 'newly allowed' (silent regression candidate)
```
HL-12: a quota control regressed v11→v12 (allow where v11 denied), a workload was admitted, exhausted capacity. C4's retrospective differential finds the class of admissions newly allowed by the regression across the whole window — feeding the §17E.4 Simulation Report (rendered by C5/E1). **C4 owns the "over a historical window" framing; E1 owns the per-event differential evaluation.**

---

## 5. Failure modes
- **No reconstructable input** (deep bypass, no audit object, only an inventory object): reconstruct from inventory spec at `confidence=low`; disclose; never fabricate a `high`-confidence verdict.
- **Bundle-at-time unknown** (no deployment record for the scope at `timestamp`): `policy_version` inferred → reduces confidence; if uninferable, the violation is `indeterminate` (reported, not counted).
- **External-data value irrecoverable at the historical timestamp:** reconstruct `best_effort` with `external_data_value_unavailable` reason; the replay is indicative, disclosed.
- **Window too large** (performance): chunk by scope+time; materialize incrementally; the dataset is the unit of reuse.

## 6. Decisions
| ID | Decision | Rationale |
|---|---|---|
| D-C4-01 | C4 = on-demand window sweep; **shares C3's detector library** | One bypass logic, two cadences; cannot diverge (C3 adversarial D7). |
| D-C4-02 | Reconcile **three independent views** (decision/audit/inventory) | Catches "deep bypass" where the audit source itself was disabled (C3 adversarial D1). |
| D-C4-03 | Reconstructed events capped at `best_effort`; `confidence` scored by fidelity | §13.1 / N-C2-SYNTH; DT-30/DT-78. |
| D-C4-04 | `confidence=low` violations excluded from auto-totals pending confirmation | DT-78 SC; avoids manufacturing false violation counts. |
| D-C4-05 | Replay delegated to **E1**; C4 reconstructs only | Single evaluator (N-C3-5 sibling); auditor independence via E1 re-execution. |
| D-C4-06 | Retrospective **differential** sweep handles the HL-12 silent-regression class | §19 + §17.4; "did a change silently start allowing?" |
| D-C4-07 | Sweep output is a **materialized C2 dataset** (digest-addressable) | Reuse by eng + auditor without re-running (DT-46/DT-78). |

## 7. Open questions (defaults)
- **OQ-1:** Sweep cadence vs purely on-demand? *Default:* on-demand for arbitrary windows + a scheduled nightly sweep over the trailing day (DT-25/DT-78 nightly replay pattern), feeding the live violation report.
- **OQ-2:** How far back can a sweep go? *Default:* bounded by C2 raw-event retention (≥30 days POC); older windows are best-effort from whatever survives, disclosed.

## 8. Dependencies
- **Consumes:** C2 (events, correlation-members, datasets, raw-event retention for reconstruction); C3 detector library (shared); runtime scanner inventory (independent observation source); deployment/drift record (bundle-at-time inference); D1 (token-issuance events for JWT reconstruction).
- **Requests from:** E1 (replay + differential evaluation of reconstructed inputs).
- **Produces (consumed by):** C5 (Audit-Derived Violation Report §17E.3, Simulation Report §17E.4 inputs); C1 (governance-noncompliance → `not_satisfied` verdict); auditors (DT-78/HL-18 re-execution).
- **Blocks on:** C2 schema freeze (M-FREEZE), C3 detector library, E1 replay engine.
