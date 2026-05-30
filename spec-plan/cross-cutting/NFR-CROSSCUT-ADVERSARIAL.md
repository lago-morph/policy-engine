# NFR Cross-Cut Adversarial Reconciliation — `NFR-CROSSCUT-ADVERSARIAL`

**Date:** 2026-05-30 · **Author role:** NFR cross-cut adversarial reconciler (hostile + integrative, final NFR pass over the G-domain).
**Inputs read in full:** all 8 `components/G*/ADVERSARIAL-REVIEW.md` (G1–G8) + skim of each `G*/SPEC.md`; `cross-cutting/CROSSCUT-ADVERSARIAL.md` (functional XD-1..XD-22 register); `cross-cutting/C2-v1.0-rc-RECONCILED.md` (the open audit-schema rc); `cross-cutting/META-ADVERSARIAL-SECOND-OPINION.md`.

**Mandate.** The functional pass (`CROSSCUT-ADVERSARIAL.md`) reconciled A–F. The G-domain (G1–G8) was authored *in parallel*, each component attacking its own numbers and flagging cross-component contracts it does not own. **Nobody has reconciled the G-components against each other or against the functional corpus.** That is this pass. The dominant failure mode is the one META M-1 named: each G-review correctly identifies that its fix lands on *another* component (most often C2), files it as its own backlog, and the contradiction between two G-components' demands on the same shared contract is never surfaced. This document surfaces them.

**One-line verdict.** The G-domain's individual engineering is strong, but **the eight components make mutually-unsatisfiable demands on three shared substrates — the C2 chain, the admission hot path, and the per-tenant key — and the C2-rc as currently written adopts only the *functional* NFR changes, not the *operational* ones (sharding, per-tenant chains, epoch boundaries, restore/fork events, crypto-shred reasons, KTL binding).** The chain-model changes **do compose, but only under one specific composite identity** that no single G-component states and the rc does not yet encode. See §3.

---

## 0. The structural finding that frames everything below

Every G-component independently concluded that **its fix is a change to C2's chain model**, and each treated C2 as a willing downstream:

- **G1** (XD-G1-4/5): shard the chain by `(source.system, cluster)` + add a cross-shard roll-up super-checkpoint.
- **G3** (A-1a, A-8c): add a `chain_epoch` baked into `chain_seq` (`(epoch, seq)`) gated by an external arbiter; add `chain.restore_boundary` + `fork_reconciliation` event types + `infrastructure_degraded` disposition.
- **G5** (G5-A3, A13): re-architect the chain from global to **per-tenant**, plus a global signed **chain-head meta-log** registry.
- **G6** (G6-D1/D3): seal **epoch boundaries** per-source-chain (rolling, never big-bang), refuse-to-seal-over-a-gap.
- **G7**: layer a per-field crypto-shred envelope inside the event, add `lifecycle.tombstone` events + `insufficient:erased_input` reason, never re-hash.
- **G4**: bind every signature to a `key_id` resolvable via an append-only **Key Transparency Log (KTL)** that must outlive the evidence.

**The C2-rc that exists (`C2-v1.0-rc-RECONCILED.md`) adopts NONE of these.** It landed XD-1, XD-2, XD-3, XD-8, XD-11, XD-12, XD-13 — the **functional** build-blocking subset. The chain model in the rc is still **per-source single chain, `chain_seq` is a bare monotonic integer, no epoch, no shard identity, no per-tenant identity, no roll-up checkpoint, no restore/fork event types, no tombstone/erasure reason wired into the chain, no KTL field on the signature.** The rc's own §6.6 field 34/35/36 (`prev_hash`/`chain_seq`/`signature`) are unchanged from the frozen v1.0. **So the G-domain produced six independent re-architectures of a contract that its own reconciliation pass already re-opened — and they were never folded in.** This is META M-1 repeating one level down, exactly as G3-A8c, G5-A13, G6-D4, and G7-A3 each predicted it would.

---

## 1. Ranked register of cross-G and G↔(A–F) contradictions

Severity: **C** = correctness (the NFR claim is *false as composed*, must resolve before any regulated go-live) · **H** = high · **M** = medium. "Composes?" = whether the two components' demands can both be true simultaneously without a third change.

| # | ID | Title | Components in tension | Sev | Composes? |
|---|---|---|---|---|---|
| 1 | **NX-1** | **The chain-model quadruple-bind: shard (G1) × per-tenant (G5) × epoch (G6) × `(epoch,seq)` arbiter (G3) — four independent re-architectures of one `chain_seq`, none reconciled, C2-rc encodes zero of them** | G1,G3,G5,G6 ↔ C2 | **C** | only under a composite identity §3 |
| 2 | **NX-2** | **RPO=0 (G3) vs lossless-ingest SLO (G6) vs async-chaining/batch-commit (G1): G1's batched group-commit loses ≤10ms of evidence on crash; G6 says drop-count MUST be 0; G3 says D0 RPO=0 — all three cannot hold on a fast local buffer** | G1,G3,G6 | **C** | NO — pick two (trilemma) |
| 3 | **NX-3** | **Crypto-shred (G7/G5) vs replay-completeness (G1/E1) vs lossless/completeness SLO (G6) vs split-chain deletion-detectability (G1-G5): erasure is mandatory, replay-`complete` is the differentiator, and "no evidence ever lost" is the SLO — erasure is deliberate evidence destruction that must NOT trip the loss alarms but MUST be distinguishable from tampering** | G7,G5,G1,G6,C2 ↔ E1 | **C** | only via `erased_input` reason wired into chain §3 |
| 4 | **NX-4** | **Per-tenant chain (G5) destroys platform-wide deletion-completeness (G1 insider-deletion threat); chain sharding (G1) destroys whole-source deletion-detectability — the two isolation/scale fixes each reintroduce the *other's* undetectable-deletion hole** | G1,G5 ↔ C2 §7.1 | **C** | only with a global roll-up meta-log §3 |
| 5 | **NX-5** | **KMS/KTL audit-root (G4) vs region-loss key availability (G3) vs key custody in same failure domain: G3's restore-and-verify needs G4's keys; if KTL/keystore shares the disaster, the restored chain is unverifiable and the deletion certs/tombstones are unprovable** | G4,G3 ↔ G5,G7 | **C** | NO without co-located public-key backup |
| 6 | **NX-6** | **The cost cliff does NOT compose: G2's cure (capture 15KB replay-complete events for only 5–15% of namespaces) DELETES the full-population replay differentiator; G5 hard-tenancy (per-bucket/per-cluster) multiplies storage; G3 active-active is a 2nd/3rd full copy; G4 value-capture (XD-1, now a MUST) and G7 per-(subject,tenant) DEKs both INCREASE the bytes G2 must afford** | G2,G5,G3,G4,G7,C2 ↔ E1 | **C** | NO — the dollars do not compose at Scale C |
| 7 | **NX-7** | **Admission hot-path budget is over-subscribed by stacked synchronous calls: G1 cosign external-data (deploy-storm cold-cache), G6 evidence-value-capture (now a MUST via XD-1/N-C2-EDV), D1 JWKS-on-miss — all on the ≤250ms (G6) / ≤2s (B5) webhook, with `failurePolicy: Fail` ⇒ deploy-storm bricks the fleet** | G1,G6 ↔ B2,B3,D1,C2 | **C** | NO without async/pre-warm split |
| 8 | **NX-8** | **The circuit breaker (G3 XD-7) is a remotely-triggerable enforcement kill-switch AND its safety rests on human criticality classification (G8 can't validate Rego competence) — the availability fix (don't mass-deny) is a security hole (DoS the dependency → enforcement off)** | G3,G8 ↔ B5,D4 | **H** | partially (default-deny-on-ambiguity) |
| 9 | **NX-9** | **Restore re-opens closed security state: G3 D1-snapshot restore re-animates revoked exceptions/denied approvals; G7 backup-restore re-animates a destroyed DEK (un-erases a subject) — restore-to-point-in-time of *security/erasure* state is a rollback of deliberate decisions** | G3,G7 ↔ D3,G5 | **H** | only if D0 log authoritative over D1 snapshot |
| 10 | **NX-10** | **Reconciliation (G3 edge-buffer/fork replay) is an event-injection channel for fabricated history; per-tenant chains (G5) + sharding (G1) + epoch rolls (G6) each ADD legitimate "append old/odd events out of order" paths that mask injection** | G3,G1,G5,G6 ↔ C2 | **H** | only with pre-disaster spoke-buffer signatures |
| 11 | **NX-11** | **The eval:recorded-event ratio (G1 XD-G1-12) swings ingest 170× and is unstated; "record every decision" (a compliance demand C2/G7 may force) blows G1 sizing, G2 cost, G6 ingest SLO, G3 RPO-0 quorum throughput simultaneously** | G1,G2,G6,G3,C2,G7 | **C** | NO until the ratio is a stated knob |
| 12 | **NX-12** | **Day-2 un-runnability (G6 ≤2 pages/shift over 14 services) vs everything G1–G7 adds: sharded chains, per-tenant chains+keys (millions of DEKs G7-A9), epoch rolls, DR drills, circuit breakers, KTL — each new mechanism is new on-call surface; the operational budget is defined-away, not engineered-down** | G6 ↔ G1,G3,G4,G5,G7 | **H** | NO — managed-tier is the honest answer |
| 13 | **NX-13** | **Portability thesis (G8) is unprovable where it matters: B1-R30 conformance suite unbuilt; Kyverno isn't Rego-portable; the G-domain's scale/cost/DR/key NFRs all assume the "one Rego decides everywhere" substrate G8 says is partially abandoned** | G8 ↔ B1,B4,G1,G2 | **H** | thesis must be scoped, not proven |
| 14 | **NX-14** | **Issuer→tenant binding (G5-A4) is the new identity keystone editable by the platform operator; combined with G4's single forge-anything trust-root (A-2) and in-cluster signer-invoke (A-3), a colluding operator can mint a tenant, mint a key, and forge that tenant's chain — three "insider" holes that compose into total compromise** | G5,G4 ↔ D1,D2 | **H** | needs operator-independent channels |
| 15 | **NX-15** | **G7 hands G4 a per-(subject,tenant) DEK grain G4 never agreed to; G5 hands G4 per-tenant keys; G3 needs region-available keys; G4's KMS-cost is unmodeled — four components specify G4's key topology and G4 reconciles none of them** | G7,G5,G3,G2 ↔ G4 | **H** | NO without a joint key-grain contract |
| 16 | **NX-16** | **Index DB / CQRS bottleneck (G1 XD-G1-10) vs G2's unpriced searchable-index growth (A-5, 25TB warm at Scale C) vs per-tenant index partitioning (G5) — the read-model is a co-equal scale bottleneck AND an unbudgeted cost AND must be tenant-scoped, three demands on one tier** | G1,G2,G5 | **H** | yes, but needs explicit CQRS split |
| 17 | **NX-17** | **Legal-hold (G7-A7) is an unbounded erasure-defeat switch that collides with G5 tenant-offboard crypto-shred and G2 retention economics — a standing `tenant=*` hold prevents every erasure AND every retention-expiry GC indefinitely** | G7,G5,G2 | **M** | yes with hold proportionality bound |
| 18 | **NX-18** | **Telemetry/evidence firewall (G6-D5/D7) vs every G-component's debug need: SRE debugging a slow shard/tenant/epoch must cross into tenant-isolated (G5) evidence; "audit every op" (G6 RUN-0) vs "no op noise in chain" (OBS-3) — the operability/integrity boundary is drawn four incompatible ways** | G6 ↔ G5,G7,C2 | **M** | yes with scoped break-glass |
| 19 | **NX-19** | **DR-drill false confidence (G3-A7): RTO grows with chain length (G1 scale), drills run at staging scale; the restore-and-prove-chain time is unproven at the production volume G2 prices and G1 sizes — the one DR number that matters is the one not tested** | G3,G1,G2 | **M** | yes (volume-tied drill cadence) |
| 20 | **NX-20** | **Behavioral/AI-judge tier (F4/XD-16) is non-deterministic + latency-heavy on the path G6 budgets at p99<250ms and G1 sizes for deterministic eval; the G-domain NFRs never model the async best-effort tier F4 needs** | F4 ↔ G1,G6,G3 | **M** | yes if tier is async-isolated |

---

## 2. Per-defect detail: tension, severity, reconciliation, owner

### NX-1 — The chain-model quadruple-bind · CRITICAL · the headline NFR finding

- **Tension.** Four G-components each re-architect `chain_seq`/chain identity, independently, against a C2-rc that encodes none of it:
  - G1 (XD-G1-4): chain identity = **shard** `(source.system, cluster)` (else one global writer absorbs 5,000 ev/s burst at 100× and ingest lag blows the C3 reconciliation window).
  - G5 (G5-A3/R26): chain identity = **tenant** (else erasing tenant A breaks tenant B's chain; a *shared* chain "makes deletion impossible").
  - G6 (G6-D3): chain identity carries an **epoch**, sealed per-source-chain on a rolling basis (else the 1.0→1.0-rc migration either rewrites history or strands late-arriving buffered events in the wrong epoch).
  - G3 (A-1a): `chain_seq` becomes **`(epoch, seq)`** where epoch is a fence-token from an external arbiter (else the default P2 async topology split-brains — two live quorums advance the *same* chain).
- **Impact.** Each is correct in isolation; **the C2-rc `chain_seq` is a bare integer** (`field 35: "Monotonic per-source sequence"`). A build team handed the rc + the four G-reviews has four different definitions of what a chain is and what `chain_seq` means. They are not independently bolt-on-able because they all redefine the *same two fields* (34/35) and the *same checkpoint semantics* (36).
- **Reconciliation.** They **do compose into one coherent model** — see §3 — but only if C2 adopts a single **composite chain identity** `(tenant, source.system, cluster, epoch)` with `chain_seq` monotonic *within that tuple*, plus a global roll-up. No single G-component states this; the C2-rc must.
- **Owner:** C2 (the rc must absorb the operational chain-model delta, exactly as it absorbed the functional one) + G1/G3/G5/G6 jointly ratify. **Class: correctness.**

### NX-2 — RPO=0 vs lossless-ingest SLO vs batch-commit · CRITICAL · NOT mutually satisfiable as written

- **Tension.** Three "never lose evidence" promises that fight on the commit path:
  - **G6** (Surface-2): `dropped_events` MUST be 0, hard, "a dropped audit event is a compliance incident."
  - **G3** (R-G3-RPO-1): D0 RPO=0 — an event is committed only after **write-majority quorum across ≥2 failure domains** before `chain_seq` advances; the edge buffer is "D0-pending" and must survive node loss.
  - **G1** (XD-G1-6/OQ-G1-2): to keep up at 100× burst, ingest uses **batched group-commit** — "the last ≤10 ms / ≤100 events are not durably chained when the worker crashes," and the gap-recorder may die with the data.
- **Impact.** G1's throughput fix **directly violates G3's RPO=0 and G6's drop-count-0** for the in-flight window. G3 itself concedes (A-3b) the irreducible floor: you cannot have RPO=0 for D0 *and* a fast local-only buffer *and* survive correlated node+region loss — **pick two.** G1 implies the fast buffer + batch; G3 implies cross-domain-replicate-before-ack; G6 forbids any loss. The three together are over-constrained.
- **Reconciliation.** Make the trilemma **an explicit, per-criticality dial**, not three independent MUSTs: (a) for D0-critical scopes, the buffer MUST replicate to ≥2 failure domains *before the admission decision returns* (couples admission latency to replication — feeds NX-7); (b) batch-commit is permitted **only** with an **out-of-band gap detector** (G1 XD-G1-6) that did not crash with the ingest worker, comparing `chain_seq` continuity against the source's own sequence (e.g. K8s audit seq); (c) any unrecoverable in-flight loss is the disclosed `evidence_lost_in_failover` reason (G3 R-G3-RS-6), which G6's SLO must **exempt from drop-count-0 the same way G7's `erased_input` is exempted** (NX-3) — a *disclosed, attested* loss is not a silent drop. The honesty tenet is the only thing that makes all three coexist: not "we never lost it" but "every loss is bounded, attested, and queryable."
- **Owner:** G3 (RPO contract) + G1 (batch + out-of-band detector) + G6 (SLO exemption for attested loss) + C2 (the attested-gap reason already exists). **Class: correctness.**

### NX-3 — Crypto-shred vs replay-completeness vs lossless SLO vs tamper-evidence · CRITICAL

- **Tension.** Four-way:
  - **G7/G5** (G5-R25, G7-§5): GDPR erasure is achieved by **destroying a DEK** — *deliberate, mandatory evidence destruction* mid-retention-window.
  - **G1/E1**: authoritative `complete` replay is the differentiator; G7-A1 shows crypto-shred regresses identity-conditioned replays to `insufficient` (the `complete:memoized_post_erasure` problem), and META G-2 notes that's "exactly the regulated namespaces where replay is most valuable."
  - **G6** (Surface-2): drop-count MUST be 0, chain gap ⇒ Sev-1 page.
  - **C2 §7.1 / G7-§5**: a `chain_seq` gap or `content_hash` change is **indistinguishable from insider tampering** and destroys tamper-evidence for *every* subject.
- **Impact.** Erasure must (a) not change `content_hash` (G7's "entire trick": shred the key, not the ciphertext — chain still verifies), (b) not trip G6's drop/gap alarms (it's lawful, not a drop), (c) be distinguishable from tampering (G7 tombstone), and (d) honestly down-label the now-unreplayable event **without** that down-label being a laundering channel for genuine capture defects (G7-A5: `insufficient:erased_input` must reference a valid tombstone + key-destruction cert or it's a `:never_captured` defect).
- **Reconciliation.** This **composes only if** the C2 chain model adopts G7's three additions: the `lifecycle.tombstone` event type, the `insufficient:erased_input` reason **bound to a tombstone + G4 key-destruction cert**, and the crypto-envelope (PII out-of-line in CAS so GC never touches `content_hash` — G7-A4's inline-PII hole must be closed). **The C2-rc does not have these.** G6's lossless SLO must add a fourth exemption class (alongside attested-failover-loss from NX-2): **lawful-erasure is not a drop.** And G1/E1 must accept that `complete` for an erased-input event becomes the `complete:memoized_post_erasure` sub-state, never plain `complete` — else it's the XD-6 declared-vs-verified sin at the erasure layer.
- **Owner:** C2 (chain additions) + G7 (envelope + reason + cert binding) + G6 (SLO exemption) + E1 (memoized sub-state). **Class: correctness.**

### NX-4 — Per-tenant chain and sharding each reintroduce the *other's* undetectable-deletion hole · CRITICAL

- **Tension.** The single global chain's one virtue is that **deleting any source is detectable as a gap** (C2 §7.1 insider threat). G1's sharding splits it into N chains by `(source, cluster)` — now **dropping a whole shard leaves the other shards' checkpoints valid** (XD-G1-5). G5's per-tenant split does the same per tenant — a malicious operator **drops a whole tenant's chain undetected** (G5-A3). Each fix solves its own problem (scale / deletion-collateral) by *creating the deletion hole the other was worried about*.
- **Impact.** The more you partition the chain (for scale, for tenancy, for erasure), the more the platform-wide "nothing was deleted" guarantee erodes — and the G-domain partitions it **four ways at once** (NX-1).
- **Reconciliation.** Both reviews independently prescribe the **same fix**: a **global, append-only, signed roll-up meta-log** that commits to the set of shard/tenant chain-heads. G1 calls it a "cross-shard super-checkpoint" (XD-G1-5); G5 calls it a "per-tenant chain-head registry … in a global, append-only, signed meta-log" (G5-A3). **They are the same object.** Define it once: a periodic super-checkpoint signs `{(chain_identity, epoch, head_seq, head_hash)}` for every existing chain, so *chain existence* is globally tamper-evident even though *content* is partitioned. This is the keystone that makes §3's composite identity safe.
- **Owner:** C2 (owns the integrity primitive, XD-18) + G1 + G5. **Class: correctness.**

### NX-5 — Key availability under region loss vs audit-root durability · CRITICAL

- **Tension.** G3's restore story (A-8a) needs G4's public keys to verify the restored chain; G4's whole edifice (A-1.2) rests on the KTL being durable — "KTL loss = total verification loss," strictly worse than losing evidence. G7's tombstones and G5's deletion certificates are unprovable without G4 keys. **If the keystore/KTL shares a failure domain with the evidence store, a region loss takes both** — restored chain unverifiable, erasure certs unverifiable, deletion-detectability gone.
- **Impact.** Every G-component's integrity claim terminates in a G4 key, and G4 (A-1.2) admits the KTL "has no DR owner." G3 assumes G4 recovers first but makes it a runbook hand-off, not a contract (G6-D4: dangling pointer at the highest-severity step).
- **Reconciliation.** (a) Embed the **public** verification key history *in* each WORM backup segment (G3-A8a + G7-R10's `key-history.json` already gestures at this — make it the load-bearing mechanism so verification is self-contained); (b) give the **KTL its own DR/RPO at least as strong as the evidence store** (G4-D-G4-1) — it is a second system-of-record; (c) make the G4↔G3 key-recovery-ordering a **named MUST contract**, not a pointer (G6-D4).
- **Owner:** G4 (KTL durability) + G3 (co-located public-key backup + ordering contract). **Class: correctness.**

### NX-6 — The dollars do not compose: the cost cliff is the differentiator, and four components inflate it · CRITICAL

- **Tension.** G2-A1 already found its own cheapest cure (capture the 15KB replay-complete event for only 5–15% of namespaces) **deletes full-population authoritative replay = the product**. Now stack the other G-demands on top of the *same* storage line:
  - **G4/XD-1 (N-C2-EDV):** external-data **value** capture is now a MUST for volatile providers — *more* bytes per event, on exactly the flagship events.
  - **G7:** per-(subject,tenant) crypto-envelope + CAS-out-of-line PII (NX-3) — *more* bytes + millions of DEKs (G7-A9, unsized).
  - **G5:** hard-tenancy (T2 per-bucket / T3 per-cluster) — storage no longer dedups/compresses across tenants; G2's 6× compression + 5× dedup (A-6, already optimistic) degrade further per-tenant.
  - **G3:** active-active sync (the *regulated* default per A-3a) is a **second or third full copy** of a 7-year-retained growing store.
  - **G2-A4/A5:** the thaw/exam reserve (~$100–300k/yr) and the searchable-index growth (~$3k+/mo, 25TB warm) are *already* unpriced.
- **Impact.** Every G-component's correctness fix **increases** the very storage cost whose own cheapest reduction *is abandoning the differentiator*. The dollars do not compose at Scale C: regulated TCO moves from G2's ~$490k/yr to ~$700–900k/yr *before* you add the per-tenant + active-active multipliers, which can each ~double the storage line.
- **Reconciliation.** This is a **first-order product/strategy decision, not an NFR knob** (G2-A1 P0): the cost cliff and the moat are the *same* decision. Surface it explicitly — either full-population replay is the product (and the regulated price reflects active-active × per-tenant × value-capture × 7yr) or it is sampled (and "authoritative replay" is honestly scoped to a population fraction, the XD-6 sin avoided by *disclosure*). The eval:recorded-event ratio (NX-11) is the single largest lever and must be decided first.
- **Owner:** G2 + product (strategic) + C2/G7 (the ratio + value-capture cost). **Class: correctness (strategic).**

### NX-7 — Admission hot path over-subscribed by stacked synchronous calls · CRITICAL

- **Tension.** G6 budgets the webhook at **p99<250ms** (decision must feel instant); B5 at ≤2s. On that path, three synchronous external calls now stack: G1's cosign external-data verify (XD-G1-1: 100s of ms cold, deploy-storm = 0% cache hit), G6's own evidence-**value** capture (G6-D6: now a MUST via XD-1/N-C2-EDV, "a cosign/registry round-trip alone can exceed 250 ms"), and D1's JWKS-fetch-on-miss (XD-G1-3). With `failurePolicy: Fail`, a deploy storm saturates the verifier and **bricks the fleet's deploys at the worst moment** — and NX-2's "replicate buffer before returning the decision" would add a *fourth* cross-domain hop.
- **Impact.** The platform's NFR (fast, safe admission) and its evidence model (capture the value synchronously, durably) are in direct tension on the one path that, when it fails, takes down the customer's deploys.
- **Reconciliation.** Both G1 and G6 independently prescribe the **same split**: the *decision* (allow/deny) is synchronous and fast; the *evidence enrichment* (external-data value capture, completeness scoring, durable chaining) is **async/out-of-band but lossless** (G6-D6 fix). Plus G1's **build-time cache-seeding** (XD-G1-1: B3 verifies at CI → seed the admission cache before the image ever reaches admission). The N-C2-EDV value MUST is satisfied in the ingest path, not the admission path. This is a new **B2↔B3↔C2 contract** the functional pass flagged (CROSSCUT G1↔B2/B3) but no one owns.
- **Owner:** B5 (flow split) + B2/B3 (cache seed) + C2 (value captured in ingest, not admission). **Class: correctness.**

### NX-8 — Circuit breaker: availability fix is a security kill-switch, safety rests on unvalidated human classification · HIGH

- **Tension.** XD-7 / G3 add an "infrastructure-degraded" mode so a degraded shared dependency drops to warn instead of mass-deny. G3-A4 shows the same mechanism is **a remotely-triggerable enforcement kill-switch** (DoS the cosign verifier → breaker opens → unsigned images admit-with-warning), and its only real defense (C-CRITICAL stays hard-deny) **rests entirely on correct human criticality classification** — which G8 shows the platform cannot validate (median authors ship semantically-wrong-but-lint-clean policies; criticality is a governance act G8 can't enforce).
- **Reconciliation.** **Default-deny-on-ambiguity** (G3-A4): any control bound to a regulated/safety Gemara category is C-CRITICAL-for-breaker-purposes *regardless* of namespace classification, so misclassification fails safe; absolute max-degraded-duration (bound the attacker's hold-open window); dual-control on manual *open* (symmetric with force-close); fast-path in-window re-eval for net-new degraded admits. Couples to the functional XD-7 (the "infrastructure-degraded mode" the functional pass already assigned to B5).
- **Owner:** G3 (breaker) + B5 (flow) + D4 (break-glass) + G8 (criticality-default). **Class: high (correctness-adjacent).**

### NX-9 — Restore re-opens deliberately-closed security/erasure state · HIGH

- **Tension.** G3-A5: restoring D1 (config) from a morning snapshot re-animates a **revoked exception** / re-fires a **denied approval** — a rollback of a security decision. G7-A10b: restoring the keystore from a backup taken *before* a DEK destruction **un-erases the subject** — crypto-shred is reversible by restore. Both are restore-to-point-in-time of state that encodes *deliberate, irreversible decisions*.
- **Reconciliation.** Make the **append-only D0 log authoritative over the D1/keystore snapshot** for both: (a) G3 — replay the security-decision log forward from the backup point; never restore a grant to a more-permissive state than D0 supports; (b) G7 — on restore, re-apply the tombstone log to re-destroy any key whose tombstone post-dates the backup (G7-A10b). One principle (D0 > snapshot) closes both holes.
- **Owner:** G3 (config restore) + G7 (key restore) + D3 (approval/exception state). **Class: high.**

### NX-10 — Reconciliation/partition paths are event-injection channels · HIGH

- **Tension.** G3-A6: edge-buffer + fork reconciliation **legitimately append old, out-of-order events** after a disruption — perfect cover to inject fabricated "recovered" history (idempotency dedupes *real* events, not *fake* ones). NX-1's partitioning makes this worse: per-tenant chains, shards, and epoch rolls each add *more* legitimate "append irregular events" paths, each a new injection surface.
- **Reconciliation.** G3-A6's fix generalizes: every recovered/reconciled event MUST be **covered by a pre-disaster spoke-buffer signature** whose `signed_at` predates the disaster (each edge buffer is a signed sealed mini-chain); an event "recovered" without such coverage is **rejected, not appended**. This must apply uniformly across all four partition mechanisms, not just region-failover.
- **Owner:** G3 (reconciliation) + C2 (integrity primitive) + G1/G5/G6 (apply to shard/tenant/epoch joins). **Class: high.**

### NX-11 — The eval:recorded-event ratio is the master variable and it's unstated · CRITICAL

- **Tension.** G1-XD-G1-12 names it the single biggest unstated assumption: if the platform records **every** decision (which a compliance auditor may *require* — "prove nothing was admitted unaudited"), ingest ≈ eval ≈ **100k ev/s at 100×**, not ~580/s — a **170× larger** problem. That single choice simultaneously: blows G1 sizing, multiplies G2 cost (NX-6), violates G6's ingest freshness SLO, and overwhelms G3's RPO-0 quorum-commit throughput (NX-2).
- **Reconciliation.** Make it a **stated, defended knob** (`record all decisions` vs `record denials + sampled allows`) decided **before** any sizing/cost/SLO number is quoted, because it moves all of them by two orders of magnitude — and the *compliance* requirement may force "record all," which the current G2 model cannot afford.
- **Owner:** C2 + G7 (compliance requirement) + G1 (sizing) + G2 (cost). **Class: correctness — gates the whole NFR layer.**

### NX-12 — Day-2 un-runnability vs everything G1–G7 adds · HIGH

- **Tension.** G6-D2: the ≤2-pages/shift budget over 14 services is a *target defined-away*, met as easily by under-alerting (silently broken chain) as by reliability. Meanwhile G1 adds sharded chains, G5 adds per-tenant chains + millions of DEKs, G3 adds epoch arbiters + DR drills + circuit breakers, G4 adds the KTL + rotation runbook, G7 adds the crypto-envelope + tombstones. **Every G-component's correctness fix is new on-call surface.** The operational budget is the one thing all of them spend and none of them credit.
- **Reconciliation.** G6-D2's honest answer: integrity alerts (chain-gap, ingest-drop) are **exempt from the page budget** (they always page) so the budget can't be met by silencing them; and **managed-tier is the real answer below N SREs** (G6-DEC-G6-2), with self-hosted-by-a-2-person-team carrying an explicit "not recommended" warning. The G-domain collectively proves the stack is **not runnable by its stated target operator** without the managed tier.
- **Owner:** G6 + every G-component (each must declare its on-call surface). **Class: high.**

### NX-13 — Portability thesis unprovable where the NFRs assume it · HIGH

- **Tension.** G8-AR-1/AR-4: "one Rego decides everywhere" is defended by an unbuilt conformance suite (B1-R30) and a heuristic lint, and **Kyverno isn't Rego-portable at all**. Every G-NFR (G1 eval sizing, G2 build-vs-buy, G4 signing) assumes the uniform-PDP substrate. If controls are re-authored per engine, the scale/cost models are computing the wrong denominator.
- **Reconciliation.** Scope the thesis honestly (G8-AR-1): portability guaranteed only across Rego-executing engines; Kyverno controls are re-authored; make B1-R30 a **blocking prerequisite**, not a co-built MUST. The G-NFRs inherit the scoped claim.
- **Owner:** B4 + B1 (conformance) + G8 (scope statement). **Class: high.**

### NX-14 — Three insider holes compose into total compromise · HIGH

- **Tension.** G5-A4: the issuer→tenant binding is editable by the platform operator (2-admin dual control, no detection). G4-A2: a single offline trust-root forges any evidence; its multi-channel publication has only one operator-independent channel, marked "ideally." G4-A3: at POC, a cluster-admin with signer-invoke rights drives K1 to sign fabrications (rate/context limits GA-deferred). **Composed: a colluding operator mints a tenant (G5), mints a key into the KTL (G4), and signs a forged chain for it (G4) — and it all verifies.** Each component rated its own hole survivable; together they are the exact insider threat (C2 §7.1) the product exists to counter.
- **Reconciliation.** Make ≥1 **operator-independent** publication channel a MUST (G4-D-G4-4); promote signer-invoke-identity-distinct-from-humans + rate/context limits to **MUST@POC** (G4-D-G4-3); make issuer-binding changes alertable security events requiring the *tenant's* sign-off (G5-A4); pull a minimal external anchor (public-log commitment of the KTL head) into POC (G4-D-G4-5). Until then, the POC's tamper-evidence is honestly **"outsider/mistake-grade, not malicious-operator-grade."**
- **Owner:** G4 + G5 + D1/D2. **Class: high.**

### NX-15 — Four components specify G4's key topology; G4 reconciles none · HIGH

- **Tension.** G7 demands per-(subject,tenant) DEKs (G7-A3); G5 demands per-tenant signing/encryption keys (G5-R3); G3 demands region-available keys (A-8a); G2 must price KMS ops at production volume (G4-A4/A8, unmodeled). G4's SPEC (A-3 review) is written around K1/K2 checkpoint keys, not a millions-of-DEK population. **G7-A3 nails it: G7 hands G4 "a key grain G4 has not agreed to" as a fait accompli** — META M-3 repeating.
- **Reconciliation.** A **joint key-grain contract** authored *now*, not imposed: G4 owns whether per-(subject,tenant) DEKs are feasible (envelope encryption / KEK-per-tenant + DEK-per-subject), the per-write unwrap latency on the ingest hot path (feeds NX-7/G1), and the KMS cost at volume (feeds G2). G7's `M-SHRED` gate cannot pass against a mock G4 (G7-A3).
- **Owner:** G4 (key model) + G7/G5 (grain requirements) + G2 (cost) + G1 (latency). **Class: high.**

### NX-16 — The read-model is a co-equal bottleneck, an unbudgeted cost, and must be tenant-scoped · HIGH

- **Tension.** G1-XD-G1-10: the 9-index OLTP event store is a co-equal (arguably worse) scale bottleneck under write-amplification + analytical-scan contention, needing a CQRS/columnar split. G2-A5: the searchable index over 128B events stays warm/hot (~25TB, ~$3k+/mo growing), unpriced. G5: that index must be tenant-partitioned (per-tenant RLS or per-bucket). Three demands on one tier that the functional XD-5 already flagged as bypassing the D2 scope interceptor for aggregate reads.
- **Reconciliation.** Mandate the **write-path/analytical-path split (CQRS)** G1 prescribes; price the index-growth line (G2-A5); route the analytical reads through the D2 scope-predicate library *and* tenant partition (functional XD-5 + G5). One architectural decision (split + scope) serves all three.
- **Owner:** G1 (CQRS) + G2 (cost line) + G5/D2 (scope). **Class: high.**

### NX-17 — Legal-hold is an unbounded erasure- and retention-defeat switch · MEDIUM

- **Tension.** G7-A7: an unbounded, over-broad legal hold (`scope: tenant=*`) suspends *every* erasure indefinitely (itself a GDPR violation — Art. 17(3) exemptions are narrow). It also collides with G5 tenant-offboard crypto-shred (a held tenant can't be erased) and G2 retention economics (held data can't GC at expiry — the cost line grows unbounded).
- **Reconciliation.** Bound hold scope + duration; proportionality check; audit over-broad holds; expiry + renewal (G7-A7). The hold is allowed to *delay* erasure, never to *prevent* it forever.
- **Owner:** G7 + G5 (offboard) + G2 (retention). **Class: medium.**

### NX-18 — The operability/integrity boundary is drawn four incompatible ways · MEDIUM

- **Tension.** G6-D5: "telemetry MUST NOT carry audit content" vs the SRE's need to debug a specific event's slow replay across tenant-isolated (G5) evidence. G6-D7: "audit every op" (RUN-0) vs "no op noise in chain" (OBS-3). The four rules draw the operability/integrity firewall in mutually-inconsistent places.
- **Reconciliation.** SRE evidence access is a **scoped, audited break-glass** that itself emits a C2 event, bounded by G5 tenancy, on labels/timings not payload by default (G6-D5). "Governed C2 event" is scoped to *enforcement/integrity-affecting* ops only (break-glass, key rotation, epoch transition, policy change), not routine scaling (G6-D7).
- **Owner:** G6 + G5 + C2. **Class: medium.**

### NX-19 — The one DR number that matters is the one not tested · MEDIUM

- **Tension.** G3-A7: restore RTO grows with chain length (G1 scale); drills run quarterly at staging scale; the prod-scale restore-and-prove-chain is explicitly *not* drilled. G2 prices and G1 sizes a volume the DR drill never restores.
- **Reconciliation.** Tie drill cadence to data-volume growth, not calendar; make restore-RTO a continuously *projected* metric (measured rate × current chain size); ≥1 prod-scale drill/year (G3-A7).
- **Owner:** G3 + G1 (scale) + G2 (the restored volume's cost). **Class: medium.**

### NX-20 — The behavioral/AI-judge tier breaks the NFRs that assume determinism · MEDIUM

- **Tension.** Functional XD-16: F4 puts non-deterministic, latency-heavy model-call evaluators inline. The G-NFRs all assume deterministic, fast eval — G6's p99<250ms, G1's eval sizing, G3's replay determinism. None models the async best-effort tier.
- **Reconciliation.** F4-ALT two-tier async architecture; the behavioral tier is C2-marked `best_effort` (analogous to N-C2-SYNTH) and never on the synchronous budgeted path. Inherits functional XD-16.
- **Owner:** F4 + G6 (separate SLO) + C2 (tier marker). **Class: medium.**

---

## 3. The consolidated C2 `v1.0-rc` ratification list — every NFR-driven chain/schema change, and whether they compose

The functional rc (`C2-v1.0-rc-RECONCILED.md`) landed the **functional** subset (XD-1/2/3/8/11/12/13). The **operational/NFR** subset below is **not in the rc** and must be added before re-freeze. Sources: the eight G-reviews.

### 3.1 The NFR-driven changes the rc pass must adopt

| # | Change | Driver | Touches |
|---|---|---|---|
| **N-1** | **Composite chain identity** `(tenant, source.system, cluster, epoch)`; `chain_seq` monotonic *within* that tuple. Replaces field 35's bare "per-source" identity. | G1 (shard), G5 (per-tenant), G3 (epoch), G6 (epoch roll) | fields 34/35 |
| **N-2** | **`chain_epoch`** as a first-class part of the sequence (`(epoch, seq)`), incremented on failover, gated by an external arbiter's epoch lease; revived old primary refuses to commit on stale epoch. | G3 A-1a | field 35 + new commit invariant |
| **N-3** | **Global roll-up super-checkpoint** event type: a periodic signed commitment over `{(chain_identity, epoch, head_seq, head_hash)}` for *every* existing chain — makes chain *existence* tamper-evident across all partitions. | G1 XD-G1-5, G5 A3 | new event type + checkpoint semantics |
| **N-4** | **`chain.restore_boundary`** + **`chain.fork_reconciliation`** event types; a discontinuity *with* a valid signed boundary (referencing the WORM backup digest + cross-checked vs buffers/replica) is legitimate, *without* is tamper. | G3 A-2, A-1b | new event types + verifier rule |
| **N-5** | **`infrastructure_degraded`** disposition (or `disposition_context`): "platform is down" ≠ "policy says no"; degraded admissions are labeled, not silently warn. | G3 XD-7, A-8c | field 37 `disposition` enum or context |
| **N-6** | **`lifecycle.tombstone`** event type + **`insufficient:erased_input`** reason, the reason **bound to a tombstone + G4 key-destruction cert** (else `:never_captured`). PII out-of-line in CAS so GC never alters `content_hash`. | G7 §5, A4, A5 | field 29 reasons + new event type + envelope |
| **N-7** | **`key_id` → KTL binding**: the signature's `key_id` (field 36) MUST resolve via the append-only Key Transparency Log; the per-package **trust slice + `revocation_freshness`** travels in signed exports; KTL gets its own DR/RPO. | G4 A-1, D-G4-1/2 | field 36 + KTL durability contract |
| **N-8** | **Reconciled-event provenance**: any event appended via a recovery/reconciliation/epoch-join path MUST carry a pre-disaster spoke-buffer signature (`signed_at` predates the disaster) or be rejected. | G3 A-6 | new ingest invariant |
| **N-9** | **Lossless-SLO exemption classes**: `evidence_lost_in_failover` (attested) and `erased_input` (lawful) are NOT "drops"; integrity alerts (chain-gap, drop) are budget-exempt and always page. | G6-D2, G3 R-G3-RS-6, G7 | SLO contract (G6) referencing C2 reasons |
| **N-10** | **Quorum-commit-before-`chain_seq`** + out-of-band gap detector for batch-commit: `chain_seq` advances only after write-majority across ≥2 failure domains; batch-commit permitted only with an independent continuity checker. | G3 R-G3-RPO-1/4, G1 XD-G1-6 | commit-path invariant |

### 3.2 Do the chain-model changes compose? — **Yes, but only under one composite identity that no single component states.**

This is the brief's crux question. The answer is **conditional yes**:

**They compose** because the four partition axes are *orthogonal dimensions of one identity*, not four competing identities:

```
chain identity  =  (tenant, source.system, cluster, epoch)
chain_seq       =  monotonic WITHIN that tuple
global integrity =  roll-up super-checkpoint over ALL {(identity, epoch, head)}
```

- **Sharding (G1)** = the `(source.system, cluster)` dimensions — for write parallelism.
- **Per-tenant (G5)** = the `tenant` dimension — for deletion-collateral isolation + crypto-shred.
- **Epoch (G6/G3)** = the `epoch` dimension — for schema migration (G6 rolling seal) *and* failover fencing (G3 `(epoch,seq)` arbiter). **These two uses of epoch are compatible**: a schema-migration seal and a failover-fence are both "increment epoch, start a fresh `seq` lineage, cross-sign the old head into the new genesis." One mechanism, two triggers.
- **Crypto-shred (G7)** rides *inside* an event in any chain without touching identity (it shreds a key, never `content_hash`) — orthogonal by construction.

**They compose ONLY IF three conditions hold — and none is in the current rc:**

1. **The global roll-up (N-3) is mandatory.** Without it, every added partition axis (tenant, shard) *subtracts* deletion-detectability (NX-4). The roll-up is the single object that makes partitioning safe. **It is the keystone of the composite model.**
2. **Epoch is reconciled to mean ONE thing** across G6 (migration) and G3 (failover). If G6 seals epoch-0→1 for the schema migration *while* G3 is incrementing epoch for a failover, two components mint epoch transitions with different semantics on the same chain. They must share **one epoch counter with one transition mechanism** (cross-sign old-head→new-genesis) and two documented triggers. The rolling per-source seal (G6-D3) and the per-source epoch fence (G3-A1a) are then the same rolling operation.
3. **`chain_seq` monotonicity is scoped to the full tuple, not "per-source."** The rc's field 35 says "Monotonic per-source sequence" — that is **under-specified for the composite model** and will be read four incompatible ways (NX-1). It must read: *monotonic within `(tenant, source.system, cluster, epoch)`; gaps within a tuple are tamper; the roll-up attests tuple existence.*

**Where they FIGHT (and the fight is resolvable):**

- **Sharding vs the single-writer tamper-property:** sharding is *fine* (each shard is still a single-writer chain); it only breaks the *global* deletion guarantee, which N-3 restores. ✔ resolvable.
- **Per-tenant vs platform completeness:** same — N-3 restores it. ✔ resolvable.
- **Epoch-seal-over-a-gap (G6-D1):** the seal MUST refuse over an unverified chain (promote G6 RUN-10 to MUST) — else the composite identity *launders* a pre-existing break into a signed root. ✔ resolvable with the refuse-to-seal rule.
- **Late-arriving buffered events at an epoch boundary (G6-D3):** the rolling per-source seal (seal a source only after its buffer drains) handles this — but it means **the platform is in mixed epochs during a roll**, and the verifier (N-4) must tolerate it. ✔ resolvable, must be stated.
- **Crypto-shred vs the roll-up:** the roll-up commits to `head_hash`; shredding a key inside a historical event doesn't change any `head_hash` (the trick), so the roll-up stays valid. ✔ composes cleanly.

**Net:** the chain-model changes **compose into one coherent model** — `(tenant, source, cluster, epoch)` + roll-up + one epoch mechanism — but the C2-rc as written encodes *none* of it, states `chain_seq` in a way that **invites four incompatible readings**, and omits the roll-up that is the *load-bearing keystone* making partitioning safe. **The rc is not done. It landed the functional half and must now land N-1..N-10 before re-freeze**, ratified jointly by G1/G3/G5/G6/G7/G4 — exactly the propagation step META M-1 warned the process keeps skipping.

---

## 4. The single most important NFR-level finding

**The C2-rc reconciled the *functional* contradictions and declared the audit schema a release candidate — but the eight G-components then independently re-architected that same chain six different ways for scale, tenancy, DR, migration, erasure, and key-trust, and not one of those changes was folded back into the rc.** The rc's `chain_seq` is still a bare "per-source monotonic integer"; there is no shard identity, no tenant identity, no epoch, no global roll-up, no restore/fork event, no tombstone reason, no KTL binding. So the platform has **two un-reconciled forks of its own keystone contract**: the functional rc (frozen-pending-signoff) and the implied operational chain model scattered across G1/G3/G5/G6/G7/G4 — and they make demands on the same three fields (34/35/36) that a build team cannot satisfy from the rc alone.

This is **META M-1 reproduced one level down, and every G-review predicted it would happen to itself** (G3-A8c "G3 is asserting C2-additivity it doesn't own"; G5-A13/A14 "G5 is a contract dressed as a component, its mandates will be filed as G5 backlog and never land in C2"; G6-D4 "hand-offs to C2/G3/G4 dangle"; G7-A3 "G7 hands G4 a grain G4 hasn't agreed to"). The fix is the same fix META prescribed and the corpus keeps not executing: **one Wave-3 NFR reconciliation pass that folds N-1..N-10 into the C2-rc and the G4 key-grain contract, makes all eight G-components agree on the composite chain identity, and only then re-freezes.** Until that pass runs, the NFR layer is internally inconsistent in exactly the way the functional layer was before its rc — and the operational architecture, which for a 7-year regulated evidence product is *at least half the product*, is not build-ready.

The good news mirrors the functional verdict: **the chain-model changes do compose** (§3) — the bones are right, the partition axes are orthogonal, the roll-up + single-epoch-mechanism + composite-`chain_seq` make them one coherent model. The defect is propagation, not topology. Run the pass.

---

## 5. Report-back — Top 10 NFR cross-cut defects + chain-model verdict

| # | ID | Defect | Sev |
|---|---|---|---|
| 1 | **NX-1** | Chain-model quadruple-bind: shard (G1) × per-tenant (G5) × epoch (G6) × `(epoch,seq)` arbiter (G3) — four re-architectures of one `chain_seq`, **C2-rc encodes none** | CRITICAL |
| 2 | **NX-2** | RPO=0 (G3) vs lossless-ingest-SLO (G6) vs batch-commit (G1) — **not mutually satisfiable**; pick-two trilemma on the fast local buffer | CRITICAL |
| 3 | **NX-3** | Crypto-shred (G7/G5) vs replay-`complete` (G1/E1) vs drop-count-0 SLO (G6) vs tamper-evidence — lawful erasure must dodge the loss alarms yet stay distinguishable from tampering | CRITICAL |
| 4 | **NX-4** | Per-tenant chain (G5) and sharding (G1) each reintroduce the *other's* undetectable-whole-chain-deletion hole; both need the **same global roll-up meta-log** | CRITICAL |
| 5 | **NX-6** | The dollars don't compose: G2's own cost-cliff cure deletes the differentiator, and G4 value-capture + G5 hard-tenancy + G3 active-active + G7 DEKs each inflate the same storage line | CRITICAL |
| 6 | **NX-7** | Admission hot path over-subscribed: G1 cosign + G6 value-capture (now a MUST) + D1 JWKS stack on the ≤250ms webhook ⇒ deploy-storm bricks the fleet with `failurePolicy: Fail` | CRITICAL |
| 7 | **NX-11** | The eval:recorded-event ratio (unstated) swings ingest/cost/SLO/RPO by 170×; "record-all" compliance demand blows G1+G2+G6+G3 at once | CRITICAL |
| 8 | **NX-5** | KMS/KTL audit-root (G4) and region-loss key availability (G3) share a failure domain ⇒ restored chain + erasure certs unverifiable; KTL has no DR owner | CRITICAL |
| 9 | **NX-14** | Three insider holes compose: issuer→tenant edit (G5) + single forge-anything trust-root (G4) + in-cluster signer-invoke (G4) ⇒ colluding operator forges a whole tenant's chain | HIGH |
| 10 | **NX-12** | Day-2 un-runnability: G6's ≤2-pages/shift over 14 services is defined-away, while G1–G7 each add new on-call surface (shards, per-tenant chains, DEKs, epochs, KTL, breakers) | HIGH |

**Do the chain-model changes compose? — YES, conditionally.** Shard (G1) + per-tenant (G5) + epoch (G6) + failover-fence (G3) + crypto-shred (G7) compose into **one coherent model**: chain identity = `(tenant, source.system, cluster, epoch)`, `chain_seq` monotonic within that tuple, with a **global signed roll-up super-checkpoint** over all chain-heads as the keystone that keeps deletion globally detectable. The partition axes are orthogonal; the two uses of "epoch" (schema-migration seal + failover fence) unify into one cross-sign-old-head→new-genesis mechanism with two triggers; crypto-shred rides inside any chain without touching `content_hash`. **But this composite model is in NO single document, and the C2-rc as written states `chain_seq` as a bare "per-source integer" with no shard/tenant/epoch/roll-up — inviting four incompatible build readings.** They compose in principle and conflict on paper. The rc landed the functional half (XD-1/2/3/8/11/12/13) and must now land the operational half (N-1..N-10) and the G4 key-grain contract before re-freeze. **The single most important NFR finding: META M-1 has reproduced itself one level down — the audit schema was re-opened to fix functional defects, six G-components then re-architected it for operational ones, and those changes were never folded back. Run one NFR reconciliation pass over the rc, or hand builders a contract that four G-components define four different ways.**
