# C2 — ADVERSARIAL REVIEW (red-team)

**Target:** C2 SPEC v1.0 (frozen schema) + PLAN + ALT.
**Reviewer persona:** hostile auditor + insider-threat modeler + production-SRE who has seen audit pipelines lie. Goal: find where this schema will mislead an auditor, lose evidence, or be gamed.

The C2 schema is the contract the whole domain stands on. If it is wrong, every downstream "compliance" claim inherits the flaw. Below: assumptions attacked, gaps, abuse cases, cross-component contradictions, and a prioritized defect list.

---

## 1. Attacks on the core premise

### A1. "Replay-capable" is only as honest as the policy-dependency catalog — and the catalog can be wrong or stale.
Completeness scoring (`complete`) depends entirely on the policy-dependency catalog (SPEC §4.2/D-C2-06): "we captured everything the engine used" is judged against "what the catalog says the policy needed." **If the catalog is stale or wrong, an event is labeled `complete` while actually missing an input the policy really consulted.** The Rego changed to read a new external-data provider, the catalog wasn't regenerated, and now you have *confidently wrong* `complete` events — the worst failure mode, because `complete` is treated as authoritative and excluded from scrutiny. The SPEC's honesty tenet is undermined by a single stale dependency.
**Severity: CRITICAL.** *Demand:* the catalog version MUST be pinned **per `policy_version`** and bound into the event (or `content_hash`), and a mismatch between the bundle's actual external-data reads (observed at runtime) and the catalog MUST downgrade to `best_effort` with a `catalog_stale` reason. Trusting the catalog blindly is a hole.

### A2. The completeness scorer is the single point of truth and it runs twice (live + replay) — divergence is silent.
SPEC §5 requires the live scorer and the E1/C4 replay-time scorer to "agree." PLAN T-DET-2 tests this with golden vectors, but **golden vectors only cover cases someone thought of.** In production the two code paths *will* drift (different library versions, different catalog snapshots). When they disagree, which wins? The SPEC doesn't say. An event stored `complete` by the live scorer but scored `insufficient` at replay time is an unhandled contradiction that lands in an auditor's lap.
**Severity: HIGH.** *Demand:* define a tie-break: the **more conservative** label wins (`insufficient` > `best_effort` > `complete`), the divergence is itself recorded as a finding, and the catalog version that produced each label is captured.

### A3. `correlation_id = Admission Review UID` assumes the UID is trustworthy and unique. It may be neither.
DT-28/D-C2-03 anchor everything on the Admission Review UID. But (a) the UID is client/apiserver-generated and could collide across clusters in a federated store (HL-20 dedups on `correlation_id` — a cross-cluster collision would *merge two unrelated admissions*); (b) a malicious actor who controls the admission request could *spoof* a UID to make their event collide with a benign one. The SPEC scopes nothing into the correlation key.
**Severity: HIGH.** *Demand:* `correlation_id` for K8s flows MUST be namespaced by cluster (`<cluster>:k8s-admrev:<uid>`), and the federated dedup (HL-20) MUST dedup on the cluster-scoped id, never the bare UID. Document the spoofing risk; the UID is an integrity input, not just a join key.

## 2. Tamper-evidence is weaker than it looks

### A4. Hash chaining proves *internal* consistency, not *completeness*. An insider who never writes the embarrassing event leaves no gap.
The HL-06 scenario *is* the attack: Gatekeeper is disabled, the deployment slips through, **no Gatekeeper event is ever emitted.** The hash chain is perfectly intact — there's nothing missing *from the chain* because the event never entered it. The chain detects edits/deletes of events that *were* recorded; it does nothing about events that were *never* recorded. The SPEC's tamper-evidence section (§7) could mislead a reader into thinking the chain guarantees completeness. It guarantees no such thing — completeness comes only from **cross-source correlation** (the K8s-audit event exists even when Gatekeeper's doesn't), which is a C3/C4 concern, not a C2 integrity guarantee.
**Severity: HIGH (framing/expectation defect).** *Demand:* SPEC §7 must explicitly state the chain proves *integrity of recorded events, not completeness of the record*, and that completeness depends on independent sources (K8s API audit) plus C3/C4 correlation. Otherwise the integrity story oversells.

### A5. Signed Merkle checkpoints depend on a signing key whose compromise is catastrophic and unmodeled.
If the platform signing key is compromised, an attacker can edit history *and re-sign every checkpoint* — total defeat. The SPEC says "platform signing key (§23)" and stops. No key rotation, no HSM requirement, no detached external timestamp anchoring (e.g. RFC 3161 TSA or a public transparency log) that would survive key compromise.
**Severity: HIGH.** *Demand:* checkpoints SHOULD be additionally anchored to an external time-stamping authority or transparency log so that even with key compromise, *backdating* is bounded by an independent clock. Defer the HSM but not the external anchor decision.

### A6. Redaction-aware hashing (§7.5) leaks via the salt and is a tamper vector.
"Salted hash so a cleartext holder can prove a match" — but who holds the salt? If the salt is per-deployment-static, the hashes are dictionary-attackable for low-entropy claims (a `tenant` claim from a known set). If the salt is per-event-random, the "prove a match" property breaks unless the salt is stored — and a stored salt is itself redactable, reopening the leak. The scheme is under-specified.
**Severity: MEDIUM.** *Demand:* specify a keyed MAC (HMAC with a redaction key held separately from the signing key) rather than "salted hash," and state explicitly that the verifier needs the redaction key, not just cleartext.

## 3. Schema gaps and conditional-field traps

### A7. The R/C/O distinction makes "valid event" and "useful event" diverge dangerously.
Only 14 fields are required. `control_id`, `policy_version`, `external_data_refs`, `jwt_claims` are all **conditional**. A producer can emit a technically-valid event with none of them and the schema validator passes. The SPEC leans on "required when the condition holds," but the *condition* is judged by the same producer that might be cutting corners. A lazy/compromised adapter emits skeletal events that pass validation but are useless for replay — and `replay_completeness` would catch *some* of this, but not a missing `control_id` (which just silently produces "ungoverned enforcement" noise) or a missing `policy_version` on a non-decision event.
**Severity: MEDIUM.** *Demand:* the no-field-drop test (T-SCH-1) must be **per-source-and-event-type** with explicit required-field profiles (e.g. "a Gatekeeper `policy.decision` MUST have control_id, policy_version, policy_ref, external_data_refs-or-explicit-empty"), not just the global 14. Conditional-required is only as strong as the per-type profile enforcing it.

### A8. `external_data_refs: []` is ambiguous: "policy used no external data" vs. "we failed to capture it."
An empty array means both "this policy genuinely consults nothing external" and "we dropped it" (the DT-25 bug). The completeness scorer disambiguates *using the catalog* — but per A1 the catalog can be wrong. Without the catalog, empty is indistinguishable, and the scorer's only safe move is `best_effort`, which the SPEC half-acknowledges. This ambiguity is load-bearing for DT-25 and it rests on the catalog being right.
**Severity: MEDIUM.** *Demand:* distinguish explicitly: `external_data_refs: []` MUST mean "policy consults no external data (per catalog vX)"; a capture failure MUST be `external_data_refs: null` + reason code, never `[]`. Empty-array-as-failure is a footgun.

### A9. `before_state`/`after_state`/`request_object` stored "canonical or by reference" — but who guarantees the referenced blob still exists at replay time?
SPEC §8.4 allows large objects to be content-addressed blobs. If the CAS blob is GC'd or its retention lapses before the replay window ends, an event that was `complete` at write time becomes effectively `insufficient` at replay time — but its stored label still says `complete`. The label and reality diverge over time, silently.
**Severity: HIGH.** *Demand:* `complete` is a claim about *current* recoverability; either (a) blobs referenced by `complete` events MUST be retention-locked to the event's retention, or (b) the replay path MUST re-verify blob existence and downgrade the *effective* label, surfacing the divergence. The SPEC must pick one; "stored once, trusted forever" is wrong.

## 4. Cross-component contradictions

### A10. The `partial` vs `best_effort` rename (D-C2-01) contradicts the authoritative spec text and the scenarios.
The spec (§13.3) and **the detailed scenarios themselves** use `replay_completeness=partial` (DT-30, DT-42 explicitly: "carry `replay_completeness=partial`"). This SPEC renames it `best_effort` and demotes `partial` to a deprecated alias. **Every downstream component (C3/C4/C5/E1) and every scenario success-criterion that literally asserts `=partial` now depends on the alias being honored everywhere.** If one consumer checks `== "best_effort"` and a producer (or a scenario test) emits `"partial"`, the join silently fails. This is a self-inflicted cross-domain hazard introduced by the rename.
**Severity: HIGH.** *Demand:* either (a) revert to `partial` to match the authoritative spec and scenarios and avoid the alias-everywhere burden, or (b) make the alias normalization mandatory **at ingest** (always rewrite `partial`→`best_effort` before storage) so no consumer ever sees `partial`, and add a conformance test that no stored event carries `partial`. Pick (b) and enforce it, or the rename is a net new bug surface. *(Recommend (b) with strict ingest normalization.)*

### A11. C2 claims to own the "export integrity envelope" but C1 (Privateer) and C5 (reporting) also produce signed packages — overlapping ownership.
DT-24 has *Privateer* (C1) export a signed package; DT-46/DT-78 have *reporting* (C5) export signed packages; SPEC §7.6 says **C2** owns the integrity envelope. Three components touch signed exports. If each implements its own signing, you get three incompatible "signed package" formats and an auditor who can't verify uniformly.
**Severity: MEDIUM.** *Demand (cross-component):* C2 owns the **integrity primitive** (Merkle + sign + manifest format); C1/C5 own **content assembly** and call C2's primitive. The domain index must state this division explicitly or it will be re-implemented three ways.

### A12. Coverage feed (§10.4) needs the "in-scope expectation set" from A1/inventory — a dependency C2 doesn't control and can't validate.
`installed_no_events` vs `not_installed` vs `n/a` classification (DT-33/DT-80) requires knowing which (namespace × control) cells are *expected*. C2 supplies only the *observed* side. If A1's governance store or the K8s inventory is wrong/stale, coverage reports are wrong, and C2 has no way to detect it. A namespace that should be in-scope but is missing from the inventory shows as `n/a` (silently uncovered) — the exact DT-33 failure, but now invisible.
**Severity: MEDIUM.** *Demand:* the coverage feed contract must record the *source and timestamp* of the expectation set, and C3 must alert on inventory staleness; "no expectation = n/a = silently fine" is a coverage-gap blind spot.

## 5. Abuse / gaming cases

- **A13.** A producer games `replay_completeness` by emitting `complete` with a fabricated `external_data_refs` digest that resolves to a value favorable to a benign verdict. Defense: the digest must be verifiable against the *retained raw provider response* (N-C2-402), and the verify endpoint must check digest↔value, not just presence. Otherwise `complete` is forgeable. **Severity: MEDIUM.**
- **A14.** Backdating via `timestamp`: `timestamp` is producer-supplied ("time of original action"). A producer can lie. Only `ingest_timestamp` is platform-controlled, and the checkpoint `signed_at` bounds insertion. The SPEC should state that **`timestamp` is producer-asserted and not integrity-guaranteed**; analytics that depend on ordering must consider `ingest_timestamp` and checkpoint bounds. **Severity: LOW/MEDIUM (documentation).**
- **A15.** Chain-of-custody for *raw* events: `source.raw_event_digest` binds the raw event, but the raw store itself (§8.3) is not described as append-only/chained. An insider edits the raw external-data response after the fact; the digest no longer matches and DT-25 backfill produces a *different* (favorable) result, undetected unless someone notices the digest mismatch. **Severity: MEDIUM.** *Demand:* raw store digests must be pinned at ingest and any later mismatch is a tamper finding.

## 6. Prioritized defect list

| # | Defect | Severity | Fix owner |
|---|---|---|---|
| D1 | Stale/wrong policy-dependency catalog → confidently-wrong `complete` (A1) | **CRITICAL** | C2 §4.2: pin catalog version per policy_version; runtime mismatch → `best_effort:catalog_stale` |
| D2 | `partial`→`best_effort` rename hazard across consumers + scenarios (A10) | **HIGH** | C2 §3.7: mandatory ingest normalization + "no stored `partial`" conformance test |
| D3 | `complete` blob GC'd → label/reality divergence over time (A9) | **HIGH** | C2 §8.4: retention-lock blobs to event retention OR re-verify + downgrade at replay |
| D4 | Live vs replay scorer divergence undefined (A2) | **HIGH** | C2 §5: conservative-label-wins tie-break + record divergence + catalog version |
| D5 | Correlation-id collision/spoof in federated store (A3) | **HIGH** | C2 §6: cluster-namespace the id; dedup on scoped id (HL-20) |
| D6 | Chain proves integrity, not completeness — overselling (A4) | **HIGH** | C2 §7: explicit caveat; completeness = cross-source + C3/C4 |
| D7 | Signing-key compromise unmodeled (A5) | **HIGH** | C2 §7.4: external timestamp/transparency anchor + rotation plan |
| D8 | `external_data_refs: []` ambiguous (empty vs dropped) (A8) | **MEDIUM** | C2 §3.5/§5: `null`+reason for capture failure; `[]` only = genuinely none |
| D9 | Per-source/per-type required-field profiles missing (A7) | **MEDIUM** | C2 §3/PLAN T-SCH-1: typed required-field profiles |
| D10 | Export-envelope ownership overlaps C1/C5 (A11) | **MEDIUM** | Domain index: C2 owns primitive, C1/C5 own content |
| D11 | Coverage expectation-set staleness blind spot (A12) | **MEDIUM** | C2 §10.4: stamp expectation source+time; C3 alerts on staleness |
| D12 | `complete` forgeable via unverified external-data digest (A13) | **MEDIUM** | C2 §10.6: verify digest↔retained raw value |
| D13 | Redaction "salted hash" under-specified/leaky (A6) | **MEDIUM** | C2 §7.5: keyed HMAC + separate redaction key |
| D14 | Raw store not described as tamper-evident (A15) | **MEDIUM** | C2 §8.3: pin raw digests at ingest; mismatch = finding |
| D15 | `timestamp` producer-asserted, not guaranteed (A14) | **LOW** | C2 §3.1: document; analytics use ingest_timestamp + checkpoint bounds |

**Top-3 must-fix before any downstream codes against the contract:** D1 (catalog staleness corrupts the central `complete` claim), D2 (the rename will silently break joins/scenarios), D4 (scorer divergence has no defined resolution). All three are about the *honesty* of the completeness label — which is the entire value proposition.
