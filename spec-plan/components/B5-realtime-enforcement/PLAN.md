# B5 — Real-Time Enforcement Flow — PLAN

**Component:** B5 · **Pairs with:** SPEC.md · **Date:** 2026-05-30

B5 is an **integration component** — most of its "build" is contracts + end-to-end wiring + the
flow-level tests that prove the pieces (B1/B2/B4/D1/C1/C2/C3) compose correctly.

---

## 1. Dependency DAG

```
[D1 JWT mapping] ─┐
[B1 decision+log]─┼─► W1 correlation_id contract (mint+propagate everywhere) ──► (touches all)
[B2 admission]────┤                         │
[B4 taxonomy/CRD]─┘                         ▼
                              W2 Disposition branch (allow/deny/warn/exception/deny-approval)
                                            │
                                            ▼
                              W3 Latency budget + external-data timeout enforcement
                                            │
                                            ▼
                              W4 Evidence-emission wiring (sync decision / async evidence) ──► [C1,C2,C3]
                                            │
                                            ▼
                              W5 "Expected decision set" export ──► [C4 §19]
                                            │
                                            ▼
                              W6 Generalization to app / identity PDPs
```

Critical path: **W1 → W2 → W3** (the synchronous path must be correct + within budget before evidence/§19 wiring).

---

## 2. Parallelizable workstreams

| WS | Title | Deps | Parallel with |
|---|---|---|---|
| W1 | correlation_id mint+propagate contract (B5-R1) | B1/B2/D1 stubs | W2 |
| W2 | Disposition branch incl. deny-with-approval (B5-R2/R5) | B4 taxonomy, B2 admission | W1 |
| W3 | Latency budget + bounded external-data (B5-R6/R7) | W2 | W4 |
| W4 | Evidence emission: sync/async split, buffering (B5-R3/R4) | W1, C1/C2 stubs | W3, W5 |
| W5 | Expected-decision-set export for §19 (B5-R11) | W2, C4 stub | W4 |
| W6 | App + identity PDP generalization (B5-R8/9/10) | W1, E3 | W4 |

---

## 3. Critical path & milestones

- **M1 — One id end-to-end (W1):** a single admission produces decision-log + audit event + Privateer
  evidence all sharing one correlation_id; trace query returns the full chain (DT-13).
- **M2 — All dispositions (W2):** allow/deny/warn/exception + deny-with-approval all exercised on
  SC-IMG-001; require_approval returns within budget and creates the CRD (the §17B.4 invariant).
- **M3 — Budget held under load (W3):** p99 ≤ 2s with external-data signature verification; verifier
  timeout falls to failurePolicy, never blows the deadline.
- **M4 — Evidence decoupled (W4):** kill the audit store → admission decisions unaffected, evidence
  buffered + backfilled with correlation_id intact (DT-42 correlation gap handled).
- **M5 — §19 expectation export (W5):** active-policy match-scope exported; C4 can compute "should
  there have been a decision" → confident bypass detection (DT-30, HL-06).
- **M6 — App/identity PDPs (W6):** an application OPA call + a Keycloak token decision both emit the
  same correlation_id-keyed, replayable evidence (DT-12, HL-16).

---

## 4. Test strategy (flow-level — the heart of B5)

1. **End-to-end trace (DT-13):** authenticated kubectl apply → assert decision-log, 17-field audit
   event, Privateer evidence, analytics correlation ALL share one correlation_id; full chain queryable.
2. **Disposition matrix:** allow/deny/warn/exception/deny-approval each produce correct admission
   result + correct evidence; require_approval returns within budget + creates CRD (no held webhook).
3. **Latency/budget (M3):** load test t3→t4 p99 ≤ budget; inject slow external-data → timeout to
   failurePolicy, deadline never exceeded (B5-R6); inject slow D1 mapping → same.
4. **Sync/async decoupling (M4):** audit store down → decisions unaffected; evidence buffered;
   backfilled later with correlation_id; missing-correlation flagged (DT-28).
5. **Fail-closed flow (F1/F2):** JWT unresolved → identity-aware control denies (runtime), field
   present-with-reason; decision undefined → fail-closed deny, `decision.error` emitted, never silent-allow.
6. **§19 expectation (M5):** disable Gatekeeper, create violating resource → no decision evidence;
   C4 + the exported expected-set → confident bypass alert (not a false positive from out-of-scope).
7. **Replay equivalence (B5-R2):** capture a real runtime decision → replay via E1 → identical result
   (nd cache makes it deterministic); proves "the decision a user hit can be replayed exactly."
8. **App/identity PDP (M6):** app PDP without input logging → flagged non-replayable (B5-R9/DT-25);
   with input logging → replayable; identity PDP token decision joins evidence stream (HL-16).
9. **Approval retry flow (B5-R7):** denied-approval → approve → retry admits within budget; approval
   state consulted via bounded cache; retry never blocks.

---

## 5. What blocks what

- **Blocked by:** B1 (decision+log+nd), B2 (admission+audit+failurePolicy+deny-approval), B4
  (taxonomy+approval CRD), D1 (JWT mapping). B5 is the *last* B-component to fully integrate.
- **Blocks:** C4 (needs the expected-decision-set export, B5-R11); E1 (needs replay equivalence
  guarantee, B5-R2); C3 (needs correlation contract to correlate).
- **Co-developed with:** B2-W4 (deny-with-approval) and B4-W4a (approval CRD) — the approval flow
  spans all three and should be built as one vertical slice.

---

## 6. Risks & mitigations

| Risk | Mitigation |
|---|---|
| correlation_id dropped/duplicated somewhere in the chain | Server-side mint (OQ1); contract-test every hop; DT-28 negative test |
| Latency budget blown by external-data | Bounded timeout → failurePolicy (B5-R6); budget load test gates release |
| Evidence coupling sneaks onto sync path | Architectural rule: steps 7–10 are async-buffered; assert in M4 |
| §19 false positives from naive absence-detection | Expected-decision-set export (B5-R11) before C4 ships absence-detection |
| Approval flow spans 3 components → integration seams | Build approval as one vertical slice (B2+B4+B5) with shared owner |
| App-PDP replayability gap | Mandatory input logging (B5-R9); flag non-replayable PDPs loudly |

---

## 7. Estimated sequencing (relative)

Week 1: W1 correlation contract + W2 disposition (needs B1/B2/B4 stubs). Week 2: W2 deny-approval
vertical slice (with B2/B4) + W3 budget. Week 3: W4 evidence decoupling + W5 §19 export. Week 4:
W6 app/identity generalization + replay-equivalence + load/hardening.
