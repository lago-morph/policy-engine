# G6 — Observability, SLOs & Day-2 Operations — PLAN

**Component:** G6 · **Date:** 2026-05-30 · **Status:** Authored
**Companion:** `SPEC.md` (requirements), `ADVERSARIAL-REVIEW.md` (attack surface).

This plan exploits parallelism: G6 has **four largely-independent workstreams** (telemetry, SLO framework, migration tooling, runbook authoring) that converge only at integration and game-day rehearsal. The critical path runs through migration tooling, because the C2 epoch-migration tool is the one deliverable that (a) is novel, (b) gates the corpus's #1 unowned risk, and (c) cannot be validated without the C2 chain implementation existing.

---

## 1. Dependency DAG

```
                          ┌─────────────────────────────────────────────┐
   EXTERNAL PREREQS       │  C2 chain + checkpoints (N-C2-301/302)       │
   (other components)     │  F2 service inventory + operators            │
                          │  B4 CRD versions (R-B4-22)                   │
                          │  G1 capacity model · G3 DR/buffer · G4 keys  │
                          └───────────────┬─────────────────────────────┘
                                          │
        ┌─────────────────────┬───────────┴────────────┬──────────────────────┐
        ▼                     ▼                        ▼                       ▼
  WS-A TELEMETRY        WS-B SLO FRAMEWORK       WS-C MIGRATION TOOLING   WS-D RUNBOOKS
  (self-observability)  (SLIs→SLOs→alerts)       (epoch + CRD + expand-   (authoring,
        │                     │                   contract tooling)        on-call, IR)
        │  golden signals     │  needs WS-A        │  needs C2 chain        │  needs WS-A/B/C
        │  per service        │  signals to        │  (CRITICAL PATH)       │  outputs to
        │  (R-G6-OBS-6)       │  compute SLIs      │                        │  reference
        ▼                     ▼                    ▼                        ▼
  A1 instrument 14     B1 define SLIs from    C1 epoch-migration       D1 upgrade/rollback
     services             golden signals          tool (RUN-7)            choreography (RUN-1..4)
  A2 telemetry/audit   B2 example SLOs +      C2 epoch-aware           D2 CRD-migration (RUN-5/6)
     pipeline split       error budgets          verifier (RUN-9)      D3 cert/secret/key
     (OBS-1)           B3 burn-rate alerts    C3 CRD conversion-          rotation (RUN-11/12)
  A3 chain-integrity   B4 error-budget          webhook + storage-     D4 capacity scale (RUN-13)
     + key-health         policy (SLO-3)         migrator (RUN-5)      D5 on-call model + IR
     monitor (OBS-9/10)                        C4 expand-contract        (ONCALL/IR)
  A4 tracing on           ┌──────────────────────┴──────────┐         D6 managed-vs-self
     governance_txn_id    │                                 │            packaging (§6)
     (OBS-7)              ▼                                 ▼
                    INTEGRATION (I): dashboards + alerts + runbooks wired together
                                          │
                                          ▼
                    GAME-DAY (G): rehearse epoch migration, key rotation, full
                                  upgrade, incident drills on staging
```

**Hard prerequisite edges (cannot start until satisfied):**
- WS-C (migration tooling) needs a working C2 hash chain + checkpoints to migrate.
- WS-B (SLOs) needs WS-A golden-signal instrumentation to compute SLIs against.
- WS-D (runbooks) references all three but can *draft* in parallel and *finalize* after.
- Game-day needs everything + a staging stack (F2 install).

**Soft edges (parallelizable with stubs):** WS-A/B/D can develop against synthetic/stubbed signals before the real services emit them.

---

## 2. Workstreams

### WS-A — Telemetry / self-observability (parallel-internal)
- **A1** Instrument all 14 services with the §2.3 golden-signal table (OpenMetrics `/metrics` + readiness≠liveness). *Highly parallel* — one task per service, independent. Owner each service team; G6 owns the conformance checklist (R-G6-OBS-6).
- **A2** Build the **two-pipeline split** (R-G6-OBS-1): platform telemetry → Prometheus/Tempo/Loki (or OTLP egress to customer's stack, OQ-G6-1); audit evidence → C2 (untouched). Enforce the no-leak rules (OBS-2/3). This is the integrity firewall — single most important WS-A deliverable.
- **A3** Chain-integrity + signing-key-health monitors (OBS-9/10) — wraps the C3 chain-verify check as the top-severity platform signal. Depends on C2 existing.
- **A4** Distributed tracing keyed on `governance_transaction_id`/`correlation_id` (OBS-7). Reuses C2 join keys; no new primitive.

### WS-B — SLO framework (depends on WS-A signals)
- **B1** Define the three-surface SLIs from golden signals.
- **B2** Set example SLOs + error budgets (admission 99.9%/250ms; ingest lossless+60s; console 99.5%/1s).
- **B3** Implement multi-window/multi-burn-rate alerts (§3.2).
- **B4** Write the error-budget policy + the non-negotiable carve-outs (ingest losslessness, chain integrity never relaxed).

### WS-C — Migration tooling (CRITICAL PATH)
- **C1** The **C2 chain-epoch migration tool** (R-G6-RUN-7): seal Epoch 0 (final checkpoint), open Epoch 1 with cross-signed genesis, switch producers to 1.0-rc. The novel, highest-risk deliverable.
- **C2** The **epoch-aware + key-aware verifier** (R-G6-RUN-9, MIG-4): extends `POST /verify` (C2 §10) to verify per-epoch under per-epoch schema and per-segment under per-`key_id`, plus cross-links.
- **C3** CRD conversion-webhook + storage-migrator job + tested inverse (RUN-5/6).
- **C4** Generic expand-contract harness (MIG-1): additive-deploy → migrate-readers → flip → retire-later, with dry-run-on-copy (MIG-3).

### WS-D — Runbook authoring + on-call (drafts in parallel, finalizes last)
- **D1** Upgrade/rollback choreography (RUN-1..4) — the dependency-ordered 8-step.
- **D2** CRD-migration runbook (RUN-5/6).
- **D3** Cert/secret/signing-key rotation runbook + G4 hand-off contract (RUN-11/12).
- **D4** Capacity-scale runbook + G1 hand-off (RUN-13/14).
- **D5** On-call model (page budget ≤2/shift, severity tiers) + incident response + evidence-gap scoping (ONCALL/IR).
- **D6** Managed-vs-self packaging decision + the customer-held-key invariant (§6, DEC-G6-2).

---

## 3. Critical path

```
C2 chain exists ──► C1 epoch-migration tool ──► C2 epoch-aware verifier ──►
  ──► Game-day: rehearse 1.0→1.0-rc epoch transition on staging ──►
  ──► prove chain still verifies end-to-end across the boundary ──► SIGN-OFF
```

The critical path is **WS-C**, because:
1. The epoch-migration tool is the only G6 deliverable with no prior art in the corpus and the one that discharges META Risk #8 (the unowned breaking migration).
2. It cannot be validated without the real C2 chain — it is gated on an external component.
3. Its acceptance gate (chain verifies clean across the epoch boundary with only the published public key) is the single most important G6 sign-off.

Everything else (telemetry, SLOs, most runbooks) is off the critical path and parallelizable.

---

## 4. Milestones

| # | Milestone | Gate | Depends on |
|---|---|---|---|
| **M1** | Golden signals emitted by all 14 services | Conformance checklist green (R-G6-OBS-6) | services exist |
| **M2** | Two-pipeline split live; no-leak verified | A telemetry flood drops 0 audit events; no PII in telemetry | M1, C2 |
| **M3** | Three-surface SLOs + burn-rate alerts live | Synthetic SLO violation pages correctly; ≤2 pages/shift baseline | M1, M2 |
| **M4** | Chain-integrity + key-health monitors live | A simulated chain break pages Sev-1 within the cadence window | C2 chain |
| **M5** | **Epoch-migration tool + epoch-aware verifier** | **Dry-run 1.0→1.0-rc on a staging copy; chain verifies clean across the boundary; all prior signatures still valid** | C2 chain (**critical**) |
| **M6** | CRD-migration tooling + tested inverse | Round-trip fuzz `v1alpha1→v1beta1→v1alpha1` is identity; no state loss | B4 CRDs |
| **M7** | All runbooks authored + linked to alerts | Every Sev-1/2 alert links a runbook | M1–M4 |
| **M8** | **Game-day rehearsal** | Live epoch migration + key rotation + full stack upgrade + incident drill on staging, all pass | M5, M6, M7 |
| **M9** | Managed-tier packaging decision ratified | Customer-held-key invariant documented + enforced in both tiers | D6 |

---

## 5. Test strategy

- **Telemetry:** assert the no-leak invariants as *tests* — a test that greps telemetry for any C2 content field and fails if found (OBS-2); a back-pressure test that floods telemetry and asserts zero audit-event drops (OBS-1).
- **SLO:** inject synthetic latency/errors; assert burn-rate alerts fire at the right window; assert the page budget holds under a normal-noise replay (ONCALL-1).
- **Migration (the crown jewel):** generate a v1.0 chain with signed checkpoints; run the epoch migration; assert **(a)** every v1.0 event's content/signature is byte-identical and still verifies, **(b)** Epoch 1 genesis `prev_hash` == Epoch 0 final checkpoint root, **(c)** the epoch-aware verifier passes end-to-end with only the published public key, **(d)** a 1.0-rc-aware consumer can read v1.0 events via the additive mapping. Adversarial test: attempt an in-place rewrite and assert the verifier *detects it as tampering* (proving the epoch path is the only clean one).
- **CRD:** fuzz round-trip conversion = identity; assert no open approval / unexpired exception is lost across a storage-version flip (RUN-6).
- **Runbooks:** game-day execution on staging is the test — a runbook that can't be executed under drill conditions is a failed test.

---

## 6. What can be built concurrently / what blocks what

**Fully concurrent (no inter-dependencies):**
- A1 (per-service instrumentation) — 14 independent tasks.
- D1–D6 runbook *drafting* (finalize after their tooling exists).
- B1–B4 *design* (implement after A signals exist).

**Blocks others:**
- C2 chain implementation **blocks** C1/C2/A3 (the migration tool + integrity monitor have nothing to operate on without it).
- A1 **blocks** B1–B3 (no SLIs without signals).
- C1/C2 **block** M5 and game-day M8.
- B4 CRDs **block** C3/M6.

**The single longest pole:** the epoch-migration tool (C1) → epoch-aware verifier (C2) → game-day rehearsal (M8). Start C1's *design* immediately (it's pure mechanism — seal/cross-sign/open, derivable from the C2 chain spec today) so that the moment the C2 chain implementation lands, the tool is ready to validate against it.
