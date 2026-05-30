# G7 — Data Lifecycle, Retention & Privacy — ADVERSARIAL REVIEW

**Component ID:** G7 · **Role:** red-team / first-contact reviewer (not the SPEC author).
**Target:** `SPEC.md`, `PLAN.md` (this directory), and the upstream contracts they lean on (C2 1.0-rc, D2, F4, G4).
**One-line verdict:** The crypto-shred resolution is the *correct* shape and it genuinely preserves the chain — but the SPEC papers over how much of the platform's headline differentiator it quietly demolishes on erasure, leans on a G4 that may not deliver, and assumes a per-subject-identity-at-ingest that the platform frequently does not have. The decision-matrix conclusion is sound; several of its load-bearing premises are softer than stated.

---

## 0. The headline collision the whole corpus missed (and G7 now owns)

The brief is right that this is the deepest unexamined contradiction in the corpus: **"we keep a tamper-evident immutable record of everything"** (C2's reason to exist, D-C2-04, N-C2-301/302) is on a direct collision course with **"delete this user's data on request"** (GDPR Art. 17 / CCPA §1798.105), and *not a single one of the 23 functional components noticed*. C2 §8.1 even says "Deletion only at retention expiry, recorded as a signed tombstone" — it modeled *time-based* expiry and never once modeled *subject-requested* erasure mid-window. D2's DT-57 redaction is an *export-copy* projection that explicitly leaves the store untouched — so the corpus's only "privacy" mechanism doesn't erase anything from the system of record at all. **The corpus shipped a multi-year immutable PII lake with no erasure story and called it a compliance product.** G7 is the first place this is confronted. That alone justifies the component. The rest of this review attacks G7's *answer*.

---

## 1. CRITICAL findings

### A-1 (CRITICAL): Crypto-shred breaks authoritative replay, and the SPEC's mitigation is optimistic about coverage.
The SPEC honestly names the regression (§5.6, G7-R16) — good — but then leans on **D-G7-03 (derive-and-store non-PII policy-input facts)** to claim the *common* case still replays `complete`. This is doing a lot of work and it is shakier than presented:
- It requires **B1/B5 to pre-compute and persist the evaluated input-fact set for every identity-conditioned decision**, forever, *before* anyone knows which subjects will later request erasure. That is a new MUST on the busiest hot path (admission), it inflates every decision's evidence payload, and B1's SPEC does not yet own it. If B1 doesn't ship it, G7-R18 is vapor and erasure regresses replay to `insufficient` for *every* identity-conditioned policy — which META-ADVERSARIAL G-2 already says is "exactly the regulated namespaces where replay is most valuable."
- Even with D-G7-03, the input-fact (`namespace_allowed_for_subject: true`) records *that the policy said yes*, not *why* — it is a memoized boolean, not a re-executable input. An auditor who wants to *re-run* the policy against the (now-erased) identity to confirm the boolean cannot. So D-G7-03 yields a `complete`-*looking* replay that is actually "trust our cached answer" — which is uncomfortably close to the **declared-vs-verified sin (XD-6)** the platform exists to prevent. The SPEC should not let a D-G7-03-backed replay carry the unqualified `complete` label; it needs a `complete:memoized_post_erasure` sub-state, or it is integrity theater.

**Severity:** the brief's own framing — "crypto-shredding breaks replay … regressing the G1/E1/C2 thesis" — is *correct and under-mitigated*. The SPEC is honest that it happens but oversells how cleanly D-G7-03 saves it.

### A-2 (CRITICAL): The whole resolution assumes a per-subject identity exists at ingest. Often it does not.
D-G7-02 (one DEK per `(subject, tenant)`) and the entire crypto-envelope presume the normalizer can *attribute every PII-bearing field to a single identified subject at write time*. But:
- Many C2 events are **`unknown`-disposition / reconstructed / synthetic** (C4, DT-30) — a bypassed deployment with `jwt_claims_completeness=reconstructed`, or no subject at all. Which DEK encrypts the PII of an event whose subject is *itself uncertain*? The SPEC has no answer; it implicitly assumes clean attribution.
- An F4 agent `request_object` (a prompt + RAG docs) may contain **third parties' PII** — names, emails, account numbers of people who are *not the requesting subject* (a support agent's prompt about customer X). Crypto-shredding the *requesting subject's* DEK does **nothing** to erase customer X's PII embedded in that prompt. A DSR from customer X cannot be satisfied by per-subject keying on the agent-session subject. **The breach-magnet field is also the one the per-subject model can't cleanly erase.** This is a genuine hole, not a corner case — it is the *normal* shape of agent content.
- Group/shared identities (`type: service-account|ci-pipeline`) have no single human data subject; per-subject keying is undefined for them.

**The SPEC's per-subject crypto-shred is correct *when a clean single subject exists* and silent about the large fraction of events where it doesn't.** Needs: a per-(content, embedded-subject) erasure path for agent content, or an explicit scoping statement that agent prompts may carry un-erasable third-party PII (which then *fails* the very GDPR claim the component makes).

### A-3 (CRITICAL): G4 is load-bearing and external; if it slips, G7 ships a privacy component that cannot erase.
Every erasure guarantee terminates in **G4 destroying a DEK and issuing a key-destruction certificate**, and the long-lived archive depends on **G4 re-notarization / crypto-agility**. G4 is a *sibling, not-yet-written* component in the same parallel wave. The PLAN's mitigation (mock G4) lets G7's *tests* pass but ships **nothing legally usable** — a mock that returns a fake destruction cert does not destroy a key. If G4 lands late or chooses a key topology incompatible with per-(subject,tenant) DEKs (e.g. only per-tenant), **D-G7-02 is dead and single-subject erasure is impossible.** G7 has hard-coded a dependency on a key-management model G4 has not agreed to. This is the same "froze a contract before its consumer existed" process error META-ADVERSARIAL M-3 flagged — G7 is *producing* a key-grain contract and handing it to G4 as a fait accompli.

---

## 2. HIGH findings

### A-4 (HIGH): Pseudonymization is dismissed too fast; for some regulators crypto-shred is *not* deletion.
§5.5 demotes pseudonymization and OQ-4 hand-waves "some regimes demand physical deletion." But the legal status of crypto-shredding as "erasure" is **not settled** across regulators — some DPAs treat encrypted-with-destroyed-key data as still "stored" (the ciphertext exists; key destruction is a control, not deletion). If a regulator rules crypto-shred insufficient, G7's *entire* primary resolution collapses and the fallback (OQ-4's hard-GC of the ciphertext blob) **reintroduces the original collision** — GC'ing the ciphertext changes nothing in the *event* only if the PII was a CAS-referenced blob; if the PII was *inline* in the event body (e.g. `subject.sub`), GC'ing it changes `content_hash` and **breaks the chain after all.** The SPEC's claim that "the event/hash/chain still survive" under physical GC holds *only* for out-of-line CAS blobs, and §6.1 stores several PII fields *inline*. The fallback is not chain-safe for inline PII. Under-analyzed.

### A-5 (HIGH): "Erased ≠ never-captured" honesty label is correct but creates a new gaming surface.
G7-R17 says `insufficient:erased_input` MUST NOT count against the capture-quality SLO. Good intent — but this is now a **laundering channel**: an operator who wants to hide a genuine capture *defect* can mislabel it `erased_input` and it vanishes from the SLO. The label that protects lawful erasure also protects a liar. There is no requirement that an `erased_input` label be *backed by a verifiable DSR + key-destruction certificate*. Without that binding, the honesty distinction (D-G7-08) is unenforceable. Fix: `insufficient:erased_input` MUST reference a valid in-chain tombstone + key-destruction cert, or it is treated as `:never_captured` (defect).

### A-6 (HIGH): Rectification (G7-R32) leaves the *old* PII encrypted-but-present — is that erasure?
Art. 16 rectification appends a corrected event and leaves the original's PII "encrypted (and erasable later)." But the original (wrong) personal data is **still stored**, indefinitely, until a *separate* erasure. A subject who corrects their data has not asked for the old value to be *kept forever encrypted in the immutable log*. For inaccurate personal data, some regimes expect the inaccurate value to be *erased*, not archived. The SPEC conflates "we have a record that a correction happened" (legitimate audit need) with "we retain the inaccurate PII forever" (a data-minimization problem). Needs an explicit rule on whether rectification auto-shreds the superseded PII value.

### A-7 (HIGH): The legal-hold-vs-erasure path can become a permanent erasure denial.
G7-R14 suspends a DSR erasure under legal hold and notifies. But there is **no bound on hold duration** and no mechanism preventing a tenant from placing a *standing, broad* legal hold (`scope: tenant=*`) that suspends *every* erasure indefinitely — turning "legal hold" into a blanket erasure-defeat switch. GDPR Art. 17(3) exemptions are *narrow and specific*; a catch-all hold is itself a violation. The SPEC gives legal hold total override authority with no proportionality check, no expiry, and no audit of *over-broad* holds. The mechanism designed to lawfully *delay* erasure is also a mechanism to lawfully-looking *prevent* it forever.

### A-8 (HIGH): Cascade-to-derived-stores (G7-R19) is asserted but the derived-store inventory is open-ended.
"Erasure MUST cascade to all derived stores" — but the SPEC's list (materialized datasets, agent memory, RAG cache, C3/C5 aggregates) is **not closed**, and the corpus has *many* places cleartext PII can leak: C5 exports already delivered to regulators, SIEM/OCSF projections (N-C2-500), G6 day-2 telemetry/traces that may log `subject.sub`, backups (OQ-5 only covers keys, not cleartext snapshots taken *before* encryption was added), and any consumer that cached a decrypted reveal. **You cannot crypto-shred a PII value that was logged in cleartext in an observability trace.** The cascade is only as good as the guarantee that PII *never* exists in cleartext anywhere outside the envelope — and G7-R1's fail-safe doesn't retroactively fix data ingested before the registry classified a field. The cascade is a MUST over an unbounded, partly-out-of-G7's-control surface.

---

## 3. MEDIUM findings

### A-9 (MEDIUM): Per-(subject,tenant) DEK count and rotation are unsized and handed to G4 as someone else's problem.
At production scale (META-ADVERSARIAL: 10–100M events/day, millions of subjects) the DEK population is *millions of keys*, each needing storage, wrapping, rotation, residency-pinning (G7-R36), and destruction tracking. The SPEC says "handed to G4" but a key model this large has *cost and latency* implications on the **ingest hot path** (every PII write needs the subject's DEK) that G1 (scale) and G2 (cost) must price — and neither has. Envelope encryption (OQ-1) helps but adds a KEK-unwrap per write. Unquantified.

### A-10 (MEDIUM): Tier transitions claim "content-preserving, never re-hash" but archive crypto-agility (G7-R11) is a re-notarization that *adds* signatures — verify the auditor can still validate the *original* signature.
G7-R10/R11 are mostly right (re-notarize roots, don't re-hash events). But after a PQ migration, an archive segment carries *both* the original ed25519 checkpoint signature (whose key may be expired/revoked) and the successor notarization. An auditor in year 6 needs an unambiguous rule for *which* signature is authoritative and how to chain trust from a *revoked* original key through the notary to today. The SPEC says "G4 owns how" — but the *verification semantics the auditor follows* are a G7 evidence-contract concern, not purely G4's. Under-specified at the exact moment (revoked-original-key) that matters.

### A-10b (MEDIUM): "Don't back up a destroyed DEK" (OQ-5) races against the backup cadence.
If a DEK is destroyed at T but the keystore backup ran at T−1h, the destroyed key is **in last night's backup**. A restore re-animates it and *un-erases* the subject. OQ-5's invariant ("never back up a destroyed DEK") is necessary but not sufficient — it must also be **"on restore, re-apply the tombstone log to re-destroy any key whose tombstone post-dates the backup."** Otherwise crypto-shred is reversible by a restore, defeating the whole erasure guarantee. This is a concrete, exploitable un-erasure path the SPEC's OoQ-5 default does not close.

### A-11 (MEDIUM): SENSITIVE-CONTENT warm-tier-max GC (G7-R26) silently degrades agent-decision auditability.
GC'ing the raw agent prompt at 90 d while keeping the digest + decision is the right privacy call — but it means an auditor reviewing an agent decision at year 2 **cannot see what the prompt actually was**, only that it hashed to X and was/wasn't blocked. For a regulated AI-governance buyer who needs to *show the regulator the actual prompt that triggered a block*, "we kept the hash" is insufficient evidence. The privacy minimization (good) and the audit-completeness (the product's value) are in direct tension here and the SPEC resolves it entirely toward privacy without flagging that it weakens the AI-governance evidence story F4 is selling.

### A-12 (MEDIUM): No story for PII in `correlation_id` / `resource_id` / free-text `outcome_reason`.
The classification table treats `resource_id` as possibly PII-INDIRECT and `correlation_id` as opaque (DT-57 note). But `resource_id` like `.../Deployment/johns-personal-test` and free-text `outcome_reason` ("denied: user john.doe@x.com lacks claim") routinely embed PII in fields classified REPLAY-CRITICAL-NONPII or INTEGRITY-METADATA — which are **declared non-erasable**. PII leaks into the *un-erasable skeleton*. The SPEC has no scrubbing/detection for PII that lands in supposedly-non-PII structural fields, and those fields cannot be crypto-shredded by design (they're in `content_hash`). This is a real, common leakage path straight into the immutable core.

---

## 4. LOW / hardening

- **A-13 (LOW):** `value_digest` salting (§5.4) prevents rainbow-table re-identification, but the *salt* must be destroyed with the DEK or it's a partial oracle; the SPEC doesn't say where the salt lives or that it shares the DEK's fate.
- **A-14 (LOW):** `pii_reveal` rate-limiting (G7-R23) is named but no threshold; an attacker reveals slowly under the limit. Needs a budget + anomaly detection, not just a rate cap.
- **A-15 (LOW):** DSR SLA (G7-R34) is SHOULD; GDPR's 1-month is a hard legal deadline — for a compliance product this should be MUST with breach-alerting.
- **A-16 (LOW):** No mention of **erasure of a subject who appears in another subject's lineage** (approval chains: subject A's deploy approved by subject B; erasing B touches A's evidence). Cross-subject evidence entanglement is unaddressed.

---

## 5. Prioritized defect list

| # | Sev | Defect | Fix |
|---|---|---|---|
| **1** | CRIT | A-2: per-subject crypto-shred can't erase third-party PII embedded in agent `request_object`, nor events with `unknown`/reconstructed subjects | Add a per-(content,embedded-subject) erasure path for agent content; or explicitly scope the GDPR claim and warn that agent prompts may carry un-erasable third-party PII (which weakens the compliance claim). |
| **2** | CRIT | A-1: D-G7-03 input-fact mitigation oversold; memoized boolean ≠ re-executable replay; risks a `complete`-looking-but-unverifiable label | Make B1 input-fact capture a hard upstream MUST or drop the `complete`-after-erasure claim; introduce `complete:memoized_post_erasure` sub-state. |
| **3** | CRIT | A-3: G7 hands G4 a per-(subject,tenant) DEK grain G4 hasn't agreed to; mock G4 ships nothing legally usable | Co-design the key grain with G4 *now* (joint contract), not as a G7-imposed fait accompli; gate M-SHRED on real G4. |
| **4** | HIGH | A-4 + A-10b: crypto-shred's legal status as "deletion" is unsettled, and the physical-GC fallback breaks the chain for *inline* PII; restore can un-erase | Specify the inline-PII case explicitly (move all erasable PII out-of-line to CAS so GC is chain-safe); add "re-apply tombstones on restore" to OQ-5. |
| **5** | HIGH | A-5: `erased_input` label launders genuine capture defects | Require every `insufficient:erased_input` to reference a valid in-chain tombstone + key-destruction cert; else treat as `:never_captured`. |
| **6** | HIGH | A-7: unbounded/over-broad legal hold = permanent erasure-defeat switch | Bound hold scope + duration; audit over-broad holds; proportionality check; expiry + renewal. |
| **7** | HIGH | A-6: rectification retains inaccurate PII forever encrypted | Decide + specify whether rectification auto-shreds the superseded PII value. |
| **8** | HIGH | A-8: cascade-to-derived-stores is a MUST over an unbounded surface (traces, SIEM exports, pre-encryption backups, delivered exports) | Close the derived-store inventory; add a "PII never in cleartext outside the envelope" invariant enforced at G6/SIEM/backup boundaries; accept that already-delivered exports can't be cascaded and state it. |
| **9** | MED | A-12: PII leaks into non-erasable skeleton fields (`resource_id`, `outcome_reason`) | PII-detection/scrub on structural + free-text fields at ingest; forbid free-text PII in `outcome_reason`. |
| **10** | MED | A-11: warm-tier GC of agent prompts weakens the AI-governance evidence the product sells | Flag the privacy-vs-audit tension; let regulated tenants opt into longer encrypted retention of agent bodies (per-tenant TTL override, already in §4.1). |
| **11** | MED | A-9: millions-of-DEK population unsized on the ingest hot path | Hand G1/G2 a real key-count + per-write-unwrap-latency model, not just "G4's problem." |
| **12** | MED | A-10: auditor verification semantics through a revoked original key + notary chain under-specified | G7 owns the *verification rule* the auditor follows post-rotation/revocation; specify it. |
| **13** | LOW | A-13/14/15/16: salt fate, reveal budget, DSR-SLA-as-MUST, cross-subject lineage erasure | Hardening pass. |

**Bottom line:** the crypto-shred resolution is the right answer to the right question — and G7 deserves credit for being the first component to even *ask* the question the corpus structurally avoided. But the SPEC's confidence outruns three soft premises: (1) that a clean per-subject identity exists at ingest (often false, especially for the agent content that is the worst PII), (2) that D-G7-03 cheaply rescues replay (it memoizes rather than preserves), and (3) that G4 will deliver the exact key grain on time. Fix defects 1–4 before this is build-ready; the rest are real but schedulable.
