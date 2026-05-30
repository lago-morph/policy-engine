# C2 — Audit Schema Framework & Replay-Capable Event Schema — PLAN

**Component ID:** C2 · **Status:** schema design FROZEN (SPEC §3.13); implementation plan below.
**Critical fact:** C2 is the gating dependency for C3/C4/C5/E1/F4/C1. The **schema-freeze milestone (M-FREEZE)** is the most important date on the whole-domain critical path; everything downstream stays in design until it lands.

---

## 1. Dependency DAG

```
        ┌──────────────────────────────────────────────────────────────┐
        │  EXTERNAL INPUTS (other domains)                             │
        │  B1 Rego metadata (policy-dependency catalog)               │
        │  D1 JWT/Keycloak normalization (subject, jwt_claims)        │
        │  D2 storage-authz scope model (§17A.5)                       │
        └───────────────┬──────────────────────────────────────────────┘
                        ▼
   ┌─────────────┐   ┌────────────────────┐   ┌──────────────────────┐
   │ W1 Schema & │   │ W2 Integrity core  │   │ W3 Policy-dependency │
   │ canonical-  │   │ (hash chain,       │   │ catalog ingest       │
   │ ization     │   │ Merkle, signing)   │   │ (from B1 metadata)   │
   └──────┬──────┘   └─────────┬──────────┘   └──────────┬───────────┘
          │                    │                         │
          └─────────┬──────────┴────────────┬────────────┘
                    ▼                        ▼
            ┌────────────────┐      ┌─────────────────────┐
            │ M-FREEZE       │      │ W4 Append-only store │
            │ (schema v1.0   │      │ + indexes + raw      │
            │ contract       │      │ retention            │
            │ published)     │      └─────────┬───────────┘
            └───────┬────────┘                │
   ┌────────────────┼─────────────────────────┼───────────────────────┐
   ▼                ▼                          ▼                       ▼
┌──────────┐ ┌──────────────┐ ┌────────────────────┐ ┌──────────────────┐
│ W5 Source│ │ W6 Completen-│ │ W7 Correlation     │ │ W8 Consumer query│
│ adapters │ │ ess scorer + │ │ engine (anchor,    │ │ API + scope      │
│ (×8)     │ │ state machine│ │ members, recovery) │ │ filter + verify  │
└────┬─────┘ └──────┬───────┘ └─────────┬──────────┘ └────────┬─────────┘
     └──────────────┴───────────┬───────┴─────────────────────┘
                                ▼
                    ┌────────────────────────┐
                    │ W9 Export integrity     │
                    │ envelope (manifest,     │
                    │ Merkle, sign) + dataset │
                    │ materialization         │
                    └────────────────────────┘
                                │
        ───────── unblocks ─────┴──────────────────────────────
        C3 analytics · C4 retro · C5 reporting · E1 simulation · C1 Privateer
```

## 2. Workstreams (what can be built in parallel)

| WS | Deliverable | Depends on | Parallelizable with |
|---|---|---|---|
| **W1** Schema + JCS canonicalization + `content_hash` | SPEC §3, §7.2 | — | W2, W3 |
| **W2** Integrity core: hash chain, Merkle checkpoints, ed25519 signing | SPEC §7 | crypto lib | W1, W3 |
| **W3** Policy-dependency catalog ingest from B1 Rego metadata | SPEC §4.2 | B1 metadata schema | W1, W2 |
| **W4** Append-only store, indexes, raw-event + raw-external-data retention | SPEC §8 | W1 (event shape) | W5 stubs |
| **W5** Source adapters ×8 (k8s-audit, gatekeeper, opa, conftest, keycloak, mesh, app-sdk, scanner) | SPEC §4.1, §3.11 | M-FREEZE | each adapter parallel to the others |
| **W6** Completeness scorer + state machine + reason codes | SPEC §5 | W3 (catalog), M-FREEZE | W7 |
| **W7** Correlation engine: anchor assignment, members view, recovery flags | SPEC §6 | M-FREEZE | W6 |
| **W8** Consumer query API + scope filter (D2) + verify endpoint | SPEC §10 | W4, M-FREEZE | W9 |
| **W9** Export integrity envelope + dataset materialization | SPEC §7.6, §8.5 | W2, W4 | — |

**Maximally parallel:** W1∥W2∥W3 up front; after M-FREEZE the eight adapters (W5) and W6/W7/W8 fan out independently.

## 3. Critical path

```
B1 metadata schema ──▶ W3 catalog ──▶ W6 scorer
W1 schema ──┬──────────────────────────────▶ M-FREEZE ──▶ (C3/C4/C5/E1 begin design-against-contract)
W2 integrity┘                                    │
                                                 └──▶ W5 adapters ──▶ W8 API ──▶ W9 export ──▶ first signed auditor export (DT-24/HL-18)
```

**The critical path runs through M-FREEZE.** Pull M-FREEZE as early as possible: it needs only W1 (schema + canonicalization) settled and W2/W3 contracts stubbed — NOT their full implementation. Freeze the *contract* (this SPEC already does), publish it, and let C3/C4/C5/E1 code against it while C2's own internals (W4–W9) are still being built. **This SPEC is the freeze artifact.**

## 4. Milestones

- **M0 — Contract freeze (M-FREEZE):** SPEC §3.13 field list + §5 state machine + §6 correlation + §7 envelope + §10 API published. *(This document satisfies M0.)* Unblocks all downstream design.
- **M1 — Walking skeleton:** W1+W4 — one adapter (gatekeeper) writes `complete` events into the append-only store with a valid hash chain; verify endpoint passes. Proves the integrity story end-to-end on one source.
- **M2 — Completeness + correlation:** W6+W7 — the gatekeeper↔OPA↔k8s-audit join works on real admission flows; `complete`/`best_effort`/`insufficient` classify correctly; DT-28 correlation fix demonstrated.
- **M3 — Full source fan-out:** all 8 adapters (W5) live; §3.11/§3.12 coverage checks green; DT-16/DT-25 normalizer regression tests in CI.
- **M4 — Consumer-ready:** W8 query API + scope filter + coverage feed; C3/C4/C5 integrate against real data.
- **M5 — Auditor-grade exports:** W9 signed export + dataset materialization; DT-24, DT-46, DT-78, HL-18 pass end-to-end with independent verification.

## 5. Test strategy

### 5.1 Replay-determinism tests (the headline suite — do these first)
- **T-DET-1 Canonicalization determinism:** the same logical event serialized via two independent code paths yields the identical `content_hash`. (HL-18 auditor independence depends on this.)
- **T-DET-2 Scorer determinism:** the completeness scorer run in the normalizer (live) and in E1/C4 (replay) returns the **same** `replay_completeness` for the same inputs + same policy-dependency catalog version. Cross-process golden vectors.
- **T-DET-3 Replay equivalence:** replaying a `complete` event reproduces the recorded `decision` exactly (E1 integration). Any divergence is a defect, not "acceptable noise."
- **T-DET-4 Backfill recovery (DT-25):** seed a dropped-`external_data_refs` event → `insufficient`; fix projection + re-normalize from retained raw → recovers to `complete`; a sibling event whose raw external-data response was *not* retained stays `insufficient` (never silently promoted).

### 5.2 Schema-conformance & no-field-drop
- **T-SCH-1 No-field-drop (DT-25/DT-16 regression):** for each adapter, a golden raw event maps to a C2 event with *every* home-able field populated; the test fails if any source field with a C2 home is dropped. This is the exact bug DT-25 prevents.
- **T-SCH-2 §9.3 coverage:** a Gatekeeper raw event populates all 17 §9.3-mapped C2 locations (SPEC §3.11).
- **T-SCH-3 Required-field invariant:** every emitted event has the 14 R fields; conditional fields satisfy their rule (e.g. `decision` present iff `event_type=policy.decision`).
- **T-SCH-4 N-C2-SYNTH:** no `replay.synthetic` event can carry `replay_completeness=complete`.

### 5.3 Tamper-evidence tests
- **T-TMP-1 Chain break detection:** edit a stored event → chain verification fails at that link.
- **T-TMP-2 Sequence-gap detection:** delete an event → `chain_seq` gap detected.
- **T-TMP-3 Checkpoint forgery:** edit a past event and re-hash forward without the key → checkpoint signature verification fails.
- **T-TMP-4 Export round-trip:** sign an export → independent verifier with only the published public key validates the Merkle root + manifest signature (HL-18, DT-24).
- **T-TMP-5 Redaction-preserving hash (§7.5):** redacted claim still contributes deterministically; a cleartext holder can prove the match; redaction does not break the chain.

### 5.4 Correlation tests
- **T-COR-1 Admission join:** gatekeeper + OPA + k8s-audit events for one admission share `correlation_id`; `correlation_members` lists all three present.
- **T-COR-2 DT-28 fix:** with OPA echoing the Admission Review UID, ≥99% of denies in a window have a paired OPA decision log; revert the config → the C3 alert fires; restore → clears within one window.
- **T-COR-3 Minted-id flag:** an event with no upstream anchor is minted and flagged (`correlation_source=minted`).

### 5.5 Scope-authz tests
- **T-AZ-1:** an Auditor scoped to one cluster/control cannot read events outside scope via the query API (not just the UI) — DT-46 step 8, HL-18.

### 5.6 Performance/scale (early, not deferred)
- Ingest throughput at target admission rate; checkpoint cadence does not stall ingest; query p95 within budget for a 30-day window scoped to one control (DT-46 materializes 11k events).

## 6. Parallel build guidance / what blocks what
- **Front-loadable now (no external blocker):** W1 schema, W2 integrity core, the canonicalization + hashing library, golden-vector test harness.
- **Blocked on B1:** W3 catalog (needs the Rego-metadata schema). Mitigation: stub the catalog with a hand-authored dependency map for `SC-IMG-001` so W6 can be built and tested before B1 lands.
- **Blocked on D1:** faithful `subject`/`jwt_claims` normalization. Mitigation: ingest raw claims passthrough first; normalize when D1 lands.
- **Blocked on D2:** scope-filter enforcement in W8. Mitigation: build the filter hook with a permissive default in dev, wire D2's decision when ready.
- **Everything downstream (C3/C4/C5/E1) is unblocked at M-FREEZE** for design and contract-testing, and at M4 for live data.

## 7. Risk register (C2-specific)
- **R1 (highest): schema churn after downstream code against it.** Mitigation: this freeze + additive-only v1.x + N-C2-FWD; any breaking change is a major bump with a migration plan and a coordinated downstream PR.
- **R2: completeness scorer disagreements between live and replay** silently corrupt "authoritative" counts. Mitigation: T-DET-2 golden vectors gate every change to the scorer or catalog.
- **R3: raw-data retention gaps** make backfill impossible (the 3 unrecoverable DT-25 events). Mitigation: monitor raw-external-data retention coverage; alert when a provider's raw responses are not being retained.
- **R4: tamper-evidence theater** — chain exists but nobody verifies it. Mitigation: continuous chain-verification job (a C3 finding on break) + verify endpoint exercised in CI and by auditors.
- **R5: correlation anchor missing at the source** (OPA not echoing UID) silently degrades everything to `best_effort`. Mitigation: T-COR-2 + a C3 alert when >1% of denies are unpaired (DT-28).
