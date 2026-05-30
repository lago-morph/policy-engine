# E1 — Policy Simulation & Dry-Run Framework — ADVERSARIAL REVIEW

**Component:** E1 · **Reviewer persona:** Daniel (Auditor) + red-team hat · **Date:** 2026-05-30
**Target:** `SPEC.md`, `PLAN.md`, `ALT-*.md` in this dir.

Mandate: attack assumptions, find gaps/inconsistencies vs other components, unhandled abuse/failure cases, scope creep, and "won't survive production" findings. Prioritized defect list at the end.

---

## 1. Attacks on core assumptions

**A1. "Authoritative over the complete subset" can still mislead.** The headline matrix counts only `complete` events, but the *report consumer* (Priya, Daniel) sees big confident numbers ("47 newly blocked") while a silent `insufficient` bucket of 9,000 events is hidden below the fold. **The authoritative count is only meaningful relative to the completeness histogram.** If 95% of events are `insufficient`, "0 newly allowed" is dangerously reassuring. → The report MUST surface `authoritative_coverage = complete / events_evaluated` as a *headline* number, and the promotion gate should refuse promotion when coverage is below a threshold (not just when untagged_risky>0). The SPEC has `requireComplete` but defaults it loosely.

**A2. `replay_completeness` is computed by C2, but *only E1 knows which fields the new bundle reads.*** §17.3 completeness is policy-relative: an event can be `complete` for bundle v12 and `insufficient` for v13 (which reads a new field). The SPEC's field-introspection downgrade (§7) handles this, BUT it means C2's stored `replay_completeness` is **necessarily stale/wrong for any new policy**. → E1 MUST recompute completeness *per bundle* at run time; the C2-stored value is only a lower bound. This is under-emphasized and is a correctness landmine: a naive implementation trusting C2's flag will over-report authority.

**A3. Allow-class normalization hides tightening.** Folding `warn` and `mutate` into allow-class means a v12-`allow` → v13-`warn` (or `allow`→`mutate`) is "unchanged allowed" in the headline matrix. The SPEC's `effect_changed_within_class` secondary diff exists but is *not gated*. A reviewer tagging only the four buckets will never see a mutation that silently rewrites every payment Deployment. → within-class effect changes (esp. new mutations) MUST be promoted to a first-class reviewable bucket, not a footnote.

**A4. Suspend ≠ Deny for the operator, but the matrix conflates them.** `suspend_pending_approval` is deny-class in the SPEC. But a v12-`deny` → v13-`suspend` is "continued block" in the matrix, yet it is a *real workflow change* (now an approver gets paged). Conflation loses a meaningful, on-call-affecting transition.

---

## 2. Gaps / unhandled cases

**G1. Mutation replay is hand-waved.** Re-executing a *mutating* policy (Kyverno mutate, Gatekeeper assign) over historical input — what is the "decision"? The mutated object? The diff? The SPEC normalizes mutate→allow-class but never specifies how a mutation's *content* is compared across bundles. Two bundles that both "allow via mutate" but mutate differently are "unchanged allowed." **Mutation-diff comparison is unspecified.**

**G2. Multi-engine evidence sets.** An EvidenceSet drawn by `control_id` may span Gatekeeper, Kyverno, and Conftest events. The differential engine must evaluate each event with the *engine that owns it* — you cannot replay a Kyverno admission event through OPA `eval` and get a faithful result. The SPEC's WS-A "uniform evaluate()" glosses over the fact that **the engine is per-event, not per-run.** Cross-engine bundle pairs (v12 Gatekeeper vs v13 Kyverno) have no defined semantics.

**G3. External-data snapshot may not exist in real audit logs.** D-E1-02 downgrades to `partial` when `external_data_snapshot` is absent — but in practice §13 only requires `external_data_refs` (name + version/digest), **not the snapshot value**. If only the digest is stored, E1 must *fetch the value by digest from somewhere*. Where? A content-addressed external-data store is implied but never specified or assigned to a component. Without it, every external-data-dependent policy replays as `partial`, which is most real image-signing/CVE policies — gutting the flagship use case.

**G4. Time-window selection is unbounded.** "last 30 days, all clusters" (HL-17) over a busy fleet could be 10⁸ events. WS-C says "stream/chunk/resume" but there is no defined cap, sampling strategy, or cost estimate surfaced *before* a user launches a run. A Namespace Author could accidentally (or maliciously) launch a fleet-wide replay. → need pre-flight event-count estimate + quota + scope coupling.

**G5. Tag tampering / accountability.** Tags gate promotion. Can the same person who authors the policy tag all newly-allowed as "Intended relaxation" and self-promote? The SPEC says tags carry actor+rationale but does not require **separation of duties** (author ≠ tagger ≠ approver). This is the obvious abuse path to "blind promotion" — the very thing HL-17 claims to prevent. D3 approval may cover promotion, but *tagging* itself is unguarded.

**G6. Fixture drift.** RegressionFixtures (§17.5) are saved with a `policy_input` snapshot. When the audit schema evolves (C2 v2), old fixtures may no longer match the new input shape, silently passing/failing. No fixture-schema-version pinning is specified.

**G7. Determinism claim vs real OPA.** "D-FULL ⇒ byte-identical forever" is false if the bundle uses `time.now_ns`, `http.send`, `rand`, or `opa.runtime()`. The SPEC only addresses external data, not these builtins. ALT-decisionlog-reuse mentions `nd_builtin_cache` as the fix but the *primary* SPEC doesn't require capturing it. → D-FULL is overclaimed; needs an explicit non-deterministic-builtin handling requirement.

---

## 3. Inconsistencies vs other components

**X1. C2 schema contract is asserted, not agreed.** SPEC §7 pins to spec §13.3 field names, but C2's actual SPEC was unwritten at authoring time. If C2 renames `jwt_claims`→`identity.claims` or drops `request_object` for PII reasons, every replay mode breaks. The adapter (D-E1-03) is the right hedge but the *field-level* contract needs to be a shared, versioned artifact in cross-cutting `DATA-MODEL.md`, not embedded prose in E1.

**X2. PII / legal redaction collides with replay fidelity.** §17.3 says preserve "original raw event where legally and operationally permissible." When raw event/`before_state`/`request_object` is redacted for GDPR/PII, the policy that reads those fields can't replay → `insufficient`. The tension between D4 security (minimize stored sensitive data) and E1 (preserve everything for replay) is real and unresolved. Differential simulation's value *degrades exactly in regulated namespaces* — the ones that need it most (payments-prod).

**X3. `PolicySimulationRun` CRD ownership.** SPEC §8 (E1) and B4 (§17C.6 CRD catalog) both claim the CRD. Schema authority must be singular. Reconcile: B4 owns the CRD *definition*; E1 owns the *controller*. Stated in E1 §8 but must match B4.

**X4. Exception linkage timing.** Tagging "Potential false positive" + filing a `PolicyException` (D3/B4) then re-running — but exceptions are scoped CRDs with approval workflow. A re-run "with the exception applied" (HL-17 step 8) requires E1 to evaluate the exception *as policy input*. How an unapproved/pending exception affects a sim re-run is unspecified (does a pending exception count? only approved?).

---

## 4. "Won't survive production" findings

**P1. The completeness gate is the whole ballgame, and real audit logs are rarely complete.** In production, admission logs routinely lack `before_state`, external-data values, and full `request_object` (sampling, truncation, retention cost). The honest expectation is that a *large fraction* of historical events are `partial`/`insufficient` for any non-trivial policy. The product's headline feature (authoritative differential) will, on day one against real data, report low authoritative coverage — a credibility problem if not framed up front (ties to A1).

**P2. Cost of 2× evaluation over fleet-scale history.** Re-executing both bundles over 30 days × all clusters is expensive and slow. Without sampling/stratification (G4), the flagship demo (HL-17) is a multi-hour batch job, not the interactive console experience implied by §16.

**P3. Engine-version skew.** The "previous" bundle v12 was evaluated by OPA v0.X in production; the replay runs OPA v0.Y. Rego semantics / builtins can differ across OPA versions. Replaying v12 under a newer engine may not reproduce the original decision — silently corrupting the "previous" side. Engine version must be pinned per bundle, not just bundle version.

---

## 5. Scope-creep watch

- M3 Live Shadow drags E1 into the production hot path and a streaming infra surface — arguably a separate product (the ALT acknowledges this). Keep it opt-in/post-MVP (PLAN already does).
- "Cross-product replay" (M2 over Jenkins/GitLab/Elastic events via §17D) multiplies engine adapters; risk of E1 becoming the universal replay engine for 9 products before the core K8s case is solid.

---

## 6. Prioritized defect list

| # | Severity | Defect | Fix |
|---|---|---|---|
| D1 | **Critical** | Authoritative count meaningless without coverage headline; completeness is policy-relative and C2's stored flag is stale for new bundles (A1, A2, P1) | Recompute completeness per-bundle at run time; surface `authoritative_coverage` as headline; gate promotion on coverage threshold |
| D2 | **Critical** | External-data *value* store unspecified; most real policies replay as `partial` (G3, P1) | Specify a content-addressed external-data snapshot store; assign ownership (C2 or new); without value, mark partial explicitly |
| D3 | **High** | Mutation replay & diff semantics unspecified (G1) | Define mutated-object diff comparison; promote mutation changes to first-class bucket |
| D4 | **High** | Engine is per-event not per-run; cross-engine pairs undefined (G2) | Bind each event to its owning engine; forbid/spec cross-engine bundle pairs |
| D5 | **High** | Non-deterministic builtins break D-FULL claim (G7, P3) | Require `nd_builtin_cache` capture; pin engine version per bundle |
| D6 | **High** | No separation of duties on tagging ⇒ self-promotion abuse (G5) | Require author ≠ tagger or approver-confirms-tags before promote |
| D7 | **Medium** | Within-class tightening (allow→warn/mutate, deny→suspend) hidden (A3, A4) | Gate `effect_changed_within_class` as reviewable |
| D8 | **Medium** | PII redaction vs replay fidelity unresolved in regulated namespaces (X2) | Document degradation; consider tokenized/structural-only replay inputs |
| D9 | **Medium** | Unbounded evidence-set selection; no pre-flight estimate/quota (G4, P2) | Pre-flight count + quota + sampling/stratification |
| D10 | **Low** | Fixture schema-version pinning missing (G6) | Pin fixture to audit-schema version; migration on bump |
| D11 | **Low** | CRD ownership split E1/B4 must be reconciled (X3) | B4 defines CRD, E1 owns controller — state in both |
| D12 | **Low** | Pending vs approved exception behavior in re-run unspecified (X4) | Only approved exceptions affect authoritative re-run; pending shown as hypothetical |
