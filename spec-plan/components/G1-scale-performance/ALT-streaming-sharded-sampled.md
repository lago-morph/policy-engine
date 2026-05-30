# ALT — Streaming, Sharded-Per-Cluster, Sampled Scaling Architecture

**For:** G1 — Scale, Performance & Capacity (high-value ALT per ORCHESTRATION-PLAN §3)
**Date:** 2026-05-30 · **Author persona:** alternative-architecture
**Thesis:** The primary G1 SPEC scales an architecture that is **synchronous-at-ingest, centralized-by-default, and full-capture**. This ALT proposes the opposite on all three axes — **async/streaming ingest with deferred hash-chaining, sharded-per-cluster from the ground up, and sampled (stratified) capture+replay** — and argues it is the architecture the platform should adopt *if* it intends to sell past ~10×. The three changes are independent and can be adopted à la carte.

---

## 0. The three axes (primary vs ALT)

| Axis | Primary G1 (and C2 as frozen) | ALT |
|---|---|---|
| **Hash-chaining** | **Synchronous, in the ingest write path.** Each append needs the prior event's hash → serialization point (the #1 bottleneck, N-G1-112). | **Asynchronous / deferred.** Ingest writes events *unchained* into a log/stream at line rate; a separate *chainer* process assigns `chain_seq` + `prev_hash` downstream, off the critical write path. |
| **Topology** | **Centralized-by-default.** One hub aggregates all spokes' events into a central store; chain is per-`source.system` (potentially global). | **Sharded-per-cluster.** Each cluster owns its own chain + local store; the hub federates *queries* and *roll-up checkpoints*, never the raw write path. |
| **Capture/replay** | **Full capture.** Record (potentially) every decision; replay over the full event set, scoped by namespace to control cost. | **Stratified sampling.** Capture all denials + a statistically-valid sample of allows; replay over a materialized stratified sample with confidence intervals, full-capture only for flagged controls/scopes. |

---

## 1. Axis 1 — Asynchronous / deferred hash-chaining (the key idea)

### 1.1 The problem with synchronous chaining
The primary architecture's tamper-evidence (`prev_hash` + monotonic `chain_seq`) is computed **inline at ingest**, which means ingest is a sequential single-writer per chain (XD-G1-4). At 100× burst (5,000/s) a durably-committed single chain may not keep up, and sharding it (the primary's fix) reintroduces a whole-shard-deletion hole (XD-G1-5).

### 1.2 The ALT mechanism — write fast, chain later
```
  ingest (line rate, parallel)            chainer (single seq, off-path)         signer (off-path)
  ─────────────────────────────           ───────────────────────────────       ──────────────────
  event → content_hash (parallel,    ┌──► consume log in order →               ┌──► every 10k / 15m:
  no prev_hash yet) → append to      │    assign chain_seq, set prev_hash,     │    Merkle root over
  an ORDERED, durable LOG            │    write to chained store               │    chained segment,
  (Kafka/NATS-JetStream/             │    (this is sequential but does         │    ed25519 sign
  object-store WAL), keyed           │    NOT block ingest)                     │
  by (cluster, source)               │                                          │
```
- Ingest computes `content_hash` (which depends **only on the event's own content**, C2 §7.2 — *not* on `prev_hash`) in **parallel, at line rate**, and appends to a durable ordered log. **No serialization at ingest.**
- A **single chainer** consumes the log *in log order* and assigns `chain_seq` + `prev_hash`. This is still sequential — but it runs **downstream of durability**, so its throughput governs *chain-freshness latency*, not *ingest acceptance*. Ingest never blocks on it. The chainer can also be **per-shard-parallel** (one chainer per `(cluster, source)` partition of the log), because each partition is an independent chain.
- **Crucial property:** the durable ordered log (Kafka/JetStream) already gives you *ordered, replicated, line-rate, no-data-loss* ingest — the exact thing the synchronous chain was struggling to provide. The hash-chain becomes a *derived, verifiable view* over the log rather than a write-path constraint.

### 1.3 What this buys
- **Ingest throughput decouples from chain throughput.** Ingest = log append throughput (Kafka: 100k+ msg/s/partition trivially). Chain-freshness becomes a *latency* SLI (`chain_lag_seconds`) you can watch, not a *capacity* wall you hit.
- **The whole-shard-deletion hole (XD-G1-5) is closed naturally:** the durable log is itself replicated and append-only (the broker enforces it); the chain is a derived attestation. To delete a shard you must compromise the broker's replicated log *and* re-derive every chain *and* re-sign every checkpoint *and* the cross-shard roll-up.
- **Backpressure becomes the broker's job** (a solved problem) instead of a bespoke spill-to-disk-and-hope (N-G1-141, XD-G1-6).

### 1.4 What it costs
- **Operational weight:** you now run Kafka/JetStream (a stateful, HA, partitioned broker) — a non-trivial day-2 burden (G6) for a POC. **At POC this is over-engineering; the primary's simple synchronous chain is correct for 1×–10×.** The ALT earns its keep only at ~10×+.
- **Chain freshness lag:** the chain (and therefore tamper-evidence and C3 chain-integrity detection) is now eventually-consistent with ingest by `chain_lag_seconds`. For an evidence product this must be bounded and *itself* audited. An auditor asking "is event X chained?" may get "yes, as of checkpoint N (5 s ago)" rather than "yes, synchronously." Defensible, but a story to tell.
- **A new single-point-of-order:** the chainer per partition is still sequential. It's off the ingest path, so it doesn't bound *acceptance*, but a stalled chainer stalls chain-freshness. Needs its own liveness/failover.

---

## 2. Axis 2 — Sharded-per-cluster federation (vs centralized hub)

### 2.1 The idea
Instead of shipping every spoke's events to a central hub store, **each cluster runs its own ingest + chain + local store** (the spoke owns its evidence). The hub does **query federation** (scatter-gather across spoke query APIs) and **roll-up checkpointing** (collects each spoke's signed chain root into a hub-level Merkle-of-roots, signed — this is exactly the cross-shard super-checkpoint XD-G1-5 demanded).

### 2.2 What it buys
- **Linear horizontal scale by construction:** adding a cluster adds its own ingest+chain+store capacity. The fleet's aggregate ingest ceiling = Σ per-cluster ceilings — there is no central write bottleneck at all. This is the cleanest answer to XD-G1-4 and XD-G1-10.
- **Data-residency / sovereignty for free:** a regulated EU cluster's evidence never leaves the cluster's region (the hub federates queries, not raw data). A major selling point to the exact regulated buyer (Stack C) the corpus targets — and one the centralized model fights.
- **Blast-radius isolation:** one cluster's ingest saturation or replay job cannot starve another (partially answers G5/XD-G1-10 contention without a CQRS split).
- **DR is per-shard:** losing one cluster's store loses one cluster's evidence, not the fleet's (better RPO blast radius for G3).

### 2.3 What it costs
- **Query federation is slower and harder:** a fleet-wide coverage-gap report (HL-20) becomes scatter-gather over N spoke APIs with partial-failure semantics ("3 of 50 clusters unreachable — report is 94% complete"). Cross-cluster joins (the `correlation_id` dedup, C5 N-C5-8) are harder when data is partitioned. **Interactive cross-fleet queries get slower**, trading the centralized model's easy global reads for write scalability.
- **More moving parts per cluster:** every spoke now runs a store + chain + signer (key custody per cluster! — a real G4 burden: N signing keys, or a shared key shipped to every spoke, which is worse).
- **Roll-up checkpoint is now load-bearing for tamper-evidence** and must be highly available — if the hub roll-up stops, whole-shard deletion becomes undetectable again until it resumes.

---

## 3. Axis 3 — Stratified sampling (vs full capture)

### 3.1 The idea
Full capture records (potentially) every decision and the storage/ingest cost scales with total decisions (XD-G1-12: possibly 100k events/s at 100×). **Stratified sampling** records:
- **100% of `deny` / `warn` / `mutate` / approval decisions** (the interesting, rare, compliance-relevant tail — always full).
- **A statistically-valid sample of routine `allow` decisions**, stratified by `(control_id, scope)`, sized to give a target confidence interval on per-control coverage and allow-rate.
- **100% capture for explicitly flagged controls/scopes** (e.g. `SC-IMG-001` in `payments-prod` — the ones an auditor will actually walk).

### 3.2 What it buys
- **Ingest and storage scale with the *interesting* event rate, not the *total* rate.** If 99% of decisions are routine allows, sampling those at 1% cuts ingest/storage by ~99% on the allow path while preserving 100% of denials and a CI-bounded view of allows. This is the only axis that attacks XD-G1-12 (the 170× ratio) head-on — it makes the ratio a *tunable knob* with quantified statistical loss.
- **Replay becomes affordable:** differential replay over a stratified sample gives "≈ N ± δ events would newly break (95% CI)" in seconds, over the full fleet, without the 70-worker-day O(events) explosion (XD-G1-8). Full-confidence replay is available on demand for flagged scopes.

### 3.3 What it costs — and why it's dangerous for *this* product
- **This collides head-on with the platform's identity.** The corpus's differentiator is **authoritative, complete, replayable evidence** (META-ADVERSARIAL G-2). A *sampled* allow-log means: "prove no unaudited workload was admitted" can only be answered *statistically*, not absolutely. For many audits that is **disqualifying** — an auditor wants the *actual* record, not a 95% CI. Sampling turns a `complete`-labeled product into an admittedly-incomplete one — the exact "declared vs verified" sin (META-ADVERSARIAL G-6, XD-6).
- **Therefore sampling must be opt-in and scope-explicit**, never default for regulated scopes. The right framing: **full capture for governed/regulated scopes; sampling only for the long tail of ungoverned, high-volume, low-stakes allow traffic** — where the value is operational analytics (C3 trends), not auditor evidence. The honesty rule (C2 §1.3) extends to honesty *about the sampling itself*: a sampled scope's coverage report must carry a "sampled at rate r, CI ±δ" label, never present as complete.
- **Sampling bias against `best_effort`/`insufficient` events** (XD-G1-12 cross-ref): if the sampler drops allows that happen to be the `insufficient` ones, replay's `complete` fraction looks artificially healthy. The stratification must be *outcome-and-completeness-aware*, not just outcome-aware.

---

## 4. Combined ALT vs Primary — trade-off summary

| Property | Primary (sync / central / full) | ALT (async / sharded / sampled) |
|---|---|---|
| Ingest ceiling | Per-chain sequential (~5k/s/shard); needs sharding fix | Broker line-rate (100k+/s); chain is derived |
| Central write bottleneck | Yes (chain + 9-index DB) | None (per-cluster) |
| Tamper-evidence freshness | Synchronous (strongest) | Eventual, bounded by `chain_lag` (slightly weaker, auditable) |
| Whole-shard deletion detectable? | Only with added roll-up | Yes (broker log + roll-up native) |
| Fleet-wide replay at 100× | Disallowed / amputated (N-G1-144) | Affordable via stratified sample (CI-bounded) |
| Cross-fleet interactive query | Fast (central) | Slower (federated scatter-gather) |
| Data residency / sovereignty | Hard (central) | Native (per-cluster) |
| Day-2 operational weight | Low (good for POC) | High (broker + per-cluster stores + N keys) |
| "Complete evidence" claim | Holds (if full capture affordable) | Holds **only** for full-capture scopes; sampled scopes are explicitly statistical |
| Right when... | POC → ~10×; single-region; auditor wants absolute completeness | ~10×+; multi-region; mixed regulated+high-volume traffic |

---

## 5. Recommendation

- **Adopt Axis-1 (async/deferred chaining) at the ~10× threshold.** It is the highest-leverage, lowest-semantic-cost change: it removes the #1 bottleneck (XD-G1-4) and *closes* the shard-deletion hole rather than opening it (XD-G1-5), at the cost of a bounded, auditable chain-lag and a broker to operate. The synchronous chain stays correct for the POC; the migration is "put a durable log in front and move chaining downstream" — a contained change. **This is the one ALT axis I would build toward from day one** (choose a store/log that *can* become the broker later, rather than painting into a synchronous-chain corner).
- **Adopt Axis-2 (sharded-per-cluster) only if multi-region/sovereignty is a real buyer requirement.** It is the cleanest scale story but the heaviest operationally and the worst for key custody (G4). Decide based on whether the design partner is multi-region; don't pay for it speculatively.
- **Adopt Axis-3 (sampling) narrowly and never for regulated scopes.** Use it to make the **long-tail allow analytics** affordable, with mandatory "sampled, CI ±δ" honesty labels — and keep **full capture for every governed control and every regulated scope**, because the product's entire reason to exist is that those are *complete*. Sampling the wrong scope would be the platform committing the sin it sells against.

**One-line verdict:** the primary G1 architecture is right for the POC and through ~10×; **the async-deferred-chaining axis is the change that turns the #1 bottleneck from a wall into a latency SLI and should shape the storage/ingest choice now**, while sharding and sampling are situational levers to pull only when the buyer's topology (multi-region) or traffic mix (huge ungoverned allow tail) actually demands them.
