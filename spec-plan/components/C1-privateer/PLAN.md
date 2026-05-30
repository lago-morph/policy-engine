# C1 — Privateer Integration — PLAN

**Status:** blocks on C2 (M-FREEZE) and A1 (Gemara control model).

## 1. Dependency DAG
```
C2 M-FREEZE ─┐
A1 controls ─┼─▶ W1 Gemara Evaluation model + log (chained via C2 primitive)
C2 query API ┘        │
                      ▼
        ┌─────────────┴───────────────┐
        ▼                             ▼
 W2 Correlation engine          W3 Supply-chain adapters
 (control↔C2 events, groups)    (SBOM in-toto, signature verify)
        │                             │
        └──────────────┬──────────────┘
                       ▼
              W4 Verdict rollup + coverage + completeness rollup
                       │
              ┌────────┴─────────┐
              ▼                  ▼
       W5 Evaluation Log    W6 Export (content assembly;
       query API            calls C2 integrity primitive §7.6)
              │                  │
              └──────────────────┘
                       ▼
        consumed by C5 reports, auditors (DT-22/DT-24), HL-20
```

## 2. Workstreams (parallelism)
| WS | Deliverable | Depends on | Parallel with |
|---|---|---|---|
| W1 | GemaraEvaluation model + append-only log | C2 freeze, A1 | — |
| W2 | Correlation engine (control↔C2 join, groups) | W1, C2 query API | W3 |
| W3 | Supply-chain adapters (SBOM, signature) | W1 | W2 |
| W4 | Verdict rollup + coverage + completeness rollup | W2, W3 | — |
| W5 | Evaluation Log query API | W4 | W6 |
| W6 | Export content assembly (calls C2 primitive) | W4, C2 §7.6 | W5 |

**Parallel after W1:** W2 (correlation) ∥ W3 (supply chain). W5 ∥ W6 after W4.

## 3. Critical path
`C2 M-FREEZE + A1 controls → W1 → W2 → W4 → W6 → first signed Gemara evidence export (DT-24)`. The hard gate is C2 freeze; W3 can proceed in parallel against a stubbed evaluation model.

## 4. Milestones
- **M1:** W1 model + one control (SC-IMG-001) evaluated from live C2 events → one GemaraEvaluation row (DT-22 single-control read).
- **M2:** W2+W3 — full §11.2 source correlation incl. SBOM/signature (DT-23).
- **M3:** W4 verdict model with `indeterminate` honesty (N-C1-6); coverage + completeness rollups.
- **M4:** W5+W6 — Gemara Evaluation Log query + signed export via C2 primitive; DT-22 → DT-24 handoff end-to-end with auditor-independent verification (HL-18).

## 5. Test strategy
- **T-C1-1 Verdict honesty:** evaluation over `insufficient` C2 events yields `indeterminate`, never `satisfied` (N-C1-6).
- **T-C1-2 Correlation faithfulness:** every §11.2 source kind present for a control in a window is attached to the evaluation; a missing required kind drives `not_satisfied`/`partially_satisfied`.
- **T-C1-3 No detection duplication:** C1 emits no §14.2 bypass finding itself; it ingests C3/C4 findings (N-C1-7) — assert C1 has no independent bypass detector.
- **T-C1-4 Supply-chain digest match:** signature evidence digest equals the C2 `external_data_refs` digest the engine consumed (D-C1-05).
- **T-C1-5 Export verifiability:** exported package verifies with the published public key alone; uses C2's Merkle/sign format (one platform format, DT-24, HL-18).
- **T-C1-6 Scope authz:** Auditor cannot evaluate/export outside scope (DT-24, §17A.5).
- **T-C1-7 Control-version split:** a mid-period control change produces two version-pinned evaluations, not a blended one.

## 6. What blocks what
- W1 blocked on C2 freeze + A1 control model. Mitigation: stub A1 with a hand-authored required-evidence map for SC-IMG-001 / SC-SBOM-001.
- W6 blocked on C2 §7.6 export primitive. Mitigation: build content assembly against a primitive stub; swap in C2's when ready.
- W3 (supply chain) has no C2/A1 blocker beyond the model — front-loadable.

## 7. Risks
- **R1:** Re-implementing detections C3/C4 own → divergent bypass results. Mitigation: N-C1-7 + T-C1-3.
- **R2:** Verdict optimism (rendering `satisfied` over thin evidence). Mitigation: T-C1-1 + the `indeterminate` default.
- **R3:** Export format drift from C5/C2. Mitigation: single C2 integrity primitive (D-C1-03).
