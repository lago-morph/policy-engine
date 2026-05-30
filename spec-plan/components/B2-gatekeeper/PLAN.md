# B2 — Gatekeeper Integration — PLAN

**Component:** B2 · **Pairs with:** SPEC.md · **Date:** 2026-05-30

---

## 1. Dependency DAG

```
[B1 signed bundle + decision contract] ──► W1 Template generator (bundle → ConstraintTemplate)
[B1 correlation_id + decision log] ────────┐                    │
[D1 JWT mapping] ──────────────────────────┼──► W3 Audit-field emitter (17 fields) ──► [C2]
[B4 taxonomy + approval/exception CRDs] ───┤                    │
                                           │                    ▼
                              W2 Enforcement-mode FSM (dryrun→warn→deny) ──► [A2 gate]
                                           │
                                           ▼
                              W4 Suspend-pending-approval (deny+CRD+retry) ──► [B4 controllers]
                                           │
                                           ▼
                              W5 Audit/detective controller + reconciliation ──► [C4 §19]
                                           │
                                           ▼
                              W6 Webhook config (failurePolicy, timeouts, external-data)
```

Critical path: **W1 → W3 → W4** (the approval constraint is the riskiest integration).

---

## 2. Parallelizable workstreams

| WS | Title | Deps | Parallel with |
|---|---|---|---|
| W1 | Template generator: B1 bundle → ConstraintTemplate adapter (R-B2-1/3) | B1 decision contract | W3 |
| W2 | Enforcement-mode FSM + lifecycle gate hooks (R-B2-6/7) | W1, A2 stub | W3, W4 |
| W3 | 17-field audit emitter + correlation_id + null-with-reason (R-B2-10/11) | D1 mapping stub, C2 sink stub | W1, W2 |
| W4 | Suspend-pending-approval: deny+approval block→CRD→retry+external-data state read (R-B2-17/20) | B4 CRDs, external-data | W5 |
| W5 | Audit/detective controller + admission↔audit reconciliation (R-B2-8/9) | W1, C4 stub | W4 |
| W6 | Webhook config: failurePolicy/timeout/scoping/system-ns carve-out (R-B2-13/16) | — | all |

W1 and W3 are the unblockers; both can start once B1 freezes the decision object + correlation_id.

---

## 3. Critical path & milestones

- **M1 — Bundle→Template round-trip (W1):** `SC-IMG-001` bundle generates a working
  ConstraintTemplate that delegates to `decision`; conformance with B1 corpus (B1-R30) green.
- **M2 — 17 fields complete (W3):** every admission + audit event carries all 17 §9.3 fields,
  correlation_id matches the OPA decision log. DT-16 negative test passes (missing field flagged).
- **M3 — Mode FSM (W2):** dryrun→warn→deny promotion enforced; direct-to-deny blocked; transitions audited.
- **M4 — Approval at admission (W4):** deny-with-approval-required + PolicyApprovalRequest created
  + retry-after-approval admits. **Highest-risk milestone**; demo HL-10 break-glass + DT-58/59.
- **M5 — Detective controller + reconciliation (W5):** audit-mode scan finds resources admission
  never saw; admission↔audit divergence surfaced (DT-15/17); feeds §19 (DT-30).
- **M6 — Webhook hardened (W6):** failurePolicy matrix per env, system-ns carve-out, external-data
  timeouts + version capture, latency budget met.

---

## 4. Test strategy

1. **Template conformance:** generated templates produce decisions identical to B1 REST/Wasm for
   the golden corpus (extends B1-R30 to the Gatekeeper engine — this is the conformance gate).
2. **Audit-field completeness:** assert all 17 fields on a matrix of (deny, warn, dryrun, audit);
   negative test: drop JWT → field present as null+reason, event flagged (DT-16).
3. **Mode FSM:** attempt direct deny authoring → blocked; promotion dryrun→warn→deny → audited;
   rollback deny→warn (DT-14) and constraint rollback (DT-06).
4. **Approval path (critical):** require_approval decision → admission denied with reason, CRD
   created, no held webhook (assert webhook returns within deadline); retry pre-approval → denied;
   approve → retry admits; exception path (DT-03) → allow-with-exception; expiry → re-denied (HL-19).
5. **Detective/reconciliation:** create resource in dryrun window → audit controller flags it;
   admission-allowed vs audit-flagged divergence raised (DT-17); disable Gatekeeper, create
   violating resource → no events → C4 bypass alert (DT-30, HL-06).
6. **Failure injection:** Gatekeeper down with failurePolicy=Fail (prod ns) → denied; with system
   ns Ignore → admitted (no brick); external-data timeout → falls to failurePolicy, version captured.
7. **Cluster-brick guard:** kill bundle/external-data during a simulated cold start → assert
   system namespaces still admit (R-B2-4/14). This is the regression test for B1 AR-6.
8. **Latency:** p99 webhook decision within admission budget; slow-policy detector.
9. **Multi-cluster/coexistence:** run alongside a pre-existing managed Gatekeeper; namespace-scope
   prevents constraint collision (D-B2-04, HL-09 drift).

---

## 5. What blocks what

- **Blocked by B1:** decision contract, correlation_id, signed bundles (template Rego source).
- **Blocked by B4:** approval/exception CRDs + controllers (W4), and the engine-selection rubric
  that says when Gatekeeper vs Kyverno (B2 must not absorb mutate/generate work).
- **Blocked by D1:** JWT subject/groups mapping for fields 7/8.
- **Blocks B5:** B2 supplies the Gatekeeper leg of the realtime sequence.
- **Blocks C4:** detective controller + the *absence* semantics for §19 bypass detection.

---

## 6. Risks & mitigations

| Risk | Mitigation |
|---|---|
| Approval-at-admission complexity (W4) | Spike first; treat as M4 keystone; external-data-backed state read avoids held webhook |
| Cluster-bricking via failurePolicy=Fail | System-ns carve-out + brick guard test (#7) mandatory before any prod Fail |
| Template Rego drift from bundle | Generate-only (no hand edits); drift detector in lifecycle (R-B2-3) |
| Coexistence with cloud-managed Gatekeeper | Namespace-scope; never assume exclusive ownership |
| External-data latency on hot path | Cache + bounded timeout + version capture; budget-tested |
| Audit-event volume in audit mode | Configurable interval; tighter only where it matters (prod) |

---

## 7. Estimated sequencing (relative)

Week 1: W1 + W6 skeleton. Week 2: W3 audit fields + W2 FSM. Week 3: W4 approval spike (M4).
Week 4: W5 detective + reconciliation. Week 5: hardening, brick-guard, latency, coexistence.
