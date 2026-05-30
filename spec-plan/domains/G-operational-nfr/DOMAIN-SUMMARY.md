# Domain G — Operational / Non-Functional Requirements (NFR) — SUMMARY

**Scope:** G1 scale · G2 cost · G3 availability/DR · G4 key-management · G5 multi-tenancy · G6 day-2 ops · G7 data-lifecycle/privacy · G8 Rego-authoring. Domain G is the **operational architecture the functional corpus (A–F) lacked** — authored to close the seven "no owner" gaps and the ten new risks the independent architecture-review-board pass named (`META-ADVERSARIAL-SECOND-OPINION.md` §3–4). The functional corpus is "a *functional* architecture with no *operational* architecture"; Domain G is that operational architecture.

---

## 1. The through-lines (the patterns that recur across all 8 components)

### 1.1 Every G component ends up imposing a contract change on C2 (and often D2)
This is the single most important fact about Domain G. **C2 — the tamper-evident evidence keystone — appears as a contract-change target in 7 of the 8 components**, and **D2 (scope/storage) in the two most structural (G5, G7).** Each NFR was *discovered* to be a cross-cutting contract, not a self-contained component:

- **G1** needs C2's per-source chain *sharded* by `(source.system, cluster)` (the single chain is the #1 scale ceiling) + a cross-shard roll-up checkpoint.
- **G3** needs a new C2 `infrastructure_degraded` disposition + `chain.restore_boundary` / `fork_reconciliation` event types.
- **G4** needs C2's `key_id` to resolve through an append-only Key Transparency Log the verifier can consult years later.
- **G5** needs C2's chain re-architected *global→per-tenant* (so deletion has no cross-tenant collateral) + a global chain-head meta-log.
- **G6** needs a scoped `disposition_context` and owns the C2 1.0→1.0-rc *chain-epoch* migration.
- **G7** needs new completeness sub-states (`insufficient:erased_input`, `complete:memoized_post_erasure`) and inline PII moved out-of-line to CAS so GC is chain-safe.
- **G8** needs B1's `__control_id__`/metadata generation reconciled and the B1-R30 conformance suite (less C2, more B1).

**The recurring "this NFR is really a cross-cutting contract" pattern** is the domain's defining hazard: each fix is correct, but it lands on a *frozen* C2/D2 the NFR component does not own. If these are filed as G-backlog, they never reach the owning SPEC — the exact META M-1 propagation failure ("the flag sits in the index but the load-bearing doc still says X"). **Domain G is only real if Wave-3 reconciliation pushes each mandate into the owning component's SPEC.** Five of these are *correctness-class* and must fold into the already-open C2 `v1.0-rc` pass (see §5 and `DOMAIN-ADVERSARIAL.md`).

### 1.2 The cost-cliff = differentiator finding (G2 — the single most important sentence in the NFR wave)
G2's model gets the *shape* right: **compute-bound at POC, storage-bound at regulated scale, and the differentiator — full-population, replay-capable, value-capture audit × 7-yr retention — is *itself* the cost cliff.** The model's own cheapest cure (lever 1: capture the heavy replay-complete event for only 5–15% of namespaces) **deletes full-population replay = the product.** So the cost cliff and the moat are the *same decision*. This must be surfaced to product as a first-order strategic constraint, not buried in a cheaper-architecture appendix. Corollary: build-vs-buy comes out *buy* and the honest re-costing (the 8 G-domain components alone are +10–15 eng-years the build column omitted) *widens* the buy advantage from ~4.7× to ~6–7×.

### 1.3 The managed-service tier as the honest answer to un-runnability (G6)
META Risk #10 — "14 services is un-runnable by a 2–4 person team (Jess)" — is not dischargeable by G6 alone. A ≤2-pages/shift *budget* is a target, not a mechanism, and can be met by the dangerous failure mode (under-alerting a silently-broken chain) as easily as the safe one. The honest conclusion the SPEC circles: **for sub-floor teams, the managed tier is the real answer** — but only if it means a *customer-controlled KMS/HSM the vendor process calls but cannot exfiltrate* and *customer-owned storage*, with the residual "vendor compute touches plaintext in memory" stated as a bounded trust surface, not zero. Self-hosted-below-N-SREs should carry an explicit "not recommended" warning.

### 1.4 The four crux resolutions (the mechanisms Domain G is built on)
- **Chain-epoch (G6):** the C2 schema migration is *impossible in place* on an append-only hash chain. Resolution: never rewrite history; seal Epoch 0 with a final signed checkpoint, open a re-normalized Epoch 1, cross-sign the old root into the new genesis. (Refinement: the seal must *refuse over a pre-existing break*, and the seal is **per-source-chain rolling**, not a single global instant — spokes buffer asynchronously.)
- **Crypto-shred (G5/G7):** erasure-vs-immutability is resolved by per-`(subject, tenant)` field-level encryption keys; destroying the DEK erases the subject's PII while leaving the event/`content_hash`/chain intact. (Refinement: must cover *all* derived stores + backups + KMS escrow, or the signed deletion certificate is a false attestation — the platform's own cardinal sin at the deletion layer.)
- **KTL (G4):** rotation breaks historical verification unless `key_id`-resolves-key against an append-only Key Transparency Log where public keys are *never removed*. This is the spine that lets a 2-year-old signed package still verify after the key rotated. (Refinement: the KTL is now a *second* system-of-record whose loss makes *all* history unverifiable — it has no DR owner and must get one from G3.)
- **Circuit-breaker / `infrastructure_degraded` (G3):** composed fail-closed defaults across B1/B2/B3/B5 = a correlated fleet-wide mass-deny with no breaker, that can even prevent recovery. Resolution: a per-criticality breaker that downgrades enforcement under shared-dependency degradation while keeping C-CRITICAL hard-deny, with a distinct audited `infrastructure_degraded` disposition ("the platform is down" ≠ "policy says no"). (Refinement: the breaker is a remotely-triggerable kill switch unless misclassification fails safe and the degraded window is absolutely bounded.)

### 1.5 The honesty discipline recurs as the integrity tripwire
The platform's "declared vs verified" sin (XD-6) re-appears at *every* NFR layer: a `full` cost lever that is `best_effort` for 90% of traffic (G2-A1), a `complete`-looking memoized replay after erasure (G7-A1), a "platform-wide" attack view that silently excludes residency-opted-out tenants (G5-A9), a signed deletion certificate that didn't actually erase (G5-A2), an epoch seal that launders a broken chain into a "verified" root (G6-D1), an `erased_input` label that launders a genuine capture defect (G7-A5). The domain's uniform rule: **a label must mean what it says, and partial coverage must surface its own denominator.**

---

## 2. Internal dependencies (the G-to-G seams)

| From → To | Contract |
|---|---|
| G1 → G2/G3/G5/G6 | the *paper* capacity model (events/day, KB/event, append-eps, dedup) feeds cost (G2), backup-window sizing (G3), per-tenant quota (G5), SLI targets (G6). Consumable *before* G1's load tests finish — downstream NFR is not blocked on G1's measurement tail. |
| G3 ↔ G4 | restore-verify needs G4 keys; if both share a failure domain they die together → **public verification keys embedded in the WORM backup** (self-contained verify). G4's KTL needs G3 to give it a DR/RPO contract. |
| G4 → G7 | G7's per-`(subject,tenant)` DEK *grain* is a contract G4 must agree to (G7 currently hands it to G4 as a fait accompli — must be co-designed). |
| G5 → G7 | G5's tenant offboard *invokes* G7's erasure machinery; per-tenant chain (G5) and crypto-shred (G7) interact at backups/derived stores. |
| G5 → G1/G3/G4/G7 | G5 is a *tenancy contract* that configures G1's quota, G3's per-tenant RPO, G4's per-tenant keys, G7's per-tenant erasure. |
| G6 → G1/G3/G4/G5 | day-2 runbooks reference these as hand-off contracts; G6 must turn pointers into *named MUST-requirements on the other component* or the runbook's highest-severity branch dangles. |
| G7/G2 ← G1 | the millions-of-DEK population (G7) and per-event-signing/verify (G2) need G1's hot-path latency + G2's cost model — currently unsized. |

**The seam hazard:** these were authored in parallel as empty-directory siblings; several G components write MUSTs against *other G components that did not yet exist at authoring time* (G6→G3/G4/G1/G5; G7→G4; G8→G3/E1). The contracts must be ratified, not assumed.

---

## 3. The 5 hardest decisions in Domain G

1. **Shard the C2 hash chain (G1) — buys scale, costs a tamper-evidence hole.** A single per-source chain serializes (fsync caps at hundreds/append/s; even in-memory ~5–20k/s) and is overwhelmed at 100× multi-cluster ingest. Sharding by `(source.system, cluster)` removes the ceiling but trades the single global ordering for N independent chains — an insider who drops a whole shard leaves the others valid. *Resolution:* shard **plus** a periodic cross-shard **roll-up super-checkpoint** signing the set of shard roots (C2-owned), so whole-shard deletion stays detectable. This is a C2 contract change C2 has not accepted → C2-rc.

2. **RPO=0 for the regulated buyer vs the latency-vs-durability trilemma (G3).** Any lost decision event is a lost piece of the auditor's evidence; "we honestly disclosed a 60-second gap" is not evidence a control operated. So the SPEC's *default* (P1/P2 async, RPO≤60s) is **wrong for the actual buyer.** But true RPO=0 requires committing the edge buffer to ≥2 failure domains *before* returning the admission decision — coupling admission latency to cross-domain replication on the ≤2s hot path. *Resolution:* mandate RPO=0 (P3/sync-D0) as the **default for the regulated profile**, state the irreducible floor = commit latency, and confront (not imply-away) the "fast local buffer / RPO=0 / survive correlated node+region loss — pick two" trilemma.

3. **T1 cannot protect the raw evidence log to a regulated standard (G5).** Row-level security makes a forgotten predicate fail closed — *for the relational read-model*. But the system of record is the object-store log, and object stores have no RLS; at T1 the raw evidence falls back to a per-request scoped-credential minter — **exactly the app-layer choke-point the board condemned**, just relocated. *Resolution:* "regulated ⇒ T2 (per-bucket) minimum" is a **correctness requirement, not sales positioning**; T1 is sold only where a raw-evidence cross-tenant leak is tolerable.

4. **The signed deletion certificate is a regulatory liability if crypto-shred is incomplete (G5/G7).** Crypto-shred assumes the tenant/subject key encrypts *all* at-rest evidence and no plaintext escaped — but derived analytics views, the relational read-model, G3 backups taken before destruction, and KMS key-escrow/backups can each leave a readable residue. A certificate asserting erasure that didn't happen is a false compliance attestation. *Resolution:* key *every* derived store under the same key; define the backup-erasure interaction with G3; KMS destruction must cover escrow/backups; **the certificate scopes its claim to what was actually destroyed** ("primary + read-model erased; backups expire by <date>"). Plus: physical-GC fallback only stays chain-safe if erasable PII is *out-of-line in CAS* — inline PII GC breaks `content_hash`.

5. **The platform thesis may require that humans largely *not* author Rego directly (G8).** Every G8 mechanism (lint, sim, templates, review, metrics) is an admission that direct Rego authoring is dangerous; when the whole toolchain exists to stop the author from doing what the platform asks, the abstraction is wrong. The M9 "author escape rate" near-zero comes from the *gates*, not author competence — a *containment* claim, not a *competence* claim. *Resolution:* either (a) narrow direct Rego authoring to the expert team and make everything else generated/template-and-fill with no raw-Rego novice affordance, or (b) keep broad authoring but market "we catch your Rego mistakes," not "anyone can write Rego." What G8 must not do is claim competence while building a six-layer apparatus that proves the opposite.

---

## 4. Consolidated open questions (decided defaults; revisit at GA)

| # | Question | Default / direction | Owner |
|---|---|---|---|
| G-OQ-1 | eval:recorded-event ratio (ingest ~580/s vs ~100k/s — a 170× swing; compliance may force "record all") | explicit knob; defended assumption; "record denials + sampled allows" default, "record all" sizes differently | G1 + C2 + G7 |
| G-OQ-2 | Single durable chain append ceiling | measure first; if per-append fsync caps at 100s/s, the ALT streaming/sharded chain is mandatory not optional | G1 |
| G-OQ-3 | CAS dedup + compression multipliers (load-bearing, asserted not measured) | best/expected/worst sensitivity bands; quote a range, not a point | G2 + G1 |
| G-OQ-4 | Recurring audit/exam thaw reserve (mis-labeled "one-time") | add a ~$100–300k/yr line at Scale C; price request fees + cross-region egress | G2 |
| G-OQ-5 | Default DR profile set before G2 cost trade is visible | don't finalize default pre-G2; regulated default = P3 (RPO=0) | G3 + G2 |
| G-OQ-6 | KTL durability / DR owner (loss = ALL history unverifiable) | give KTL its own RPO ≥ the evidence store; embed per-package trust slice | G4 + G3 |
| G-OQ-7 | POC tamper-evidence is self-referential (platform signs+timestamps+publishes root) | label POC as "outsider/mistake-grade, not malicious-operator-grade", or pull a minimal external anchor + operator-independent root channel into POC | G4 + D4 |
| G-OQ-8 | Per-subject identity at ingest often absent (agent third-party PII, `unknown`-disposition events) | per-(content, embedded-subject) erasure path for agent content; or scope the GDPR claim honestly | G7 + F4 |
| G-OQ-9 | Crypto-shred's legal status as "deletion" is unsettled across DPAs | document the regime-dependent risk; ensure physical-GC fallback is chain-safe (out-of-line PII) | G7 |
| G-OQ-10 | Restore can un-erase (destroyed DEK in last night's backup) | "on restore, re-apply the tombstone log to re-destroy any key whose tombstone post-dates the backup" | G7 + G3 |
| G-OQ-11 | Legitimately multi-tenant principals (MSP/consultant) under T2+ separation | per-tenant credential + tenant-context-switch model; unspecified today | G5 + D1 |
| G-OQ-12 | Managed-tier integrity boundary | customer-controlled KMS/HSM + customer-owned storage; acknowledge residual plaintext-in-memory trust | G6 + G4 |
| G-OQ-13 | B1-R30 cross-engine conformance suite (the portability proof) is unbuilt; Kyverno isn't Rego-portable | make B1-R30 a blocking prerequisite; scope portability to Rego-executing engines; Kyverno controls are re-authored | G8 + B1 + B4 |
| G-OQ-14 | NS-authoring "most-restrictive-wins" combinator is asserted, owned by no component | pin the B-layer combinator, or restrict NS authoring to additive denials in a disjoint control namespace | G8 + B-layer |

---

## 5. What Domain G hands to the rest of the platform (and what it asks back)

- **To C2 (the keystone):** five *correctness-class* contract changes that must fold into the already-open **`v1.0-rc`** pass — per-source chain *sharding* (G1), `infrastructure_degraded` + `chain.restore_boundary` (G3), per-tenant chain (G5), `insufficient:erased_input` completeness sub-state (G7), `key_id`/KTL resolution (G4). These are the domain's headline asks and are itemized in `DOMAIN-ADVERSARIAL.md`.
- **To D2:** RLS-mandatory, no-direct-store-handle enforcement (architectural + lint), inline-PII-to-CAS migration.
- **To D4:** signer custody (KMS/HSM, IAM split, signer-invoke identity distinct from human cluster-admins) promoted from SHOULD to **MUST@POC**.
- **To B1/B4:** the B1-R30 conformance suite as a *blocking prerequisite*; `__control_id__` generation reconciled with B1-R1; a policy-cost linter at authoring.
- **To E1:** the mandatory differential-sim gate as the load-bearing authoring guardrail; sampling/materialized-stratified-sample to keep fleet-wide replay (the differentiator) affordable at scale.
- **To F2:** day-2 (the gap F2 left at install-time), region spokes for residency, a plugin `/metrics` golden-signal SPI.
- **To product/business:** the cost-cliff = differentiator strategic constraint, the build-vs-buy = *buy* conclusion (now ~6–7×), and the managed-tier-for-sub-floor-teams recommendation.

**The domain's structural risk is singular and repeated:** like the WHERE clause G5 replaces, the *enforcement* of Domain G's own boundaries currently rests on lints, abstractions-by-convention, GA-deferred controls, and contracts written against components that do not yet exist. Domain G succeeds only if Wave-3 propagates each NFR mandate into the owning SPEC and ratifies the C2-rc asks.
