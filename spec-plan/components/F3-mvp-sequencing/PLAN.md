# F3 — POC Scale, MVP Scope & Sequencing — PLAN

**Component:** F3 · **Source:** §22, §26, §27 · **Status:** Authored (domain-lead fallback)

This PLAN contains **two things**: (1) F3's own implementation plan, and (2) the **first-draft whole-platform build sequence across all 23 components (A–F)** — the deliverable the primary will reconcile into the cross-cut MASTER-PLAN. The whole-platform sequence is §3 onward.

---

## 1. F3's own plan

F3 produces artifacts, not code: the cut line (SPEC §4), the sizing math (SPEC §2), the acceptance criteria (SPEC §6), and this sequence. Its only "implementation" is keeping the cut line honest as components evolve and feeding the primary's MASTER-PLAN.

- **Milestone F3-M1:** cut line + sizing ratified (this doc).
- **Milestone F3-M2:** whole-platform sequence reconciled with the 6 domain leads' PLANs by the primary.
- **Milestone F3-M3:** acceptance criteria wired into CI as the MVP definition-of-done.

---

## 2. The 23 components (recap) and their MVP status

| Comp | Name | MVP? | Thin-slice note |
|---|---|---|---|
| A1 | Governance model & Gemara + lineage | **MVP** | controls + lineage records (no graph DB) |
| A2 | Policy lifecycle (author→simulate→promote) | **MVP** | draft→dry-run→warn→enforce |
| B1 | OPA/Rego + signed bundles | **MVP** | engine + replay over §13 |
| B2 | Gatekeeper | **MVP** | admission deny/warn |
| B3 | Conftest | **MVP** | CI gate |
| B4 | Engine selection, action taxonomy, CRDs | **MVP (config)** | route OPA/GK/Conftest; CRD owner |
| B5 | Real-time enforcement flow | **MVP** | the §18 admission flow |
| C1 | Privateer integration | Phase-2 | evidence collectors beyond audit |
| C2 | Audit schema framework | **MVP** | the authoritative §13 replay schema |
| C3 | Compliance analytics | **MVP (2 detections)** | bypass + JWT drift (§14.2) |
| C4 | Retrospective detection | Phase-2 | advanced beyond the 2 examples |
| C5 | Reporting | **MVP (1 category)** | one human-readable report |
| D1 | Keycloak/JWT + mapping | **MVP** | §15.4 mapping; multi-IdP later |
| D2 | Scoped roles + storage authz | **MVP (thin)** | scope predicate (the hard one) |
| D3 | Approval-gated decisions | **MVP (thin)** | suspend_pending_approval + 1 webhook; mesh later |
| D4 | Security requirements | **MVP (baseline)** | §23 TLS/OIDC/signing/audit |
| E1 | Simulation & dry-run | **MVP** | differential replay (§17.4) |
| E2 | Console / Headlamp GUI | **MVP** | graph, replay, simulation, scoped authoring |
| E3 | Per-product PDP libraries | **MVP (K8s + 1–2)** | full §17D catalog phased |
| F1 | API | **MVP (the 8)** | §21.1 endpoints |
| F2 | Deployment & extensibility | **MVP (thin)** | operator install + core CRDs |
| F3 | MVP scope & sequencing | **MVP (meta)** | this doc |
| F4 | AI/agent governance | Phase-3 | deltas on the base, base-first |

---

## 3. Whole-platform dependency DAG (first draft)

```
                       ┌─────────────────────────── FOUNDATION (build first, mostly parallel) ───────────────────────────┐
                       │                                                                                                  │
   [C2 audit schema §13] ──────────────┐         [D1 JWT mapping §15.4] ──────┐        [A1 Gemara + lineage §6] ──┐       │
        │  (THE contract everyone emits)│              │ (identity contract)  │             │ (governance spine)  │       │
        v                               │              v                      │             v                     │       │
   [B1 OPA/Rego engine] <───────────────┘     [D2 scoped authz + storage pred]│      [A2 policy lifecycle] <───────┘       │
        │   (decision + replay engine)               │ (scope predicate lib)  │             │  (promote pipeline)         │
        │                                             │                        │             │                            │
        ├─────────────> [B4 engine selection/CRDs] <──┴────────────────────────┴─────────────┤  (CRD owner, action matrix)│
        │                       │                                                             │                            │
        v                       v                                                             v                            │
   [B2 Gatekeeper] [B3 Conftest] [B5 realtime flow §18]                                  [F2 operator+CRDs install] <──────┘
        │                       │                       │                                      │
        └───────────────────────┴───────────────────────┴──────────────────────────────────> [F1 API (the 8) §21]
                                                                                                      │
   ┌──────────────────────────────────────────────────────────────────────────────────────────────┐ │
   │                          UPPER TIER (depends on foundation contracts)                          │ │
   │                                                                                                │ v
   │   [E1 simulation §17.4] <── needs: C2 schema + B1 engine + D2 scope + A2 lifecycle             │
   │           │                                                                                    │
   │   [C3 analytics §14] <── needs: C2 schema (+ B2/B1 logs)                                        │
   │   [E3 PDP libraries §17D] <── needs: B-engines + C2 schema                                      │
   │   [D3 approval gates §17B] <── needs: B4 CRDs + F1 webhook                                      │
   │   [C5 reporting §17E] <── needs: C2 + C3 + E1                                                   │
   │   [D4 security §23] <── cross-cutting, lands incrementally across all                           │
   │   [E2 console §16] <── needs: F1 API + A1 lineage + E1 sim + C3 violations                      │
   └────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                       │
   ┌───────────────────────────────────────────────────┴─────────────────────────────────────────────┐
   │   PHASE-2:  C1 Privateer · C4 retro detection · D3 full mesh · C5 rich reports · E3 full catalog   │
   │   PHASE-3:  F4 AI/agent governance — DELTAS on C2/D1/D2/D3/E1/E3/F1 (no base refactor)             │
   │   PHASE-2/3 spine: lineage graph DB · export adapters (SIEM/GRC) · multi-IdP · cross-cloud         │
   └─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.1 The three foundation contracts (everything keys off these)
The platform has **three load-bearing contracts** that must stabilize first, because every other component either emits to or consumes them:
1. **C2 — the §13 replay-capable audit schema** (every engine emits it; E1/C3/C5 consume it).
2. **D1+D2 — the §15.4 JWT→subject mapping + §17A scope predicate** (every read/write/decision is scoped by it).
3. **A1 — Gemara controls + lineage records** (every policy and event traces to it).

**Locking these three early is the single highest-leverage scheduling move.** They are the API between domains.

---

## 4. Critical path (MVP)

```
A1 (Gemara+lineage contract) ─┐
C2 (§13 schema contract) ─────┼─> B1 (OPA engine emitting §13) ─> B2/B5 (Gatekeeper deny + realtime flow)
D1+D2 (identity+scope) ───────┘            │
                                           v
                              E1 (differential simulation over §13 replay)  ─> E2 (console shows graph+replay+sim)
                                           │
                                           └─> C3 (bypass/drift detection) ─> AC-7
```

**The critical path is: `C2 schema → B1 engine → E1 simulation → E2 console`.** Everything else can be built in parallel around it. E1 (differential simulation) is the longest single high-novelty build and gates the headline demo (AC-5); start it the moment C2's schema and B1's replay interface are stable, even with stubbed data.

---

## 5. Parallel workstreams (maximize concurrency)

**Wave 1 — Foundation contracts (all parallel, no inter-deps beyond schema agreement):**
- WS1a: **C2** authoritative §13 schema + ingest.
- WS1b: **D1** JWT mapping + **D2** scope-predicate library (shared with F1/F2).
- WS1c: **A1** Gemara model + lineage records.
- WS1d: **B4** action taxonomy + CRD group (single owner; unblocks B2/B3/F2).
- WS1e: **F2** operator skeleton + core CRDs (needs B4's CRD group).

> These five run concurrently. The only synchronization point is agreeing the three contracts (§3.1). Recommend a 1-week "contract freeze" before Wave 2.

**Wave 2 — Engines + API + lifecycle (parallel, behind Wave-1 contracts):**
- WS2a: **B1** OPA/Rego + signed bundles (emits §13).
- WS2b: **B2** Gatekeeper + **B5** realtime flow (emits §13).
- WS2c: **B3** Conftest CI gate.
- WS2d: **A2** policy lifecycle/promotion.
- WS2e: **F1** API (the 8 endpoints) over D2 scope predicate.
- WS2f: **D4** security baseline (TLS/OIDC/signing) — cross-cutting, starts here.

**Wave 3 — Upper tier (parallel, behind engines+schema):**
- WS3a: **E1** simulation/differential (critical path — start early with stubs).
- WS3b: **C3** analytics (2 detections).
- WS3c: **D3** approval-gate thin slice (suspend_pending_approval + 1 webhook).
- WS3d: **E3** K8s PDP library (+ 1–2 more).
- WS3e: **C5** one report category.

**Wave 4 — Console + integration:**
- WS4a: **E2** Headlamp console (graph + replay + simulation + scoped authoring) — integrates F1/A1/E1/C3.
- WS4b: end-to-end AC-1..8 wiring + the 3–6 integrations.

**Phase-2 (post-MVP, parallel):** C1 Privateer · C4 advanced retro detection · D3 full approval mesh · C5 rich/multi-framework reports · E3 full §17D catalog · lineage **graph DB** · export adapters (SIEM/GRC) · multi-IdP.

**Phase-3 (AI deltas — F4):** layered on C2 (evaluator_results/trace fields), D1/D2 (agent subject chain), D3 (approval patterns), E1 (behavioral differential), E3 (agent PDP catalog), F1 (agent sub-resources). **No base refactor** — the reframe doc's thesis. F4 can begin design in parallel with Phase-2 but ships after the base MVP validates the architecture.

---

## 6. What blocks what (the hard edges)

- **C2 blocks B1/E1/C3/C5** — nobody can emit or replay without the schema. Freeze it first.
- **D2 scope predicate blocks F1 real reads + E1 scoped replay + every list endpoint** — and it's the same predicate §26.1 defers (storage out of scope). **This is the #1 schedule + correctness risk** (F1 DEFECT-1, F2 DEFECT-1). Pull D2 forward; build the predicate as a library even over ordinary storage.
- **B4 CRD ownership blocks F2 + D3** — one CRD group, one owner; resolve before Wave 1 ends.
- **B1 engine blocks B2/E1** — replay interface must be stable.
- **E1 is the longest pole** on the critical path — start with stubbed data the moment C2/B1 interfaces exist.
- **F4 blocked by nothing structurally** (it's deltas) but **gated by judgment**: ship base-first so agents validate against a proven platform.

## 7. Test strategy (platform-level, owned with F3 acceptance)

- **Contract-freeze tests:** golden §13 event fixtures, golden JWT→subject mappings, golden lineage records — every component validates against these so the three contracts can't silently drift.
- **End-to-end MVP acceptance (AC-1..8)** wired into CI as the definition of "MVP done."
- **Scope-isolation tests** (cross-namespace/tenant leakage) across F1+D2+E1 — the highest-risk correctness property.
- **Capacity tests to §22 envelope** (owned with F2): 500k events/day, 10k-event sim latency, 50 GUI users.
- **Differential-simulation correctness:** known policy diff over a fixed replay set yields the exact newly_blocked/newly_allowed classification (§17.4).

## 8. Sequencing risks (top 5)

1. **D2 scope predicate vs deferred storage (§26.1)** — schedule + breach risk; pull D2 forward.
2. **C2 schema churn** — if §13 changes after engines emit it, mass rework; freeze early, declare it explicitly extensible (F1 DEFECT-8).
3. **E1 underestimated** — differential simulation is the novel core and the long pole; resource it first.
4. **CRD ownership collision (B4/F2)** — duplicate CRDs if unowned.
5. **F4 scope creep into MVP** — keep base-first; F4 is deltas, Phase-3.

---

## 9. Phased roadmap summary (MVP → GA)

| Phase | Theme | Components | Gate |
|---|---|---|---|
| **Wave 1** | Contract freeze | C2, D1, D2, A1, B4, F2(skeleton) | 3 contracts frozen |
| **Wave 2** | Engines + API | B1, B2, B3, B5, A2, F1, D4 | enforce + emit §13 |
| **Wave 3** | Simulation + analytics | E1, C3, D3(thin), E3(K8s), C5(1) | AC-5/AC-7 pass |
| **Wave 4** | Console + integrate | E2, integrations | AC-1..8 pass = **MVP** |
| **Phase 2** | Depth | C1, C4, D3-mesh, C5-rich, E3-full, graph DB, export adapters | wedge features |
| **Phase 3** | AI deltas | **F4** on C2/D1/D2/D3/E1/E3/F1 | agent governance, no base refactor → **GA** |
