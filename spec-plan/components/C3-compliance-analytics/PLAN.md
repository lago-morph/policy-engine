# C3 — Compliance Analytics Engine — PLAN

**Status:** blocks on C2 (M-FREEZE) for the contract; on C2 M4 (live query API + derived views) for live data.

## 1. Dependency DAG
```
C2 M-FREEZE (contract) ─▶ all detector design + contract tests
C2 M4 (query API + correlation-members §10.3 + coverage feed §10.4 + verify §10.6)
        │
        ├─▶ W1 Detector framework (reconciliation runner, finding model, scope filter, determinism harness)
        │        │
        │        ├─▶ W2 D-BYPASS + D-CORRELATION (correlation-members based)
        │        ├─▶ W3 D-INCONSISTENT + D-DRIFT + D-RUNTIME-VS-BUILD (version/verdict based)
        │        ├─▶ W4 D-JWT-DRIFT (claim-presence based)        ← needs D1 required-claim lists
        │        ├─▶ W5 D-COVERAGE (matrix; selector eval)        ← needs A1 in-scope + inventory
        │        └─▶ W6 D-CHAIN + D-LATENCY + D-UNGOVERNED (integrity/support)
        │
        └─▶ W7 Finding store + state machine (open/ack/resolved/suppressed) + handoff to C4/C5/C1
```

## 2. Workstreams (parallel)
All detectors (W2–W6) are independent once the framework (W1) and finding model exist — they consume different C2 views. Build the **framework first**, then fan out the six detector groups in parallel.

| WS | Detectors | Extra dep | Parallel with |
|---|---|---|---|
| W1 | framework + finding model + determinism harness | C2 contract | — |
| W2 | D-BYPASS, D-CORRELATION | webhook-health signal | W3–W6 |
| W3 | D-INCONSISTENT, D-DRIFT, D-RUNTIME-VS-BUILD | A2/B1 SoT version | W2,W4,W5,W6 |
| W4 | D-JWT-DRIFT | D1 required-claim lists | others |
| W5 | D-COVERAGE | A1 in-scope + inventory | others |
| W6 | D-CHAIN, D-LATENCY, D-UNGOVERNED | C2 verify endpoint | others |
| W7 | finding store + state + handoffs | W1 | after W1 |

## 3. Critical path
`C2 M-FREEZE → W1 framework → W2 D-BYPASS (the headline §14.2/§19 detection) → handoff to C4/C5`. D-BYPASS is the first detector to land because it drives the platform's signature retrospective scenario (HL-06/DT-30). Everything else parallelizes off W1.

## 4. Milestones
- **M1:** W1 framework + finding model + determinism harness; one trivial detector (D-UNGOVERNED) end-to-end.
- **M2:** W2 — D-BYPASS emits within ≤15 min (DT-30 SC); D-CORRELATION fires at >1% unpaired (DT-28).
- **M3:** W3+W4+W5 — drift, inconsistency, JWT-drift, coverage matrix (DT-31/32/33/80, HL-09).
- **M4:** W6+W7 — chain-integrity detector live (makes C2 tamper-evidence actionable); finding state machine + handoffs to C4/C5/C1; federated rollup dedup (HL-20).

## 5. Test strategy
- **T-C3-DET (determinism):** each detector version over the same window + same C2 evidence reproduces the identical finding set (N-C3-4). Golden windows per detector. This is the analytics analog of C2's replay-determinism.
- **T-C3-BYPASS:** seed a K8s-audit create with no GK/OPA members → `enforcement_bypass` within one interval; webhook-unhealthy in window → `infrastructure_induced`; webhook deliberately removed → `suspected_malicious` (DT-30, DT-42, HL-06).
- **T-C3-CORR:** revert OPA UID-echo config → D-CORRELATION fires >1%; restore → clears within one window (DT-28).
- **T-C3-DRIFT:** one cluster stuck on v11 while SoT=v12 → `policy_drift`; same input deny-on-a/allow-on-b → `inconsistent_enforcement`, linked (DT-32, HL-09).
- **T-C3-JWT:** drop `tenant` from a fraction of JWTs above threshold → `jwt_policy_drift` with correct omit_rate + allow/deny split (DT-31).
- **T-C3-COV:** constraint present but `excludedNamespaces` covers the ns → classified effectively `not_installed`, not `enforced` (DT-33 selector subtlety); full matrix classification (DT-80); stale inventory → `stale_inventory` subtype not silent `n/a`.
- **T-C3-CHAIN:** tamper a stored event → `chain_integrity` critical finding (C2 adversarial R4).
- **T-C3-CONF:** detection over `insufficient` C2 events carries reduced confidence + disclosure (N-C3-3).
- **T-C3-SCOPE/FED:** federated rollup dedups on cluster-scoped correlation_id, not bare UID (HL-20, C2 adversarial D5).

## 6. What blocks what
- W1 needs only the C2 *contract* (front-loadable at M-FREEZE with stub data).
- W2 needs the correlation-members view + a webhook-health feed (stub the latter).
- W4 needs D1 required-claim lists (stub with SC-IMG/tenant-isolation maps).
- W5 needs A1 in-scope map + K8s inventory (stub both).
- W3 needs A2/B1 source-of-truth versions (fall back to modal observed version).
- W7 handoffs need C4/C5/C1 ingestion contracts — define the finding schema (W1) early so they can integrate.

## 7. Risks
- **R1:** Threshold false positives erode trust (analyst tunes out real findings). Mitigation: per-scope thresholds; findings carry the firing threshold; T-C3 golden windows include benign baselines.
- **R2:** Detection over low-fidelity evidence asserts confident-but-wrong findings. Mitigation: N-C3-3 confidence honesty + T-C3-CONF.
- **R3:** Coverage selector subtlety (DT-33) missed → silent uncovered namespaces. Mitigation: D-C3-02 selector evaluation + T-C3-COV.
- **R4:** Overlap with C1/C4 re-implementing detection. Mitigation: D-C3-04 single-ownership; C4/C1 ingest C3 findings, don't re-derive.
