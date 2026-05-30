# G1 — Scale, Performance & Capacity — ADVERSARIAL REVIEW

**Component ID:** G1 · **Reviewer role:** red-team / SRE-who-has-been-paged-at-3am persona · **Date:** 2026-05-30
**Mandate:** attack G1's *own* numbers, and attack the functional components G1 sizes. Find the budget that is unachievable, the hot-path call that blows the deadline, the ingest path that overwhelms the hash-chain, and the replay that explodes. Prioritized defect list at the end.

---

## 1. Attack the admission latency budget

### XD-G1-1 (CRITICAL): The admission hot path makes a synchronous external-data call, and the budget only holds at 95% cache hit — which is exactly false at the moment it matters most.
G1 §2.2 budgets the external-data call (step 5) at p99 400 ms *assuming ≥95% cache hit* (N-G1-103). But consider **when** the cache is cold:
- **Deploy storms / CI bursts:** a new release pushes 200 new image digests in a minute. Every one is a **cache miss** on `image-signature-status` (cosign verification, a network call to a registry + Rekor/Fulcio). At 0% hit ratio the per-call latency is provider-RTT + verification (cosign signature verification against a transparency log is **100s of ms**, sometimes seconds). 200 simultaneous cold misses against one provider instance → provider saturation → calls queue → **p99 approaches the 500 ms timeout, then the 2 s webhook budget, then the K8s `timeoutSeconds`.**
- With `failurePolicy: Fail` (B2-R14, prod default), a saturated external-data provider during a deploy storm **fails admission closed for the entire cluster** — i.e. the governance platform **bricks deploys precisely during a release**, the worst possible moment. This is not hypothetical; it is the canonical failure mode of synchronous admission-time external verification.
- **The budget is achievable in steady state and unachievable during the exact bursts the system exists to govern.** The §2.2 "p99 67 ms" headline is a warm-cache number presented without the cold-cache asterisk being load-bearing enough.
- **Fix:** the external-data result for image signatures should be **pre-warmed at build/push time (Conftest/B3 already verifies at CI — reuse that result as the cache seed) and verified asynchronously**, so admission reads a *cache that was populated before the image ever reached admission*. Admission should *never* be the first place a signature is checked. G1 should mandate **cache-seed-from-build** as a MUST, not leave it to a 95%-hit assumption. (Cross-ref B3 build-time evidence, C2 §3.10 Conftest event.)

### XD-G1-2 (HIGH): OPA eval p99 ≤ 80 ms is asserted for "25–100 controls" but the cost is driven by the *request_object size*, not the control count.
N-G1-101 budgets OPA eval at 80 ms for the bundle. But OPA eval cost scales with **input document size and rule complexity**, not just the number of controls. A large `request_object` (a Deployment with 50 containers, init-containers, big env blocks, or a CRD with a multi-MB spec) iterated by a `deny[msg] { some i; input.spec.template.spec.containers[i]... }` comprehension can be **O(containers × rules)**. A pathological policy over a large object can blow 80 ms easily. The budget needs to be stated as **"≤ 80 ms for inputs ≤ N KB and policies without unbounded comprehension over large arrays"**, plus a **policy-cost linter** (reject/flag Rego whose worst-case eval over the largest admissible object exceeds budget) in B1's CI. As written, N-G1-101 is a budget with no enforcement mechanism — it will be discovered violated in production, not at authoring time.

### XD-G1-3: D1 JWT mapping "cached, 2 ms" assumes the cache never misses; on miss it calls the IdP synchronously inside the webhook.
§2.2 step 3 budgets D1 at p50 2 ms / p99 15 ms "cached." On a cold cache or after a Keycloak realm key rotation, JWT validation may need a JWKS fetch from Keycloak. If that fetch is synchronous and in-band, it is a **second external call in the hot path** the budget doesn't account for, and a Keycloak brownout becomes an admission brownout. The budget must state that JWKS/key material is **pre-fetched and refreshed out-of-band**, never lazily inside admission.

---

## 2. Attack the ingest / hash-chain (THE bottleneck)

### XD-G1-4 (CRITICAL): Audit ingest at 500k/day × multi-cluster overwhelms C2's single hash-chain — the spec mandates a serialization bottleneck and never sharded it.
This is the defect META-ADVERSARIAL #2 pointed at, made precise:
- C2 §7.3 (N-C2-301) mandates a **per-source append-only chain** with `prev_hash` = previous event's `content_hash` and **strictly monotonic `chain_seq`**. Appending event N **requires** event N−1's hash. This is a hard serialization point: **a single chain cannot be appended in parallel, by construction** — that property is what makes it tamper-evident.
- C2 §8.1 partitions the *store* by `(source.system, time)`, but says nothing about the **chain** being partitioned the same way. Read literally, "per-source" could mean one chain per `source.system` (e.g. one global `gatekeeper` chain across **all 5–500 clusters**). At 100× that is **one sequential writer absorbing 5,000 events/sec of burst** across the whole fleet.
- A naïve durable chain (fsync per append, to make "the event is committed and chained" durable) caps at **hundreds of appends/sec** on commodity storage. Even an in-memory chain caps at ~5,000–20,000/sec single-threaded. **At 100× burst (5,000/sec) a single global, durably-committed chain does not keep up.** Ingest lag grows unboundedly, `ingest_lag_seconds` blows past the 15-min C3 reconciliation window, detection operates on stale data, and the spilled buffer (N-G1-141) grows until disk fills.
- **G1 §1.1/N-G1-113 prescribes sharding the chain by `(source.system, cluster)` — but this is a CHANGE to C2's contract that C2 has not accepted.** C2 as frozen does not guarantee a shardable chain identity. **This is a cross-component contract defect: G1's scaling fix requires a C2 schema/contract clarification (chain identity = shard identity) that the frozen C2 SPEC does not provide.** It must go to the C2 reconciliation pass (it is adjacent to the already-open C2 un-freeze, META-ADVERSARIAL M-1). Until C2 commits to a sharded chain, **the central evidence spine has an unfixed scale ceiling.**

### XD-G1-5 (HIGH): Sharding the chain weakens the tamper-evidence guarantee, and G1 doesn't address the new attack surface.
Sharding (N-G1-113) trades the single global ordering for N independent chains. But now:
- An attacker who can **drop an entire shard** (delete one cluster's chain) leaves the *other* shards' checkpoints perfectly valid — the deletion is only detectable if there is a **higher-order index of which shards should exist** and a **cross-shard checkpoint** that commits to all shard roots. G1 §7/N-G1-114 says signing isn't a bottleneck but doesn't add the **shard-roll-up checkpoint** that sharding *requires* to preserve "deleting a whole source is detectable" (C2 §7.1 insider-deletion threat). **Sharding without a roll-up Merkle-of-shard-roots reintroduces an undetectable-deletion hole.** G1 must require a periodic **cross-shard super-checkpoint** signing the set of shard roots, and C2 must own it. This is a real gap G1's own §7 glosses.

### XD-G1-6 (HIGH): Batched group-commit (OQ-G1-2) loses up to 10 ms of evidence on crash and calls it "audited" — but the audit-of-the-gap may itself be in the lost batch.
N-G1-141/OQ-G1-2 resolve backpressure by spilling and recording a coverage gap. But the **batched-commit window means the last ≤10 ms / ≤100 events are not durably chained when the worker crashes**. The "record a coverage gap" mitigation (C3 `coverage_gap`) requires *writing an event* — but if the ingest worker just died, who writes the gap event, and into which chain (whose tail just vanished)? The honesty-over-coverage tenet says "we lost evidence here must be queryable," but the mechanism that makes it queryable is the same pipeline that just failed. **G1 needs an out-of-band liveness/gap detector** (a separate process comparing `chain_seq` continuity against expected source sequence, e.g. K8s audit's own sequence) so a lost tail is detected by something that *didn't* crash with it. Otherwise "audited evidence loss" is a promise the failure mode cannot keep.

### XD-G1-7 (MEDIUM — corrects a likely mis-attribution, kept for the record): the signer is NOT the bottleneck, despite the META-ADVERSARIAL phrasing "per-event cryptographic cost."
META-ADVERSARIAL line 65 implies per-event signing is a scale cost. It is not: per-event `signature` is **optional** (C2 field 36); only **checkpoints** are signed, every 10k events / 15 min, at ~50 µs ed25519. The real cost is the **sequential SHA-256 append (XD-G1-4)**, not signing. G1 §N-G1-114 states this correctly. **Defect logged only to prevent the build team from "optimizing" the signer (wrong target) instead of sharding the chain (right target).** The genuine crypto cost at scale is the **chain-verification full scan** (LT-4): re-verifying 1.5B events' hashes at 100× is a multi-hour sequential scan — that, not signing, is the crypto scale cost, and G1 should budget the verify scan explicitly (it does, LT-4, but the target "within reconciliation budget" is hand-wavy for 1.5B events).

---

## 3. Attack replay (the O(events × bundles) explosion)

### XD-G1-8 (HIGH): 30-day replay over a real fleet is O(events × bundles) and G1's "namespace-scope by default" only hides it.
E1 §4 is `O(|S| × bundles)`. G1 §5.1 calls this "linear, fine" and bounds `|S|` by namespace scope. But the **product feature that sells the platform is the *cluster-wide* / *fleet-wide* differential** ("if I add rule X, how many of my *org's* deploys would newly break?" — DT-49/HL-17). Namespace-scoping that to dodge the cost **defeats the differentiator**. At 100×, a 30-day fleet differential is 1.5B events × 2 evals = ~70 worker-days; even at 100 workers that's ~17 hours and a large compute bill **per simulated policy change** — and policy authors iterate (try rule X, tweak, try X′). The cost is `O(events × iterations)`. **G1's "disallow cluster replay at 100×" (N-G1-144) is honest but is a feature amputation, not a scaling solution.** The ALT's **sampling + materialized-stratified-sample** approach is the real answer and G1 under-sells it by listing it as option (d).

### XD-G1-9 (MEDIUM): The EvidenceSet materialization is "one store scan" — but at 100× that scan is itself the cost, and it re-scans per run unless dedup is perfect.
N-G1-132 says materialize-once-reuse. Good — but the *first* materialization of a 30-day fleet set scans 1.5B rows through the 9 secondary indexes (§4 / N-G1-121). That scan is I/O-bound on the index DB, the **least horizontally scalable tier** (XD-G1-10). And materialization-reuse only helps if the user replays the *same* scope twice; iterative policy authoring usually changes the candidate bundle but keeps the set — so reuse *does* help there, but the **first scan still gates the first answer** by minutes-to-hours. G1 should require **the materialized set to be built incrementally/continuously** (a rolling materialized view per common scope), not on-demand per replay.

### XD-G1-10 (HIGH): The index DB is the real central bottleneck, and G1 under-rates it relative to the chain.
G1 §7 names the hash-chain append as #1 and the index DB as "partially" scalable. I'd argue they're **co-equal** or the **index DB is worse**, because:
- The chain append is shardable (XD-G1-4 fix) and CPU/memory-bound (fast).
- The index DB carries **9 secondary indexes** (N-G1-121) that must be maintained on *every write* (write amplification ~10×) **and** serve every analytics/replay/reporting/GUI read. It is a **single logical writer per shard** with read-replica reads, but the **9-index write amplification at 580 events/sec sustained (100×) plus heavy analytical scans is a classic OLTP-meets-OLAP contention** that read replicas don't fix (replicas lag, and analytics wants fresh data). **G1 should mandate splitting the write-path index store from the analytical/replay store (CQRS / a columnar analytics mirror), which it currently doesn't.** The §4 storage math counts the bytes but not the **IOPS and the read/write contention**, which is what actually pages an SRE.

---

## 4. Attack the capacity model itself

### XD-G1-11 (MEDIUM): The model's constants are assumed, not measured, and the worked examples inherit ±wide error bars presented as precise.
§9 presents `64 GB`, `735 TB`, `125 webhook_replicas`, `1,500 provider qps` as if precise. Every one rests on assumed constants (2.5 KB/event, 50% CAS dedup, 95% cache hit, 5,000 append/s, 2 ms eval, 800 eval/s/replica). **The CAS dedup ratio in particular is a wild guess** — it depends entirely on workload (a monorepo redeploying the same 10 images dedups 95%; a fleet of unique images dedups ~0%). At 0% dedup the storage numbers ~double. The model is sound as a *framework* but its outputs should carry **explicit ±error bands** and be labeled "validate via LT-7 before quoting to a customer." PLAN M-G1-6 (calibrate within ±25%) addresses this, but the SPEC §9 worked examples read as committed numbers. **Add error bars to every worked example.**

### XD-G1-12 (MEDIUM): The eval-rate vs ingest-rate decoupling (N-G1-105) is load-bearing and unproven.
G1's central scaling claim is that 100k evals/sec produces only ~580 recorded events/sec because "most evals are allow on steady-state traffic" and only a decision subset is recorded. **This ratio (eval:recorded-event) is never sourced.** If the platform records an audit event for *every* admission decision (including the millions of routine allows — which an auditor might *require* for "prove nothing was admitted unaudited"), then ingest rate ≈ eval rate ≈ **100k events/sec at 100×, not 580/sec.** That is a **170× larger ingest problem** and blows the entire §3/§4 sizing. **The eval:event ratio must be a stated, defended assumption with a knob** ("record all decisions" vs "record denials + sampled allows"), because the choice changes the storage/ingest sizing by two orders of magnitude — and the *compliance* requirement may force "record all," which the model cannot afford. This is the single biggest unstated assumption in G1.

### XD-G1-13 (LOW): "100× is the ceiling, >100× is a re-architecture" is asserted without showing where 100× breaks.
N-G1-ENV caps the model at 100×. Fine as scope, but G1 should name *which* component breaks first past 100× (it's the index DB / the chain-verify scan) so the re-architecture trigger is a measured threshold, not a round number.

---

## 5. Cross-component contradictions surfaced

- **G1 ↔ C2:** G1 requires a **sharded, batch-committed, roll-up-checkpointed** chain (N-G1-113, XD-G1-4/5). C2 frozen mandates a **per-source single chain, no batching contract, no shard roll-up**. **These conflict.** → C2 reconciliation pass.
- **G1 ↔ C2/compliance:** G1's storage/ingest sizing assumes **denials + sampled allows recorded** (XD-G1-12); a "record every decision" compliance requirement contradicts the sizing. → must be decided in C2/G7/compliance, not in G1 alone.
- **G1 ↔ B2/B3:** G1's hot-path budget (XD-G1-1) requires **build-time cache seeding** of signature status; B3 verifies at build but there is **no contract that the build result seeds the admission external-data cache.** → new B2↔B3 contract.
- **G1 ↔ E1 / market thesis:** G1 disallows cluster-wide 30-day replay at scale (N-G1-144); the differentiator *is* fleet-wide differential (XD-G1-8). → the ALT (sampling) must be promoted, or the differentiator is scale-bounded.
- **G1 ↔ META-ADVERSARIAL G-2:** G1 sizes replay assuming events are `complete`; META-ADVERSARIAL G-2 says most real events are `best_effort`/`insufficient`. **E1 §4 skips `insufficient` events** — so the *replayable* `|S|` may be far smaller than total events (cheaper compute, but a *smaller, possibly-unrepresentative* sample — a correctness problem masquerading as a perf win).

---

## 6. Prioritized defect list

| # | Sev | Defect | Fix owner |
|---|---|---|---|
| **1** | **CRITICAL** | **XD-G1-4** — per-source single hash-chain is an unsharded serialization bottleneck; overwhelmed at 100× multi-cluster ingest. G1's sharding fix requires a C2 contract change C2 hasn't accepted. | C2 (un-freeze) + G1 |
| **2** | **CRITICAL** | **XD-G1-1** — synchronous external-data (cosign) call in the admission hot path saturates during deploy storms (0% cache hit) and, with `failurePolicy: Fail`, bricks deploys cluster-wide at the worst moment. Needs build-time cache seeding. | B2 + B3 + G1 |
| **3** | **HIGH** | **XD-G1-12** — the eval:recorded-event ratio (governing whether ingest is ~580/s or ~100k/s at 100×) is an unstated assumption that swings sizing by 170×; compliance may force "record all." | C2 + G7 + G1 |
| **4** | **HIGH** | **XD-G1-10** — the 9-index OLTP event store is a co-equal (arguably worse) bottleneck under write-amplification + analytical-scan contention; needs CQRS / columnar mirror split. | C2 + G1 |
| **5** | **HIGH** | **XD-G1-8** — fleet-wide 30-day differential replay (the differentiator) is O(events×iterations); namespace-scoping dodges it by amputating the feature. Needs sampling (ALT). | E1 + G1 |
| **6** | **HIGH** | **XD-G1-5** — sharding the chain reintroduces undetectable whole-shard deletion unless a cross-shard roll-up super-checkpoint is added. | C2 + G1 |
| **7** | **HIGH** | **XD-G1-6** — batched-commit evidence loss; the gap-recording mechanism can die with the data it should record. Needs out-of-band gap detector. | C2 + G1 |
| **8** | **MED** | **XD-G1-2** — OPA eval 80 ms budget driven by request_object size, not control count; needs a policy-cost linter at authoring. | B1 + G1 |
| **9** | **MED** | **XD-G1-9** — EvidenceSet first-materialization scan gates the first replay answer; needs continuous/rolling materialization. | E1 + C2 |
| **10** | **MED** | **XD-G1-11** — capacity-model constants assumed, not measured; worked examples need ±error bands and a "validate before quoting" label. | G1 |
| **11** | **MED** | **XD-G1-3** — D1 JWKS fetch can become a second synchronous hot-path call on cache miss / key rotation. | D1 + G1 |
| **12** | **MED** | **XD-G1-7** — record-keeping: don't optimize the signer (not the bottleneck); the chain-verify full scan IS a real crypto scale cost needing an explicit budget. | G1 |
| **13** | **LOW** | **XD-G1-13** — name the component that breaks first past 100× (index DB / verify scan) as the re-architecture trigger. | G1 |

**Bottom line:** G1's framework is correct and its #1-bottleneck identification (the hash-chain) matches the META-ADVERSARIAL finding. But (a) the chain fix is a C2 contract change not yet accepted, (b) the admission hot path has a real deploy-storm outage mode, and (c) the **single most consequential number in the whole model — the eval:recorded-event ratio — is unstated and could be wrong by 170×.** Fix those three before any customer-facing capacity claim.
