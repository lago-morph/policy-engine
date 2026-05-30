# E1 — Policy Simulation & Dry-Run Framework — PLAN

**Component:** E1 · **Spec:** `SPEC.md` (this dir) · **Date:** 2026-05-30

Implementation plan exploiting parallelism: dependency DAG, workstreams, critical path, milestones, test strategy.

---

## 1. Dependency DAG

```
            ┌─────────────────────────────────────────────────────────┐
            │  C2 Audit Schema + replay_completeness (HARD BLOCKER)     │
            └───────────────┬─────────────────────────────────────────┘
                            │ (ReplayEventV1 adapter decouples)
   ┌───────────────┐        ▼
   │ B1..B5 engines│──► [WS-A Eval Harness]──┐
   │ eval/dry-run  │     (engine adapters)   │
   └───────────────┘                         ▼
   [WS-B Evidence Layer]──► [WS-C Differential Engine]──► [WS-D Tagging] ──► [WS-F Report §17E.4]
        ▲ (D2 scope)            │                              │                 ▲
   ┌─────────┐                  ├──► [WS-E Mode adapters M1..M9]                  │
   │ D2 §17A.5│                 │                                                │
   └─────────┘                  └──► [WS-G PolicySimulationRun controller]───────┘
                                                                                 │
                                                  [WS-H Fixtures §17.5] ──────────┘
```

**Critical path:** C2 schema (or adapter stub) → Evidence Layer (WS-B) → Eval Harness (WS-A) → Differential Engine (WS-C) → Report (WS-F). Everything else hangs off these.

---

## 2. Parallel workstreams

| WS | Scope | Blocks on | Parallel-with |
|---|---|---|---|
| **WS-A Eval Harness** | Uniform `evaluate(bundle, input, ext_data)` over OPA/`gator`/Kyverno/Conftest | B1–B5 CLIs/APIs | WS-B |
| **WS-B Evidence Layer** | Materialize immutable EvidenceSets; digest; replay_completeness histogram; scope-enforced query | C2 adapter, D2 | WS-A |
| **WS-C Differential Engine** | §17.4 matrix, allow/deny normalization, reconciliation invariant, external-data pinning | WS-A, WS-B | WS-E |
| **WS-D Tagging** | SimulationTag state machine, untagged_risky gate, exception linkage | WS-C | WS-F |
| **WS-E Mode adapters** | M1..M9 input contracts; each mode is independent (embarrassingly parallel) | WS-A, WS-B | each other |
| **WS-F Report** | §17E.4 report producer + signing | WS-C, WS-D | WS-G |
| **WS-G CRD controller** | `PolicySimulationRun` reconcile loop, status machine | WS-B, WS-C | WS-F |
| **WS-H Fixtures** | §17.5 audit→fixture, regression store, CI re-run hook | WS-A | WS-E |

**Embarrassingly parallel within WS-E:** the 9 modes share the Evidence Layer + Eval Harness but each mode's input adapter (M1 manifest parser, M2 audit selector, M3 shadow tee, M4 snapshot reader, M8/M9 target picker) is independent and can be built by separate developers concurrently.

---

## 3. Milestones

- **MVP-0 (unblocks everything):** `ReplayEventV1` adapter + stub EvidenceSet from fixture files (no C2 yet). Lets WS-A/C/E proceed against the spec §13 field list.
- **MVP-1:** M1 Manifest + M2 Historical Replay + WS-A eval harness for OPA/Conftest. (DT-45, DT-46.)
- **MVP-2:** M5 Differential + WS-C matrix + WS-F report + reconciliation invariant. (DT-49, HL-17.) **This is the differentiator** — prioritize.
- **MVP-3:** WS-D tagging + untagged_risky promotion gate + exception linkage. (DT-49/HL-17 SC.)
- **MVP-4:** M6 Namespace scope (D2 wired) + M8/M9 tests + WS-H fixtures. (DT-50, DT-51, DT-52, HL-08.)
- **MVP-5:** M3 Live Shadow + M4 Snapshot + WS-G CRD controller. (DT-47, DT-48.)
- **GA:** Gatekeeper `gator` + Kyverno engine adapters; report signing; streaming/resumable large runs.

**Sequencing rationale:** differential (M5) is the market gap (research §8) and the flagship scenario (HL-17), so it precedes shadow/snapshot which are lower-differentiation and operationally heavier.

---

## 4. What can be built concurrently / what blocks what

- **Concurrent from day 1:** WS-A (engine adapters) and WS-B (evidence) once the `ReplayEventV1` adapter contract is frozen — neither needs the other.
- **The 9 mode adapters** are concurrent once WS-A+WS-B exist.
- **Blocks:** WS-C blocks WS-D blocks WS-F (tagging needs a diff; gate needs tags). WS-G (CRD) blocks on WS-C but not on WS-D.
- **Hard external blocker:** authoritative replay (M2,M5–M9) cannot ship real until C2 emits `replay_completeness`. Until then, runs are advisory and the adapter feeds synthetic fixtures.

---

## 5. Test strategy

| Layer | Tests |
|---|---|
| Unit | classify() over all 4 quadrants + within-class; reconciliation invariant property test (sum == evaluated for random inputs); allow/deny normalization of warn/mutate/suspend |
| Determinism | re-run same EvidenceSet+bundles ⇒ byte-identical DiffResult (D-FULL); external-data pinning regression |
| replay_completeness | complete/partial/insufficient routing; field-introspection downgrade (bundle references absent field ⇒ event downgraded) |
| Mode contracts | one golden test per M1..M9 wired to its scenario (DT-45..52) |
| Tagging gate | untagged_risky>0 ⇒ promotion blocked; exception filing flips fp_candidate; rationale-required for intended-relaxation |
| Scope (security) | M6 cannot read out-of-namespace events even with crafted selector (storage-layer enforced; GUI-bypass attempt fails) |
| E2E | HL-17 end-to-end: author v13 → diff over 30d → 47 newly blocked → tag → exception → re-diff → promote → 24h clean |
| Report | §17E.4 all-fields-present contract test; signature verify |

**Coverage anchors:** every DT-45..52 success-criterion becomes an automated assertion; HL-17 and HL-08 become E2E suites.

---

## 6. Risks / mitigations

| Risk | Mitigation |
|---|---|
| C2 schema not ready / diverges | `ReplayEventV1` adapter + synthetic fixtures; spec §13 is contract of record |
| External-data replay non-determinism | Pin to historical snapshot (D-E1-02); fail loudly if absent |
| Large evidence sets blow memory | Stream/chunk/resume; persist intermediate EvaluatedEvents |
| Engine eval semantics differ across OPA/Gatekeeper/Kyverno | Normalize to allow/deny-class at adapter boundary; per-engine golden tests |
| Shadow mode mistaken as reproducible | Mark D-LIVE; archive observed decisions to immutable EvidenceSet |
| "Authoritative" over incomplete data | Hard gate: incomplete never counts as authoritative; requireComplete flag |
