# G3 — Availability, DR & Resilience — IMPLEMENTATION PLAN

**Component ID:** G3 · **Date:** 2026-05-30 · Pairs with `SPEC.md`.

This plan turns the SPEC into a buildable, maximally-parallel program of work. It identifies the dependency DAG, the four workstreams that can proceed concurrently, the critical path, milestones, and the test/drill strategy. The two crux deliverables — **chain-continuous backup/restore** and **the circuit-breaker / infrastructure-degraded mode** — are on the critical path and are sequenced first because everything else (FMEA validation, multi-region, drills) depends on them existing.

---

## 1. Dependency DAG (what blocks what)

```
                         ┌─────────────────────────────────────────────┐
   UPSTREAM (must exist   │  C2 §7-§8 chain/checkpoint/CAS/retention      │  (consume, don't rebuild)
   or be stubbed first):  │  F2 §2-§6 hub/spoke topology + failurePolicy │
                          │  B5 §5 flow failure modes                    │
                          │  G4 key-validity-history API (for restore-verify)
                          └───────────────┬─────────────────────────────┘
                                          │
        ┌─────────────────────────────────┼───────────────────────────────────┐
        ▼                                 ▼                                     ▼
 ┌───────────────┐               ┌──────────────────┐                  ┌──────────────────┐
 │ WS-A           │               │ WS-B              │                  │ WS-C              │
 │ Data-class +   │  (foundation) │ Circuit breaker / │  (parallel,       │ FMEA + DR runbook │  (parallel,
 │ quorum-commit +│──────┐        │ infra-degraded    │   needs only      │ + observability   │   needs SPEC only)
 │ buffer durab.  │      │        │ mode (X-4 resolve)│   F2/B5 contracts)│   metrics (G6 seam)│
 │ (D0 RPO=0)     │      │        └─────────┬─────────┘                  └─────────┬─────────┘
 └───────┬────────┘      │                  │                                      │
         │               │                  │                                      │
         ▼               ▼                  ▼                                      ▼
 ┌───────────────────────────────┐  ┌───────────────────────┐          ┌────────────────────────┐
 │ WS-D: Backup/Restore chain-    │  │ §19 catch-up hand-off  │          │ Drill catalog wired to │
 │ continuity (restore_boundary, │  │ to C4 (degraded gaps)  │          │ each FMEA row (§8)     │
 │ WORM, Restore Attestation)    │  └───────────────────────┘          └────────────────────────┘
 │  ── THE CRUX, on crit path ── │
 └───────────────┬───────────────┘
                 ▼
 ┌───────────────────────────────┐
 │ WS-E: Multi-region (split-brain│  (needs WS-A quorum + WS-D restore + WS-B breaker)
 │ fence, failover, fork-reconcile│
 │ reconciliation)               │
 └───────────────┬───────────────┘
                 ▼
 ┌───────────────────────────────┐
 │ WS-F: Chaos/DR drills + GA gate│  (needs A,B,D,E built; C runs alongside)
 └───────────────────────────────┘
```

**Edges that matter:**
- WS-D (restore) **requires** WS-A (you can't define "restore to last quorum-committed checkpoint" without the quorum-commit + chain-advance discipline) and the C2 checkpoint primitive + G4 key-validity API.
- WS-E (multi-region) **requires** WS-A (single-writer/quorum linearization is the split-brain defense) + WS-D (failover *is* a restore-to-checkpoint + reconcile) + WS-B (a region/hub loss must engage degraded mode at spokes, not mass-deny).
- WS-B (breaker) is **independent of** WS-A/WS-D — it only needs the F2 `failurePolicy`/CRD-status surface and the B5 flow contracts. **This is the key parallelism unlock: the X-4 resolution can be built concurrently with the durability work.**
- WS-C (FMEA/runbook/metrics) is **paper+observability** and can start immediately from the SPEC, feeding WS-F's drill criteria.

---

## 2. Workstreams (parallelizable)

### WS-A — Evidence durability core (D0 RPO=0)  ·  *foundation, critical path*
- A1. Quorum-commit gate on the C2 chain: `chain_seq` issued only after write-majority across ≥2 failure domains (R-G3-RPO-4); CAS-blob-durable-before-event ordering (R-G3-RPO-5).
- A2. Durable edge buffer (D0-pending): disk-backed, bounded, spill-to-spool, back-pressure, idempotent flush on `raw_event_digest` (R-G3-BUF-1/2/3). Resolves B5-F4 durably.
- A3. Data-class labeling (`governance.example.io/data-class`) enforced at admission for platform PVs (R-G3-DC-1/2).
- A4. RPO metrics: `g3_d0_buffer_oldest_age_seconds` (R-G3-BUF-3).
- **Owner discipline:** tight, correctness-critical, **single-author core** (mirrors the META caution against parallelizing integrity-critical cores). Do not fan this out.

### WS-B — Circuit breaker / infrastructure-degraded mode (X-4 resolution)  ·  *parallel, high-value*
- B1. Degraded-mode controller (HA, leader-elected): probes for bundle-server, verifier, Keycloak, OPA-control-plane, C2-sink (R-G3-CB-1).
- B2. Breaker state machine CLOSED→OPEN→HALF-OPEN, scoped (dependency × control × criticality) (R-G3-CB-2/3/4); breaker-state CRD (`GovernancePlatform.status.degraded[]`) the spokes watch (OQ-G3-5: hub-aggregate / spoke-apply).
- B3. Spoke admission integration: apply degraded disposition from the §4.1 matrix instead of blanket fail-closed; criticality class on scope CRDs.
- B4. `infrastructure_degraded` C2 disposition + `degraded_session_id`, degraded-transition audit events (R-G3-AV-3, R-G3-CB-5). **Additive C2 contribution.**
- B5. §19 catch-up hand-off to C4, delimited by `degraded_session_id` (R-G3-CB-6/7).
- B6. Anti-self-brick: C-SYSTEM carve-out + bootstrap exemption + breaker-never-blocks-own-deps (R-G3-CB-9).
- B7. Manual override / break-glass with dual-control on C-CRITICAL force-close (R-G3-CB-8).
- B8. Metric `g3_breaker_state{dependency,control}`.

### WS-C — FMEA, DR runbook, DR observability  ·  *parallel, starts immediately*
- C1. FMEA table maintained as living doc; each row maps to a drill (R-G3-FMEA-1).
- C2. DR runbook authoring + dual-control Restore-Attestation ceremony + fence-before-promote ordering (R-G3-TEST-5).
- C3. DR observability metric set + alerts handed to G6 (R-G3-TEST-4): replication-lag, buffer-age, chain-verify, checkpoint-age, breaker-state, last-drill-age.

### WS-D — Chain-continuous backup & restore (THE CRUX)  ·  *critical path*
- D1. WORM/object-lock continuous backup at checkpoint granularity, separate credential boundary (R-G3-BK-1/2/3).
- D2. Restore protocol: load-in-seq, verify every `prev_hash`/`chain_seq`/checkpoint-signature against G4 historical keys, fail-loud on mismatch (R-G3-RS-1).
- D3. `chain.restore_boundary` signed marker + Restore Attestation (dual-control) appended as continuation-of-chain (R-G3-RS-2/3).
- D4. Chain-verify logic update: valid signed restore marker = legitimate discontinuity; bare discontinuity = tamper (R-G3-RS-4). **No re-hash/re-chain/re-sign of history (R-G3-RS-5).**
- D5. Post-restore validation gate: full `/verify` + export-manifest re-verify before serving (R-G3-RS-8).
- D6. G4 co-restore ordering: keys/key-validity restorable before/independent of data (R-G3-RS-7).

### WS-E — Multi-region & split-brain  ·  *depends on A+D+B*
- E1. Single-writer-per-source-chain lease/fence (R-G3-MR-3/4).
- E2. Profiles P1/P2/P3 deploy paths (R-G3-MR-1); spoke regional independence (R-G3-MR-2).
- E3. Failover: fence-before-promote, restore_boundary for lag window, edge-buffer reconciliation, validation gate (R-G3-MR-6, R-G3-RS-6).
- E4. `evidence_lost_in_failover` disclosure path (R-G3-RPO-2).
- E5. Fork-reconciliation procedure (preserve both tails, `fork_reconciliation` marker, never delete) (R-G3-MR-5).
- E6. Metric `g3_d0_replication_lag_seconds`.

### WS-F — Chaos/DR drills + GA gate  ·  *integration, last*
- F1. Wire each FMEA row to a drill (DRILL-1..3, CHAOS-4..11).
- F2. Automate drills in staging; game-day harness with blast-radius bound + abort (R-G3-TEST-1/2).
- F3. GA acceptance gate: DRILL-1, DRILL-2, DRILL-3, CHAOS-4, CHAOS-6, CHAOS-7, CHAOS-9, CHAOS-10 green with measured RPO/RTO (R-G3-TEST-3).

---

## 3. Critical path

```
C2 checkpoint primitive (upstream)
   → WS-A quorum-commit + durable buffer (D0 RPO=0)
      → WS-D restore-to-checkpoint + restore_boundary + validation gate   ← THE CRUX
         → WS-E failover (= restore + reconcile + fence)
            → WS-F DRILL-1/DRILL-2/DRILL-3 GA gate
```

WS-B (circuit breaker) runs **fully in parallel** off the F2/B5 contracts and joins at WS-F (CHAOS-4/6/7). It is *not* on the durability critical path, so the X-4 resolution and the evidence-durability resolution proceed concurrently — the single biggest schedule win. WS-C is paper+metrics from day 0.

**Longest chain:** quorum-commit → restore-continuity → failover → DR drill. The restore-continuity work (WS-D) is the gating intellectual difficulty (it must be *provably* indistinguishable-from-legitimate yet *distinguishable-from-tamper*), so it gets the most senior owner and the earliest start after WS-A's commit discipline lands.

---

## 4. Milestones

| ID | Milestone | Gates on | Exit criterion |
|---|---|---|---|
| **M-G3-1** | **D0 RPO=0 commit discipline** | WS-A1/A2 | A killed node loses zero committed events; CHAOS-10 (buffer) passes in staging. |
| **M-G3-2** | **X-4 resolved (degraded mode live)** | WS-B | CHAOS-4 + CHAOS-5 pass: bundle/verifier outage → warn-don't-mass-deny; degraded admissions audited; §19 catch-up queued. |
| **M-G3-3** | **Restore proven chain-continuous** *(crux)* | WS-D | DRILL-1 + DRILL-3 pass: restore verifies end-to-end; restore boundary ≠ tamper; forged truncation = tamper. |
| **M-G3-4** | **Anti-self-brick proven** | WS-B6 | CHAOS-7 passes: shared-dep fix deploys *during* the outage. |
| **M-G3-5** | **Multi-region failover** | WS-E | DRILL-2 + CHAOS-9 pass: failover w/ fence-before-promote; split-brain prevented/reconciled without deleting events. |
| **M-G3-6** | **GA DR gate** | WS-F3 | All eight GA-gate drills green with measured RPO/RTO within target; DR runbook drill-validated; metrics live in G6. |

---

## 5. What can be built concurrently / what blocks what (summary)

- **Concurrent from day 0:** WS-A (durability core), WS-B (circuit breaker), WS-C (FMEA/runbook/metrics). These touch different surfaces (storage-commit vs. admission-degradation vs. docs/observability) and share no code path.
- **Blocked until WS-A:** WS-D (restore needs the commit/chain-advance discipline + checkpoint primitive).
- **Blocked until WS-A+WS-D+WS-B:** WS-E (failover is restore+reconcile+fence and must engage degraded mode).
- **Blocked until A,B,D,E:** WS-F GA drills (you can only drill what exists), though WS-C's drill *definitions* and WS-F's staging harness can be scaffolded early.
- **External unblockers needed early:** G4's historical-key-validity API (for WS-D restore-verify) and C2's checkpoint cadence config. Stub both behind interfaces so WS-A/WS-D don't block on G4/C2 delivery.

---

## 6. Test strategy

- **Unit/contract:** restore protocol against synthetic chains incl. adversarial inputs (truncated-without-marker, forged-marker, key-rotated-across-checkpoint, fork). Breaker state machine incl. flap/anti-flap and the per-request-timeout-vs-aggregate-breaker separation (R-G3-CB-4).
- **Integration:** the eight GA-gate drills (§8.3) as automated staging jobs; the rest of the catalog monthly.
- **Property tests:** "no committed event is ever lost across {node, zone, region, restore}"; "every chain discontinuity is either an attested restore/fork marker or a tamper finding — never ambiguous"; "no event is ever deleted to resolve a fork."
- **Adversarial (drives the drills):** CHAOS-6 (breaker-as-bypass), CHAOS-11 (stale-exception-on-restore), DRILL-3 (restore-as-truncation) — these come directly from `ADVERSARIAL-REVIEW.md` and are mandatory.
- **Acceptance:** R-G3-TEST-3 GA gate; every Sev-1/Sev-2 FMEA row has a passing drill (R-G3-FMEA-1).
