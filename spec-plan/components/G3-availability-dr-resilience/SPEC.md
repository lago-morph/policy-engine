# G3 — Availability, Disaster Recovery & Resilience — SPEC

**Component ID:** G3 · **Domain:** G — Operational / Non-Functional Requirements
**Spec sources:** META-ADVERSARIAL-SECOND-OPINION §3 (NFR gap: "DR / RPO / RTO for the evidence store is undefined") and Risk #4; Domain-B `DOMAIN-ADVERSARIAL.md` X-4 ("composed fail-closed defaults = correlated self-amplifying outage; no circuit breaker"); C2 SPEC §7–§8 (tamper-evidence, append-only log, hash chain, signed Merkle checkpoints); B5 SPEC §5 (flow-level failure modes); F2 SPEC §2–§6 (topology, HA, failure modes).
**Status:** DRAFT v1 · **Date:** 2026-05-30 · **Author persona:** cooperative engineering author (NFR/operational architecture).
**Scenarios exercised (durability/availability slices of):** HL-06 (bypass detection presupposes the evidence survived), HL-18 / DT-24 (auditor independence requires the chain still verifies after restore), DT-25 (backfill presupposes raw retention survived), DT-46 (materialized dataset durability), plus the X-4 mass-deny outage mode.

> **One-line thesis.** The platform's product *is* a tamper-evident system of record. Therefore the two non-functional properties that matter most are **(a) the audit log must never lose committed events and must still cryptographically verify after a restore (RPO=0, chain-continuous DR)**, and **(b) when the platform's own infrastructure degrades, the enforcement fleet must enter a distinct, audited "infrastructure-degraded" mode rather than mass-denying the whole estate ("policy says no" ≠ "the platform is down").** Everything below serves those two properties.

---

## 0. Why this component exists (the two unowned failures it resolves)

1. **DR/RPO/RTO for the audit log was undefined.** The independent review board's Risk #4: *"Loss or corruption of the append-only log unbacks every verdict and export. For an evidence product this is the top durability property."* C2 SPEC §7–§8 specifies how the log is **tamper-evident in steady state** but is silent on what happens when the store, a region, or the cluster is lost. A signed hash chain that cannot be restored without breaking its own signatures is worse than no backup — it manufactures a false tamper signal. G3 owns the **backup + restore design that preserves C2 hash-chain and checkpoint-signature continuity** (§5).

2. **The composed-fail-closed mass-deny outage (Domain-B X-4).** B1/B2/B5/B3 each independently chose a security-correct "fail closed / fail loud" default. *Composed*, when a **shared dependency** (signed-bundle server, signature verifier, Keycloak, the OPA control plane, external-data providers) degrades, every spoke in the fleet times out to `failurePolicy=Fail` simultaneously → **correlated fleet-wide mass-deny** with no circuit breaker, and the fail-closed defaults can even prevent recovery of the dependency that failed (e.g. you can't deploy the fix because admission denies it). Domain B explicitly delegated the resolution to a "domain-wide infrastructure-degraded mode" that "needs to be a flow-level decision, not five independent component choices." **G3 owns that mode** (§4) because it is fundamentally an availability/resilience control, not a policy-engine feature.

These two are the spine. RPO/RTO targets (§3), FMEA (§6), multi-region/multi-cluster (§7), and the chaos/DR-drill plan (§8) hang off them.

---

## 1. Scope

### 1.1 In scope
- **Data-class taxonomy** and **explicit RPO/RTO targets per class** (audit log vs. config/CRDs vs. derived/rebuildable state) — §3.
- The **availability target for the admission hot path** and the precise **per-criticality fail-open / fail-closed disposition** when the platform (not the policy) is unavailable — §4.1.
- The **circuit breaker / "infrastructure-degraded" mode**: detection, state machine, scope of action, recovery, and its audit obligations — §4.2–§4.6. Resolution of Domain-B X-4.
- **Backup + restore design that preserves C2 integrity**: how to back up and restore an append-only signed hash chain + Merkle checkpoints **without breaking continuity or manufacturing a false tamper alarm** — §5. This is the crux.
- **FMEA** (failure-mode & effects analysis) for each platform service, with detection, effect, mitigation, and the RPO/RTO/availability target each must meet — §6.
- **Multi-region / multi-cluster resilience** topology: hub loss, spoke loss, region loss, partition (split-brain) — §7.
- **Chaos engineering & DR-drill (game-day) test plan** with explicit pass/fail acceptance criteria — §8.

### 1.2 Out of scope (delegated, with the seam named)
- **Production scale / $-per-event / throughput sizing** → **G1** (scale/performance). G3 consumes G1's volume model to size backup windows and restore time; it does not re-derive it.
- **Retention economics / cost of multi-region copies / long-term archive tiering** → **G2** (cost & retention economics). G3 states the *durability requirement* (how many copies, how far apart, how often); G2 prices it. Cross-reference is explicit in §7.5 and `ALT-DR-TOPOLOGY.md`.
- **Key custody, KMS/HSM, signing-key rotation & compromise recovery** → **G4** (key management). G3 *depends on* the signer being available and on a key being restorable; the **re-sign-after-key-compromise** procedure is G4's. G3 owns only "the checkpoint chain must still verify after a data restore with the *then-current valid* keys" (§5.4) and flags the key/data co-restore ordering (§5.6).
- **Tenant isolation / blast-radius of one tenant on shared workers** → **G5**. G3 owns *availability* blast radius (one spoke, one region), not *tenant* blast radius.
- **Day-2 ops, on-call, upgrade choreography, observability** → **G6**. G3 specifies *what must be observable for DR* (RPO lag metric, chain-verify health) and the *DR runbooks*; G6 owns the broader operability surface. The DR runbook lives in G3 (§8.5) because it is DR-specific.
- The **per-component steady-state failure behavior** of B1/B2/B5/F2 stays in those specs; G3 **composes** them into the FMEA and the degraded-mode and does not restate their internals.

---

## 2. Data-class taxonomy (the basis for differentiated RPO/RTO)

Not all platform data is equally precious. Conflating them produces either an unaffordable everything-is-RPO-0 design or a fatal lose-the-audit-log design. G3 defines **four data classes**, each with its own durability contract (§3).

| Class | Examples | Key property | Rebuildable? | If lost… |
|---|---|---|---|---|
| **D0 — System-of-record evidence** | C2 append-only event log, per-source hash chains, signed Merkle checkpoints, retained raw source events (incl. raw external-data responses, N-C2-402), CAS blobs (`before/after/request_object`, external-data values), signed export manifests, materialized replay datasets (DT-46). | **Tamper-evident, append-only, legally load-bearing.** This is the product. | **No.** A lost decision event is gone forever; it cannot be re-derived (the engine evaluation already happened). | The compliance thesis collapses (Risk #4). Every downstream verdict/export is unbacked. **RPO=0 is non-negotiable.** |
| **D1 — Governance configuration & control intent** | Governance CRDs (`GovernancePlatform`, `GovernanceBundle`, Constraints/ConstraintTemplates, `PolicyApprovalRequest`, exceptions), A1 governance store (control catalog, Gemara/OSCAL mappings), signed policy bundles + their provenance, the policy-dependency catalog (C2 §4.2), Keycloak realm config / role-claim mappings (D1/D2). | **Authoritative declared state.** Source-of-truth for *what should be enforced*. | **Partially.** CRDs live in etcd and can be re-applied from GitOps; signed bundles are re-pullable from the bundle registry. But in-flight approval state and exceptions are not trivially rebuildable. | Enforcement drifts to stale/no policy; approvals/exceptions may be lost (a security event). Recoverable from GitOps + registry, but **RPO must be tight (≤5 min) and restore fast.** |
| **D2 — Derived / queryable read-models** | C3 analytics aggregates, C2 secondary indexes (correlation_id, control_id, scope, replay_completeness…), coverage matrices, C5 cached reports, `correlation_members` views. | **Rebuildable from D0.** Pure functions of the event log. | **Yes**, by re-indexing/re-aggregating D0. Expensive (time) but not lossy. | Queries/dashboards degrade until rebuilt. **No data loss; RPO can be loose.** Cost is *rebuild time*, which sets a soft RTO. |
| **D3 — Ephemeral / operational state** | In-flight admission requests, decision-log buffers (B1-R28), worker queues, simulation job state (E1), session caches, leader-election leases. | **Transient.** Tolerates loss with at-most-bounded user impact. | **N/A** (recreated by normal operation). | A few in-flight requests retry; buffered-but-unflushed decision logs are the one risk-bearing sub-case (see §3.4 / FMEA-EV2). |

**Normative requirements:**

- **R-G3-DC-1 (MUST):** Every persistent datum in the platform MUST be assigned exactly one data class (D0–D3) in its owning component's deployment manifest metadata (`governance.example.io/data-class`). Backup, replication, and DR tooling key off this label. An unclassified persistent volume is a release-blocking defect.
- **R-G3-DC-2 (MUST):** The buffered-but-not-yet-committed decision-log path (B5-F4: C2 down, evidence buffered at the edge) is the one place where **D3 (a buffer) holds soon-to-be-D0 (evidence)**. This buffer is therefore treated as **D0-pending** and inherits D0 durability obligations until the event is committed and acknowledged into the C2 chain (§3.4). Silently dropping a buffered decision log is an RPO-0 violation, not an ephemeral-loss event.

---

## 3. RPO / RTO targets per data class (the headline contract)

**Definitions.** *RPO (Recovery Point Objective)* = maximum acceptable **data loss** measured in time (how far back the most recent recoverable state may be). *RTO (Recovery Time Objective)* = maximum acceptable **downtime** to restore service. These are stated **per data class and per failure tier**; a single platform-wide number would be either dishonest or ruinous.

### 3.1 The target table

| Class | Failure tier | **RPO** | **RTO** | Mechanism that achieves it (forward ref) |
|---|---|---|---|---|
| **D0 — evidence** | Single node / disk loss | **0** | ≤ 5 min | Synchronous quorum write (§3.3) — committed event survives any single node. |
| **D0 — evidence** | Zone loss (AZ) | **0** | ≤ 15 min | Multi-AZ synchronous quorum (≥3 AZ, write-majority) — §3.3, §7.2. |
| **D0 — evidence** | Region loss | **≤ 60 s** (async cross-region) **/ 0** (if active-active sync chosen) | ≤ 4 h (warm standby) / ≤ 15 min (active-active) | Cross-region replication of the log + CAS + checkpoint chain; topology choice priced in `ALT-DR-TOPOLOGY.md`. **The ≤60 s async-replication window is the single most scrutinized number in this spec** — see §3.2 and ADVERSARIAL-REVIEW A-3. |
| **D0 — evidence** | Full backup/restore (logical corruption, ransomware, operator error) | **0** to last committed checkpoint segment; **≤ checkpoint cadence** (default ≤ 15 min, N-C2-302) for events after the last checkpoint | ≤ 4 h to a chain-verified restore | Immutable object-lock snapshots + checkpoint-anchored restore (§5). |
| **D1 — config/CRDs** | Any tier up to region loss | **≤ 5 min** | ≤ 30 min | etcd backup + GitOps re-apply + signed-bundle re-pull (§3.5). |
| **D2 — derived** | Any tier | **N/A (rebuildable)** — bounded by D0 RPO | ≤ 24 h to fully re-indexed; **degraded-but-serving in ≤ 1 h** (priority indexes first) | Re-index/re-aggregate from restored D0 (§3.6). |
| **D3 — ephemeral** | Any tier | **N/A** except **D0-pending buffers = 0** (R-G3-DC-2) | ≤ 5 min (auto-recreated) | Normal operation; buffer durability per §3.4. |

### 3.2 Why D0 RPO is 0 for everything up to region, and the honest caveat for region loss

- **R-G3-RPO-1 (MUST):** For all single-node and single-zone failures, **committed D0 evidence RPO MUST be 0.** A decision event is "committed" only after it is durably written to a **write-majority quorum across ≥2 failure domains** and the C2 chain `content_hash`/`prev_hash`/`chain_seq` are persisted with it. The C2 normalizer MUST NOT acknowledge an event into the chain (advance `chain_seq`) until quorum-committed. *Rationale:* the entire compliance thesis (Risk #4, Risk #5) dies if "we have your decision" is ever a lie. RPO=0 here is the load-bearing promise.
- **R-G3-RPO-2 (MUST):** For **region loss**, RPO is **>0 only if** the chosen topology is async cross-region replication, and then it MUST be **bounded and continuously measured** (`g3_d0_replication_lag_seconds`, §6.3 / G6). Default bound **≤ 60 s**. The platform MUST refuse to advertise "authoritative evidence" for the lag-window's events after a regional failover until reconciliation (§7.4) confirms they replicated. **Any region-loss data loss is a disclosed, audited gap (a synthetic `replay_completeness` reason `evidence_lost_in_failover`), never a silent hole.** See A-3.
- **R-G3-RPO-3 (SHOULD, escalates to MUST for regulated/Stack-C deployments):** Deployments under a multi-year regulated retention regime (SOC2/PCAOB/financial-services) SHOULD elect **active-active synchronous cross-region** (RPO=0 even on region loss) for D0, accepting the latency/cost in `ALT-DR-TOPOLOGY.md`. The default POC profile uses async warm-standby (RPO ≤ 60 s) because the POC is functional-validation, not production durability (F3 §22). **The deployment profile MUST state which it is** — selling RPO=0 while running async is the "declared vs verified" sin the platform exists to prevent (META G-2).

### 3.3 Quorum-commit contract for the evidence log

- **R-G3-RPO-4 (MUST):** The D0 store MUST be deployed as a **≥3-replica quorum across ≥3 failure domains (AZs)** with **write-majority (W ≥ 2 of 3) acknowledgment** before chain advancement. The hash chain's `chain_seq` is the linearization point; quorum commit MUST precede `chain_seq` issuance so a partition cannot fork the chain (§7.3 split-brain defense).
- **R-G3-RPO-5 (MUST):** CAS blobs (large `request_object`/state/external-data values referenced by digest, C2 §8.4) MUST be quorum-durable **before** the referencing event is acknowledged — otherwise a `complete`-labeled event can point at a blob that did not survive (an integrity lie). The acknowledgment ordering is: **blob durable → event quorum-committed → `chain_seq` issued → checkpoint eventually signs over it.**

### 3.4 The buffer (D0-pending) durability rule — resolving B5-F4

B5-F4 says: when C2 is down, the decision is unaffected and **evidence is buffered at the edge** (B1-R28), correlation_id preserved for later backfill. G3 makes the buffer's durability a hard contract because a lost buffer = lost D0 = RPO violation.

- **R-G3-BUF-1 (MUST):** The edge decision-log buffer MUST be **persistent (disk-backed, survives pod restart)**, **bounded with a high-water mark**, and **back-pressure or spill to a local durable spool** rather than drop on overflow. Dropping a buffered decision log is an RPO-0 violation; if the spool is exhausted, the system MUST raise a P1 `evidence_buffer_saturated` alert and MAY (per scope criticality, §4) enter degraded mode — it MUST NOT silently discard.
- **R-G3-BUF-2 (MUST):** Buffered events carry their `correlation_id` and the inputs needed to reach their original `replay_completeness`; on C2 recovery they are flushed **in `chain_seq`-free form** (the chain seq is assigned at commit time on the authoritative chain, not at the edge) and committed in `timestamp` order. The flush MUST be idempotent on `(source.system, raw_event_digest)` (N-C2-105) so a partially-flushed buffer after an edge crash does not double-commit.
- **R-G3-BUF-3 (SHOULD):** The maximum buffer-drain-deficit (oldest un-flushed buffered event age) is a monitored RPO-risk metric `g3_d0_buffer_oldest_age_seconds`; sustained growth predicts an impending RPO breach before it happens.

### 3.5 D1 (config) RPO/RTO mechanism

- **R-G3-CFG-1 (MUST):** Governance CRDs and Keycloak realm config are the *declared* state; their **GitOps source repository is the primary recovery source** (re-apply to a fresh cluster), backed by **etcd snapshots every ≤5 min** for the in-cluster-only state that GitOps does not own (live approval/exception status, leader leases — though leases are D3). Signed policy bundles are re-pullable from the OCI bundle registry (itself a backed-up D1 asset or an external durable registry).
- **R-G3-CFG-2 (MUST):** **In-flight `PolicyApprovalRequest` and active exception state are D1, not D2** (they are not rebuildable from the event log alone — an approval granted but not yet consumed is live state). They MUST be included in the ≤5-min RPO backup. *Losing a pending approval re-denies a legitimately-approved deploy (annoyance); losing a granted-but-unconsumed exception could re-block production (availability incident) or, worse, a restored-stale exception could re-open a closed bypass (security incident — see A-5).*

### 3.6 D2 (derived) rebuild

- **R-G3-DRV-1 (MUST):** Every D2 dataset MUST be **deterministically rebuildable from restored D0** by a documented, tested rebuild job, with a **priority order** (chain-verification index and scope-authz index first — so the system can *safely* serve scoped reads — then analytics aggregates). "Degraded-but-serving in ≤1 h" means the security-relevant indexes are back; full analytics may lag to ≤24 h.
- **R-G3-DRV-2 (SHOULD):** D2 MAY also be backed up (not only rebuilt) to shorten RTO, but a restored D2 MUST be **re-validated against D0** (digests match) before serving, because a stale/forged D2 could otherwise misreport coverage. Rebuild-from-D0 is always the source of truth; restored-D2 is an RTO optimization only.

---

## 4. Admission hot-path availability & the "infrastructure-degraded" mode

This section resolves **Domain-B X-4**. It is the heart of G3.

### 4.1 Availability target & the fail-open/fail-closed disposition matrix

- **R-G3-AV-1 (MUST):** The **admission hot path's availability is NOT a single platform SLO**; it is governed per-namespace by a deliberately-chosen `failurePolicy` tied to a **criticality class** declared on the scope (F2-HA-1, F2 §6). The platform MUST require this choice to be explicit (no silent default) and MUST record it in the `GovernancePlatform`/scope CRD.
- **R-G3-AV-2 (MUST):** The disposition when the **platform** (a shared dependency) is unavailable — distinct from when a *policy* denies — follows this matrix. The criticality class is a property of the scope (namespace/cluster), set by governance:

| Criticality class | Example scopes | Disposition when platform/shared-dep is DOWN (and degraded mode active) | Rationale |
|---|---|---|---|
| **C-CRITICAL** (regulated, high-assurance) | `payments-prod`, `*-prod` under a regulated control | **Fail-closed, BUT** the per-control `degraded_action` may downgrade *individual* non-safety controls to `warn` while keeping **safety-critical controls hard-denied** (image-signing, privilege-escalation). The default for an unclassified critical control is **hard deny**. | You may not silently admit into a regulated namespace; but a *whole-fleet* hard-deny of *every* control because the bundle server is slow is the X-4 outage. Per-control granularity is the escape valve. |
| **C-STANDARD** (normal prod) | most `*-prod` | **Degrade to warn/advisory** with loud alerting + §19 catch-up scan. | Availability of the workload outweighs a brief enforcement gap that is detectively backfilled (C4 §19). This is the X-4 resolution: warn, don't mass-deny. |
| **C-LOW** (dev/test/non-prod) | `*-dev`, sandboxes | **Fail-open (Ignore)** with audit-event-of-bypass so C3/C4 see the gap. | Never brick developer velocity for a platform hiccup. |
| **C-SYSTEM** (platform's own namespaces) | `governance-system`, `kube-system` | **Always fail-open / carve-out** (B2-R4) — and additionally **exempt from the circuit breaker's deny actions** so the breaker can never block its own recovery. | The anti-self-brick rule (§4.6): the fix for a platform outage MUST be deployable *during* the outage. |

- **R-G3-AV-3 (MUST):** Every degraded-mode disposition (warn-instead-of-deny, fail-open admit) MUST emit a **first-class audit event** distinguishing **"admitted under infrastructure-degraded mode"** from **"admitted because policy allowed."** This is a new C2 reason/disposition (`disposition_context: infrastructure_degraded`, with `degraded_reason` and `degraded_session_id`) so that (a) the auditor can see exactly which admissions happened without full enforcement, (b) C4 §19 can drive the catch-up re-scan over precisely that set, and (c) an attacker who triggers degraded mode to slip something through is **not invisible** — the gap is loud, scoped, and time-bounded, not silent (A-4).

### 4.2 The circuit breaker / "infrastructure-degraded" mode — the core mechanism

The mass-deny outage (X-4) happens because each spoke *independently* times out to `failurePolicy=Fail` against the same failed shared dependency, with no coordinated notion of "the platform itself is sick, so don't treat every timeout as a policy decision." G3 introduces a **fleet-aware circuit breaker** that distinguishes **"this dependency is down"** from **"this request is denied."**

- **R-G3-CB-1 (MUST):** The platform MUST run a **degraded-mode controller** (a control-plane component, leader-elected, HA) that **continuously probes the shared dependencies** whose failure causes correlated outages: the **signed-bundle server / OCI registry**, the **external-data / signature-verifier providers**, **Keycloak / D1 identity broker**, the **OPA control plane / bundle freshness**, and the **C2 evidence sink reachability**. Each probe yields a per-dependency health state.
- **R-G3-CB-2 (MUST):** When a probed shared dependency crosses a failure threshold (error rate / latency over a window — a classic closed→open circuit transition), the controller **opens the breaker for the set of controls that depend on that dependency**, and pushes a **degraded-mode signal** to the affected spokes (via a CRD status field the spoke admission layer watches, e.g. `DegradedMode` CR / `GovernancePlatform.status.degraded[]`). The breaker is **scoped to (dependency × control × criticality)**, never a global kill switch — granularity is what prevents both the outage *and* the breaker-as-bypass attack (A-4).
- **R-G3-CB-3 (MUST):** While the breaker is **open** for a (dependency, control) pair, the spoke admission layer applies the **degraded disposition from the §4.1 matrix for that control's criticality** instead of timing out to a blanket `failurePolicy=Fail`. Concretely: a control whose external-data verifier is down, in a `C-STANDARD` namespace, **warns-and-admits-with-`infrastructure_degraded`-audit** instead of denying — so a slow `cosign` verifier no longer mass-denies the fleet, but the gap is recorded for §19 catch-up.
- **R-G3-CB-4 (MUST):** The breaker is **distinct from the per-request `failurePolicy` timeout.** A single slow request still respects the request-level timeout (B5-R6 ≤2s budget). The breaker engages only on the **aggregate, sustained** failure signal — so it does not flap on one slow call, and it gives the *fleet* a coherent answer rather than 10,000 spokes each guessing. This separation is the key design move.

### 4.3 Breaker state machine

```
        probe healthy (sustained, ≥ recovery window)
   ┌──────────────────────────────────────────────────────────┐
   │                                                            │
   ▼                                                            │
┌────────┐  failure threshold crossed   ┌──────────┐  half-open │
│ CLOSED │ ───────────────────────────▶ │   OPEN   │ ─probe──▶ ┌────────────┐
│(normal │                              │(degraded │  trial    │ HALF-OPEN  │
│enforce)│ ◀─────────────────────────── │ disposi- │ ◀──────── │ (test 1   │
└────────┘   probe recovered + manual    │ tion in  │  trial    │  request) │
             OR auto-close criteria       │ effect)  │  fails    └────────────┘
                                          └──────────┘
```

- **CLOSED:** normal enforcement; the §4.1 "platform UP" path.
- **OPEN:** degraded disposition in effect for the scoped (dependency, control, criticality) set; loud alerting; every affected admission emits the `infrastructure_degraded` audit event; the §19 catch-up set is being accumulated.
- **HALF-OPEN:** the controller periodically lets a **trial** evaluation hit the dependency; success path → CLOSED, failure → back to OPEN. Auto-close requires a **sustained** recovery window (anti-flap).

- **R-G3-CB-5 (MUST):** Transitions OPEN↔CLOSED are themselves **audit events** (`event_type=policy.degraded_transition`, signed into the C2 chain) with `degraded_session_id`, the triggering dependency, the affected control/scope set, and `opened_by` (auto vs. operator). This makes "the platform was in degraded mode from t1 to t2 affecting controls X over scope Y" an immutable, auditable fact — essential for the auditor and for A-4 (breaker-as-bypass cannot be invisible).

### 4.4 Recovery & catch-up (closing the detective loop)

- **R-G3-CB-6 (MUST):** On breaker close, the controller MUST **emit a §19 catch-up work item to C4** delimited by `degraded_session_id` and the affected (control × scope) set, so that every resource admitted under degraded mode is **re-evaluated against the now-recovered policy** and any that would have been denied is surfaced as a retrospective finding (HL-06 / DT-30 mechanism). Degraded mode is *availability-preserving*, but the enforcement gap it opens is **detectively closed**, not forgiven. This is the honesty contract that makes warn-don't-deny acceptable to an auditor (rebuts META G-6 partially: the gap is labeled *and* remediated, not just labeled).
- **R-G3-CB-7 (SHOULD):** The catch-up re-scan SHOULD be **prioritized by criticality** (C-CRITICAL scopes first) and SHOULD complete within an SLA (default: critical scopes re-scanned within 1 h of recovery) so the window of "admitted-but-not-yet-verified" is bounded.

### 4.5 Manual override (break-glass) of the breaker

- **R-G3-CB-8 (MUST):** An authorized operator (separation-of-duties role, D2) MUST be able to **manually open or force-close** the breaker (break-glass), e.g. force-close a flapping breaker that is hiding a real outage, or manually open to drain a dependency for maintenance. Every manual action is a signed audit event with actor identity and a required reason string. Manual *force-close* of a breaker over a **C-CRITICAL safety control** MUST require a second approver (dual-control), because it re-enables hard-deny that could itself cause an outage, and because it is exactly the lever A-4's attacker would want.

### 4.6 Anti-self-brick invariant (the recursion stopper)

- **R-G3-CB-9 (MUST — the single most important resilience invariant):** The platform's **own recovery path MUST NOT be subject to its own fail-closed enforcement.** Specifically: (a) `C-SYSTEM` namespaces are always carved out (R-G3-AV-2); (b) the degraded-mode controller, the C2 store, the bundle server, and the operator MUST be deployable/restartable **without passing through an admission gate that depends on them** (use a bootstrap exemption / a static break-glass policy that admits platform-namespace workloads even when every dynamic dependency is down); (c) the circuit breaker's deny actions MUST never apply to the components whose health the breaker itself depends on. *Rationale:* X-4's nastiest sub-case is "the composed fail-closed defaults prevent recovery of the very dependency that failed" — you cannot fix the bundle server because admission (which needs the bundle server) denies the bundle server's own deploy. This invariant breaks that recursion. It is tested explicitly in §8 (CHAOS-7).

---

## 5. Backup & restore preserving C2 hash-chain & checkpoint-signature continuity (the crux)

**The hard problem (Risk #4, A-1, A-2):** C2's value is that the log is **append-only, hash-chained, and signed** (C2 §7). A naive backup/restore breaks this in three ways: (1) restoring a *prefix* of the chain (everything up to backup time) looks identical to **malicious truncation** — a false tamper alarm, or worse, a real truncation an attacker hides as "a restore"; (2) restoring to a *fork* (two divergent tails after a partial restore) is **split-brain on the hash chain** (A-1); (3) re-writing or re-chaining events during restore would **invalidate every signed checkpoint** downstream. G3 specifies a restore design that preserves continuity by construction.

### 5.1 What gets backed up, and the immutability requirement

- **R-G3-BK-1 (MUST):** D0 backups are taken at **checkpoint boundaries** (N-C2-302 signed Merkle checkpoints). A backup is a contiguous prefix of the per-source chain **ending at a signed checkpoint**, plus all CAS blobs the prefix references, plus the checkpoint chain itself, plus the public-key/key-id metadata needed to verify (the keys themselves are G4's custody; the *key-id references* travel with the backup).
- **R-G3-BK-2 (MUST):** D0 backups MUST be written to **immutable, object-locked (WORM) storage** (S3 Object Lock / equivalent), retention-locked for the compliance retention period (G2), in a **separate failure domain and separate account/credential boundary** from the live store, so that a compromise of the live store (ransomware, malicious admin) cannot delete or rewrite the backup. The backup credential MUST NOT be able to delete within the lock window (defense against the insider-deletion threat C2 §7.1 names but did not extend to backups).
- **R-G3-BK-3 (MUST):** Backups are **continuous/incremental at the segment level** (each new sealed checkpoint segment is shipped to WORM immediately on seal), not periodic full dumps, so the backup RPO equals the checkpoint cadence (≤15 min default) for the *backed-up* copy, while the *live quorum* copy is RPO-0 (§3). Two independent durability mechanisms (quorum + WORM backup) protect against different threats (node loss vs. logical corruption).

### 5.2 The restore protocol — continuity by construction

- **R-G3-RS-1 (MUST):** Restore reconstructs the chain **as a verifiable prefix terminating at the last fully-backed-up signed checkpoint.** The restore tool MUST: (1) load segments in `chain_seq` order; (2) re-verify every `prev_hash` link and every `chain_seq` continuity; (3) re-verify every Merkle checkpoint signature against the public key valid at the checkpoint's `signed_at` (G4 provides historical key validity); (4) produce a **signed Restore Attestation** (§5.3) recording exactly which checkpoint the chain was restored *to*. A restore that cannot verify the chain end-to-end MUST fail loudly and produce a forensics report, never serve a partially-verified chain as authoritative.
- **R-G3-RS-2 (MUST):** The restored chain's **head is the last backed-up checkpoint**, NOT an arbitrary event. Events that were committed to the live quorum *after* the last backed-up checkpoint but lost in the disaster (the ≤15-min WORM window, or the ≤60s cross-region async window) are an **explicit, attested gap**, not a silent truncation. The Restore Attestation names the gap: "events with `chain_seq > S` and `timestamp > T` may be missing; this is a known restore boundary." See §5.5.

### 5.3 Restore Attestation — the anti-false-tamper-alarm primitive

- **R-G3-RS-3 (MUST):** Every restore produces a **Restore Attestation**: a signed record (signed by an operator+platform dual key, separation-of-duties) stating: the disaster timestamp, the source backup digest, the **last restored `chain_seq` and checkpoint Merkle root**, the **gap boundary** (highest known-lost `chain_seq`/timestamp if any), the restoring operator(s), and a fresh continuation pointer. This attestation is **itself appended to the chain as a genesis-of-continuation marker** (`event_type=chain.restore_boundary`). The chain after restore therefore reads: `… → last-backed-checkpoint → [signed restore_boundary marker, explaining the discontinuity] → new events`. **A verifier (auditor, HL-18 `/verify`) seeing a restore_boundary marker knows the discontinuity is an attested restore, not tampering.** This is the mechanism that prevents "restore looks like malicious truncation."
- **R-G3-RS-4 (MUST):** The chain-verification logic (C2 §7.3, C3 continuous chain-check) MUST treat a **valid signed `restore_boundary` marker** as a legitimate chain discontinuity (the `prev_hash` of the first post-restore event points at the marker, which points at the last restored checkpoint). A `prev_hash`/`chain_seq` discontinuity **without** a valid signed restore marker remains a **tamper finding**. This is the precise rule that distinguishes "we restored from backup" from "someone deleted events," resolving A-1/A-2.

### 5.4 Re-chaining is forbidden; continuation is appended

- **R-G3-RS-5 (MUST):** The restore MUST NOT **re-hash, re-chain, or re-sign** historical events or checkpoints. Their original `content_hash`/`prev_hash`/`chain_seq`/signatures are preserved byte-for-byte (RFC 8785 JCS canonical form, C2 §7.2). Restore is **load + verify + append-continuation-marker**, never rewrite. *Rationale:* re-signing history with a current key would (a) destroy the original-time attestation, and (b) be indistinguishable from an attacker re-writing history with a stolen key. Immutability of past signatures is the trust anchor; restore must respect it absolutely. (This also keeps G3 decoupled from G4's key rotation: old checkpoints verify with old keys, new continuation signs with current keys.)

### 5.5 Reconciling the post-checkpoint tail after a region failover (the RPO>0 window)

- **R-G3-RS-6 (MUST):** After a **region failover** where the async-replicated copy may lag the lost primary by ≤60 s (R-G3-RPO-2), the failover-target's chain head is its **own last-replicated checkpoint**. The events in the lag window that committed on the lost primary but didn't replicate are **lost D0**. The platform MUST: (1) emit a `chain.restore_boundary` marker on the surviving chain naming the lag window; (2) attempt **reconciliation from edge buffers** (R-G3-BUF-1/2) — any spoke that still holds buffered copies of lag-window events re-commits them on the surviving chain (recovering some/all of the gap); (3) for events that cannot be recovered, record a platform-level **`evidence_lost_in_failover` disclosure** with the count and scope, surfaced in the auditor `/verify` output and as a C3 finding. **The loss is bounded, measured, and disclosed — never silent.** This is the honest face of RPO>0 (A-3).

### 5.6 Co-restore ordering with keys (the G3↔G4 seam)

- **R-G3-RS-7 (MUST):** Restore of D0 evidence and the availability of **verification keys** (G4) MUST be ordered: the **public keys / key-validity history (G4) must be restorable independently of and before the data**, because verifying the restored chain (R-G3-RS-1 step 3) requires the historical public keys. If the signing private key was compromised in the disaster, restore proceeds (verification uses the *then-valid* public keys for old segments) and the **continuation** is signed with a fresh G4 key per G4's compromise-recovery procedure — G3 does not re-sign history (R-G3-RS-5); it only needs G4 to furnish a valid continuation signer and the historical public keys. This explicitly bounds the G3/G4 contract.

### 5.7 Restore validation gate

- **R-G3-RS-8 (MUST):** A restore is **not "done"** until an automated **post-restore chain-verification pass** (HL-18 `/verify` over the full restored range) returns green AND a **sample of materialized export manifests (DT-24) re-verify against the restored chain**. Only then is the restored store promoted to serving. Serving an unverified restore is a release-blocking defect. This gate is exercised in every DR drill (§8, DRILL-1).

---

## 6. FMEA — Failure Mode & Effects Analysis per service

Each row: failure mode → detection → effect → mitigation → the data class at risk and the RPO/RTO/availability target it must meet. "Sev" is platform-impact severity (1=catastrophic/evidence-loss, 2=major outage, 3=degraded, 4=minor).

| ID | Service | Failure mode | Sev | Detection | Effect | Mitigation (and where owned) | Class · target |
|---|---|---|---|---|---|---|---|
| **FMEA-EV1** | C2 evidence store | Single node/disk loss | 2 | quorum health, missing-replica alert | none if quorum holds | ≥3-replica W≥2 quorum (R-G3-RPO-4); auto-rejoin | D0 · RPO 0 / RTO ≤5m |
| **FMEA-EV2** | C2 evidence store | **Buffered (uncommitted) decision logs lost on edge crash while C2 down** | 1 | `g3_d0_buffer_oldest_age`, spill alerts | **D0 loss = RPO breach** | persistent disk-backed buffer + spool + back-pressure (R-G3-BUF-1); never drop | **D0-pending · RPO 0** |
| **FMEA-EV3** | C2 evidence store | Logical corruption / ransomware / malicious-admin delete | 1 | continuous chain-verify (C3), WORM tamper alert | chain breaks, events appear deleted | WORM object-locked backup in separate cred boundary (R-G3-BK-2); restore-to-checkpoint + attestation (§5) | D0 · RPO ≤15m / RTO ≤4h |
| **FMEA-EV4** | C2 evidence store | Region loss | 2 | region health, replication-lag breach | up to lag-window D0 at risk | cross-region replication (§7.2); edge-buffer reconciliation (R-G3-RS-6); disclosed gap | D0 · RPO ≤60s(async)/0(sync) / RTO ≤4h–15m |
| **FMEA-EV5** | C2 normalizer | Normalizer crash/backlog | 3 | ingest-lag metric, queue depth | ingest latency grows; no loss (raw retained, buffered) | HA replicas; raw-event retention (N-C2-402) enables re-normalize; buffer absorbs | D0 (deferred) · RPO 0 / RTO ≤30m |
| **FMEA-SG1** | Signer (checkpoint signing) | Signer unavailable | 3 | checkpoint-cadence-missed alert | checkpoints stop; chain still appends (hashes hold) | events keep committing (hash chain is the integrity floor); checkpoints resume on signer recovery; **gap in checkpoints ≠ data loss** | D0 · no RPO impact / RTO ≤1h |
| **FMEA-SG2** | Signer / key | **Signing key compromised** | 1 | (G4-owned detection) | future signatures untrustworthy | **→ G4 compromise-recovery**; G3 ensures restore/continuation signs with fresh key, never re-signs history (R-G3-RS-5/7) | D0 integrity · (G4) |
| **FMEA-BS1** | Signed-bundle server / OCI registry | Down/slow (shared dep) | 2 | breaker probe (R-G3-CB-1) | **without breaker: fleet mass-deny (X-4)** | **circuit breaker → degraded disposition (§4)**; spokes serve last-good cached bundle (F2 §6); §19 catch-up | enforcement avail · degraded-mode |
| **FMEA-XD1** | External-data / signature verifier | Down/slow (shared dep, the §18.1 headline) | 2 | breaker probe | **without breaker: fleet mass-deny on every signed-image check** | breaker → warn-and-admit-with-`infrastructure_degraded` for C-STANDARD; hard-deny kept only for C-CRITICAL safety controls (§4.1); §19 catch-up | enforcement avail · degraded-mode |
| **FMEA-ID1** | Keycloak / D1 identity broker | Down (shared dep) | 2 | breaker probe; B5-F1 | identity-aware controls fail-closed per-request (B5-F1) → **fleet-wide if every request needs identity** | breaker scopes the degraded disposition; cached JWKS/token introspection reduces hard dependency; C-SYSTEM exempt | enforcement avail · degraded-mode |
| **FMEA-OP1** | Operator / CRD controllers | Crash / reconcile conflict | 3 | leader-election health | reconciliation stalls; enforcement continues on last-applied | leader-elected 2-replica (F2-OP); CRDs are source of truth; restart is safe | D1 · RTO ≤30m |
| **FMEA-OP2** | Degraded-mode controller | Controller down | 2 | self-health, dead-man's-switch | **breaker can't open → reverts to raw fail-closed = X-4 risk; or breaker stuck open = enforcement gap (A-4)** | HA leader-elected; **fail-safe default = last-known breaker state persisted in CRD**, dead-man's-switch alerts; manual override (R-G3-CB-8) | control-plane avail · RTO ≤15m |
| **FMEA-CF1** | etcd / CRD store | Loss | 2 | etcd health | declared config/approvals lost | etcd snapshot ≤5m + GitOps re-apply (R-G3-CFG-1); approvals/exceptions in backup (R-G3-CFG-2) | D1 · RPO ≤5m / RTO ≤30m |
| **FMEA-CF2** | Keycloak realm/config | Loss | 3 | auth failures | identity mapping lost | realm config in GitOps/backup (D1); JWKS cacheable | D1 · RPO ≤5m / RTO ≤30m |
| **FMEA-DR1** | Derived stores (C3/indexes) | Loss/corruption | 4 | query errors, index-health | queries/dashboards degrade; **no data loss** | rebuild from D0 (R-G3-DRV-1), priority order; re-validate vs D0 | D2 · RPO n/a / RTO ≤1h degraded, ≤24h full |
| **FMEA-AP1** | Approval flow (B4 CRD) | Pending approval/exception lost in DR | 3 | reconcile diff | legit deploy re-denied; **stale exception re-opens closed bypass (A-5)** | approvals/exceptions = D1 in ≤5m RPO (R-G3-CFG-2); **exceptions re-validated against expiry on restore** (A-5 fix) | D1 · RPO ≤5m |
| **FMEA-SB1** | Spoke→hub link | Partition | 3 | link health (F2 §6) | spoke buffers logs; enforces on cached bundle | F2 §6 behavior; buffer durability (R-G3-BUF-1); reconcile on heal (§7.3) | D0-pending · RPO 0 (buffered) |
| **FMEA-NET1** | Cross-region link | Partition (split-brain risk) | 2 | replication-lag, both-sides-writing alarm | **two divergent chain tails = split-brain (A-1)** | single-writer-per-source-chain invariant (§7.3); quorum linearization; fence on partition | D0 integrity · RPO per topology |

- **R-G3-FMEA-1 (MUST):** Every FMEA row's mitigation MUST have a corresponding **chaos/DR-drill test** (§8) that exercises the failure and asserts the target is met. A mitigation with no drill is treated as unverified (and is a release-blocking gap for Sev-1/Sev-2 rows).

---

## 7. Multi-region / multi-cluster resilience

### 7.1 Topology recap (from F2 §2) and the resilience overlay

F2 establishes **hub-and-spoke**: a governance control plane (hub, `governance-system`) and enforcement edges (spokes) per cluster. G3 overlays resilience: **where does each data class live, what survives the loss of what, and how does the system avoid split-brain.**

- **Spokes** hold: D3 (in-flight), D0-pending (edge buffers), cached D1 (last-good signed bundle). Enforcement continues from cached bundle if the hub is unreachable (F2 §6).
- **Hub** holds: D0 (the C2 store), D1 (governance store, etcd, approvals), D2 (analytics/indexes). **The hub is the durability-critical tier**; its loss is the disaster G3 plans for.

### 7.2 Hub resilience tiers

- **R-G3-MR-1 (MUST):** The hub control plane (esp. the D0 store) MUST be deployable in one of three durability profiles, **explicitly selected** (priced/compared in `ALT-DR-TOPOLOGY.md`):
  - **(P1) Single-region, multi-AZ** (default POC+): ≥3 AZ quorum (RPO 0 to zone loss) + WORM backup off-region (RPO ≤15m to region loss, RTO ≤4h). Region loss = restore-from-backup.
  - **(P2) Active-passive warm standby** (cross-region async): primary region serves; standby region receives async-replicated D0 + checkpoints (RPO ≤60s) and can be promoted (RTO ≤4h, mostly DNS/failover + restore-tail reconciliation).
  - **(P3) Active-active synchronous** (regulated): D0 quorum spans regions, RPO 0 even on region loss, at the cost of cross-region write latency on the commit path and higher run cost (RTO ≤15m).
- **R-G3-MR-2 (MUST):** Spokes are **regionally independent for enforcement**: loss of the hub or a region MUST NOT stop admission at unaffected spokes — they keep enforcing on cached bundles and buffering D0-pending evidence (RPO 0 via durable buffers) until the hub returns. The blast radius of a hub/region loss is **evidence-ingest latency and control-plane features**, never fleet-wide admission failure (this is itself an X-4-adjacent property: a hub outage must not become a fleet mass-deny — guaranteed by F2 §6 cached-bundle + G3 degraded mode).

### 7.3 Split-brain prevention on the hash chain (the A-1 defense)

The append-only hash chain is **per-source** (C2 §7.3). Split-brain = two divergent tails appended to the *same source chain* in two regions during a partition, both claiming `chain_seq = N+1`.

- **R-G3-MR-3 (MUST):** Each per-source chain has **exactly one writer (linearization owner) at a time**, enforced by a **lease/fence** (leader election scoped per source-chain). On a partition, **only the side that holds the valid lease may advance `chain_seq`**; the other side **buffers (D0-pending) and does not append** to the authoritative chain. This makes a forked chain impossible by construction: there is never more than one writer issuing `chain_seq` for a given source.
- **R-G3-MR-4 (MUST):** In **active-active (P3)**, the per-source single-writer rule still holds — different source-chains may be owned by different regions, but a given source-chain's `chain_seq` is issued by exactly one region's writer (via the cross-region quorum's linearization). The quorum-commit (R-G3-RPO-4) **is** the linearization point; a minority partition cannot reach write-quorum and therefore cannot advance the chain (it buffers). On heal, the buffered side reconciles by appending its buffered events *after* the authoritative tail (re-`chain_seq`'d at commit), idempotent on `raw_event_digest`.
- **R-G3-MR-5 (MUST):** If, despite the fence, two divergent signed tails are ever detected (e.g. a botched manual failover that promoted a standby while the primary was still writing), the platform MUST treat it as a **Sev-1 integrity incident**: freeze both tails, run a **chain-reconciliation procedure** that (a) identifies the common ancestor checkpoint, (b) preserves *both* tails immutably as evidence (never silently discards one — both are real events that happened), (c) appends a signed `chain.fork_reconciliation` marker that merges them into a single forward chain by total-ordering on `(timestamp, event_id)` and re-attesting, and (d) raises an auditor disclosure. **No event is ever deleted to resolve a fork** — the fork itself is auditable history. This is the worst-case A-1 path and is drilled (CHAOS-9).

### 7.4 Regional failover reconciliation

- **R-G3-MR-6 (MUST):** Failover promotion of a standby (P2) follows: (1) **fence the old primary** (prevent zombie writes — split-brain prevention); (2) promote standby, head = last-replicated checkpoint; (3) emit `chain.restore_boundary` for the lag window (§5.5); (4) reconcile from edge buffers; (5) run the §5.7 restore-validation gate; (6) only then resume serving as authoritative. Fencing **before** promotion is mandatory and is the most error-prone manual step → it is automated and drilled (DRILL-2).

### 7.5 Cost/replication coupling to G2 (explicit seam)

- **R-G3-MR-7 (SHOULD):** The number of D0 copies, their geographic spread, the checkpoint/backup cadence, and the chosen profile (P1/P2/P3) are the **dominant cost drivers of the whole platform at production scale** (each copy is a full, growing, 7-year-retained evidence store). G3 states the *durability requirement*; **G2 owns the cost model** and `ALT-DR-TOPOLOGY.md` presents the RPO×cost×complexity trade so the buyer chooses with eyes open. G3 MUST NOT mandate P3 universally — it is correct for regulated D0 and ruinously expensive for C-LOW data.

---

## 8. Chaos engineering & DR-drill (game-day) test plan

A DR plan that has never been executed is a hypothesis. G3 mandates recurring, automated-where-possible drills with **explicit pass/fail acceptance criteria**, each tied to an FMEA row and a requirement.

### 8.1 Cadence & governance

- **R-G3-TEST-1 (MUST):** DR drills (full restore-and-verify, regional failover) run on a fixed cadence (default **quarterly** for full DR, **monthly** for chaos fault-injection in staging) and **at least once before GA**. Every drill produces a signed drill report (pass/fail per criterion, measured RPO/RTO vs. target). A missed or failed Sev-1/Sev-2 drill is a release/operations blocker.
- **R-G3-TEST-2 (SHOULD):** Chaos experiments run **continuously in staging** (steady-state hypothesis → inject → observe → auto-rollback) and **in production only with a blast-radius bound and an abort switch** (game-day, not random production chaos, given this is compliance infrastructure).

### 8.2 The drill catalog (each asserts a target)

| ID | Drill | Injects | Pass criteria (measured) | Verifies |
|---|---|---|---|---|
| **DRILL-1** | **Full evidence restore & chain re-verify** | Destroy the live D0 store in staging; restore from WORM backup | Chain `/verify` green end-to-end; sample export manifests (DT-24) re-verify; **Restore Attestation present**; RTO ≤4h; RPO ≤15m (data after last checkpoint accounted for as attested gap) | §5 (the crux), R-G3-RS-1/3/8, FMEA-EV3 |
| **DRILL-2** | **Regional failover** | Kill primary region | Standby promoted; old primary **fenced before promotion**; `restore_boundary` emitted for lag window; edge-buffer reconciliation recovers the recoverable tail; RPO ≤60s (async)/0 (sync); RTO ≤4h/≤15m | §7.4, R-G3-MR-6, FMEA-EV4 |
| **DRILL-3** | **Restore-looks-like-truncation test** | Restore a prefix, then run the auditor `/verify` | `/verify` reports a **legitimate attested restore boundary, NOT a tamper finding**; a *forged* truncation (no marker) **does** report tamper | R-G3-RS-3/4, A-1/A-2 |
| **CHAOS-4** | **Bundle-server mass-deny avoidance** | Take the signed-bundle server down fleet-wide | Breaker opens; C-STANDARD scopes **warn-and-admit** (no mass-deny); C-CRITICAL safety controls still deny; every degraded admission emits `infrastructure_degraded` audit; §19 catch-up queued | §4 (X-4 resolution), R-G3-CB-2/3/6, FMEA-BS1 |
| **CHAOS-5** | **Verifier-slow mass-deny avoidance** (the §18.1 headline) | Make the signature verifier slow past timeout | As CHAOS-4 for the signed-image control; request-level timeout still respected (B5-R6); breaker engages only on sustained signal (R-G3-CB-4) | FMEA-XD1, R-G3-CB-4 |
| **CHAOS-6** | **Breaker-as-bypass attack** | Adversary tries to trigger degraded mode to slip a non-compliant deploy | Degraded admissions are **loud, scoped, audited, time-bounded**; safety-critical controls **not** downgraded in C-CRITICAL; §19 catch-up **catches** the slipped resource; manual force-close requires dual-control | A-4, R-G3-AV-3, R-G3-CB-5/6/8 |
| **CHAOS-7** | **Anti-self-brick** | Take down a shared dep, then attempt to deploy its fix | The fix deploys: `C-SYSTEM` carve-out + bootstrap exemption admit the recovery workload **despite** the dependency being down; breaker never blocks its own recovery path | R-G3-CB-9 (the recursion stopper), FMEA-OP2 |
| **CHAOS-8** | **Degraded-mode-controller failure** | Kill the degraded-mode controller | Breaker state persisted in CRD survives; dead-man's-switch alerts; fleet does **not** silently revert to raw fail-closed mass-deny; HA failover ≤15m | FMEA-OP2 |
| **CHAOS-9** | **Hash-chain split-brain** | Force a partition + a botched double-promotion | Single-writer fence prevents the fork (expected); if forced, `fork_reconciliation` preserves **both** tails, deletes nothing, re-attests, discloses | §7.3, R-G3-MR-3/4/5, A-1, FMEA-NET1 |
| **CHAOS-10** | **Buffer-saturation / evidence-loss** | C2 down + sustained decision volume until buffers fill | Buffers spill to durable spool, **never drop**; `evidence_buffer_saturated` P1 fires; on C2 recovery all buffered events flush idempotently with RPO 0 | R-G3-BUF-1/2, FMEA-EV2 |
| **CHAOS-11** | **Stale-exception-on-restore** | Restore a backup containing an exception that expired during the outage | Restore **re-validates exceptions against current time/expiry**; expired exception does NOT silently re-open a closed bypass; triggers §19 re-scan | A-5, FMEA-AP1, R-G3-CFG-2 |

### 8.3 Acceptance gate

- **R-G3-TEST-3 (MUST):** Before GA, **DRILL-1, DRILL-2, DRILL-3, CHAOS-4, CHAOS-6, CHAOS-7, CHAOS-9, CHAOS-10** MUST pass with measured RPO/RTO within target. These eight are the load-bearing drills (the two crux problems: evidence durability + mass-deny, plus their adversarial corner cases). A POC profile MAY defer the multi-region drills (DRILL-2 in P2/P3 form) but MUST run DRILL-1, CHAOS-4/6/7/10 even single-region.

### 8.4 Steady-state observability for DR (handoff to G6)

- **R-G3-TEST-4 (MUST):** The following MUST be continuously observable (and are the metrics G6 dashboards/alerts on): `g3_d0_replication_lag_seconds` (RPO-region health), `g3_d0_buffer_oldest_age_seconds` (RPO-buffer health), `g3_chain_verify_status` (continuous chain integrity), `g3_last_checkpoint_age_seconds` (signer/checkpoint health), `g3_breaker_state{dependency,control}` (degraded-mode visibility), `g3_last_successful_drill_age_days` (DR-readiness). A stale/red value on any of these is an availability/durability risk that pages before it becomes an incident.

### 8.5 DR runbook (the human procedure)

- **R-G3-TEST-5 (MUST):** A versioned **DR runbook** MUST exist and be drill-validated, covering: declare-disaster criteria & authority; restore-from-WORM procedure (with the fence-before-promote ordering, §7.4); the Restore Attestation signing ceremony (dual-control); the post-restore validation gate (§5.7); the catch-up §19 trigger; the evidence-loss disclosure procedure; and roll-forward of edge buffers. The runbook is the human side of every MUST above; a MUST with no runbook step is incomplete. (Broader day-2 ops live in G6; the **DR-specific** runbook lives here.)

---

## 9. Decisions (decide-document-continue)

| ID | Decision | Rationale |
|---|---|---|
| **D-G3-01** | **Differentiated RPO/RTO by data class** (D0 RPO=0; D1 ≤5m; D2 rebuildable; D3 ephemeral), not one platform number. | One number is either dishonest (loses evidence) or ruinous (everything sync-replicated). The product is the D0 evidence; only it earns RPO=0. |
| **D-G3-02** | **D0 evidence RPO=0 up to zone loss** via ≥3-AZ write-majority quorum; chain `chain_seq` issued only post-quorum-commit. | The compliance thesis dies if "we have your decision" can be a lie (Risk #4). Quorum is the linearization point that also prevents chain forks. |
| **D-G3-03** | **Region-loss RPO is >0 only for the async profile, and is bounded, measured, and disclosed** (`evidence_lost_in_failover`), never silent. Regulated deployments SHOULD elect active-active (RPO 0). | RPO>0 on the audit log is the auditor's-evidence-loss fatal-to-compliance risk (A-3). The only acceptable RPO>0 is a *disclosed, reconcilable* one; the profile must be stated, not assumed. |
| **D-G3-04** | **Resolve the X-4 mass-deny via a fleet-aware circuit breaker + per-criticality "infrastructure-degraded" mode**, scoped to (dependency × control × criticality), distinct from per-request `failurePolicy` timeout. | "Policy says no" ≠ "platform is down." A coordinated breaker gives the fleet one coherent answer (degrade C-STANDARD to warn, keep C-CRITICAL safety controls denying) instead of 10k spokes each mass-denying. Granularity also defeats the breaker-as-bypass attack. |
| **D-G3-05** | **Degraded admissions are first-class audited** (`infrastructure_degraded` disposition + `degraded_session_id`) and **drive a mandatory §19 catch-up re-scan**; degraded mode is availability-preserving but the gap is *detectively closed, not forgiven*. | Makes warn-don't-deny acceptable to an auditor and makes the breaker non-bypassable: the gap is loud, scoped, time-bounded, and remediated (rebuts META G-6 for this path). |
| **D-G3-06** | **Anti-self-brick invariant** (R-G3-CB-9): the platform's own recovery path is exempt from its own fail-closed enforcement (C-SYSTEM carve-out + bootstrap exemption + breaker never blocks its own deps). | X-4's nastiest sub-case is fail-closed preventing recovery of the failed dependency. This breaks the recursion. |
| **D-G3-07** | **Restore = load + verify + append signed `restore_boundary` marker; NEVER re-hash/re-chain/re-sign history.** A discontinuity *with* a valid signed marker is a legitimate restore; *without* one it's tamper. | The only way to restore a signed append-only chain without (a) breaking every downstream signature or (b) being indistinguishable from malicious truncation. The marker is the anti-false-alarm primitive (A-1/A-2). |
| **D-G3-08** | **D0 backups go to immutable WORM storage in a separate credential/failure boundary**, continuous at checkpoint granularity. | Defends the C2 insider-deletion/ransomware threat that C2 §7.1 names but didn't extend to backups; quorum protects node loss, WORM protects logical corruption — different threats, two mechanisms. |
| **D-G3-09** | **Single-writer-per-source-chain fence** (lease/quorum-linearized) prevents hash-chain split-brain by construction; a forced fork is reconciled by preserving *both* tails (delete nothing) + a signed `fork_reconciliation` marker. | A forked signed chain is the worst integrity failure; preventing it is cheaper than reconciling it, and reconciliation must never delete real events. |
| **D-G3-10** | **G3 owns durability *requirements*; G2 prices them, G4 owns key custody/compromise, G5 owns tenant isolation, G6 owns day-2/observability.** G3 names every seam explicitly. | Avoids the META-review failure mode of NFRs with no owner; avoids G3 sprawling into cost/key/tenant decisions that belong elsewhere. |

## 10. Open questions (with decided defaults)

- **OQ-G3-1: Default DR profile for the POC?** *Default:* **P1 (single-region multi-AZ + off-region WORM backup)** — RPO 0 to zone loss, RPO ≤15m to region loss. P2/P3 are deployment elections priced in `ALT-DR-TOPOLOGY.md`. Rationale: POC is functional validation (F3 §22); production regulated buyers elect P2/P3.
- **OQ-G3-2: Checkpoint cadence vs. region-loss RPO interaction?** *Default:* keep C2's N=10k/T=15m checkpoint cadence; the WORM-backup RPO equals checkpoint cadence (≤15m), while quorum gives RPO 0 — the two are independent. Tightening cadence narrows both the backdating window (C2 OQ-2) and the backup RPO, at signing cost (G4) — revisit at scale (G1/G2).
- **OQ-G3-3: Auto-close vs. human-in-the-loop for the breaker?** *Default:* **auto-close on sustained recovery for C-STANDARD/C-LOW; require human confirmation to re-enable hard-deny on C-CRITICAL safety controls** (dual-control, R-G3-CB-8) — because auto-re-enabling hard-deny can itself cause an outage if the recovery is flaky.
- **OQ-G3-4: How aggressively to reconcile edge buffers after failover?** *Default:* best-effort recover-then-disclose: recover everything the buffers still hold; disclose the irrecoverable remainder as `evidence_lost_in_failover`. The buffer retention window (R-G3-BUF) bounds how much is recoverable — a tunable durability/cost knob (G2).
- **OQ-G3-5: Does the breaker probe live in the hub or the spoke?** *Default:* **hub-side aggregation, spoke-side application** — the degraded-mode controller (hub, HA) aggregates the fleet-wide signal and publishes breaker state to a CRD the spokes watch; spokes apply it locally so enforcement decisions stay at the edge (F2-SCALE-2) and a hub partition doesn't strand the spoke (it keeps the last-published breaker state — FMEA-OP2/CHAOS-8).

## 11. Dependencies

- **Consumes:** **C2** §7–§8 (chain/checkpoint/CAS/retention model — G3's backup/restore preserves it), **B5** §5 (per-flow failure modes G3 composes), **F2** §2–§6 (hub/spoke topology, HA, `failurePolicy`, cached-bundle behavior), **B1/B2/B4** (the per-engine fail-closed defaults composed into FMEA-BS1/XD1/ID1), **C4/§19** (the catch-up re-scan consumer of degraded-mode gaps).
- **Coordinates with (named seams):** **G1** (volume model sizes backup/restore windows), **G2** (prices the DR-copy/replication/profile choices — `ALT-DR-TOPOLOGY.md`), **G4** (key custody, historical-key validity for restore-verify, signing-key-compromise recovery — R-G3-RS-6/7, FMEA-SG2), **G5** (tenant blast-radius; G3 owns *availability* blast-radius only), **G6** (DR observability metrics handoff §8.4; broader day-2 ops; the DR runbook is G3's, day-2 is G6's).
- **Consumed by / blocks:** any production (non-POC, regulated) go-live — DRILL-1 + CHAOS-4/6/7/10 are GA gates (R-G3-TEST-3). The degraded-mode `infrastructure_degraded` disposition is a **new C2 audit reason** G3 contributes back to C2 (additive, N-C2-FWD-compatible) and a **new §19 input** to C4.
