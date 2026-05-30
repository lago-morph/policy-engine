# G7 — Data Lifecycle, Retention & Privacy — SPEC

**Component ID:** G7 · **Domain:** G — Operational / Non-Functional Architecture
**Wave:** Operational/NFR (sibling to G1 scale, G2 cost/retention-economics, G3 DR, G4 key-management, G5 multi-tenancy/residency, G6 day-2 ops, G8 Rego human-factors).
**Status:** v1.0 — decisions made and documented; ready for adversarial review.
**Spec sources / upstream contracts:**
- `C2-audit-schema/SPEC.md` + `cross-cutting/C2-v1.0-rc-RECONCILED.md` — the append-only, hash-chained, signed-Merkle-checkpoint evidence log; the `content_hash`/`prev_hash`/`chain_seq` integrity primitive; redaction-aware integrity (N-C2-303); raw-event retention (N-C2-402); CAS for large objects + external-data values (§8.4, N-C2-EDV).
- `D2-scoped-rbac-storage/SPEC.md` §5.3–5.4 — visibility levels, redaction-aware export, the DT-57 flow, scope predicates.
- `F4-ai-agent-extension/SPEC.md` — `request_object` now carries assembled prompts, conversation history, retrieved RAG documents, tool catalogs; `agent.prompt`/`agent.tool_call`/`agent.output` events.
- `D4-security/SPEC.md` §2.2 (SEC-5..8 evidence integrity), OQ-4 (hash-at-rest for `sub`/`email` with grant-gated reveal).
- `cross-cutting/META-ADVERSARIAL-SECOND-OPINION.md` §3 (audit-log retention economics unowned; the privacy explosion of agent `request_object`; key-management foundational not hardening).
- Sibling hand-offs: **G2** (retention economics / $-per-event-per-year), **G4** (key management & long-lived signatures), **G5** (multi-tenancy + data residency).
**Scenarios exercised:** DT-57 (redacted JWT-subject export), §23 evidence integrity, DT-24/HL-18 (auditor independent verification), DT-25 (backfill bounded by raw retention).

---

## 0. Why this component exists (the unowned problem)

The META-ADVERSARIAL pass found three holes that all land in G7:

1. **Retention economics are unowned.** C4-OQ2 bounds sweeps by "C2 raw retention (≥30d POC)"; real SOC2/financial-services regimes require **multi-year** retention. Raw external-data values + `before_state`/`after_state` + `request_object` + `jwt_claims`, content-addressed and signed, kept for 7 years at production volume, is a large unmodeled cost — and XD-1's "value capture is a MUST" *increases* it.
2. **The corpus keeps a tamper-evident immutable record of everything and never noticed that this is on a collision course with "delete this user's data on request."** GDPR Art. 17 (right to erasure) / Art. 5(1)(c) (data minimization) / CCPA §1798.105 (right to delete) demand that a data subject's personal data be deleted. The C2 audit log is append-only, hash-chained, and signed precisely so that **nothing can be deleted or altered without detection**. These two requirements are, on their face, mutually exclusive. **No prior component owns the resolution.**
3. **Storing agent prompts is a breach magnet.** F4's `request_object` is the assembled context: system prompt, conversation history, retrieved RAG documents, tool arguments. This is the single highest-density PII/secret/confidential payload in the entire platform, and it is written into the same immutable, multi-year-retained, content-addressed evidence log as everything else.

G7 owns: **(1) retention & tiering mechanics, (2) the erasure-vs-immutability resolution, (3) PII classification + handling, (4) data-residency hand-off to G5.** It does *not* re-own key custody (G4) or the $-cost model (G2); it defines the lifecycle contract those components instantiate.

---

## 1. Scope

### 1.1 In scope
- **Data-class taxonomy** for every field/blob the platform persists (§3) and a per-class retention TTL, tiering policy, and erasure policy.
- **Retention & tiering mechanics**: hot/warm/cold/archive tiers, TTL enforcement, the signed-tombstone deletion record, archival format that stays independently verifiable across the retention horizon (ties to G4 long-lived signatures), and **legal hold** (§4).
- **The erasure-vs-immutability resolution** (§5): crypto-shredding via per-subject field-level encryption keys, the decision matrix, and the precise replay/chain-integrity impact.
- **PII handling** (§6): field classification, redaction vs encryption-at-rest, who can see un-redacted data (ties to D2 visibility/scope), and the F4 agent-`request_object` containment regime.
- **Data-subject-request (DSR) pipeline** (§7): access / erasure / rectification request handling end-to-end, including the audit log of the erasure itself.
- **Residency hand-off contract to G5** (§8).

### 1.2 Out of scope (delegated)
- **Key custody, rotation, HSM/KMS selection, signer separation-of-duties → G4.** G7 *consumes* a per-subject-key capability and a long-lived signing/notarization service; it specifies their lifecycle semantics, not their cryptographic implementation.
- **$-per-event-per-year cost model, storage-growth projection, tier-cost arbitrage → G2.** G7 defines *what* tiers and TTLs exist; G2 prices them.
- **Physical region placement, sovereign-cloud topology, cross-region replication → G5.** G7 defines *which data classes are residency-constrained* and hands G5 the constraint; G5 places the bytes.
- **The append-only log's chain/checkpoint mechanics themselves → C2.** G7 layers lifecycle on top of C2's primitives; it must not break them.
- **Disaster recovery / RPO / RTO of the evidence store → G3.** Backup and restore-and-prove-chain are G3; G7 ensures the lifecycle (tier transitions, crypto-shred) survives a restore.

### 1.3 Design tenets
- **The audit log is never mutated.** Every G7 mechanism is layered so that no byte of a committed C2 event is rewritten and no hash-chain link is broken. Erasure is achieved by **destroying the key that decrypts a field**, not by deleting or editing the field's ciphertext. This is the load-bearing tenet (§5).
- **Erasure is itself an audited, signed event.** A deletion that leaves no trace is indistinguishable from tampering. Every TTL expiry and every crypto-shred is recorded as a signed tombstone in the same chain (extends N-C2-400).
- **Minimize at ingest, not at export.** The cheapest PII to protect is the PII you never stored verbatim. Pseudonymization/encryption happens in the normalizer (write path), not as an afterthought.
- **Honesty over coverage, applied to privacy.** A field that has been crypto-shredded MUST be representable as a first-class, queryable state (`shredded`), exactly as `replay_completeness` surfaces `insufficient`. A shredded input that breaks replay is *labeled*, never silently dropped or silently promoted.

---

## 2. Data model — the lifecycle envelope

G7 does not add top-level C2 fields (C2 is additive-only and we honor N-C2-FWD). It defines a **per-field encryption + lifecycle envelope** carried inside the existing extensible structures, plus three side records that live *outside* the immutable event body but are themselves chained.

### 2.1 The crypto-envelope for protected fields
A field classified `PII` or `SENSITIVE` (§3) is stored not as cleartext but as an **envelope**:

```json
{
  "enc": "aes-256-gcm",
  "key_ref": "subject-dek://payments/sub:user-123#v1",   // per-subject DEK reference (G4-managed)
  "iv": "base64…",
  "ct": "base64…",                                          // ciphertext of the cleartext value
  "value_digest": "sha256:…"                                // salted hash of cleartext, for content_hash + match-proof
}
```

This is a **strict superset of C2's existing redaction-aware shape** (N-C2-303, which already stores `{"redacted": true, "value_digest": "sha256:…"}`). The `value_digest` is what flows into `content_hash`, so:

- **The hash chain is computed over the envelope, not the cleartext.** Destroying `key_ref`'s key destroys the ability to read `ct`, but `value_digest`, `iv`, and the envelope structure are unchanged → **`content_hash` is unchanged → the chain still verifies.** This is the entire trick (§5).
- A holder of the cleartext can still prove a match against `value_digest` (auditor independence preserved for non-shredded subjects).

### 2.2 Three side records (chained, not in the event body)
| Record | Purpose | Integrity |
|---|---|---|
| **Tombstone event** | Records a TTL expiry, a hard delete at retention end, or a crypto-shred. `{ op, target_ref, target_digest, reason, legal_hold_check, actor, signed_at }`. | Appended to the chain as a normal C2 event (`event_type=lifecycle.tombstone`); signed; gap-detectable. |
| **Key-destruction certificate** | Issued by G4 when a per-subject DEK is destroyed. `{ key_ref, destroyed_at, dsr_id, attestation_sig }`. | Signed by the key-management service; referenced from the tombstone. Proves the erasure cryptographically. |
| **Retention manifest** | Per data class: TTL, tier schedule, legal-hold set, residency tag. Versioned, signed. | Bound into checkpoint signatures so the *policy under which data was retained/erased* is itself evidence. |

### 2.3 Tiering metadata (side index, not event body)
Each event/blob carries a derived `lifecycle_state` in a **side index** (never in `content_hash`, because it changes over time): `{ tier: hot|warm|cold|archive, retain_until, legal_hold: bool, shred_state: live|partial|shredded, residency: <region-tag> }`. Mutable; the *transitions* are the auditable artifacts (tombstones / tier-move records), not the current value.

---

## 3. Data-class taxonomy (the classification contract)

Every persisted field maps to exactly one class. Class drives TTL, tier, encryption, erasure, and residency. (Field numbers are C2 1.0-rc field indices.)

| Class | Definition | Example fields | At-rest | Erasure mechanism | Default TTL |
|---|---|---|---|---|---|
| **PII-DIRECT** | Directly identifies a data subject. | `subject.sub`, `subject.email`, `jwt_claims.sub`/`email`/`preferred_username`, `jwt_claims.groups` (where group = person), F4 `request_object` prompt text & conversation history | **Per-subject field-level encryption** (§2.1) | **Crypto-shred** (destroy per-subject DEK) | erasure on DSR; else class TTL (see §4.1) |
| **PII-INDIRECT** | Quasi-identifier; re-identifies in combination. | `subject.roles`, `subject.tenant`, `resource_id` (may embed a username), `scope.namespace` (may name a team), source IPs in `engine_context` | Per-subject encryption **if** linkable to one subject; else pseudonymized | Crypto-shred if per-subject; else pseudonym-map deletion | class TTL |
| **SENSITIVE-CONTENT** | Confidential/secret payload, not necessarily personal. | F4 `request_object` retrieved RAG docs, tool arguments; `before_state`/`after_state` of secrets/configmaps; external-data **values** that contain secrets | **Per-tenant** field-level encryption (not per-subject — no single subject owns it) | Crypto-shred per-tenant **or** drop-to-reference at warm tier (§6.4) | short (see §6.4) |
| **REPLAY-CRITICAL-NONPII** | Needed for authoritative replay; carries no personal data. | `request_object` (non-PII structural parts), `external_data_refs` value/digest, `policy_version`, `catalog_version`, `policy_ref`, `disposition`, `obligations` | Cleartext (tenant-encrypted at rest at infra level) | **Not erasable** — retained for full audit horizon | full audit TTL (§4.1) |
| **INTEGRITY-METADATA** | The tamper-evidence skeleton. | `content_hash`, `prev_hash`, `chain_seq`, `signature`, `event_id`, `correlation_id`, `governance_transaction_id`, `timestamp` | Cleartext | **Never erasable, never encrypted** (it is the chain) | full audit TTL + legal-hold horizon |
| **OPERATIONAL** | Logs/metrics/traces of the platform itself. | day-2 telemetry (G6), normalizer logs | Standard infra encryption | Standard rolling deletion | 30–90 d |

### 3.1 Normative classification requirements
- **G7-R1 (MUST)** Every field a normalizer projects MUST carry a data-class tag derived from a **central classification registry** keyed by `(source.system, field_path)`. An unclassified field is treated as **PII-DIRECT** (fail-safe: most-protective) and flagged as a classification defect (mirrors D2 "missing authz metadata ⇒ invisible").
- **G7-R2 (MUST)** The classification registry is **versioned and signed**; the registry version under which an event was ingested is recorded in `engine_context.lifecycle.classification_version` so the protection applied is itself auditable.
- **G7-R3 (MUST)** F4 agent `request_object` content is classified at sub-field granularity: prompt/conversation = PII-DIRECT; retrieved RAG docs + tool args = SENSITIVE-CONTENT; model identity / sampling params / attestation digests = REPLAY-CRITICAL-NONPII. The whole blob is NEVER stored as a single opaque cleartext object (this is the breach-magnet fix — §6.4).
- **G7-R4 (MUST)** `INTEGRITY-METADATA` fields are MUST-NOT-encrypt and MUST-NOT-erase. Any DSR or retention rule that would touch them is rejected at policy-compile time, not at runtime.

---

## 4. Retention & tiering mechanics

### 4.1 TTL per data class (the retention contract handed to G2 for pricing)
TTLs are **policy-driven and per-tenant overridable upward** (a regulated tenant may require longer; never shorter than the legal floor). POC defaults vs. production floors:

| Class | POC default | Production floor (SOC2 / fin-svc) | Notes |
|---|---|---|---|
| INTEGRITY-METADATA | 30 d | **7 y** (+ legal-hold extension) | The chain must span the full evidence horizon (§23). |
| REPLAY-CRITICAL-NONPII | 30 d | **7 y** | Replay value decays but audit value does not. G2 prices the dominant cost here. |
| PII-DIRECT | 30 d | **min(legal-retention-floor, data-minimization-ceiling)** — typically 1–3 y, **subject to DSR-triggered early crypto-shred** | The tension class (§5). |
| PII-INDIRECT | 30 d | as PII-DIRECT | |
| SENSITIVE-CONTENT (incl. agent RAG/prompt bodies) | **7 d hot, then reference-only** | **90 d max for raw bodies**; digest+attestation retained for 7 y | Aggressively minimized (§6.4). |
| OPERATIONAL | 7 d | 30–90 d | Not evidence; rolling deletion. |

- **G7-R5 (MUST)** The integrity skeleton (`content_hash`/`prev_hash`/`chain_seq`/checkpoints) is retained for the **longest** TTL of any class in the chain. You may crypto-shred a PII field at year 2 but the *event* (its hash, its place in the chain) survives to year 7 so the chain still verifies end-to-end (§5.4).
- **G7-R6 (MUST)** Retention is **never** implemented as "the field is gone." It is implemented as either (a) crypto-shred (key destroyed, ciphertext + digest remain), or (b) the field's CAS blob is garbage-collected *only after* all events referencing its digest have themselves crossed their TTL **and** no legal hold applies — and the digest reference remains in the event (the event records "this blob existed and hashed to X, it has aged out").

### 4.2 Hot / warm / cold / archive tiering
| Tier | Backing | Latency | Contents | Transition trigger |
|---|---|---|---|---|
| **Hot** | Index DB + object-store recent segments | ms | events ≤ N days; live query, active replay, open DSRs | on ingest |
| **Warm** | Object store, indexed | sub-second | events N days–M months; analytics, retrospective sweeps | age ≥ N d |
| **Cold** | Object store IA / nearline | seconds–minutes | events M months–first-year; periodic audit pulls | age ≥ M mo |
| **Archive** | Deep archive (Glacier-class) + offline signed Merkle roots | minutes–hours | events > 1 y; legal hold / regulator request only | age ≥ 1 y |

- **G7-R7 (MUST)** A tier transition is **content-preserving and chain-preserving**: it moves bytes and updates the side-index `tier`, but never re-serializes or re-hashes the event. A restored archive object MUST recompute the identical `content_hash` (RFC 8785 JCS, per N-C2-300) and verify against the chain.
- **G7-R8 (MUST)** SENSITIVE-CONTENT raw bodies (agent RAG/prompt blobs) **do not promote past warm**; at the warm→cold boundary the raw blob is crypto-shredded/garbage-collected and only its `value_digest` + source attestation survive into cold/archive (§6.4). This caps the breach window for the highest-risk class.
- **G7-R9 (SHOULD)** Tier placement is residency-aware: an event tagged for an EU-resident tenant never tiers into a non-EU archive region. Constraint handed to G5 (§8).

### 4.3 Archival format — independently verifiable across the horizon (G4 long-lived-signature tie-in)
The threat: a 7-year archive outlives ed25519 trust assumptions, key rotations, and possibly algorithm deprecation. An auditor in year 6 must still verify a checkpoint signed in year 0 by a key rotated in year 2.

- **G7-R10 (MUST)** Archived segments are stored in a **self-describing, format-stable** container: `{ events.ndjson (canonical JCS), merkle.json (leaves+root), checkpoint-chain.json (the signed checkpoints covering this segment), key-history.json (the public-key + validity-window history needed to verify), classification_version, retention_manifest_version }`. Verification requires only the archive itself plus the published key history — no live platform service (auditor independence at the 7-year horizon, extends N-C2-304).
- **G7-R11 (MUST)** **Signature renewal / notarization (G4 hand-off):** before any signing key's validity window closes, G4 **re-notarizes** the affected checkpoint Merkle roots under the successor key (a long-lived timestamp/notary chain), so the chain of trust is continuous across rotations. G7 specifies *when* re-notarization is due (key-validity-window minus a safety margin, per archived segment); **G4 owns how** (key custody, notary, algorithm-agility to a successor signature scheme). Crypto-agility (e.g. migration to a PQ-safe scheme) is a re-notarization, not a re-hash — the events never change.
- **G7-R12 (SHOULD)** Archive integrity is **periodically re-proven**: a scheduled job pulls a sample of archived segments and verifies chain+Merkle+signature, so silent bit-rot or a broken notary chain is detected long before a regulator asks. (Couples to G3 restore-and-prove-chain.)

### 4.4 Legal hold
- **G7-R13 (MUST)** A **legal hold** is a signed predicate (`{ scope, control_id?, subject?, time_window, hold_id, authority }`) that **overrides every TTL and blocks every erasure**, including DSR crypto-shred, for matching data. Legal-hold check is a precondition of *every* tombstone op (§2.2); a tombstone whose `legal_hold_check` is not `clear` is rejected and the attempted erasure is itself logged.
- **G7-R14 (MUST)** Legal hold vs. erasure conflict resolution: when a DSR erasure targets data under legal hold, the platform **suspends** the crypto-shred (does not deny silently), records a `lifecycle.erasure_suspended` event, and surfaces it to the DPO/legal queue. GDPR Art. 17(3)(b)/(e) explicitly exempt erasure where retention is required for legal claims/obligations — the platform encodes this exemption rather than violating either rule. The data subject is owed a response stating the lawful basis for continued retention.
- **G7-R15 (MUST)** Releasing a legal hold re-enters matching data into normal lifecycle; any DSR that was suspended is automatically re-queued.

---

## 5. The core tension — right-to-erasure vs. an append-only tamper-evident log

### 5.1 The collision, stated precisely
- **C2 guarantees** (D-C2-04, N-C2-301/302): the store is append-only; events are *never* edited or deleted in place; each event's `content_hash` chains to the next via `prev_hash`; a `chain_seq` gap or a `prev_hash` mismatch is detectable tampering; signed Merkle checkpoints transitively attest to all history.
- **GDPR Art. 17 / CCPA §1798.105 guarantee** the data subject the right to have their personal data **erased**.
- **Naive erasure (delete the row / null the field) breaks C2**: it either (a) creates a `chain_seq` gap → looks identical to insider tampering → destroys the tamper-evidence property for *every* subject, or (b) changes the event body → `content_hash` no longer matches → the chain fails to verify → every downstream verdict, export, and signed checkpoint over that segment is invalidated.

**Neither requirement can simply win.** Deleting breaks the audit product (the entire reason to exist). Not deleting breaks the law. The resolution must satisfy *both*: erase the personal data **and** keep the chain verifiable.

### 5.2 Candidate mechanisms (decision matrix)

| Mechanism | How it erases | Chain integrity | Replay impact | Auditor "trace to subject" | Verdict |
|---|---|---|---|---|---|
| **Hard delete / row removal** | Physically remove event | ❌ `chain_seq` gap = looks like tampering; checkpoints over it fail | ❌ event gone | ❌ | **Rejected** — destroys the product's core property for all subjects. |
| **Field nulling in place** | Overwrite cleartext with null | ❌ `content_hash` changes → chain breaks | ❌ input gone, no record it existed | ❌ | **Rejected** — same chain break, plus undetectable mutation. |
| **Tombstoning only** (mark deleted, keep data) | Flag `deleted=true`, retain bytes | ✅ append-only honored | ✅ data still present | ✅ | **Rejected as erasure** — the personal data is *still stored*; it is not erasure, it is a soft-delete flag. Regulator non-compliant. (Used *in combination*, §5.3.) |
| **Pseudonymization at ingest** (replace `sub` with an opaque token, keep a token→identity map elsewhere) | Delete the map entry | ✅ event body unchanged | ⚠️ replay of identity-dependent policy works *only* via the map; deleting the map breaks identity-conditioned replay anyway | ❌ **Defeats the auditor's "trace this decision to a JWT subject"** — the whole D1/C2 identity-edge thesis. The map is also itself a re-identification honeypot and PII store. | **Rejected as the primary mechanism** — see §5.5. Used as a *complement* for PII-INDIRECT quasi-identifiers only. |
| **Crypto-shredding via per-subject field-level encryption** (encrypt PII fields under a per-subject DEK at ingest; erase = destroy the DEK) | Destroy the per-subject key | ✅ **ciphertext + `value_digest` unchanged → `content_hash` unchanged → chain verifies** | ⚠️ shredded input becomes unreadable → events that *needed* it for replay regress to `insufficient` (§5.6) — **but the event, its structure, and its place in the chain are fully preserved and the regression is labeled** | ✅ until shredded (un-redacted reveal is grant-gated, §6.3); after shred, the *fact* a subject existed and its `governance_transaction`/decision metadata remain, only the identifying value is irrecoverable | **CHOSEN (primary).** |

### 5.3 Decision — crypto-shredding + signed tombstone (the resolution)

> **D-G7-01 (the resolution): Right-to-erasure is satisfied by CRYPTO-SHREDDING — per-subject field-level encryption applied at ingest, where erasure = irreversible destruction of the per-subject Data-Encryption-Key (DEK) by G4 — recorded as a signed tombstone + key-destruction certificate in the same append-only chain. The audit log is never mutated; `content_hash`, `prev_hash`, `chain_seq`, and all checkpoint signatures remain valid because the chain is computed over the ciphertext envelope (and its `value_digest`), not the cleartext.**

Rationale, point by point:
1. **It actually erases.** Once the per-subject DEK is destroyed (NIST SP 800-88 cryptographic erasure), the ciphertext is computationally indistinguishable from random; the personal data is unrecoverable. This is recognized as erasure under GDPR (EDPB guidance treats cryptographic erasure with destroyed keys as effective deletion). It is *real* erasure, not tombstoning's soft-delete.
2. **It preserves the chain.** The hash chain was already designed (N-C2-303) to be computed over a redaction envelope whose `value_digest` is stable. We extend that exact seam: encryption is just a stronger redaction whose envelope is byte-stable under key destruction. **No `content_hash` changes, no `chain_seq` gap, no checkpoint re-signing.** The corpus's strongest property survives untouched.
3. **It is auditable.** The erasure is a signed `lifecycle.tombstone` event + a G4 key-destruction certificate, both in-chain. "We deleted user-123's PII on DSR-4471 at time T under authority A, legal-hold clear" is itself tamper-evident evidence — which is exactly what a regulator wants to see and exactly what a naive delete destroys.
4. **It composes with what already exists.** D4-OQ-4 already chose "hash-at-rest for `sub`/`email` with grant-gated reveal"; D2 already gates un-redacted reveal by scope; DT-57 already redacts on the *export copy*. G7 promotes "hash-at-rest" to "encrypt-at-rest-per-subject" so the same field can be *revealed* (grant-gated) **until** the subject exercises erasure, after which the DEK is gone and reveal is impossible by construction.

**Granularity (D-G7-02):** one DEK **per (subject, tenant)**, wrapping the PII-DIRECT fields of all that subject's events. Erasing one subject destroys exactly that subject's readability and touches no other subject's events or keys. (Per-event keys would be 10⁸× key-management cost for no erasure benefit; per-tenant keys would make single-subject erasure impossible. Per-subject is the correct grain — handed to G4 as the key-population requirement.)

### 5.4 Why the chain still verifies — worked through
Take event E with `jwt_claims.sub` stored as an envelope (§2.1). `content_hash(E) = sha256(JCS(E))` where E contains `{"sub": {"enc":"aes-256-gcm","key_ref":"…#v1","iv":"…","ct":"…","value_digest":"sha256:H"}}`. After crypto-shred, the **key behind `key_ref` is destroyed** but every byte of E — `ct`, `iv`, `value_digest`, structure — is **identical**. Therefore:
- `content_hash(E)` is unchanged → `prev_hash` of E+1 still matches → **chain intact**.
- The Merkle leaf for E is unchanged → the checkpoint root over E's segment is unchanged → **the year-0 signature still verifies in year 6** (no re-signing needed).
- `value_digest = sha256(salt‖cleartext)` survives, so a holder of an *independent* copy of the cleartext (e.g. the subject themselves, or a regulator who already has it) can still prove "this event concerned this value" — but **no one can recover the cleartext from the store**, because `ct` is now undecryptable. The salt prevents the digest itself from being a re-identification oracle (rainbow-table resistance).

### 5.5 Why NOT pseudonymization-at-ingest as the primary mechanism
Pseudonymization replaces `sub:user-123` with `sub:tok-9af3…` and keeps a token→identity map. The brief flags the failure precisely: **it defeats the auditor's "trace this decision back to its JWT subject."** The platform's differentiator is governance-to-enforcement *traceability with identity* (D1's single identity edge into C2). If the canonical identity is a pseudonym whose map can be (and on erasure *must* be) deleted, then:
- The auditor cannot answer "who deployed the unsigned image?" without the map — and post-erasure the map is gone, so the answer is permanently unavailable even for *non-erased* legitimate audit needs that share the map's lifecycle.
- The token→identity map is itself a concentrated PII re-identification store — a *second* breach magnet — and a single point whose compromise re-identifies everyone.
- Deleting one subject's map entry still leaves the question of whether the pseudonym appears in derived analytics, exports, RAG memory, etc.

Crypto-shredding keeps the **real** `sub` in the event (encrypted, grant-gated-revealable) so traceability is *full and faithful* for the entire lawful retention period, and erasure is a clean per-subject key destruction with no shared honeypot. **Pseudonymization is therefore demoted to a narrow complementary role:** PII-INDIRECT quasi-identifiers that are not the canonical identity edge (e.g. a source IP) MAY be pseudonymized at ingest where encryption is overkill, with the pseudonym-map under the same per-subject-erasure lifecycle.

### 5.6 Replay impact of erasure (the honest cost — answer the brief's regression)
**This is the real, unavoidable cost and we name it rather than hide it.** When a per-subject DEK is destroyed, any field that subject's PII contributed to a policy *input* becomes unreadable. If a policy consulted `jwt_claims` (e.g. namespace-scoping by `sub`/`groups`), then **after erasure that event can no longer be replayed to an authoritative `complete` verdict** — the input is gone. This is exactly the G1/E1/C2 "authoritative replay" thesis regressing, and META-ADVERSARIAL G-2/Risk-5 already noted "redaction is mandatory in exactly the regulated namespaces where replay is most valuable."

We resolve it honestly using C2's *own* honesty machinery rather than pretending it doesn't happen:

- **G7-R16 (MUST)** A crypto-shred that removes a replay-input field MUST cause the affected events to be **re-scored to `insufficient`** (or `best_effort` if the missing input is non-critical), with reason code **`erased_input:<field>`** (new, additive to C2 §5.5 vocabulary). The re-score is done via C2's existing re-normalization-appends-a-new-event mechanism (N-C2-105) — **the original event is not mutated**; a new `replay.completeness-rescore` event supersedes its label via the side index. The chain is untouched; the honesty label moves.
- **G7-R17 (MUST)** The platform MUST distinguish, in every replay-coverage report, between `insufficient:never_captured` (a *defect* — we failed to capture the input) and `insufficient:erased_input` (a *lawful* erasure — the input was captured and then destroyed on a valid DSR). The latter is **not** a coverage defect and MUST NOT count against the capture-quality SLO; it is reported as "lawfully erased, replay intentionally non-authoritative." A `best_effort` product wearing a `complete` label is the sin XD-6 exists to prevent; an *erased* event honestly labeled `insufficient:erased_input` is the *correct* behavior, not a failure.
- **G7-R18 (SHOULD)** Mitigation to bound the regression: capture, **at decision time and before any erasure**, a **non-PII derived replay-sufficiency record** for identity-conditioned policies — i.e. record *the boolean facts the policy actually computed from the PII* (`policy_input_facts: { "namespace_allowed_for_subject": true }`) rather than only the raw PII. This is itself REPLAY-CRITICAL-NONPII (no personal data, just the evaluated predicate result), so it **survives erasure** and lets the decision still replay to `complete` for the *policy logic* even after the subject's identity is shredded. This recovers authoritative replay for the common case (decision was a pure function of a now-erased identity) at the cost of pre-computing and storing the predicate outcomes. **Decision D-G7-03: derive-and-store policy-input facts so erasure degrades replay gracefully rather than catastrophically.** (Hands B1 a requirement: emit the evaluated input-fact set in decision evidence. Couples to N-C2-EDV value-capture.)
- **G7-R19 (MUST)** Erasure MUST cascade to **all derived stores**: materialized replay datasets (C2 §8.5 / D2-R8) that embed the subject's PII are either re-materialized post-shred or invalidated; F4 agent-memory entries (`agent.prompt`, RAG memory) for the subject are crypto-shredded on the same DEK; C3/C5 aggregates that cached cleartext are flushed. A DSR is not complete until the cascade is proven (the DSR pipeline §7 verifies it).

### 5.7 What survives erasure (so the audit product still works)
After a subject's crypto-shred, these remain and keep the platform's value intact:
- The **event itself**, its chain position, `content_hash`, and all checkpoint signatures (integrity unbroken).
- All **REPLAY-CRITICAL-NONPII** and **INTEGRITY-METADATA**: `disposition`, `obligations`, `policy_version`, `control_id`, `resource_id` (if non-PII), `correlation_id`, `governance_transaction_id`, `timestamp`, derived policy-input facts (G7-R18).
- The **fact that an erasure occurred** (signed tombstone + key-destruction cert), which is itself audit evidence.

So "what was decided, under which policy, with what outcome, in which flow" is **fully and permanently auditable**; only "*which named human* the decision was about" becomes irrecoverable — which is exactly the GDPR-mandated outcome.

---

## 6. PII handling

### 6.1 Classification of the high-risk fields (concrete)
| Field | Class | At-rest treatment | Reveal gate |
|---|---|---|---|
| `subject.sub`, `subject.email` | PII-DIRECT | per-subject envelope (§2.1) | D2 scope + grant (§6.3) |
| `jwt_claims.sub`/`email`/`preferred_username` | PII-DIRECT | per-subject envelope | D2 scope + grant |
| `jwt_claims` (other consulted claims) | PII-DIRECT or REPLAY-CRITICAL-NONPII per claim | claims the *policy consulted* and that are personal → encrypted; non-personal consulted claims → cleartext (replay needs them) | per-claim allowlist (§6.2) |
| `before_state` / `after_state` | SENSITIVE-CONTENT if secret/PII-bearing; else REPLAY-CRITICAL-NONPII | per-tenant envelope if sensitive | D2 scope |
| `request_object` (K8s admission) | mostly REPLAY-CRITICAL-NONPII; PII sub-fields (annotations with emails) encrypted | sub-field classified | D2 scope |
| **F4 `request_object` prompt + conversation** | **PII-DIRECT** | **per-subject envelope, warm-tier-max (§6.4)** | strict grant + short TTL |
| **F4 `request_object` RAG docs + tool args** | **SENSITIVE-CONTENT** | **per-tenant envelope, warm-tier-max** | strict grant + short TTL |

### 6.2 Redaction vs encryption-at-rest (when to use which)
- **Encryption-at-rest (per-subject/per-tenant envelope)** is the **storage** mechanism for everything classified PII/SENSITIVE: the cleartext is never at rest. This is what enables crypto-shred (§5).
- **Redaction** is an **export/projection** mechanism (DT-57): when producing an evidence bundle for an external party, fields are replaced with `"<redacted>"` on the *export copy*, with `original_hash`/`redacted_hash` in the manifest. **The store is untouched; redaction never modifies the audit log.** (DT-57 is correct as written; G7 ratifies it.)
- **G7-R20 (MUST)** The two are layered: data is *encrypted* at rest and *redacted* on export. An external-share export of an *encrypted* field redacts the cleartext (decrypted under grant, then redacted in the projection) — it never ships ciphertext-plus-recoverable-key.
- **G7-R21 (MUST)** A field that has been crypto-shredded and is then included in an export is rendered as `"<erased>"` (distinct from `"<redacted>"`): redacted means "withheld from this recipient but exists"; erased means "permanently destroyed under a DSR." The distinction is material to a regulator.

### 6.3 Who can see un-redacted data (ties to D2)
- **G7-R22 (MUST)** Un-redacted / decrypted reveal of a PII-DIRECT field requires **all** of: (a) the caller's D2 scope predicate covers the event (D2-R3/R5), (b) a **role grant** for `pii:reveal` (a privilege strictly above `report:view`), and (c) the per-subject DEK still exists (not erased). All three; any miss → the field shows as envelope-redacted (`"<redacted>"`) or `"<erased>"`.
- **G7-R23 (MUST)** Every PII reveal emits a `pii_reveal` audit event (`subject_revealed, revealed_by, justification, correlation_id`) into the chain — reveal is itself audited, mirroring D2-R9 `boundary_crossing`. Mass reveal is rate-limited and alerts (anti-exfil).
- **G7-R24 (MUST)** No single role can both (a) reveal PII and (b) destroy a DEK without a second actor / approval — separation of duties on the erasure path (a malicious admin must not be able to "erase to cover tracks"; an erasure is gated by the DSR pipeline §7, not an ad-hoc admin button). Couples to G4 signer-custody SoD.

### 6.4 The agent `request_object` containment regime (the breach-magnet fix)
F4's `request_object` (prompts, conversation, RAG content) is the densest PII/secret store on the platform and is written to the multi-year evidence log. Left naive, a single read of the cold archive leaks years of every user's prompts. G7 contains it:
- **G7-R25 (MUST)** Agent prompt/conversation content is **PII-DIRECT, per-subject-encrypted at ingest**; RAG docs/tool-args are **SENSITIVE-CONTENT, per-tenant-encrypted at ingest**. The raw assembled `request_object` is **never** stored as one opaque cleartext blob (G7-R3).
- **G7-R26 (MUST)** Raw agent-content bodies are **warm-tier-max (G7-R8)**: hot 7 d, warm to 90 d max, then the body is crypto-shredded/GC'd and **only the `value_digest` + source attestation + the policy decision about it survives** into cold/archive. The platform retains, for 7 years, the *evidence that a prompt was governed* (its decision, its hash, that it was/wasn't blocked for PII) — not the prompt text. This is the data-minimization win: long-term evidence value with bounded long-term breach exposure.
- **G7-R27 (MUST)** What the *governance decision* needs to replay (e.g. "the PII-detector flagged this prompt") is captured as a **derived non-PII fact** (G7-R18: `prompt_pii_flagged: true`, `tool_args_schema_valid: true`), so the agent admission decision still replays `complete` after the raw body is shredded. The decision is replayable; the prompt is not retained.
- **G7-R28 (MUST)** F4 agent-memory writes already enforce TTL and "deny memory writes with PII" (F4 §138); G7 makes that TTL and the per-subject DEK shared with the audit-log envelope, so a single DSR crypto-shred erases the subject across audit log *and* agent memory *and* RAG cache in one key destruction.

---

## 7. Data-Subject-Request (DSR) pipeline

End-to-end handling of access / erasure / rectification, itself fully audited.

- **G7-R29 (MUST)** A DSR is a first-class, signed, chained object (`{ dsr_id, type: access|erasure|rectification, subject_ref, received_at, lawful_basis_check, legal_hold_check, status }`). Its lifecycle (received → scoped → executed → verified → closed) is recorded as chain events.
- **G7-R30 (MUST) — Access (Art. 15 / SAR):** resolve all events for `subject_ref` via the per-subject DEK + indexes, decrypt under a SoD-gated DSR-fulfillment grant, and produce a redaction-aware bundle (reusing DT-57 export). Scope-filtered by the subject's own data only.
- **G7-R31 (MUST) — Erasure (Art. 17):** (1) legal-hold check (G7-R13/14) — if held, suspend + notify (G7-R14); (2) else instruct G4 to **destroy the subject's DEK** and issue a key-destruction certificate; (3) emit the signed `lifecycle.tombstone`; (4) re-score affected replay events (G7-R16/17); (5) cascade to derived stores (G7-R19); (6) **verify** the cascade (attempt decrypt → must fail; query derived stores → must be clear); (7) close with a signed completion attestation the platform can hand the subject. **The audit log is never deleted from; only keys are destroyed and tombstones appended.**
- **G7-R32 (MUST) — Rectification (Art. 16):** never an in-place edit (chain-breaking). A correction is a **new appended event** superseding the prior (N-C2-105 supersession), with the old event's PII left encrypted (and erasable later). The chain shows both the original and the correction — which is itself the auditable record of the rectification.
- **G7-R33 (MUST)** The DSR pipeline enforces SoD: the requester-verification, the legal-hold sign-off, and the DEK-destruction are not all performable by one principal (G7-R24).
- **G7-R34 (SHOULD)** DSR SLA timers (GDPR 1-month) are tracked and surfaced; legal-hold suspensions stop the erasure clock with a recorded lawful basis.

---

## 8. Data residency — hand-off contract to G5

G7 owns *classifying which data is residency-constrained*; G5 owns *placing the bytes and the keys*.

- **G7-R35 (MUST)** Every event carries a **residency tag** in the lifecycle side-index, derived at ingest from `scope.tenant`/`scope.region` and the tenant's residency policy (e.g. `eu-only`, `us-gov`, `unrestricted`). G7 computes the tag; G5 enforces placement.
- **G7-R36 (MUST)** The **per-subject/per-tenant DEKs (§5) are themselves residency-constrained**: an EU subject's DEK MUST be held in an EU key region (G4 + G5). Crypto-shred in-region. This is load-bearing: data-residency for crypto-sharded data is really *key*-residency — if the key crosses a border, so effectively does the readable data.
- **G7-R37 (MUST)** Tiering (§4.2) is residency-bounded (G7-R9): no event tiers/archives/replicates into a region its residency tag forbids. G7 hands G5 the per-event constraint; G5 owns the cross-region replication topology and the DR placement (with G3).
- **G7-R38 (SHOULD)** Cross-border evidence export (e.g. an EU tenant's events to a US regulator) goes through the DT-57 redaction + an explicit residency-transfer authorization (SCCs/adequacy basis recorded as a chained event). G7 provides the mechanism; the lawful-transfer basis is a policy decision.
- **G7-R39 (MUST)** The hand-off interface to G5 is the **residency-tag + key-region + tier-eligibility tuple** per event/blob; G5 MUST NOT place or replicate any object in violation of it, and a violation is a P0 (mirrors D2 scope-escape severity).

---

## 9. Decisions (decide-document-continue)

| ID | Decision | Rationale |
|---|---|---|
| **D-G7-01** | **Crypto-shredding** (per-subject field-level encryption; erasure = DEK destruction) + signed tombstone is the erasure-vs-immutability resolution. | Only mechanism that *actually erases* AND keeps `content_hash`/chain/checkpoints valid (chain is over ciphertext+digest, never cleartext). Extends C2's existing redaction-envelope seam (N-C2-303). |
| **D-G7-02** | **One DEK per (subject, tenant).** | Single-subject erasure isolation; per-event = unaffordable key count, per-tenant = can't erase one subject. Handed to G4 as the key-population requirement. |
| **D-G7-03** | **Derive-and-store non-PII policy-input facts at decision time** so identity-conditioned decisions still replay `complete` after erasure. | Bounds the §5.6 replay regression; the predicate result is REPLAY-CRITICAL-NONPII and survives the shred. Hands B1/B5 a value-capture requirement. |
| **D-G7-04** | **Pseudonymization is a *complement* (PII-INDIRECT only), not the primary mechanism.** | Pseudonymizing the canonical identity defeats the auditor's trace-to-subject and creates a re-identification-map honeypot (§5.5). |
| **D-G7-05** | **Agent `request_object` raw bodies are warm-tier-max; only digests + decisions survive long-term** (G7-R26). | Caps the breach window on the densest PII/secret payload while preserving 7-y evidence that a prompt *was governed*. |
| **D-G7-06** | **Erasure is itself a signed, in-chain event; the log is never deleted from.** | A traceless delete is indistinguishable from tampering; the erasure record is evidence a regulator wants. |
| **D-G7-07** | **Encrypt at rest, redact on export — two layers, store untouched by either erasure or redaction.** | DT-57 redaction is an export-copy projection; crypto-shred is key destruction; neither mutates the chain. |
| **D-G7-08** | **Erased ≠ redacted ≠ never-captured.** Three distinct labels (`<erased>`, `<redacted>`, `insufficient:never_captured`). | Lawful erasure must not be mistaken for a capture defect (preserves capture-quality SLO honesty, anti-XD-6). |

## 10. Open questions (with decided defaults)
- **OQ-1: DEK wrapping topology?** *Default:* per-subject DEK wrapped by a per-tenant KEK wrapped by a platform root in HSM/KMS (envelope encryption); G4 owns the hierarchy. Destroying the DEK is sufficient for shred; KEK/root are for tenant-/platform-wide kill.
- **OQ-2: Re-notarization cadence for long-lived signatures?** *Default:* re-notarize archived checkpoint roots at key-validity-window minus 90 d; G4 owns the notary/timestamp chain and crypto-agility to a successor (incl. PQ) scheme.
- **OQ-3: Production PII-DIRECT TTL floor before forced minimization?** *Default:* min(legal-retention-floor, data-minimization-ceiling); per-tenant, never below the legal floor, DSR can shred earlier. G2 prices it.
- **OQ-4: Does crypto-shred satisfy *every* regulator, or do some demand physical deletion?** *Default:* crypto-shred (NIST 800-88 cryptographic erasure, EDPB-aligned) is the platform standard; for regimes mandating physical destruction, a scheduled hard-GC of the *ciphertext blob* (not the event skeleton) after shred is offered as an add-on — the event/hash/chain still survive (the ciphertext is already unrecoverable, GC is belt-and-suspenders). Flagged for legal review per deployment.
- **OQ-5: Backup/restore interaction (G3)?** *Default:* backups store **ciphertext only**; a DEK destroyed in primary is destroyed in backup by construction (the key is not in the data backup) — crypto-shred is automatically honored across restores **iff** the keystore backup excludes destroyed keys. G7 hands G3/G4 this invariant: **never back up a destroyed DEK.**

## 11. Dependencies
- **Consumes:** C2 (append-only chain, redaction envelope N-C2-303, CAS §8.4, re-normalization N-C2-105, reason-code vocabulary §5.5); D2 (scope predicates, visibility, DT-57 redaction-export); D4 (SEC-5..8 integrity, OQ-4 hash-at-rest); F4 (`request_object` agent content, agent-memory TTL); **G4** (per-subject DEK lifecycle, key-destruction certificates, long-lived signature re-notarization, crypto-agility); B1/B5 (derived policy-input-fact capture, D-G7-03).
- **Consumed by / hands off to:** **G2** (TTL/tier table → cost model); **G5** (residency-tag + key-region + tier-eligibility tuple, G7-R35..39); **G3** (don't-back-up-destroyed-DEK invariant, restore-and-prove-chain after tier moves); C3/C5 (erased-vs-defect distinction in coverage reports, G7-R17).
- **Cross-domain contract surface:** the data-class taxonomy (§3), the crypto-envelope shape (§2.1), the `erased_input`/`<erased>` labels, the DSR pipeline (§7), and the G5 residency tuple (§8).
