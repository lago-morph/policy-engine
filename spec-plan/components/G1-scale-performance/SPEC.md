# G1 — Scale, Performance & Capacity — SPEC

**Component ID:** G1 · **Domain:** G — Operational / Non-Functional Requirements (NFR)
**Spec sources:** §22 (POC scale), §24–25 (deployment/F2), §18/§17B/§17C.6 (real-time enforcement & admission deadline), §12–13 (C2 audit schema), §14 (C3 analytics), §17 (E1 simulation), §17E (C5 reporting).
**Companion NFR components:** G2 (cost/retention economics), G3 (availability/DR), G4 (key management), G5 (multi-tenancy isolation), G6 (observability/day-2), G7 (data lifecycle/privacy), G8 (Rego authoring).
**Provenance:** authored to close META-ADVERSARIAL-SECOND-OPINION finding #2 — *"No production-scale or cost model exists (only §22's 6/sec POC)."* This SPEC is the missing performance/capacity layer.
**Status:** DRAFT v1 · **Date:** 2026-05-30
**Scenarios touched:** DT-46 (30-day replay), DT-77/DT-80 (reporting at scale), HL-06/DT-28 (bypass detection latency), HL-20 (federated reporting), DT-49/HL-17 (differential sim).

---

## 0. Why this component exists

Every functional component in the corpus is sized for *correctness*, not *throughput*. F3 §22 explicitly states the POC success metric is "correct, traceable, replayable decisions, not throughput" (R-F3-PERF-4). That is the right call for a POC and the wrong basis for selling to a regulated enterprise whose K8s fleet emits 10–100M admission events/day. G1 supplies:

1. **Concrete latency/throughput budgets** for every hot path, with the hard Kubernetes admission timeout as the immovable constraint.
2. **A capacity-planning model** (formulas + worked examples at POC, 10×, 100×) so the architecture's scale ceiling and per-event cost are knowable *before* contracts are signed.
3. **Backpressure / load-shedding contracts** so the system degrades predictably instead of cascading.
4. **A per-component horizontal-scale-vs-bottleneck map** identifying exactly where the architecture serializes.
5. **A benchmarking / load-test plan** with named SLIs, so the budgets are *measured*, not asserted.

### 0.1 Assumed reference hardware (decided defaults — state and continue)

All budgets below assume this reference footprint unless stated. These are **assumptions** the build must validate; they are chosen to be unremarkable cloud-commodity sizes.

| Tier | Instance class (assumed) | vCPU / RAM | Notes |
|---|---|---|---|
| OPA replica (centralized) / Gatekeeper webhook pod | 4 vCPU / 8 GiB | per replica | OPA eval is CPU-bound; Rego compiled to bytecode in-process. |
| Audit normalizer / ingest worker | 4 vCPU / 8 GiB | per worker | I/O + hashing + signing bound. |
| Event store node (index DB, e.g. Postgres) | 8 vCPU / 32 GiB / NVMe SSD | per node | Secondary indexes per C2 §8.1. |
| Object/CAS blob store | S3-class object storage | n/a | `request_object`, `before/after_state`, external-data values. |
| E1 simulation worker | 8 vCPU / 16 GiB | per worker | Embeds OPA eval engine; bursty. |
| C3 analytics worker | 4 vCPU / 8 GiB | 1–2 at POC | Batch reconciliation. |
| Governance API (F1) | 2 vCPU / 4 GiB | 2–3 replicas, HPA | Stateless. |

**D-G1-HW:** Reference instance = AWS `m6i.xlarge`-class (4 vCPU/16 GiB) as the unit of capacity math; storage = `gp3`/NVMe for the index DB and S3-standard for CAS. Stated so all $/event and node-count math is reproducible. Other clouds substitute equivalent classes.

---

## 1. The POC scale envelope (the baseline all budgets are relative to)

Restated from §22 / F3-SPEC line 23–29, because every G1 number is a multiple of this:

| Dimension | POC (1×) | 10× | 100× | Source |
|---|---|---|---|---|
| Clusters | 1–5 | 10–50 | 50–500 | §22 |
| Namespaces (per cluster) | 10–100 | 100–1,000 | 1,000–10,000 | §22 |
| **Policy evals/sec (edge, peak)** | **100–1,000** | 1,000–10,000 | 10,000–100,000 | §22 |
| **Audit events/day** | **10k–500k** (~6/s avg, ~50/s burst) | 100k–5M (~58/s avg) | 1M–50M (~580/s avg, ~5,000/s burst) | §22 + F3 |
| 30-day stored events | ~15M | ~150M | ~1.5B | derived |
| 30-day stored bytes (see §4) | 30–75 GB | 300–750 GB | 3–7.5 TB | F3 line 26 + §4 |
| Concurrent GUI users | 5–50 | 50–200 | 200–1,000 | §22 |
| Controls modeled | 25–100 | 100–500 | 500–2,000 | §22 |

**N-G1-ENV:** "100×" in this document means **100× the audit-event and eval rate**, i.e. ~50M events/day and ~100k evals/sec peak. That is a mid-size regulated enterprise fleet, not hyperscale; G1 deliberately does *not* size for >100× (that is a re-architecture, see ALT).

---

## 2. Latency budgets — the synchronous hot path (admission)

The admission webhook is the **only** synchronous, user-blocking, hard-deadline path in the platform. Everything else (ingest, analytics, replay, reporting) is asynchronous and throughput-bound, not latency-bound. So the latency budget chapter is almost entirely about admission.

### 2.1 The immovable constraint

- **N-G1-100 (MUST):** The Kubernetes API server enforces a per-webhook `timeoutSeconds` of **1–30 s** (default 10 s; cluster-configurable, hard ceiling 30 s). B2-R13 sets the governance webhook timeout to a **default budget ≤ 2 s**. If the webhook (including all external-data calls) does not return within `timeoutSeconds`, the API server applies `failurePolicy` (B2-R14: `Fail`/fail-closed for prod runtime). **A blown latency budget on a `failurePolicy: Fail` webhook is a hard production outage — it blocks every admission cluster-wide.** This is the single highest-severity performance failure mode in the platform.

### 2.2 Admission latency budget (the decomposition)

Total synchronous budget = **p99 ≤ 1,000 ms**, p50 ≤ 150 ms, hard-stop = 2,000 ms (B2-R13), well inside the 10 s K8s default. Decomposed (each line is a measured SLI in §8):

| # | Step | Owner | p50 budget | p99 budget | Scales how |
|---|---|---|---|---|---|
| 1 | API-server → webhook network + TLS | K8s | 5 ms | 30 ms | n/a |
| 2 | Gatekeeper request decode + match | B2 | 5 ms | 20 ms | webhook replicas |
| 3 | D1 JWT→subject mapping (cached) | D1 | 2 ms | 15 ms | in-proc cache |
| 4 | OPA eval (no external data) | B1 | **10 ms** | **80 ms** | CPU / replicas |
| 5 | **External-data call (if policy reads it)** | B2 | **40 ms (cache hit ~2 ms)** | **400 ms** | provider + cache |
| 6 | Decision assembly + disposition | B4 | 2 ms | 10 ms | n/a |
| 7 | Response encode + return | B2 | 3 ms | 15 ms | n/a |
| | **Sum (external-data path)** | | **~67 ms** | **~570 ms** | |
| | **Sum (no external data)** | | **~27 ms** | **~170 ms** | |

- **N-G1-101 (MUST):** OPA policy evaluation (step 4) MUST hold **p99 ≤ 80 ms** for the standard governance bundle (25–100 controls). OPA bytecode eval of typical admission policies is sub-millisecond to single-digit-ms; 80 ms is generous headroom for a large bundle + a complex `request_object`. Policies exceeding this are caught by the latency test suite (B2-F6, §8).
- **N-G1-102 (MUST):** Every external-data provider call (step 5) MUST have its **own bounded timeout (default 500 ms)** and a **cache with a configured TTL** (B2-R15, B2-AR-1). The external-data timeout MUST be **strictly less than `webhook_timeout − reserved_budget`** so a slow provider falls back to `failurePolicy` *before* the webhook deadline, never after. **N-G1-102 is the load-bearing budget rule of the hot path** (see ADVERSARIAL XD-G1-1).
- **N-G1-103 (MUST):** External-data results MUST be cached; the **cache-hit ratio is a first-class SLI** (`extdata_cache_hit_ratio`, target ≥ 95% in steady state). The p99 budget in §2.2 *only holds at ≥ 95% hit ratio*; at 0% hit ratio the p99 external-data path is provider-RTT-bound and can approach the 500 ms timeout, consuming half the entire budget.
- **N-G1-104 (SHOULD):** Webhook scope (`namespaceSelector`/`objectSelector`, B2-R16) SHOULD be narrowed so the governance webhook is invoked only for in-scope resources — this removes evals from the hot path entirely (the cheapest scaling lever: don't evaluate what you don't govern).

### 2.3 Why admission throughput is *not* a central-plane problem

- **N-G1-105:** Admission evaluation is **edge-served** (F3 line 25: "Control plane sees logs, not evals"). 100–1,000 evals/sec (POC) and even 100k/sec (100×) are absorbed by Gatekeeper's embedded OPA *per spoke cluster*, horizontally scaled by webhook replicas. **The control plane never sees an eval; it sees the resulting audit event (~1 per decision).** Therefore the eval rate (100k/sec at 100×) and the ingest rate (~580/sec avg at 100×) are different numbers by 2–3 orders of magnitude, because only a sampled/decision subset of evals produce a *recorded* audit event, and most evals are `allow` on already-admitted steady-state traffic. **This decoupling is the single most important scaling property of the architecture** and is why the central evidence spine is sizeable rather than impossible.

---

## 3. Throughput budgets — the asynchronous ingest path (audit)

### 3.1 Ingest is async by contract

- **N-G1-110 (MUST):** Per B5-R3, evidence emission (steps 7–10 of the enforcement flow) MUST NOT be on the synchronous admission critical path. The decision is synchronous; the *evidence delivery* is async, buffered, best-effort-with-retry (B1-R28). **Consequence: a slow or backpressured ingest pipeline never delays or fails an admission decision.** It can, however, drop or delay *evidence*, which is a compliance-integrity problem (see §6 backpressure).

### 3.2 Ingest throughput targets

| Metric | POC (1×) | 10× | 100× |
|---|---|---|---|
| Avg ingest rate | 6 events/s | 58 events/s | 580 events/s |
| Burst ingest rate | 50 events/s | 500 events/s | 5,000 events/s |
| Ingest workers (per §22 / F3) | 1–2 | 4–8 | 20–40 (sharded, see §7) |
| Per-event normalize+hash+chain cost (budget) | ≤ 5 ms CPU | ≤ 5 ms | ≤ 5 ms |

- **N-G1-111 (MUST):** A single normalizer worker MUST sustain **≥ 200 events/sec** (normalize + canonicalize per RFC 8785 JCS + SHA-256 content_hash + chain append). At ≤ 5 ms/event CPU this is comfortably met on the reference 4-vCPU worker (a single SHA-256 over a ~2 KB event is microseconds; JCS canonicalization dominates at ~0.5–2 ms). 200/sec/worker means: POC needs 1 worker, 100× needs ~25 workers at avg and ~25 *sharded* chains to absorb the 5,000/sec burst (see §3.3, the bottleneck).

### 3.3 THE BOTTLENECK — hash-chain serialization (per-source append-only log)

- **N-G1-112 (CRITICAL):** C2 §7.3 (N-C2-301) mandates a **per-source append-only chain**: `prev_hash` of event N = `content_hash` of event N−1, with a **strictly monotonic `chain_seq`**. This is a **serialization point**: to append event N you must know event N−1's hash, so appends to one chain are **inherently sequential** — they cannot be parallelized within a single chain. A single sequential SHA-256-and-append loop tops out around **5,000–20,000 appends/sec** on the reference node (memory-bound; faster if the prior hash is hot in cache, slower if each append round-trips to durable storage with fsync, which can drop it to **hundreds/sec**).
  - **At POC (6/s avg, 50/s burst):** trivially fine on one chain.
  - **At 100× (580/s avg, 5,000/s burst):** a **single global chain with durable-fsync-per-append is the binding constraint** and may not keep up with burst. The chain, not the eval engine, is the scale ceiling of the *central* spine. This is exactly the META-ADVERSARIAL finding #2 concern and the ADVERSARIAL §XD-G1-3 defect.
- **N-G1-113 (MUST):** To scale the chain, the source key MUST be **sharded** so that "per-source" means per `(source.system, cluster, shard)` rather than one global chain. C2 §8.1 already partitions the store by `(source.system, time)`; G1 requires the **chain identity** to be sharded the same way, giving N independent sequential chains that append in parallel. With S shards each sustaining ~5,000 appends/sec, aggregate ingest = S × 5,000/sec. **100× (5,000/s burst) needs only S ≈ 1–2 shards if durability is batched, or S ≈ 10–25 if every append is fsynced individually.** See ALT for the async-batched-chain alternative that removes the per-append fsync entirely.
- **N-G1-114 (MUST):** Checkpoint signing (C2 §7.4, N-C2-302, ed25519 every 10,000 events / 15 min) is **off the append path** — it runs over a chain segment asynchronously. ed25519 sign is ~50 µs; even at 100× (5,000 events/s → one 10k-event checkpoint every 2 s) signing cost is negligible (~50 µs every 2 s). **Signing is NOT a bottleneck; the sequential append is.** (This corrects a common mis-attribution — see ADVERSARIAL XD-G1-7.)

---

## 4. Storage growth & event-size math

### 4.1 C2 event size (the 41-field event)

The C2 frozen schema lists 36 numbered core fields (§3.1–3.6); with the `disposition`/`obligations[]` axis the cross-cut reconciliation adds (META-ADVERSARIAL M-1 / XD-3), the event is **~41 logical fields**. Size is dominated not by field count but by four **embeddable large objects**: `request_object`, `before_state`, `after_state`, and external-data `value_ref` payloads.

| Event shape | Inline size (canonical JSON) | Notes |
|---|---|---|
| Skinny event (decision-only, large objects by reference) | **~1.5–2.5 KB** | identity/envelope/decision/scope/integrity fields, `request_object` stored as `cas://` digest. |
| Fat event (small `request_object` inlined, e.g. a Deployment) | **~4–8 KB** | the canonical §3.8 example is ~3 KB. |
| Worst case (full `before_state`+`after_state`+`request_object` inlined) | **20–100 KB** | MUST be content-addressed out-of-line per C2 §8.4. |

- **N-G1-120 (MUST):** Large objects (`request_object`, `before_state`, `after_state`, external-data values) MUST be **content-addressed to a CAS blob store** when they exceed a configured inline threshold (default **4 KB**), with only the `sha256` digest inline (C2 §8.4). This keeps the indexed event-store row **bounded at ~2.5 KB** regardless of payload size — critical because the index DB (not the CAS) is the expensive, hard-to-scale tier.
- **D-G1-SIZE:** Capacity math uses **2.5 KB/event indexed + an average 3 KB/event of CAS blob** (many decisions reference a shared image-digest blob; CAS dedups identical `request_object`s and external-data values, so realized CAS bytes are *lower* than naïve event×blob). Net **~5 KB/event of stored bytes** matches F3's "30–75 GB / 15M events" (= 2–5 KB/event), validating the assumption.

### 4.2 Storage growth formula

```
indexed_bytes/day   = events/day × indexed_event_size (2.5 KB)
cas_bytes/day       = events/day × avg_blob_size × (1 − cas_dedup_ratio)
total_bytes/day     = indexed_bytes/day + cas_bytes/day
stored_bytes(window)= total_bytes/day × retention_days × (1 + index_overhead)
```
where `index_overhead` ≈ 0.4–1.0 for the secondary indexes C2 §8.1 mandates (correlation_id, control_id, scope×3, resource_id, policy_version, replay_completeness, timestamp — **~9 secondary indexes**, a material multiplier the corpus never costed).

### 4.3 Worked storage examples

| Scenario | events/day | retention | indexed (×1.7 for indexes) | CAS (3 KB, 50% dedup) | **Total** |
|---|---|---|---|---|---|
| **POC, 30d** | 500k | 30 d | 500k×2.5KB×30×1.7 ≈ **64 GB** | 500k×1.5KB×30 ≈ 22 GB | **~86 GB** (F3 says 30–75 GB at lower event rate — consistent) |
| **POC, 1yr** | 500k | 365 d | ≈ 776 GB | ≈ 274 GB | **~1.05 TB** |
| **10×, 1yr** | 5M | 365 d | ≈ 7.8 TB | ≈ 2.7 TB | **~10.5 TB** |
| **100×, 1yr** | 50M | 365 d | ≈ 78 TB | ≈ 27 TB | **~105 TB/yr** |
| **100×, 7yr (SOC2/finserv)** | 50M | 2,555 d | ≈ 543 TB | ≈ 191 TB | **~735 TB** |

- **N-G1-121 (MUST):** Because the **9-secondary-index overhead (~1.7×) applies only to the indexed tier**, the architecture MUST support **tiering**: hot indexed store for the recent re-normalization/analytics window (≥30 d, C2 §8.3) and **cold, compressed, index-light archival** (object storage) for the multi-year retention tail. At 100×/7yr, keeping 735 TB *fully indexed in the hot DB* is economically irrational; only the recent window needs the 9 indexes. **This tiering boundary is owned jointly with G2 (cost) and G7 (lifecycle); G1 owns the performance contract that cold-tier queries are allowed to be slow (§5).**

---

## 5. Analytics, replay & reporting query-latency targets

These are **asynchronous, non-user-blocking** paths (except interactive GUI queries). Targets are tiered by interactivity.

| Query class | Owner | Scope | Target latency | Mode |
|---|---|---|---|---|
| GUI interactive event lookup (by correlation_id / event_id) | C2 §10 / E2 | single flow | **p99 ≤ 300 ms** | indexed point query |
| GUI interactive scoped event list (namespace × control, 1 page) | C5/C2 | ≤ 1k rows | **p99 ≤ 1 s** | indexed range query |
| Coverage-gap matrix (namespace × control, DT-80) | C5+C3 | scope-bounded window | **≤ 5 s** | pre-aggregated from C3 feed |
| C3 detector reconciliation (one 15-min window) | C3 | one window in scope | **≤ window interval (15 min)** so it never falls behind | batch |
| Single-policy simulation over 10k events (R-F3-PERF-1) | E1 | 10k events | **interactive (≤ 30 s) or short bg job** | replay |
| **30-day differential replay (DT-46)** | E1 | up to 30d of scope-filtered events | **bg job; ≤ minutes for namespace scope, longer for cluster** | replay |
| Weekly exec report render + sign (DT-34) | C5+C2 | 7-day window | **≤ 60 s** scheduled | batch + sign |

### 5.1 The replay cost model (E1) — the second-biggest scaling risk

E1's differential algorithm (E1 §4) is, for evidence set S and the 2 bundles (previous, new):

```
cost(differential) = |S| × (2 × eval_cost + reconstruct_cost + classify_cost)
```

- **N-G1-130 (MUST):** Differential replay is **O(events × bundles)** with `bundles = 2`. This is *linear in events*, which is fine — **but the ADVERSARIAL review flags the real risk: O(events × candidate_bundles) when a user batch-evaluates a candidate policy set, and O(events × policies-in-bundle) if eval is not memoized.** The mitigations:
  - **N-G1-131 (MUST):** Replay MUST be **namespace-scoped by default** (R-F3-PERF-2); full-cluster replay only for small POC datasets. Namespace scoping bounds `|S|` to a single tenant's events (10²–10⁴, not 10⁷).
  - **N-G1-132 (MUST):** The `EvidenceSet` MUST be **materialized once, immutably, with a digest** (E1 §4, C2 §8.5) and reused across runs (engineering replay + auditor walkthrough). Materialization is the expensive step (one scan of the store); re-running a different candidate bundle over the *same* materialized set avoids re-scanning storage.
  - **N-G1-133 (MUST):** Replay eval MUST run on a **horizontally scalable worker pool** (F2 line 38: "scale workers for replay jobs"). E1 eval is embarrassingly parallel across events (each event is independent); a 30-day, 5M-event cluster-scope replay parallelizes across W workers at `|S|/W × 2 × eval_cost`. **This is the parallelism META-ADVERSARIAL line 65 worried E1's "serial core" forbade — G1 clarifies: the *differential-diff authoring* may be serial, but the *evaluation loop over events is parallel* and MUST be parallelized.**

### 5.2 Worked replay example

- **POC, 30-day namespace replay, 1 namespace ≈ 50k events, 1 worker, eval ~2 ms:** 50k × 2 evals × 2 ms = 200 s materialize+eval ≈ **3–4 min** → "short background job" ✓ (R-F3-PERF-1).
- **100×, 30-day cluster replay, 50M events/day × 30 = 1.5B events:** at 2 ms × 2 evals = **~70 worker-days single-threaded**. **This is the O(events) explosion the ADVERSARIAL review names.** Mitigation: (a) namespace-scope to ~1.5B/Nnamespaces, (b) parallelize across W=100 workers → ~17 hrs, (c) **only replay `complete` events** (insufficient events are skipped per E1 §4), (d) **sample** for advisory diffs (ALT). **Cluster-wide 30-day replay at 100× is not interactive and arguably should be disallowed** — a hard product boundary, not a perf bug.

---

## 6. Backpressure & load-shedding (degrade predictably)

The platform has one synchronous path (admission) and several async pipelines (ingest, analytics, replay, reporting). Each needs a defined behavior when overloaded.

### 6.1 Admission path (synchronous) — shed by failing closed/open per policy

- **N-G1-140 (MUST):** Under overload the admission webhook does **not** queue — it is hard-deadline-bounded (§2). If OPA eval or external-data exceeds budget, the request **falls back to `failurePolicy`** (B2-R14): `Fail`/deny for prod runtime (fail-closed, secure), `Ignore`/allow for system namespaces (availability). There is **no backpressure on admission** — load is shed by the K8s timeout itself. The capacity lever is **webhook replicas** (B2/F2: 2+ replicas, scale horizontally).

### 6.2 Ingest path (async) — buffer, then shed *evidence* (never decisions)

- **N-G1-141 (MUST):** The evidence emitter (B1-R28) MUST buffer with a **bounded local queue + disk spill**. When the buffer fills (ingest backpressured), behavior is, in order: (1) spill to local disk, (2) emit a `D-LATENCY`/`audit_latency` signal (C3 §2.9) so the lag is *itself audited*, (3) **never** block or fail the originating decision. **Shedding evidence is a compliance-integrity event, not a silent drop** — a dropped/delayed event MUST be recorded as a coverage gap (C3 `coverage_gap`) so "we lost evidence here" is queryable. This is the honesty-over-coverage tenet (C2 §1.3) applied to backpressure.
- **N-G1-142 (MUST):** The ingest queue depth and oldest-unprocessed-event age (`ingest_lag_seconds`) are first-class SLIs (§8) with alerting thresholds. `ingest_lag` > the C3 reconciliation interval (15 min) means detection is operating on stale data — an operational alarm.

### 6.3 Analytics / replay / reporting (async batch) — admission-control jobs

- **N-G1-143 (MUST):** Replay/report jobs (E1/C5) MUST run on a **bounded worker pool with a job queue** and **per-tenant fairness** (G5 dependency — no single tenant's 30-day cluster replay starves others). A job exceeding a resource budget MUST be **checkpointed/paged or rejected at submit time** with an estimated-cost preflight (`|S| × 2 × eval_cost` is knowable before running). **N-G1-144 (MUST):** the API MUST reject a replay whose estimated cost exceeds a configured ceiling and suggest narrowing scope, rather than accept-and-OOM.

---

## 7. Per-component scale map — horizontal vs bottleneck

| Component | Hot path role | Scales horizontally? | Bottleneck / serialization point | Lever |
|---|---|---|---|---|
| **B1 OPA eval** | sync decision | **Yes** — stateless, per-replica/sidecar | none (CPU-bound, edge-served) | replicas / sidecars |
| **B2 Gatekeeper webhook** | sync decision | **Yes** — webhook replicas, per spoke | external-data provider latency (§2.2 step 5) | replicas + extdata cache |
| **External-data provider** | sync, in hot path | depends on provider | **provider RTT is in the 2 s budget** (XD-G1-1) | cache TTL, async refresh |
| **C2 normalizer/ingest** | async | **Yes** — N workers | **per-source hash chain append (N-G1-112)** | shard the chain (N-G1-113) |
| **C2 event store (index DB)** | async query | partially — read replicas | **write-serialized chain + 9 indexes** (N-G1-121); single-writer per shard | shard by (source,cluster); tier cold data |
| **C2 CAS blob store** | async | **Yes** — object storage is effectively infinite | none | n/a (cost only, G2) |
| **C2 checkpoint signer** | async, off-path | single signer (key custody) | ed25519 sign is ~50µs — **not** a bottleneck (N-G1-114) | n/a |
| **C3 analytics** | async batch | **Yes** — partition detectors/windows | 15-min window must finish in 15 min; chain-verify is a full scan | parallelize per scope/detector |
| **E1 simulation** | async batch | **Yes** — eval loop parallel across events (N-G1-133) | EvidenceSet materialization (one store scan); O(events) | parallel workers; materialize-once |
| **C5 reporting** | async/interactive | **Yes** — stateless render | aggregation scans; signed export is one Merkle pass | pre-aggregate; reuse C3 feeds |
| **F1 API** | sync, thin | **Yes** — HPA on CPU/QPS (F2 line 37) | downstream store | replicas |
| **D1 JWT mapping** | sync, in hot path | **Yes** — in-proc cache | IdP availability on cache miss | cache + IdP HA |
| **Operator/CRD controllers** | control | single active (leader-elected) | reconcile loop; not throughput-critical | n/a |

**The #1 bottleneck is the per-source hash-chain append (N-G1-112).** It is the *only* mandatory serialization point on the central evidence spine, it is the property that makes the audit log tamper-evident (so it cannot simply be removed), and it is unsharded in the C2 spec as written. Everything else scales horizontally.

---

## 8. Benchmarking & load-test plan (named SLIs)

### 8.1 Service Level Indicators (the named SLIs)

| SLI | Definition | Target (POC) | Target (100×) | Owner |
|---|---|---|---|---|
| `admission_p50_ms` / `admission_p99_ms` | end-to-end webhook latency | 150 / 1000 ms | 150 / 1000 ms (unchanged — edge-served) | B2 |
| `opa_eval_p99_ms` | OPA eval only | 80 ms | 80 ms | B1 |
| `extdata_call_p99_ms` | external-data provider call | 400 ms | 400 ms | B2 |
| `extdata_cache_hit_ratio` | hot-path cache hits | ≥ 95% | ≥ 95% | B2 |
| `admission_timeout_rate` | fraction hitting `timeoutSeconds` | < 0.01% | < 0.01% | B2 |
| `ingest_throughput_eps` | events/sec sustained per worker | ≥ 200 | ≥ 200 | C2 |
| `chain_append_eps` | appends/sec per chain shard | ≥ 5,000 | ≥ 5,000 | C2 |
| `ingest_lag_seconds` | age of oldest unprocessed event | < 60 s | < 900 s | C2 |
| `evidence_drop_rate` | shed events / total (coverage gap) | 0 (target) | < 0.1% with audited gap | C2/C3 |
| `c3_window_completion_ms` | one 15-min reconcile duration | < 15 min | < 15 min | C3 |
| `replay_events_per_worker_sec` | replay eval throughput | ≥ 500 | ≥ 500 | E1 |
| `report_render_p99_s` | scheduled report render+sign | < 60 s | < 120 s | C5 |
| `gui_query_p99_ms` | interactive scoped list | < 1000 ms | < 1500 ms | C2/E2 |

### 8.2 Load-test scenarios

- **LT-1 Admission soak:** drive 1,000 evals/sec (POC) / 100k/sec (100×) through Gatekeeper with a mix of 70% no-external-data, 30% external-data policies, at 0%/50%/95% cache-hit. **Pass:** `admission_p99_ms ≤ 1000`, `admission_timeout_rate < 0.01%`. **Specifically test the 0%-cache-hit cold-start** (provider cold) — this is where XD-G1-1 says the budget blows.
- **LT-2 External-data brownout:** inject 1 s / 5 s provider latency and provider 50x errors mid-soak. **Pass:** webhook falls back to `failurePolicy` within budget; **no** request exceeds `webhook_timeout`; the timeout/fallback is audited.
- **LT-3 Ingest burst:** 50/sec (POC) / 5,000/sec (100×) burst for 10 min into 1 vs N chain shards. **Pass:** `ingest_lag_seconds` recovers to < 60 s within one minute of burst end; **measure the single-chain ceiling explicitly** to validate N-G1-112.
- **LT-4 Chain-integrity verification scan:** full chain re-verify over 15M (POC) / 1.5B (100×) events. **Pass:** completes within the C3 reconciliation budget; record throughput (verify is the same sequential-hash cost as append).
- **LT-5 30-day replay:** namespace-scope (POC) and cluster-scope (must be *rejected* at 100× per N-G1-144). **Pass:** namespace replay ≤ minutes; cost-preflight correctly rejects oversized cluster replay.
- **LT-6 Reporting at scale:** weekly exec report (DT-34) + signed export over a 7-day window at each tier. **Pass:** ≤ 60 s (POC) / ≤ 120 s (100×).
- **LT-7 Storage growth validation:** run for a fixed window, measure realized bytes/event and CAS dedup ratio; **validate the §4 model** (target 2–5 KB indexed, 50% CAS dedup).

### 8.3 Benchmark harness requirements

- **N-G1-150 (MUST):** A reproducible load-test harness (k6/Locust for admission + a synthetic event generator for ingest) MUST live in the repo and run in CI as a **non-gating nightly perf job** at POC scale, with a **scale-multiplier flag** to run 10×/100× on demand against a provisioned cluster.
- **N-G1-151 (MUST):** Every SLI in §8.1 MUST be exported as a Prometheus metric (G6 dependency) so the load tests assert against *the same* metrics ops watches in production — no separate measurement path.

---

## 9. Capacity-planning model (formulas)

### 9.1 Admission tier

```
webhook_replicas      = ceil(peak_evals_per_sec / per_replica_eval_capacity)
per_replica_eval_capacity ≈ 1000 / opa_eval_p50_ms × concurrency  (e.g. 1000/10 × 8 = 800 eval/s/replica)
extdata_provider_qps  = peak_evals_per_sec × extdata_policy_fraction × (1 − cache_hit_ratio)
```
Worked: **100× peak 100k eval/s** → `webhook_replicas = ceil(100000/800) = 125` per fleet (spread across spokes). `extdata_provider_qps = 100000 × 0.3 × 0.05 = 1,500 qps` to the provider — **provision the provider for 1,500 qps, not 30,000**, because of the 95% cache.

### 9.2 Ingest tier

```
ingest_workers   = ceil(burst_events_per_sec / per_worker_eps)          # per_worker_eps ≥ 200
chain_shards     = ceil(burst_events_per_sec / per_shard_append_eps)     # per_shard ≈ 5000 batched
```
Worked: **100× burst 5,000 events/s** → `ingest_workers = ceil(5000/200) = 25`; `chain_shards = ceil(5000/5000) = 1` if batched-durability, but **≥ 10 recommended** for headroom + parallel verify + per-cluster isolation.

### 9.3 Storage tier (see §4.2 formula)

Worked POC: `500k × 2.5KB × 30 × 1.7 = 64 GB indexed + 22 GB CAS = 86 GB`. Worked 100×/7yr: **~735 TB** → mandates cold tiering (N-G1-121).

### 9.4 Replay tier

```
replay_workers = ceil(target_replay_seconds_budget⁻¹ × |S| × 2 × eval_ms / 1000)
```
Worked 100× cluster 30-day: |S|=1.5B, eval 2 ms → 70 worker-days single-threaded → **disallow cluster-scope; namespace-scope to ~5M events → ~5.5 hrs on 1 worker, ~3 min on 100 workers.**

---

## 10. Dependencies & open questions

### 10.1 Dependencies
- **C2** (event size, hash-chain, checkpoint, storage contract) — G1 sizes it; G1 *requires* the chain be shardable (N-G1-113).
- **B2/B1** (admission budget, external-data cache) — G1 sets the budgets; B2 enforces timeouts.
- **E1** (replay cost) — G1 requires parallel eval + cost-preflight.
- **C3/C5** (query latency) — G1 sets targets.
- **F2** (topology, replica counts) — G1's capacity model feeds F2's deployment sizing.
- **G2** (cost/$ per event from §4 volumes), **G3** (DR/RPO over the same store), **G5** (per-tenant fairness in §6.3), **G6** (SLI metrics), **G7** (tiering/lifecycle boundary).

### 10.2 Open questions (decided defaults)
- **OQ-G1-1:** Single global chain vs sharded chain? → **Sharded by `(source.system, cluster)`** (N-G1-113). Rationale: removes the only hard serialization ceiling; matches C2's existing partition key. *Requires C2 to clarify chain identity = shard identity (a contract note, not a schema change).*
- **OQ-G1-2:** Per-append durable fsync vs batched group-commit? → **Batched group-commit** (default 10 ms / 100 events) — trades ≤10 ms of evidence-loss-on-crash window for 10–100× append throughput. The lost window is bounded and *audited* as a coverage gap (N-G1-141). See ALT for the fully-async variant.
- **OQ-G1-3:** Is cluster-wide 30-day replay a supported product feature at 100×? → **No** (N-G1-144 rejects it); namespace-scope or sample. A hard product boundary.
- **OQ-G1-4:** Reference instance sizing? → **m6i.xlarge-class** (D-G1-HW), stated as an assumption to be validated by LT-1..LT-7.
