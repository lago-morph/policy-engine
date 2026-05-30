# G6 — Observability, SLOs & Day-2 Operations — SPEC

**Component:** G6 · **Domain:** G (Operational / NFR wave) · **Spec source:** META-ADVERSARIAL §3 (Risk #10 "day-2 operational burden of a 14-service stack is unowned"; Risk #8 "C2's first real event is an unplanned breaking migration") · F2 (service inventory, operators, CRDs) · C2 / C2-v1.0-rc-RECONCILED (the schema migration this component owns the runbook for) · B4 (CRD versioning) · G1 (capacity/SLIs, hand-off target) · G3 (availability/DR/resilience, failure-mode reference) · G4 (key management, rotation hand-off target).
**Status:** Authored · **Date:** 2026-05-30
**Persona lens:** **Jess** — the small (2–4 person) SRE/platform team at a regulated mid-market buyer who must *run* the governance platform that governs everyone else. If Jess can't operate it on-call without a dedicated team, the buyer churns regardless of feature quality.

> **One-line thesis.** The platform governs 14 other services; G6 answers *"who governs the platform, who watches the watchmen, and can a 2–4 person team keep it alive on-call?"* It owns three things the functional corpus never did: **(1) self-observability** (the platform's own golden signals, kept cryptographically and operationally distinct from customer audit evidence), **(2) SLOs + error budgets** for the three surfaces that matter (admission hot path, audit ingest, console), and **(3) day-2 runbooks** — chief among them the **C2 1.0→1.0-rc schema migration of a tamper-evident append-only hash-chained log**, which is impossible *in place* and is done instead as a **chain-epoch boundary** (never rewrite history; append a re-normalized continuation chain and cross-sign the old root into the new genesis).

---

## 1. Scope

**In scope.**
1. **Self-observability of the platform** — metrics, traces, logs the platform emits *about itself* (not about customer workloads); the golden-signal set per service; the hard separation between **platform telemetry** (operational, mutable, short-retention, the SRE's) and **customer audit evidence** (C2, tamper-evident, long-retention, the auditor's).
2. **SLO / error-budget framework** — SLIs, example SLOs, burn-rate alerting, and the error-budget policy for the admission hot path, audit ingest, and the console.
3. **Day-2 runbooks** (numbered, executable) — upgrade/rollback choreography for the ~14-service stack; **CRD version migration** and the **C2 append-only-log schema migration**; certificate/secret rotation (hand-off to **G4**); capacity scaling (hand-off to **G1**); on-call model and incident response.

**Out of scope (owned elsewhere, referenced here).**
- *Capacity sizing math / autoscaling targets* → **G1** (G6 defines the *signal* that triggers a scale event and the *runbook*; G1 owns the numbers).
- *Key custody, HSM/KMS, signing-key rotation cryptography* → **G4** (G6 owns the *operational hand-off*: when to rotate, how to drain, how to verify the chain still verifies after a key roll).
- *DR / RPO / RTO / backup-and-prove-the-chain* → **G3** (G6 references G3's RPO/RTO targets in the incident runbooks; G3 owns the durability contract).
- *Retention economics / cost model* → **G2** (G6's telemetry retention budget is sized here; multi-year evidence retention cost is G2).
- *Multi-tenant isolation* → **G5** (G6's per-tenant dashboards/alerts consume G5's tenant boundary; it does not define it).
- *Install-time topology* → **F2** (G6 is explicitly **day-2**, the gap F2 §4 left: "F2 is install-time only").

---

## 2. Self-observability — "who watches the watchmen"

### 2.1 The core separation: platform telemetry ≠ customer audit evidence (R-G6-OBS-1)

The single most important rule in this component. The platform emits two utterly different streams that the corpus has so far conflated by omission:

| Property | **Platform telemetry** (G6) | **Customer audit evidence** (C2) |
|---|---|---|
| Question it answers | "Is the platform healthy? why is admission slow?" | "Did control SC-IMG-001 operate effectively on 2026-05-12?" |
| Audience | Jess (SRE), platform on-call | The customer's auditor (PCAOB/SOC2), the customer's compliance team |
| Integrity model | Best-effort, **mutable**, sampled, lossy under pressure | **Tamper-evident**: hash-chained, signed Merkle checkpoints (C2 §7) |
| Retention | Short (13mo metrics, 7–30d traces/logs — §2.5) | Long (multi-year; G2 owns the economics) |
| Storage | Prometheus/Mimir + Tempo + Loki (or vendor equiv.) | C2 append-only object-store log + index DB (C2-OQ1) |
| Sampling | **MUST** be sampled/dropped under load | **MUST NEVER** be sampled/dropped (audit-ingest is lossless — §4) |
| Cardinality | High (per-pod, per-request) | Controlled (per the C2 schema) |
| On loss | Page, but no compliance impact | Compliance incident; G3 DR territory |

- **R-G6-OBS-1 (MUST):** Platform telemetry and customer audit evidence MUST flow through **physically separate pipelines, separate stores, and separate access-control scopes.** A platform-telemetry outage MUST NOT drop a single C2 audit event, and conversely a flood of telemetry MUST NOT be allowed to back-pressure the audit-ingest path. They share *no* queue, *no* store, and *no* signing key.
- **R-G6-OBS-2 (MUST):** No customer audit content (jwt_claims, request_object, before/after state, external-data values, control verdicts) may ever appear in platform telemetry. Telemetry that needs to reference an event references it by **opaque id only** (`event_id`, `correlation_id`) — never by content. This is both a privacy control (G7) and the integrity firewall: the SRE's mutable dashboards must never become a second, un-chained copy of the evidence.
- **R-G6-OBS-3 (MUST):** The reverse leak is also forbidden — platform operational metrics MUST NOT be written into the C2 chain. The chain is for governance decisions, not for "ingest worker p99 latency." (A naive implementation that "audits everything" would pollute the evidence chain with operational noise and inflate G2's retention cost.)

### 2.2 The recursion problem: the platform observes itself with the same tools it sells

The platform's enforcement (Gatekeeper/OPA admission) runs **in the same cluster** it governs. So the platform is subject to its own admission webhooks. This creates a bootstrap/deadlock surface that ordinary apps don't have:

- **R-G6-OBS-4 (MUST):** The `governance-system` namespace (F2 §2.1, the hub) MUST be **exempt from its own enforcing policies** by a break-glass exclusion, OR the admission webhook MUST `failurePolicy: Ignore` for the governance namespace specifically. Otherwise a bad policy push can lock the operator out of fixing the bad policy push (a self-inflicted P0 with no remediation path). This exemption is itself audited (it is a governed exception, recorded as a C2 event with `obligations:[exception]`).
- **R-G6-OBS-5 (MUST):** Platform self-monitoring (the Prometheus/agent scraping the platform) MUST run **outside the platform's own control loop** — a separate, minimal monitoring stack that does *not* depend on Keycloak for auth, does *not* depend on the operator, and does *not* depend on the evidence store. If the platform is down, its monitoring must still be up to tell you *why*. (Falsifier for a bad design: "the dashboard that tells me OPA is down is itself behind OPA-gated SSO." That is the watchmen-watching-themselves trap.)

### 2.3 Golden signals per service (R-G6-OBS-6)

Every one of the ~14 services emits the four golden signals (latency, traffic, errors, saturation), plus a per-service **liveness/readiness** distinction. The table is the conformance floor — a service is not "done" until it emits these.

| Service (F2 §2.2) | Latency SLI | Traffic | Errors | Saturation | Critical custom signal |
|---|---|---|---|---|---|
| **Gatekeeper / OPA (admission)** | webhook eval p50/p95/p99 | admission reqs/s | webhook 5xx + timeout rate | eval queue depth, CPU | **webhook timeout rate** (a timeout = fail-open/closed event — couples to G3) |
| **Kyverno (optional effector)** | mutation/validate latency | admission reqs/s | policy error rate | CPU | mutation failure rate |
| **Audit normalizer / ingest (C2)** | ingest lag (event ts → ingest ts) | events/s in | normalize error rate, **drop count (MUST be 0)** | queue depth, backlog age | **lossless-ingest invariant** (drop count, chain-gap count) |
| **Evidence store (C2)** | write p99, read p99 | writes/s, reads/s | write failures, **chain-verify failures** | disk %, IOPS, object-store 429s | **chain-integrity check pass/fail** (continuous, C3) |
| **Governance API (F1)** | request p95/p99 by route | RPS | 4xx/5xx | replica CPU, conn pool | authz-deny rate (D2) |
| **Console / Headlamp backend (E2)** | page-load + API p95 | active users, RPS | JS errors, API 5xx | backend CPU | live-user count |
| **Simulation engine (E1)** | job completion time | jobs/s, queue len | job failure rate | **worker pool saturation** (serial core — META G-2/MASTER-PLAN-ALT) | replay-`complete` fraction (gating AC, G-2) |
| **Analytics (C3)** | detection lag | events processed/s | detector error rate | worker CPU | chain-verify finding rate |
| **Operator + CRD controllers (F2)** | reconcile latency | reconciles/s | reconcile error rate, requeue rate | work-queue depth | **CRD conversion-webhook error rate** (critical during migration — §5.2) |
| **Keycloak / IdP (D1)** | token-issue p95 | auth reqs/s | auth failures | session count | token-issuance error rate (blocks *everything* if down) |
| **Privateer** | eval latency | evals/s | eval errors | CPU | collector lag |
| **Approval controller (B4/D3)** | approval-CR reconcile | open approvals | controller errors | queue | **stuck-approval age** (a deadlocked approval blocks a deploy) |
| **Bundle/plugin controller (F2)** | bundle-verify latency | pulls/s | **signature-verify failures** | CPU | unsigned-bundle-refused count (supply chain) |
| **Export adapter (F2/C5)** | export latency | exports/s | export failures | CPU | signed-export failure rate |

- **R-G6-OBS-6 (MUST):** Every service emits its row of golden signals on a `/metrics` endpoint (OpenMetrics/Prometheus) and a readiness probe distinct from liveness. A service whose readiness depends on a *downstream* service MUST expose **both** ("I am alive" vs "my dependency is down") so cascading-failure root cause is one dashboard hop, not a guess.
- **R-G6-OBS-7 (SHOULD):** **Distributed tracing** spans the governance transaction end-to-end using the **retry-stable `governance_transaction_id`** (C2 field 40) and `correlation_id` (field 3) as the trace/span correlation keys — so an SRE can pull the full admission→deny→approval→retry→admit flow as one trace. This reuses the exact join keys C2 already defines (no new primitive); the trace is platform telemetry (mutable, short-lived), distinct from the C2 chain that records the same flow as evidence.
- **R-G6-OBS-8 (SHOULD):** A single **"is the platform healthy?" status page** rolls the 14 services' readiness into one composite, with explicit *dependency ordering* (Keycloak → API → console; normalizer → store) so a red light points at a cause, not a symptom.

### 2.4 Self-observability of the integrity mechanism itself

The tamper-evidence (hash chain, signed checkpoints) is the product's reason to exist; its health is a first-class signal:

- **R-G6-OBS-9 (MUST):** A **continuous chain-integrity monitor** (the C3 chain-verification check, N-C2-301/§7.3) exposes, as platform telemetry: time-since-last-successful-full-chain-verify, chain-gap count, broken-link count, and **time-since-last-signed-checkpoint** (N-C2-302). Any non-zero gap/break, or a checkpoint cadence miss (default N=10 000 events / T=15 min), pages immediately — this is the single highest-severity platform alert because a silently broken chain voids evidence the customer is paying to be able to trust.
- **R-G6-OBS-10 (MUST):** **Signing-key health** is a monitored signal (key age vs G4 rotation policy, checkpoint-signing success rate, key-id in use). Hand-off to G4 for the rotation *procedure*; G6 owns the *alert* that says "key is approaching rotation age" and "checkpoint signing is failing."

### 2.5 Telemetry retention budget (sized here, costed by G2)

- **R-G6-OBS-11 (SHOULD):** Default retention: **metrics 13 months** (so YoY SLO trends and one full audit cycle are visible), **traces 7 days** (debugging window; sampled at 1–10% steady-state, 100% on error), **platform logs 30 days**. These are operational defaults, *independent* of and far shorter than C2 evidence retention (multi-year, G2). This separation is what keeps platform-telemetry cost bounded while evidence retention grows.

---

## 3. SLOs and error budgets

### 3.1 Framework (R-G6-SLO-1)

- **R-G6-SLO-1 (MUST):** Each SLO is `(SLI definition, objective %, measurement window, error budget, burn-rate alerts, error-budget policy)`. SLIs are computed from §2 golden signals. Objectives below are **POC/MVP defaults** sized to the §22 envelope (100–1,000 evals/s, 500k events/day ≈ 6/s avg / 50/s burst, 5–50 console users); production targets are renegotiated with the design partner and inherit G1's scale numbers.
- **R-G6-SLO-2 (MUST):** SLOs are **per surface, not global.** A 99.9% "platform uptime" number is meaningless when the admission hot path, the audit ingest, and the console have utterly different criticality and failure semantics. The three surfaces below are budgeted independently.

### 3.2 The three SLO surfaces

#### Surface 1 — Admission hot path (the highest-stakes, lowest-latency surface)

The platform is *inline* in the customer's deploys. If admission is slow, every deploy in every governed namespace is slow; if it fails the wrong way, deploys either break (fail-closed) or go ungoverned (fail-open). This is where the platform can hurt the customer most.

| | |
|---|---|
| **Availability SLI** | fraction of admission requests answered (allow/deny/mutate) **within the webhook timeout** (not 5xx, not timed-out) |
| **Availability SLO** | **99.9%** over 30 days (≈ 43 min/mo budget) for fail-closed (high-assurance) namespaces; **99.5%** for fail-open namespaces |
| **Latency SLI** | webhook evaluation p99 |
| **Latency SLO** | **p99 < 250 ms** added latency at the webhook (admission must feel instant) |
| **Error budget** | 0.1% of admission requests/30d |
| **Why this number** | A webhook timeout forces the `failurePolicy` decision (F2 OQ-4): fail-closed namespaces block deploys (availability pain), fail-open namespaces go ungoverned (a **silent control bypass** C3 must detect — couples G3). Both are budget-burning events. The tighter 99.9% for fail-closed reflects that *its* failure visibly breaks the customer's deploys. |
| **Alert (multi-window burn rate)** | Fast: 2% budget burned in 1h → page. Slow: 5% in 6h → page. 10%/3d → ticket. (Google SRE multi-window/multi-burn-rate.) |

#### Surface 2 — Audit ingest (the lossless surface — the evidence spine)

| | |
|---|---|
| **Completeness SLI** | `1 − (dropped_events / received_events)` — the lossless invariant |
| **Completeness SLO** | **100.000% (zero dropped events), hard.** Not a percentage to spend down — a **dropped audit event is a compliance incident**, not budget burn. |
| **Freshness SLI** | ingest lag = `ingest_timestamp − timestamp` (C2 fields 6,5) p95 |
| **Freshness SLO** | **p95 < 60 s** (event observable in console/analytics within a minute); **p99 < 5 min**; backlog **MUST drain** (lag bounded, not growing) |
| **Integrity SLI** | chain-gap count, broken-link count, checkpoint-cadence adherence |
| **Integrity SLO** | **zero gaps, zero broken links; checkpoint within cadence** (hard) |
| **Error budget** | **There is no completeness/integrity budget.** The only budget is on *freshness* (lag), where 0.5% of events may exceed the 60 s p95 target. Completeness and integrity are step-function-fail, not budgeted. |
| **Why this asymmetry** | This is the META-ADVERSARIAL G-6 lesson made operational: an honest evidence product **cannot** treat lost evidence as "within budget." Buffer (don't drop) under back-pressure; if you truly cannot buffer, **fail the producer closed** rather than silently drop (and audit the fact). G3 owns the buffer/DR mechanism; G6 owns the SLO that forbids silent loss. |
| **Alert** | Any drop > 0 → immediate page (Sev-1). Lag p95 > 60 s for 10 min → page. Backlog age growing for 30 min → page. Chain gap/break → immediate page (Sev-1, see R-G6-OBS-9). |

#### Surface 3 — Console / read & query (the human surface — graceful-degradation allowed)

| | |
|---|---|
| **Availability SLI** | fraction of console API requests served < 5xx |
| **Availability SLO** | **99.5%** over 30 days (≈ 3.6 h/mo) |
| **Latency SLI** | console interactive API p95 |
| **Latency SLO** | **p95 < 1 s** for read/query; simulation/replay jobs are **async** with their own job-completion SLO (p90 single-policy replay over 10k events **< 60 s**, per F2 R-F2-SCALE-4) |
| **Error budget** | 0.5%/30d |
| **Why this is the loosest** | The console is **read-mostly and not inline** in any customer enforcement path. A console outage is painful for the compliance team but does **not** break deploys (admission) or lose evidence (ingest). It is allowed to degrade gracefully (stale-but-served reads) while the other two surfaces hold. This is the correct place to absorb operational pain. |
| **Alert** | 2%/1h fast-burn → page; 5%/6h → ticket. Replay-job p90 > 60 s → ticket (capacity signal → G1). |

### 3.3 Error-budget policy (R-G6-SLO-3)

- **R-G6-SLO-3 (MUST):** When a surface's error budget is exhausted, a written policy governs the response — it is not advisory:
  - **Admission hot path budget exhausted** → freeze risky changes to the admission path (no policy-engine upgrades, no webhook config changes) until budget recovers; root-cause review mandatory.
  - **Audit-ingest completeness/integrity violated (any drop or chain break)** → **Sev-1 incident**, not budget accounting; the customer is notified (it may be a reportable compliance event); evidence-gap is scoped and recorded.
  - **Console budget exhausted** → prioritize reliability work over console features next sprint; no deploy freeze (it's not safety-critical).
- **R-G6-SLO-4 (SHOULD):** SLOs are reviewed with the design partner quarterly; the *ingest losslessness* and *chain integrity* SLOs are **non-negotiable** and never relaxed — they are the product's integrity contract, not an operational dial.

---

## 4. Day-2 runbooks

All runbooks are numbered, ordered, and written for **Jess** (a 2–4 person team, on-call). The design principle (R-G6-RUN-0): **every runbook has a pre-flight check, a stepwise procedure, an explicit rollback, and a post-condition verification** — because a small team executing under pressure cannot improvise.

- **R-G6-RUN-0 (MUST):** Every day-2 procedure that touches enforcement, the evidence chain, or the signing key MUST be (a) dry-runnable, (b) reversible or explicitly-irreversible-with-a-gate, and (c) emit a C2 `policy.decision`/`obligations:[exception]` event marking the operational change, so day-2 ops are *themselves* governed and auditable.

### 4.1 Runbook: upgrade / rollback of the ~14-service stack

The stack is not independently versioned — services share contracts (the C2 schema, the F1 envelope, the CRD versions, the JWT mapping). Naive "upgrade everything" breaks contracts mid-flight. The order is dictated by the dependency DAG (F2 §7, MASTER-PLAN).

- **R-G6-RUN-1 (MUST):** Upgrades follow a fixed **dependency-ordered choreography**, leaf-contracts first:
  1. **Pre-flight:** verify chain integrity passes (R-G6-OBS-9), full backup taken (G3), error budgets healthy, no open Sev incident, CRD storage versions recorded.
  2. **Schema/contract layer first (expand phase of expand-contract):** deploy the new **C2 schema** producers/consumers in *additive, backward-compatible* mode (the 1.0-rc additive-only rule, §5) and new CRD versions in *served-but-not-storage* mode — so old and new coexist.
  3. **Identity (Keycloak/D1)** — upgrade first within the runtime tier because everything authenticates against it; verify token issuance before proceeding (it blocks everything).
  4. **Evidence store + normalizer (C2)** — upgrade with the **ingest never stopping** (R-G6-RUN-2); this is the most sensitive step.
  5. **Engines (OPA/Gatekeeper/Kyverno)** — rolling, per-spoke, one spoke at a time; the `governance-system` self-exemption (R-G6-OBS-4) must be in place first so a bad bundle can't lock you out.
  6. **Operator + CRD controllers** — leader-elected rolling upgrade; conversion webhooks must be healthy (R-G6-OBS row) before flipping any CRD storage version.
  7. **API (F1) → analytics (C3) → simulation (E1) → console (E2)** — consumers last; they tolerate the additive schema by N-C2-FWD.
  8. **Contract phase:** once all consumers read the new contract, retire the deprecated aliases (e.g. `decision`→`disposition`) on a later release — never in the same upgrade.
  - **Post-condition:** chain still verifies; all readiness green; a synthetic admission + a synthetic audit event round-trip end-to-end.
- **R-G6-RUN-2 (MUST):** During the evidence-store upgrade, **audit ingest MUST NOT stop** — the spoke-side buffer (F2 failure-mode: "spoke buffers decision logs locally") absorbs the gap; ingest resumes and drains the buffer with **no dropped events and no chain gap** (the new normalizer continues the *same per-source chain* — `prev_hash` links across the upgrade boundary). If the store must be briefly unavailable, producers buffer; they never drop.
- **R-G6-RUN-3 (MUST):** **Rollback** is per-tier and bounded by the expand-contract discipline: because the upgrade was additive (new fields/CRD-versions served alongside old), rolling a consumer *back* is safe (it ignores the new fields, N-C2-FWD). The **one-way gates** are explicitly flagged: (a) flipping a CRD *storage version* (requires conversion-back, §5.2), (b) retiring a deprecated alias, (c) a signing-key rotation (G4). These gates require their own runbook and are never bundled into a routine upgrade.
- **R-G6-RUN-4 (SHOULD):** Canary one spoke before fleet-wide engine upgrades; the hub is upgraded last among hub services and only after a spoke canary holds for a defined soak window.

### 4.2 Runbook: CRD version migration (`v1alpha1 → v1beta1 → v1`)

Owns the operational side of B4-R22 ("conversion/upgrade path before any CRD becomes a durable contract") and F2 R-F2-UPGRADE-1 (conversion webhooks).

- **R-G6-RUN-5 (MUST):** CRD migration uses the standard Kubernetes **served-versions + conversion-webhook + storage-version-flip** sequence, never a destructive replace:
  1. Add the new version as **served, not stored**; deploy the conversion webhook; verify its error rate is zero (R-G6-OBS row) against real stored objects.
  2. Migrate stored objects lazily (read-write round-trips) or with a one-shot storage-migrator job; verify all objects readable at the new version.
  3. **Flip the storage version** to the new version (the one-way gate, R-G6-RUN-3a).
  4. Mark the old version deprecated; remove it only a full release later, after confirming no client uses it.
  - **Rollback (before storage flip):** drop the served new version; trivial. **After storage flip:** requires the conversion webhook to convert *back*, which MUST be tested before the flip — otherwise the flip is irreversible and must be gated as such.
- **R-G6-RUN-6 (MUST):** CRDs that carry **enforcement-relevant durable state** (`PolicyException`, `PolicyApprovalRequest`, `GovernanceBundle`) MUST be migrated with **zero loss of in-flight state** — an open approval or an unexpired exception cannot be dropped by a conversion bug, or a deploy silently changes verdict mid-migration. The conversion webhook is fuzz-tested round-trip (`v1alpha1→v1beta1→v1alpha1` is identity) before any storage flip.

### 4.3 Runbook: C2 schema migration of an append-only hash-chained log (the 1.0 → 1.0-rc breaking change)

**This is the flagship day-2 procedure and the one META-ADVERSARIAL Risk #8 says nobody owns.** The challenge: you cannot mutate a tamper-evident append-only hash-chained, signed-Merkle log to "fix" historical events, because (a) it is append-only by invariant (N-C2-400/§7.3 — events are *never* edited in place), and (b) rewriting any past event breaks every subsequent `prev_hash` and invalidates every signed checkpoint after it (N-C2-302). **In-place schema migration of this log is not "hard" — it is forbidden by the data structure's own integrity guarantee.** Any tool that rewrites history has, by definition, performed the exact tamper the chain exists to detect.

#### 4.3.1 Why a naive migration silently breaks every prior signature

If you re-serialize old events into the 1.0-rc field set and recompute `content_hash` (now binding `catalog_version`, C2-rc field 41, §6.6), every old event's hash changes → every `prev_hash` link downstream is now wrong → every signed checkpoint that attested to the old hashes is now a signature over content that no longer exists. An auditor re-verifying the chain gets **"signature valid, but content does not match"** for all history — which is indistinguishable from tampering. The migration would *itself trip the tamper alarm*. So:

#### 4.3.2 The approach: a **chain-epoch boundary** (append, never rewrite) — R-G6-RUN-7

- **R-G6-RUN-7 (MUST):** The 1.0→1.0-rc migration is performed as a **chain epoch transition**, not a rewrite. The mechanism (which the C2 model *already supports* — this runbook operationalizes N-C2-105 re-normalization and N-C2-302 checkpoints):

  1. **Freeze the old epoch (Epoch 0).** Take a final signed Merkle checkpoint over the complete `1.0` chain (per source). This checkpoint's root is the **immutable, eternally-verifiable attestation of all v1.0 history.** v1.0 events are **never touched again.** Their signatures stay valid forever because their content never changes.

  2. **Open a new epoch (Epoch 1) with an epoch-genesis event** whose `prev_hash` is **not** the zero hash but the **final Epoch-0 checkpoint root** (the "cross-sign"). This single act cryptographically binds the new chain to the old: Epoch 1's genesis transitively attests to all of Epoch 0. The chain is *continuous across the epoch* even though the field schema changed — because the boundary is a content-addressed link, not a re-serialization. Epoch 1 events use `schema_version: "c2.audit-event/1.0-rc"`; Epoch 0 events remain `"c2.audit-event/1.0"`. Both verify independently and the boundary verifies the join.

  3. **New events emit under 1.0-rc.** No historical event is re-hashed or re-signed. Because 1.0-rc is *additive-only* (§5/C2-rc §0: all 36 v1.0 fields retained; `decision` survives as a deprecated alias of `disposition`; the only new required field is `disposition`), an Epoch-0 event read by an Epoch-1-aware consumer is still fully interpretable: the consumer maps the old flat `decision` onto `disposition`/`obligations` with the canonical table (C2-rc §1.3). **No old event needs migrating to be read** — the additive design is precisely what makes the epoch boundary lossless.

  4. **Optional, lineage-preserving re-normalization of selected old events** (only where a *replay-fidelity upgrade* is wanted — e.g. retroactively capturing an external-data value that the raw retention still holds, DT-25 backfill, N-C2-105): the re-normalized event is **appended to Epoch 1 as a NEW event** with `parent_correlation_id`/supersession-lineage to the Epoch-0 original. The original stays in Epoch 0, untouched and still signed. This is the *only* sanctioned "migration of historical data," and it is an **append, not an edit** — it adds a corrected record alongside the original, with both verifiable, exactly as a corrected ledger entry works in double-entry accounting.

- **R-G6-RUN-8 (MUST):** The epoch boundary is **recorded as a first-class governed event** (a checkpoint event + an epoch-genesis event), signed, so an auditor can independently verify: "Epoch 0 ran schema 1.0, was sealed by checkpoint root R0 at time T0; Epoch 1 began at T0 under schema 1.0-rc with genesis `prev_hash = R0`." The schema change is thus *itself* tamper-evident evidence, not a silent operational mutation.
- **R-G6-RUN-9 (MUST):** Verification tooling (`POST /verify`, C2 §10) MUST be **epoch-aware**: it verifies each epoch's chain under that epoch's schema, then verifies the cross-epoch link (Epoch 1 genesis `prev_hash` == Epoch 0 final checkpoint root). The auditor-independence primitive (verify with only the published public key) is preserved across the boundary.
- **R-G6-RUN-10 (SHOULD):** Pre-flight before any epoch transition: full chain verifies clean (no pre-existing gaps — you cannot seal a broken chain), raw-event retention window (N-C2-402) is confirmed for any planned backfill, signing key is healthy and not due for rotation (don't combine an epoch transition with a key roll — §4.4), and the transition is dry-run on a copy.

> **Decision (DEC-G6-1):** *The append-only-log "schema migration" is solved by a chain-epoch boundary, not by data rewriting. This both honors the integrity invariant and turns the META-ADVERSARIAL "additive-only is broken by the rc" finding into a non-event: the rc IS additive, so the epoch boundary is the clean way to start emitting the new fields without ever touching — or invalidating the signatures over — history.* This generalizes: **every future C2 schema change, breaking or not, is an epoch transition.** The append-only log never migrates; it only ever opens new epochs cross-signed to old ones.

#### 4.3.3 Note: this is why "additive-only" was the right invariant *and* why it needed an epoch escape hatch

META-ADVERSARIAL M-3 / Risk #8: the corpus declared C2 FROZEN before consumers existed, then its #1 fix broke "additive-only." The epoch mechanism dissolves the contradiction: **the 1.0-rc change is genuinely additive** (reconciled in C2-v1.0-rc), so it needs no rewrite; and even if a *truly* breaking change arrives later (a field's meaning is incompatibly redefined), the epoch boundary handles it without rewriting history. The data structure's integrity guarantee and its evolvability are reconciled by never conflating "the log" with "the schema of the current epoch."

### 4.4 Runbook: certificate & secret rotation (hand-off to G4)

- **R-G6-RUN-11 (MUST):** G6 owns the **operational choreography**; **G4 owns the key custody/cryptography**. The hand-off contract:
  - **TLS/webhook certs** (admission webhooks, F1 ingress, mesh): rotate with overlapping validity (new cert trusted before old retired); admission webhook cert rotation MUST be zero-downtime (a webhook with an expired cert = fail-open/closed storm). Monitored: cert expiry countdown alerts at 30/14/7/1 days.
  - **Service secrets** (DB creds, object-store creds, Keycloak client secrets): rotate via the secret store with rolling restart; verify each service re-authenticates before retiring the old secret.
  - **The C2 signing key (the integrity foundation, G4/D4-OQ-1):** this is the highest-stakes rotation. G6's runbook: drain checkpoint signing to a clean boundary, **open a new chain epoch (R-G6-RUN-7) at the key roll** so the new `key_id` starts a clean signed segment, publish the new public key, and **retain the old public key forever** (old checkpoints must stay verifiable). Critically — *the old key is never used to re-sign history and the new key never claims to have signed the old epoch.* Key rotation is thus a special case of the epoch boundary. **Key *compromise* recovery** (re-establishing trust after a leak) is a G4 incident procedure that G6's incident response (§4.7) invokes; G6 does not own the cryptographic recovery, only the operational drain/cutover and the alerting.
- **R-G6-RUN-12 (SHOULD):** Rotation is rehearsed on a schedule (game-day), not first attempted during a real expiry/compromise.

### 4.5 Runbook: capacity scaling (hand-off to G1)

- **R-G6-RUN-13 (MUST):** G6 owns the **signals that trigger a scale event**; **G1 owns the capacity model and targets.** The hand-off: when a §2.3 saturation signal crosses a G1-defined threshold (ingest backlog growing, simulation worker pool saturated, admission eval queue depth rising, evidence-store disk/IOPS), G6's runbook executes the scale action (HPA limits raised, worker pool grown, store volume expanded, additional ingest workers) and verifies the saturation signal recovers. The **serial E1 core** (META G-2 / MASTER-PLAN-ALT line 327: "tight, serial, single-author core") is flagged as a **non-horizontally-scalable bottleneck** — when it saturates, the runbook *queues* (degrades replay latency gracefully) rather than promising a scale-out that the architecture can't deliver; this is escalated to G1 as a capacity-ceiling finding, not silently absorbed.
- **R-G6-RUN-14 (SHOULD):** The audit-ingest path is sized to **never** be the bottleneck that drops events (the lossless SLO, §3.2 Surface 2): scaling ingest is *automatic and aggressive* (buffer + add workers) precisely because the alternative — dropping evidence — is forbidden.

### 4.6 On-call model (sized for Jess — the 2–4 person team)

The META-ADVERSARIAL Risk #10 core worry: *"a governance platform that needs a platform team to govern it is a hard sell"* to the very small SRE teams it targets. G6's answer is an explicit, deliberately-minimal on-call model.

- **R-G6-ONCALL-1 (MUST):** **Alert volume is budgeted.** The platform MUST be operable by a single on-call engineer. The page budget is **≤ 2 actionable pages per on-call shift in steady state**; anything above is an alerting bug, not an acceptable baseline. Only the *symptom-of-customer-impact* signals page (admission failing, ingest dropping/lagging, chain broken, Keycloak down, signing failing); everything else is a ticket/dashboard. This is enforced by tiering alerts (page / ticket / dashboard) and by the multi-window burn-rate discipline (§3.2) that suppresses flapping.
- **R-G6-ONCALL-2 (MUST):** A **tiered severity model**:
  - **Sev-1 (page, all-hands):** audit event dropped, chain broken/gap, signing-key compromise, admission fully down in a fail-closed namespace, evidence store down. These threaten the product's integrity promise or break customer deploys.
  - **Sev-2 (page, single on-call):** admission degraded, ingest lag breaching SLO, Keycloak degraded, console down, a stuck approval blocking a deploy.
  - **Sev-3 (ticket):** single replica down (HA absorbs it), non-blocking detector errors, console latency, budget-burn tickets.
- **R-G6-ONCALL-3 (SHOULD):** Every Sev-1/Sev-2 alert links **directly to its runbook** (§4.1–4.5); there are no "figure it out" pages. The runbook is the on-call's primary artifact.
- **R-G6-ONCALL-4 (SHOULD):** **Managed-service tier exists as the answer for teams that can't run it** (see §6): for buyers below the SRE-headcount floor, the vendor runs the control plane and the customer runs only the thin spoke agents. This is not a cop-out — it is the honest acknowledgment that a 14-service governance stack has a real operational floor, and that floor is a *product packaging* decision, not a thing to hand-wave.

### 4.7 Incident response

- **R-G6-IR-1 (MUST):** Incident response follows declare → triage(severity) → mitigate(runbook) → communicate → resolve → **blameless post-mortem**. The post-mortem feeds back into the alerting budget (R-G6-ONCALL-1) and the runbooks.
- **R-G6-IR-2 (MUST):** Any incident that **touches the evidence chain** (a drop, a gap, a verify failure, a key compromise) triggers a **mandatory customer notification path** and an **evidence-gap scoping** step — what window of evidence is affected, whether it is recoverable from raw retention (N-C2-402) or is a permanent gap. This is a compliance incident, handled jointly with the customer, not a routine ops ticket. G3 (DR) owns recovery; G6 owns the declaration and the integrity-impact scoping.
- **R-G6-IR-3 (SHOULD):** A **break-glass procedure** (the F2/G3 fail-closed override, the R-G6-OBS-4 self-exemption) is itself a runbook with its own audit trail — using break-glass is a governed, recorded action (C2 `obligations:[exception]`), so "we bypassed our own controls to fix an outage" is auditable rather than invisible.

---

## 5. Migration-tooling normative requirements (consolidated)

- **R-G6-MIG-1 (MUST):** All schema/CRD/contract evolution uses **expand-contract** (additive deploy → migrate readers → flip → retire-later), never in-place destructive change. (Generalizes R-G6-RUN-1/5.)
- **R-G6-MIG-2 (MUST):** The C2 append-only log is **never rewritten**; schema evolution is by **chain-epoch boundary** (R-G6-RUN-7) with cross-signed epoch genesis. Historical signatures remain valid forever.
- **R-G6-MIG-3 (MUST):** Every migration tool is **dry-runnable on a copy**, emits a verifiable record of what it did, and (for CRDs) has a tested inverse before any one-way gate is crossed.
- **R-G6-MIG-4 (MUST):** Verification is **epoch-aware and key-aware** — it verifies each epoch under its schema and each segment under its `key_id`, plus the cross-links between them.

---

## 6. Managed-service vs self-hosted operations (short comparison)

The blunt operational truth behind META Risk #10: the 14-service stack has a real run-cost floor, and not every target buyer can clear it.

| Dimension | **Self-hosted (customer runs the control plane)** | **Managed (vendor runs the control plane; customer runs spokes)** |
|---|---|---|
| Who holds the SRE burden | Jess's 2–4 person team | Vendor SRE; customer runs only thin spoke agents (Gatekeeper/OPA + log shipper) |
| 14-service upgrade choreography (§4.1) | Customer's problem | Vendor's problem; customer upgrades only the spoke agent |
| Signing-key custody | Customer (G4) — full control, full responsibility | Split-trust question: **who holds the key?** If the vendor holds it, the customer is trusting the vendor with the integrity primitive — arguably defeating "auditor-independent" evidence. Resolution: **customer-held signing key even in managed mode** (vendor runs compute, customer holds the key + verifies independently). |
| Data residency / evidence sovereignty | Fully in customer's boundary | The evidence (C2 log) **MUST remain in the customer's storage/tenancy** (G5/G7) even when the vendor runs the control plane — the auditor must trust the customer's evidence, not the vendor's. |
| Fit | Buyers with a real platform team; high-sovereignty regulated buyers who won't let a vendor near the evidence | Jess-class buyers below the SRE-headcount floor; faster time-to-value |
| The honest trade | Maximum control, maximum run-burden — the META Risk #10 burden lands here | Lower run-burden, but introduces a vendor-trust surface that **must not** extend to the signing key or the evidence store, or the product's integrity claim collapses |

**Decision (DEC-G6-2):** offer **both tiers**, but with a hard invariant: in *either* tier, the **signing key and the evidence store stay in the customer's trust boundary.** The vendor may operate compute; it may never become a party the auditor has to trust for the evidence's integrity. This keeps "auditor-independent evidence" true regardless of who runs the pods, and makes the managed tier a legitimate answer to the "Jess can't run 14 services" risk rather than a dodge.

---

## 7. Dependencies

- **F2:** the service inventory, operators, CRDs, hub-spoke topology, failure modes — G6 is the day-2 layer F2 explicitly didn't cover.
- **C2 / C2-v1.0-rc-RECONCILED:** the audit schema, the hash chain, signed Merkle checkpoints, re-normalization (N-C2-105), raw retention (N-C2-402) — the substrate the epoch-migration runbook operates on.
- **B4:** CRD versioning (R-B4-22) — the CRD-migration runbook (§4.2) operationalizes it.
- **G1:** capacity model + SLI definitions — G6 defines scale-trigger *signals*; G1 owns the *numbers* (hand-off, §4.5).
- **G3:** availability/DR/RPO/RTO + buffer mechanism — referenced in upgrade/ingest/incident runbooks; G3 owns durability, G6 owns the operational procedures and SLOs that consume it.
- **G4:** key management — cert/secret/signing-key rotation cryptography (hand-off, §4.4); G6 owns the operational drain/cutover/alerting.
- **G5 / G7:** tenant boundary (per-tenant dashboards/alerts) and privacy (telemetry-must-not-carry-PII, R-G6-OBS-2).
- **C3:** the continuous chain-verification check is a C3 detector; G6 surfaces it as the highest-severity platform signal (R-G6-OBS-9).
- **E1:** the serial-core saturation signal + the replay-`complete` fraction gating metric (G-2).

---

## 8. Open questions — decided defaults

| # | Question | Decided default | Rationale |
|---|---|---|---|
| OQ-G6-1 | Self-monitoring stack: reuse customer's existing observability or ship one? | **Ship a minimal, self-contained one + integrate with customer's** (OpenMetrics/OTLP egress) | R-G6-OBS-5: monitoring must not depend on the platform it watches; but a regulated buyer already has Splunk/Datadog — emit OTLP so they can ingest. Don't force a second observability platform on Jess. |
| OQ-G6-2 | Are platform telemetry and audit evidence ever co-located for cost? | **No, never** (R-G6-OBS-1) | The integrity firewall and the back-pressure isolation are non-negotiable; the cost saving isn't worth a path where telemetry load can drop evidence. |
| OQ-G6-3 | In-place schema migration of the C2 log? | **Forbidden — epoch boundary only** (DEC-G6-1) | The append-only integrity invariant makes in-place migration the exact tamper it exists to detect. |
| OQ-G6-4 | Combine key rotation with an epoch transition? | **Key rotation IS an epoch transition; never combine with a *schema* epoch transition in one step** | Each is a one-way gate; bundling two one-way gates makes rollback impossible. One at a time. |
| OQ-G6-5 | On-call page budget | **≤2 actionable pages/shift; anything more is an alerting defect** | The product targets Jess (2–4 SREs); an un-runnable alert volume kills the deal (META Risk #10). |
| OQ-G6-6 | Managed vs self-hosted | **Both; signing key + evidence store always customer-held** (DEC-G6-2) | Answers "Jess can't run 14 services" without breaking "auditor-independent evidence." |
| OQ-G6-7 | Does day-2 ops itself get audited? | **Yes — every enforcement/chain/key-touching op emits a governed C2 event** (R-G6-RUN-0) | The platform that governs others must govern its own operators; break-glass especially must be visible. |

---

## 9. Normative requirements summary

Self-observability: **R-G6-OBS-1..11**. SLOs: **R-G6-SLO-1..4**. Runbooks: **R-G6-RUN-0..14**. On-call/IR: **R-G6-ONCALL-1..4, R-G6-IR-1..3**. Migration tooling: **R-G6-MIG-1..4**. The MUST set is the day-2 operational conformance floor; the three non-negotiables — **lossless audit ingest, chain integrity, and customer-held signing key** — are never relaxed under any SLO, packaging, or capacity pressure.
