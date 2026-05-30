# C5 — Reporting — PLAN

**Status:** blocks on C2 (M-FREEZE + integrity primitive §7.6) and on C1/C3/C4 outputs per report type.

## 1. Dependency DAG
```
C2 M-FREEZE + query API + coverage feed + integrity primitive
        │
        ├─▶ W1 Report framework (query, scope-filter, schedule, deliver, export)
        │        │
        │        ├─▶ W2 R1 Real-Time Enforcement (C2 events)          ← C2 only
        │        ├─▶ W3 R4 Coverage-Gap & Drift (C3 findings + C2 feed)← C3
        │        ├─▶ W4 R2 Audit-Derived Violation (C4 population)     ← C4
        │        └─▶ W5 R3 Simulation (E1 + C4 differential)          ← E1/C4
        │
        ├─▶ W6 Export: signed package (calls C2 primitive) + PDF + CSV/NDJSON + OCSF profile
        └─▶ W7 Scheduling + delivery + federated rollup (HL-20)
```
Report types unblock in **dependency order of their sources**: R1 (C2 only) first, then R4 (C3), then R2 (C4), then R3 (E1/C4).

## 2. Workstreams
| WS | Deliverable | Source dep | Parallel with |
|---|---|---|---|
| W1 | Report framework (query/scope/schedule/deliver/export shell) | C2 contract | — |
| W2 | R1 Real-Time Enforcement | C2 events | W3 (after W1) |
| W3 | R4 Coverage-Gap & Drift | C3 findings + C2 feed | W2 |
| W4 | R2 Audit-Derived Violation | C4 population | W5 |
| W5 | R3 Simulation | E1 + C4 differential | W4 |
| W6 | Export (signed via C2 primitive, PDF, CSV/NDJSON, OCSF) | C2 §7.6/§9 | W7 |
| W7 | Scheduling, delivery, federated rollup | W1 | W6 |

## 3. Critical path
`C2 M-FREEZE → W1 → W2 (R1, the simplest, C2-only) → W6 export`. R1 ships first and proves the query→render→scoped→signed-export pipeline end-to-end before the harder report types (which wait on C3/C4/E1).

## 4. Milestones
- **M1:** W1 framework + R1 (DT-77) with scope filtering and counts-by-engine; signed CSV/NDJSON export.
- **M2:** W6 — signed evidence package via C2 primitive verifiable with the public key alone (DT-24, HL-18); PDF + OCSF profile.
- **M3:** W3 R4 coverage-gap matrix (DT-80) + drift (HL-09) + missing-audit-fields (DT-25).
- **M4:** W4 R2 audit-derived violation (DT-78) with recorded-vs-inferred distinction + low-confidence backlog + re-execute.
- **M5:** W5 R3 simulation (DT-79, HL-12) + W7 scheduling (DT-34) + federated rollup (HL-20).

## 5. Test strategy
- **T-C5-1 Field completeness:** R1 rows carry all §17E.2 fields (mutation_diff iff mutate; approval correlation iff suspend/require_approval); R2 rows carry all 9 §17E.3 fields; R3 carries all §17E.4 fields (DT-77/78/79).
- **T-C5-2 Scope enforcement:** a Compliance Analyst scoped to `tenant=payments` sees only payments rows; an Auditor sees only authorized controls/period; over-scope query → truncated + notice, never silent-complete (DT-77/78, §17A.5).
- **T-C5-3 Signed export verifiability:** export verifies with the published public key alone; uses C2's Merkle/sign format; tamper → verification fails (DT-24, HL-18).
- **T-C5-4 Honesty:** `insufficient` events excluded from R3 authoritative counts + disclosed (DT-46); low-confidence R2 rows shown but not auto-counted (DT-78); recorded-vs-inferred visibly distinct (C4 D2).
- **T-C5-5 "No findings" ≠ "couldn't query":** source-down renders "unavailable", not zero (D-C5-05).
- **T-C5-6 Coverage matrix:** (namespace × control) cells classified correctly incl. selector-aware and stale-inventory (DT-80, DT-33).
- **T-C5-7 Federated dedup:** rollup dedups on cluster-scoped correlation_id (HL-20, C2 D5).
- **T-C5-8 Scheduling:** cron report runs, materializes a reproducible dataset, delivers, and a missed run is logged+alerted (DT-34).
- **T-C5-9 Round-trip:** every rendered datum dereferences to a C2 event_id or a cited finding id (no orphan numbers).

## 6. What blocks what
- W1+W2 need only C2 (front-loadable at M-FREEZE; R1 is the unblocked first report).
- W3 needs C3 findings (stub finding fixtures to build the matrix UI early).
- W4 needs C4 population (stub violation fixtures).
- W5 needs E1 simulation + C4 differential (stub).
- W6 needs C2 §7.6 integrity primitive + §9 OCSF mapping (stub the primitive; real verifiability at C2 M5).

## 7. Risks
- **R1:** A blank/green report that falsely implies compliance (source was down, or scope hid everything). Mitigation: D-C5-05 + T-C5-5; "unavailable" and "truncated by scope" are first-class render states.
- **R2:** Reports re-implement detection/reconstruction instead of consuming C3/C4. Mitigation: D-C5-02; C5 is presentation only.
- **R3:** Three signing formats across C1/C2/C5. Mitigation: D-C5-01 single C2 primitive.
- **R4:** Inferred violations misread as recorded denies in a PDF. Mitigation: N-C5-1 visible distinction + T-C5-4.
