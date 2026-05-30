# G1 — Scale, Performance & Capacity — PLAN

**Component ID:** G1 · **Status:** DRAFT v1 · **Date:** 2026-05-30
Implementation plan for the performance/capacity layer. Maximizes parallelism: the harness, the per-component load tests, and the capacity model are largely independent workstreams that converge on a single dashboard.

---

## 1. Dependency DAG

```
                         ┌─────────────────────────────────────────────┐
                         │ W0: SLI catalog + metric contract (§8.1)    │  ← blocks everything (defines what "measured" means)
                         └───────────────┬─────────────────────────────┘
                                         │
        ┌────────────────────────────────┼────────────────────────────────────────┐
        ▼                                ▼                                          ▼
┌───────────────────┐      ┌──────────────────────────┐            ┌──────────────────────────────┐
│ W1: Bench harness │      │ W2: Synthetic event-gen  │            │ W3: Capacity model (formulas │
│ (k6/Locust admit) │      │ (ingest/replay corpus)   │            │  + worked examples, §9) —    │
│ — depends W0      │      │ — depends W0 + C2 schema │            │  PAPER, depends only on §22  │
└─────────┬─────────┘      └────────────┬─────────────┘            └──────────────┬───────────────┘
          │                             │                                          │
          ▼                             ▼                                          │
┌───────────────────┐      ┌──────────────────────────┐                           │
│ W4: LT-1/LT-2     │      │ W5: LT-3/LT-4 ingest +    │                           │
│ admission + ext-  │      │ chain-integrity (validate │                           │
│ data brownout     │      │ THE bottleneck N-G1-112)  │                           │
│ (needs B2/B1)     │      │ (needs C2 ingest)         │                           │
└─────────┬─────────┘      └────────────┬─────────────┘                           │
          │                             │                                          │
          ▼                             ▼                                          ▼
┌───────────────────┐      ┌──────────────────────────┐            ┌──────────────────────────────┐
│ W6: LT-5 replay   │      │ W7: LT-6 reporting +      │            │ W8: Capacity model           │
│ (needs E1)        │      │ LT-7 storage growth       │            │ CALIBRATION — feed measured  │
│                   │      │ (needs C5/C2)             │            │ constants back into §9 model │
└─────────┬─────────┘      └────────────┬─────────────┘            └──────────────┬───────────────┘
          └──────────────┬──────────────┴───────────────────────────────────────┘
                         ▼
              ┌────────────────────────────────────────┐
              │ W9: Perf dashboard + nightly CI job     │  ← converges all SLIs (G6 dep)
              │ + GO/NO-GO budget report                │
              └────────────────────────────────────────┘
```

---

## 2. Workstreams (what can be built concurrently)

| WS | Deliverable | Depends on | Parallelizable with | Owner skill |
|---|---|---|---|---|
| **W0** | SLI catalog + Prometheus metric names (§8.1), shared with G6 | §22, C2/B2 SPECs | — (gate) | perf/obs |
| **W1** | Admission load harness (k6/Locust) | W0 | W2, W3 | perf eng |
| **W2** | Synthetic C2-event generator + replay corpus (parametric by scale-multiplier) | W0, C2 frozen schema | W1, W3 | perf eng |
| **W3** | Capacity model spreadsheet/notebook (formulas §9, worked examples) — **paper, no running system needed** | §22 only | everything | architect |
| **W4** | LT-1 admission soak + LT-2 ext-data brownout | W1 + B1/B2 deployable | W5, W6, W7 | perf eng |
| **W5** | LT-3 ingest burst + LT-4 chain-integrity scan — **validates the #1 bottleneck** | W2 + C2 ingest deployable | W4, W6, W7 | perf eng |
| **W6** | LT-5 replay (namespace + reject cluster) | W2 + E1 deployable | W4, W5, W7 | perf eng |
| **W7** | LT-6 reporting + LT-7 storage-growth | W2 + C5/C2 deployable | W4, W5, W6 | perf eng |
| **W8** | Calibrate the §9 model with measured constants (eval_ms, append_eps, KB/event, dedup ratio) | W4–W7 results | — | architect |
| **W9** | Unified perf dashboard + nightly non-gating CI perf job + GO/NO-GO budget report | W0, W4–W8 | — | perf/obs |

**Fully parallel from day 1:** W1, W2, W3 (harness, generator, paper model) — none blocks the others, all block on W0 only.

---

## 3. Critical path

```
W0 (SLI catalog)
  → W2 (event generator)
    → W5 (ingest burst + chain-integrity)   ← the bottleneck-validation, longest-pole experiment
      → W8 (model calibration)
        → W9 (dashboard + GO/NO-GO)
```

**Rationale:** W5 is on the critical path because N-G1-112 (the per-source hash-chain serialization) is the platform's single hard scale ceiling and the META-ADVERSARIAL #2 finding. Everything else either scales horizontally (low risk, parallel) or is paper (W3). The longest experiment is establishing the single-chain append ceiling and proving sharding (N-G1-113) removes it. W4 (admission) is *higher business risk* (a production outage mode) but *lower architectural uncertainty* (well-understood timeout semantics), so it runs in parallel off the critical path.

---

## 4. What to measure first (priority order)

1. **Single-chain append ceiling (`chain_append_eps`, LT-3).** This is the number that decides whether the central evidence spine survives 100×. Measure it before anything else architectural. If batched-commit gets ≥5,000/s/shard, the architecture holds with modest sharding; if per-append fsync caps at hundreds/s, the ALT (async streaming chain) is mandatory, not optional.
2. **Admission p99 at 0% external-data cache (`admission_p99_ms`, LT-1 cold-start + LT-2 brownout).** This is the production-outage mode (N-G1-100). Measure the *cold* and *brownout* cases first — steady-state warm-cache is the easy case and proves nothing.
3. **Realized bytes/event and CAS dedup ratio (LT-7).** Feeds every G2 cost number and the §4 storage model. Cheap to measure, high leverage.
4. **OPA eval p99 for the real governance bundle (`opa_eval_p99_ms`, LT-1).** Validates N-G1-101's 80 ms headroom against the *actual* 25–100-control bundle, not a toy policy.
5. **Replay throughput per worker + the cost-preflight accuracy (LT-5).** Validates N-G1-130/144 and the "no cluster-wide replay at 100×" boundary.

---

## 5. Milestones

| ID | Milestone | Exit criterion |
|---|---|---|
| **M-G1-1** | SLI contract frozen | §8.1 metrics named, agreed with G6, exported skeletons exist |
| **M-G1-2** | Paper capacity model | §9 formulas + POC/10×/100× worked examples reviewed |
| **M-G1-3** | Bottleneck quantified | LT-3/LT-4 give measured `chain_append_eps`; sharding (N-G1-113) demonstrated to scale linearly |
| **M-G1-4** | Hot-path budget proven | LT-1/LT-2 pass: p99 ≤ 1 s, timeout-rate < 0.01%, brownout falls back within budget |
| **M-G1-5** | Replay boundary proven | LT-5: namespace replay ≤ minutes; cost-preflight rejects oversized cluster replay |
| **M-G1-6** | Model calibrated | §9 constants replaced with measured values; predictions within ±25% of LT results |
| **M-G1-7** | Continuous perf gate | nightly CI perf job + dashboard live; GO/NO-GO budget report published |

---

## 6. Test strategy

- **Unit-level micro-benchmarks** (Go bench / criterion-style): JCS canonicalization, SHA-256 content_hash, ed25519 sign, single chain append. Establish per-op costs that the §9 model multiplies.
- **Component load tests** (LT-1..LT-7, §8.2) on a provisioned cluster, parameterized by `SCALE_MULTIPLIER ∈ {1,10,100}`.
- **Chaos/brownout** (LT-2): inject provider latency/errors, kill ingest workers mid-burst, verify graceful degradation and audited evidence-loss.
- **Regression gate:** nightly perf job at POC scale (1×) is non-gating but **alerts on >20% regression** in any §8.1 SLI; 10×/100× runs are on-demand against provisioned infra.
- **Cross-validation:** the same Prometheus SLIs (N-G1-151) are asserted by both the load tests and production monitoring — no separate measurement path.

---

## 7. Concurrency summary (the "what blocks what")

- **Independent from day 1 (block only on W0):** harness (W1), event generator (W2), paper model (W3).
- **Independent of each other once their component is deployable:** the four LT clusters W4 (admission), W5 (ingest/chain), W6 (replay), W7 (reporting/storage) run fully in parallel — they touch different subsystems.
- **Serial tail:** W8 (calibration) needs all LT results; W9 (dashboard/GO-NO-GO) needs W8. These two are the only forced-serial steps.
- **Cross-component handoffs:** G1 hands the capacity model to F2 (replica sizing), G2 (cost), G3 (DR over same volumes), G5 (fairness), G6 (SLI metrics). These can consume G1's paper model (W3/M-G1-2) *before* the load tests finish — so downstream NFR components are not blocked on G1's measurement tail.
