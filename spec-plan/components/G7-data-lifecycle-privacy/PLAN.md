# G7 — Data Lifecycle, Retention & Privacy — PLAN

**Component ID:** G7 · **Domain:** G — Operational / NFR
**Companion:** `SPEC.md` (this directory), `ADVERSARIAL-REVIEW.md`.
**Posture:** the lifecycle layer sits *on top of* the C2 evidence spine; almost everything here is gated by C2's redaction-envelope seam (N-C2-303) and by G4's key services. The critical path is **classification registry → crypto-envelope at ingest → per-subject DEK lifecycle (G4) → crypto-shred + tombstone → erasure replay re-score**. Tiering and residency parallelize off to the side once the envelope exists.

---

## 1. Dependency DAG

```
                ┌─────────────────────────────────────────────────────────┐
                │ UPSTREAM CONTRACTS (must be stable before G7 builds)     │
                │  C2 1.0-rc: chain, N-C2-303 redaction envelope, CAS,     │
                │             N-C2-105 re-normalization, §5.5 reasons      │
                │  D2: scope predicates, visibility, DT-57 export          │
                │  G4: per-subject DEK API, key-destruction cert, notary   │
                │  F4: request_object agent content, agent-memory TTL      │
                └───────────────┬─────────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────────────────────┐
        ▼                       ▼                                        ▼
 ┌──────────────┐      ┌─────────────────────┐                 ┌───────────────────┐
 │ WS-A          │      │ WS-B                │                 │ WS-F (residency)  │
 │ Classification│      │ Crypto-envelope     │                 │ residency tagging │
 │ registry      │─────▶│ at ingest (§2.1)    │                 │ → G5 tuple (§8)   │
 │ (§3)          │      │ per-field encrypt   │                 │ (needs §3 tags)   │
 └──────┬────────┘      └─────────┬───────────┘                 └─────────┬─────────┘
        │ (class drives           │ (envelope is the                     │
        │  encrypt/TTL/tier)      │  prerequisite for shred)             │
        │                         ▼                                      │
        │               ┌─────────────────────┐                         │
        │               │ WS-C                │                         │
        │               │ per-subject DEK      │◀── G4 key API           │
        │               │ binding + reveal     │                         │
        │               │ gate (§6.3, G7-R22)  │                         │
        │               └─────────┬───────────┘                         │
        │                         ▼                                      │
        │               ┌─────────────────────┐                         │
        │               │ WS-D (CRITICAL PATH) │                         │
        │               │ crypto-shred + signed│◀── G4 destroy + cert    │
        │               │ tombstone + DSR (§5, │                         │
        │               │ §7)                  │                         │
        │               └─────────┬───────────┘                         │
        │                         ▼                                      │
        │               ┌─────────────────────┐                         │
        │               │ WS-E                │                         │
        │               │ erasure replay       │◀── C2 §5.5 + N-C2-105   │
        │               │ re-score + cascade   │    B1 input-fact capture │
        │               │ (§5.6, G7-R16..19)   │                         │
        │               └─────────────────────┘                         │
        ▼                                                                ▼
 ┌──────────────┐                                              ┌───────────────────┐
 │ WS-G (tiering)│  needs §3 classes + WS-B envelope            │ hand-off to G5    │
 │ hot/warm/cold/│  (SENSITIVE warm-max GC of agent bodies)     │ + G2 cost table   │
 │ archive (§4)  │──────────────────────────────────────┐      │ + G3 backup inv.  │
 └──────┬────────┘                                       │      └───────────────────┘
        ▼                                                ▼
 ┌──────────────┐                              ┌────────────────────┐
 │ WS-H archive  │  needs G4 long-lived sig     │ legal hold (§4.4)  │
 │ format + re-  │◀── G4 re-notarize / agility  │ overrides TTL+shred│
 │ notarize (§4.3)│                             │ (gates every WS-D) │
 └──────────────┘                              └────────────────────┘
```

**Edges that matter most:**
- WS-B (crypto-envelope at ingest) is the **single prerequisite** for WS-C/D/E. Nothing about erasure works until PII is encrypted-at-rest per-subject.
- WS-D depends on **G4** (external) for DEK destruction + certificates. If G4 slips, G7's erasure path stubs the destroy call but cannot ship real shred.
- Legal hold (§4.4) is a **cross-cutting gate** on WS-D — it must be wired before any real erasure can run, or you risk erasing held data.

## 2. Workstreams

| WS | Title | Deliverable | Depends on | Parallelizable with |
|---|---|---|---|---|
| **WS-A** | Classification registry | Versioned signed `(source,field_path)→class` registry; fail-safe-to-PII-DIRECT; classification_version stamping (G7-R1/R2/R3) | C2 field list, F4 sub-field map | WS-F, WS-G scaffolding |
| **WS-B** | Crypto-envelope at ingest | Normalizer writes PII/SENSITIVE fields as envelopes (§2.1); `value_digest` flows into `content_hash`; chain unchanged | WS-A, C2 N-C2-303, G4 DEK API | WS-F |
| **WS-C** | Per-subject DEK binding + reveal gate | DEK provisioning per (subject,tenant); grant-gated decrypt (G7-R22/23); `pii_reveal` audit; SoD (G7-R24) | WS-B, G4, D2 scope | WS-G, WS-H |
| **WS-D** | Crypto-shred + tombstone + DSR pipeline (**critical path**) | DEK destruction call; signed `lifecycle.tombstone`; DSR object + lifecycle (§7); legal-hold gate (§4.4) | WS-C, G4 destroy+cert | — (serializes E) |
| **WS-E** | Erasure replay re-score + cascade | re-score to `insufficient:erased_input` via N-C2-105 (G7-R16/17); derived-store cascade + verify (G7-R19); B1 input-fact capture (D-G7-03) | WS-D, C2 §5.5, B1/B5 | — |
| **WS-F** | Residency tagging → G5 hand-off | residency-tag computation (G7-R35); key-region constraint (G7-R36); residency tuple to G5 (G7-R39) | WS-A | WS-B, WS-G |
| **WS-G** | Tiering hot/warm/cold/archive | tier engine; content-preserving moves (G7-R7); SENSITIVE warm-max GC of agent bodies (G7-R8/R26); residency-bounded (G7-R9) | WS-A, WS-B | WS-C, WS-H |
| **WS-H** | Archive format + long-lived verification | self-describing container (G7-R10); re-notarization schedule → G4 (G7-R11); periodic re-prove (G7-R12) | WS-G, G4 notary | WS-C |

## 3. Critical path

```
C2 envelope seam stable
  → WS-A classification registry
  → WS-B crypto-envelope at ingest        ← the gate for everything erasure
  → WS-C per-subject DEK + reveal gate
  → WS-D crypto-shred + tombstone + DSR    ← needs G4 destroy+cert (external dep)
  → WS-E erasure replay re-score + cascade ← proves the regression is handled honestly
```

**Longest pole:** WS-B → WS-D, because WS-D cannot be *demonstrated* (a real erasure that keeps the chain verifying) until both the envelope (WS-B) and G4's key-destruction certificate exist. The single biggest schedule risk is **G4 readiness**; mitigate by building WS-D against a G4 mock that returns deterministic destruction certs, so G7's chain-still-verifies-after-shred test runs before G4 is real.

## 4. What can be built concurrently / what blocks what

**Concurrent from day 1 (no inter-dependency):**
- WS-A (classification registry) and WS-F (residency tagging) — both consume only the C2 field list + tenant policy.
- WS-G tier-engine scaffolding and WS-H archive-format definition — can be specced/stubbed before WS-B lands, then wired.

**Hard serialization:**
- WS-C **blocks on** WS-B (no reveal/shred without an envelope).
- WS-D **blocks on** WS-C **and** G4 (external).
- WS-E **blocks on** WS-D (you can't re-score until something was shredded) **and** on B1 emitting input-facts (D-G7-03).

**Fan-out opportunity:** WS-F/G/H (residency, tiering, archive) form a second independent track that runs alongside the WS-A→B→C→D→E erasure spine. Two sub-teams: an **erasure/privacy** team (A,B,C,D,E) and a **retention/residency** team (F,G,H), syncing only at the WS-A classification registry and the WS-B envelope.

## 5. Milestones

| ID | Milestone | Gate / exit criterion |
|---|---|---|
| **M-CLASS** | Classification registry frozen | Every C2 1.0-rc field + every F4 sub-field has a class; unclassified ⇒ PII-DIRECT + flagged. Signed + versioned. |
| **M-ENVELOPE** | Crypto-envelope at ingest live | A PII field round-trips through encrypt→store→decrypt; `content_hash` over the envelope is stable; chain verifies. |
| **M-SHRED** (key gate) | First crypto-shred that keeps the chain verifying | Destroy a test subject's DEK; the event's `content_hash` is unchanged; the checkpoint signed *before* the shred still verifies *after*; ciphertext no longer decrypts. **This is the headline acceptance test.** |
| **M-REPLAY-HONEST** | Erasure replay regression handled | A shredded identity-input event re-scores to `insufficient:erased_input` (not `:never_captured`); a decision with a derived input-fact (D-G7-03) still replays `complete` after shred. |
| **M-DSR** | End-to-end DSR | Access + erasure + rectification run through §7; legal hold suspends erasure correctly; the erasure produces a signed completion attestation; cascade verified. |
| **M-HOLD** | Legal hold overrides | A DSR erasure against held data is suspended + logged, never silently denied or executed. |
| **M-ARCHIVE** | Long-lived verifiable archive | A year-0-signed segment verifies after a simulated key rotation via G4 re-notarization, offline, from the archive container alone. |
| **M-RESIDENCY** | G5 hand-off | Residency tuple emitted per event; a forbidden-region tier/replicate is blocked (P0 test). EU-subject DEK stays in EU key region. |

## 6. Test strategy

- **Chain-survival tests (highest value).** For every lifecycle op (crypto-shred, tier-move, blob-GC, rectification-supersede): assert `content_hash` unchanged, `chain_seq` gap-free, pre-op checkpoint signature still verifies post-op. **A regression here is a P0 — it means a lifecycle op broke tamper-evidence**, the exact failure the whole component exists to avoid.
- **Erasure-completeness tests.** Post-shred: (a) decrypt MUST fail; (b) every derived store (materialized datasets, agent memory, RAG cache, aggregates) MUST be clear; (c) the tombstone + key-destruction cert MUST be present and verify. A DSR that doesn't prove the cascade does not close (G7-R31 step 6).
- **Honest-label tests.** `insufficient:erased_input` is distinct from `insufficient:never_captured` in coverage reports; the former does NOT count against capture-quality SLO (G7-R17). `<erased>` vs `<redacted>` vs cleartext rendered distinctly on export (G7-R21).
- **Legal-hold tests.** Held data survives TTL expiry and DSR erasure; release re-queues suspended DSRs; every blocked erasure is logged.
- **Replay-graceful-degradation test.** Decision that was a pure function of an erased identity still replays `complete` via the stored input-fact (D-G7-03); decision that genuinely needed the raw PII regresses to `insufficient:erased_input` and is reported as lawful, not defect.
- **Residency tests.** Forbidden-region placement/replication blocked (P0, mirrors D2 scope-escape); EU DEK never leaves EU key region; cross-border export carries a recorded transfer basis.
- **Agent-content containment tests.** `request_object` is never stored as one opaque cleartext blob; raw agent bodies are GC'd at warm→cold while the decision + digest survive; a single DSR shred clears audit-log + agent-memory + RAG-cache for the subject.
- **Independent-verification test (auditor at horizon).** Archive container verifies offline (no live service) after a key rotation, using only its embedded key-history (G7-R10/R11).

## 7. Risks & mitigations (plan-level; full red-team in ADVERSARIAL-REVIEW.md)

| Risk | Mitigation |
|---|---|
| **G4 not ready** (external dep on key-destruction cert + notary) | Build WS-D/H against a G4 mock returning deterministic certs; chain-survival tests run before G4 is real. |
| **Replay regression undersold** — erasure quietly tanks `complete` coverage | D-G7-03 input-fact capture + the `erased_input` vs `never_captured` split (G7-R17) make the regression explicit and lawful, not a hidden SLO hit. |
| **DEK-per-subject key population explodes cost** | Hand G4/G2 the per-(subject,tenant) grain (D-G7-02) for sizing; envelope-encryption (OQ-1) bounds KMS calls via per-tenant KEK. |
| **Backup re-introduces a "deleted" key** | Invariant to G3/G4: never back up a destroyed DEK (OQ-5); shred honored across restores by construction. |
| **Agent prompts leak from cold archive** | Warm-tier-max GC of raw agent bodies (G7-R26); only digest + decision survive long-term. |
| **Classification gap silently leaves PII in cleartext** | Fail-safe: unclassified ⇒ PII-DIRECT + flagged defect (G7-R1); classification_version stamped for audit. |
