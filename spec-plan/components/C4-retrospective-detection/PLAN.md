# C4 — Retrospective Audit Detection — PLAN

**Status:** blocks on C2 (M-FREEZE + raw retention), C3 (shared detector library), E1 (replay engine).

## 1. Dependency DAG
```
C2 M-FREEZE + raw retention ─┐
C3 detector library ─────────┼─▶ W1 Three-view reconciliation (decision/audit/inventory)
runtime scanner inventory ───┘        │
                                      ▼
                          W2 Bypass classification (classic/deep)
                                      │
deployment/drift record ──▶ W3 Input reconstruction (synthetic C2 event, confidence scoring)
D1 token-issuance ────────┘           │
                                      ▼  (request replay)
                          E1 ◀──── W4 Replay driver + differential (HL-12)
                                      │
                                      ▼
                          W5 Outputs: 4 artifacts + §17E.3 rows + materialized dataset
                                      │
                          consumed by C5, C1, auditors (DT-78/HL-18)
```

## 2. Workstreams
| WS | Deliverable | Depends on | Parallel with |
|---|---|---|---|
| W1 | Three-view reconciliation engine | C2, C3 library, scanner inventory | — |
| W2 | Bypass classification (classic vs deep) | W1 | W3 (partly) |
| W3 | Input reconstruction + confidence scoring | C2 raw retention, D1 tokens, deployment record | W2 |
| W4 | Replay driver (calls E1) + retrospective differential | W3, E1 | — |
| W5 | Output artifacts + §17E.3 rows + dataset materialization | W4, C2 §8.5 | — |

## 3. Critical path
`C2 freeze + C3 library + E1 replay → W1 → W2 → W3 → W4 → W5 → first audit-derived violation population (DT-78)`. C4 is **the most-blocked component in the domain** (depends on C2, C3, *and* E1). It is on the domain critical path's tail; front-load W1/W2/W3 design against contracts and stubs so only the final replay wiring waits on E1.

## 4. Milestones
- **M1:** W1 three-view reconciliation over a seeded window; classic bypass found (DT-30 historical).
- **M2:** W2 deep-bypass classification (inventory-only object with no audit event — the C3-A1 hole closed).
- **M3:** W3 reconstruction producing `best_effort` synthetic events with correct confidence bands (DT-78 a/b/c).
- **M4:** W4 replay via E1 → verdicts; retrospective differential for HL-12 silent regression.
- **M5:** W5 — full §17E.3 population + materialized dataset; DT-46/DT-78 reuse + HL-18 auditor independent re-execution (≥95% tie-out).

## 5. Test strategy
- **T-C4-1 Three-view reconciliation:** seed (audit∧inventory∧¬decision) → classic bypass; (inventory∧¬audit∧¬decision) → deep bypass; (audit∧decision∧¬inventory) → no finding (HL-06, C3-A1 hole).
- **T-C4-2 Reconstruction cap:** every reconstructed event is `best_effort`, never `complete` (N-C4-1/N-C2-SYNTH).
- **T-C4-3 Confidence bands:** full audit object + known bundle + re-resolvable external data → `high`; defaulted field → `medium` + `missing_fields`; undisclosed verdict → `low` (DT-78 a/b/c).
- **T-C4-4 Low-confidence not auto-counted:** `confidence=low` rows excluded from totals until confirmed (N-C4-2, DT-78 SC).
- **T-C4-5 §17E.3 completeness:** every row has all 9 fields; `source_audit_log` round-trips to the raw row (DT-78 SC).
- **T-C4-6 Auditor independence:** independent re-execution of reconstructed inputs via E1 ties out to stored verdicts ≥95%; divergence flagged (DT-78 SC, HL-18, §17.4).
- **T-C4-7 Retrospective differential:** over a v_old→v_new window, the regression (`deny`→`allow`) class is found (HL-12).
- **T-C4-8 Dataset reuse:** sweep dataset is digest-addressable and reused by a second consumer without re-running (DT-46).
- **T-C4-9 Determinism:** the same window + same C2 evidence + same bundle reproduces the same violation population (replay determinism, sealed-window — inherits C3-A5 watermark fix).

## 6. What blocks what
- W1 blocked on C3's detector library (shared) + a runtime scanner inventory feed (stub the inventory).
- W3 blocked on C2 raw retention (needs raw audit objects + token-issuance events). Mitigation: seed raw events.
- W4 blocked on E1 replay engine. Mitigation: build the replay-driver interface against an E1 stub returning canned verdicts; swap in E1 at integration.
- W5 export/dataset blocked on C2 §8.5 + §7.6. Mitigation: dataset stub.

## 7. Risks
- **R1 (highest): deep-bypass requires an independent inventory source that may not exist in the POC.** Mitigation: degrade gracefully — without runtime inventory, C4 still catches classic bypass (audit∧¬decision); document the residual blind spot explicitly (don't claim coverage C4 can't deliver).
- **R2:** Reconstruction fabricates a confident-but-wrong input → a false violation in an audit report. Mitigation: confidence bands + N-C4-2 (low not auto-counted) + auditor re-execution tie-out.
- **R3:** Bundle-at-time inference wrong → replay against the wrong policy. Mitigation: deployment/drift record is authoritative; uninferable → `indeterminate`, not a guess.
- **R4:** Divergence from C3 (different windows find different bypasses). Mitigation: shared detector library (D-C4-01).
- **R5:** E1 dependency slips → C4 cannot replay. Mitigation: stubbed replay driver keeps W1–W3 unblocked; reconstruction value exists even before replay (the violation *candidates* are useful).
