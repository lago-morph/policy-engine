# C3 — Compliance Analytics Engine — SPEC

**Component ID:** C3 · **Domain:** C — Evidence, Audit & Analytics
**Spec sources:** §14 (Compliance Analytics Engine: §14.1 Responsibilities, §14.2 Example Detections), with inputs from §13 (C2 schema — consumed), §9.3, §15.2, §17E (reports), §6 (governance hierarchy/scope), §7 (lifecycle).
**Status:** SPEC (depends on C2 frozen schema v1.0).
**Scenarios exercised:** DT-30 (Gatekeeper bypass), DT-31 (JWT policy drift), DT-32 (cross-cluster inconsistency), DT-33/DT-80 (coverage gaps), DT-28 (correlation gap), HL-09 (multi-cluster drift), HL-20 (federated).

---

## 1. Scope

C3 is the **detection engine** that runs continuously over the C2 event stream/store and emits **findings**. It does **not** store raw events (C2), render reports (C5), produce control verdicts (C1), or run policy replay (E1). It consumes the frozen C2 schema and the derived views (correlation members §10.3, coverage feed §10.4) and produces typed findings that C5 reports, C1 ingests as evidence, and operators act on.

### 1.1 Responsibilities (§14.1, normative)
C3 SHALL: detect policy bypasses · detect inconsistent enforcement · correlate runtime and build-time outcomes · identify drift between governance and runtime · detect missing enforcement coverage · generate compliance-evidence inputs (findings) for reports.

### 1.2 Finding model
```
Finding {
  finding_id: UUIDv7
  type: enum   // enforcement_bypass | inconsistent_enforcement | policy_drift |
               // jwt_policy_drift | coverage_gap | correlation_gap | ungoverned_enforcement |
               // chain_integrity | audit_latency
  severity: enum  // critical | high | medium | low
  control_id?: string
  scope: {cluster?, namespace?, tenant?, region?, environment?}
  detected_at: rfc3339
  window: {from, to}            // the detection window that produced it
  evidence_refs: [c2_event_id | correlation_id | coverage_cell]
  detail: object                // type-specific payload (algorithm output)
  confidence: high|medium|low
  state: open | acknowledged | resolved | suppressed
  reconciliation_interval: duration   // the cadence that found it (e.g. 15m)
}
```
All findings carry the `evidence_refs` that let an operator (or C5/C1) trace the finding back to the underlying C2 events. **N-C3-1: every finding MUST be reproducible from its `evidence_refs` + the named detector version** (determinism — the same inputs reproduce the same finding).

---

## 2. The detections (concrete algorithms)

Each detector has: trigger cadence, inputs (C2 queries), the algorithm, the emitted finding, and the scenario it satisfies. Detectors run on a **reconciliation interval** (default 15 min, configurable per detector) over a sliding window, and additionally on backfill of historical windows.

### 2.1 D-BYPASS — Enforcement bypass via missing event (§14.2, §19; DT-30, DT-42, HL-06)
**Trigger:** every reconciliation interval (≤15 min, DT-30 success criterion).
**Inputs (C2 queries):**
- K8s-API audit events: `event_type=resource.change`, `operation ∈ {create,update}`, `resource_type ∈ {Pod,Deployment,StatefulSet,…}`, in scope, in window.
- For each, the correlation-members view (§10.3) for its `correlation_id`.
**Algorithm:**
```
for each resource.change R where R.control_id ∈ in-scope-controls(R.scope):   // control applies
   members = C2.correlations(R.correlation_id).present
   if 'gatekeeper' ∉ members AND 'opa' ∉ members:        // no admission evaluation, no decision log
        // the three §14.2 conditions: deployment exists, no GK event, no OPA decision
        emit Finding{
          type: enforcement_bypass, severity: high,
          control_id: applicable_control(R), scope: R.scope,
          detail: { resource_id: R.resource_id, missing: ['gatekeeper','opa'],
                    k8s_audit_event: R.event_id },
          confidence: high }
```
**Refinement (root-cause classification):** join the window to Gatekeeper webhook health (an operational signal); if the webhook was `unhealthy` during the window, tag `detail.root_cause=infrastructure_induced` (DT-42 step 4); else `suspected_malicious` (HL-06: webhook deliberately removed).
**Handoff:** the finding triggers C4 to reconstruct + replay the input (the synthetic `best_effort` event), and feeds C5's §17E.3 Audit-Derived Violation Report. **C3 detects the gap; C4 reconstructs/replays; C5 reports.** (Clean ownership split.)
**Edge cases:** a resource updated many times shares one `resource_id` across correlation groups — evaluate per-`correlation_id`, not per-`resource_id`, so a single bypassed *change* is caught even if other changes were enforced.

### 2.2 D-INCONSISTENT — Inconsistent enforcement across clusters (§14.1; DT-32, HL-09)
**Trigger:** reconciliation interval; also on cross-cluster rollups (HL-20).
**Inputs:** `policy.decision` events grouped by `(constraint_name + control_id)` across clusters in scope, in window.
**Algorithm:**
```
for each (control_id, constraint_name) group G across clusters:
   // (a) same input, divergent verdict
   key(e) = hash(canonical(e.request_object) + e.jwt_claims_relevant + e.external_data_refs)
   for each input-equivalence class K in G:
      verdicts = set(e.decision for e in K)
      if |verdicts| > 1:
         emit Finding{ type: inconsistent_enforcement, severity: high,
           detail: { control_id, divergence: per-cluster decision counts,
                     example_input: K.sample, divergence_rate } }   // DT-32: deny on a, allow on b
   // (b) version drift as the usual cause
   versions = { cluster: most_common(e.policy_version) for clusters in G }
   if |distinct(versions.values)| > 1:
      emit Finding{ type: policy_drift, severity: high,
        detail: { control_id, expected_version: source_of_truth_version(control_id),
                  observed_versions: versions } }   // HL-09: cluster-b on v11, rest on v12
```
DT-32 establishes the version delta (v11 vs v12) as the *cause* of the divergent verdict; C3 emits **both** an `inconsistent_enforcement` (the symptom) and a `policy_drift` (the cause) finding, linked.

### 2.3 D-DRIFT — Governance↔runtime policy drift (§14.1; HL-09)
**Trigger:** reconciliation interval.
**Inputs:** observed `policy_version` per `(control_id, cluster, constraint_name)` from C2 vs. the **source-of-truth bundle version** from the governance/lifecycle store (A1/A2/B1).
**Algorithm:**
```
for each (control_id, cluster):
   observed = most_recent(e.policy_version) in window
   expected = SoT.deployed_version(control_id, cluster)
   if observed != expected:
       emit Finding{ type: policy_drift, severity: high,
         detail: { control_id, cluster, expected, observed,
                   drift_age: now - first_seen(observed) } }
```
This is the "runtime is not running what governance says" detector (the v11-stuck-cluster of HL-09). Distinct from D-INCONSISTENT (which compares clusters to *each other*); D-DRIFT compares runtime to *source of truth*.

### 2.4 D-JWT-DRIFT — JWT policy drift (§14.2; DT-31)
**Trigger:** reconciliation interval (DT-31: "within one reconciliation interval").
**Inputs:** for each policy package, the §15.2 required-claim list (from policy metadata / D1 mapping); the `jwt_claims` actually present across `policy.decision` events in window.
**Algorithm:**
```
for each policy_package P with required_claims R:
   events = C2.query(policy_ref.rego_package=P, window)
   for each required claim c in R:
      omit_rate = count(e where c ∉ e.jwt_claims) / count(events)
      if omit_rate > threshold (default 5%):
         emit Finding{ type: jwt_policy_drift, severity: high,
           detail: { policy_package: P, claim: c, omit_rate,
                     example_subjects: sample,
                     decision_split: { allow: n, deny_missing_claim: m } } }
```
DT-31: a Keycloak realm change re-scopes the `tenant` mapper; ≥18% of JWTs omit `tenant`; deny-missing-claim events spike. C3 fires when omit_rate crosses threshold and includes the allow/deny-missing-claim split so the compliance impact (enforcement degradation) is visible.

### 2.5 D-COVERAGE — Missing enforcement coverage (§14.1; DT-33, DT-80)
**Trigger:** scheduled (e.g. daily) + on inventory change.
**Inputs:** C2 coverage feed (§10.4) — observed decision counts per `(namespace × control)`; the in-scope expectation set from A1 governance + K8s inventory.
**Algorithm:**
```
build matrix (namespace × control) for scope:
  for each cell (ns, c):
     installed = binding_exists(ns, c)         // constraint/policy bound
     decisions = count(C2 decisions ns,c,window)
     applicable = A1.in_scope(ns, c)            // control applies to this namespace tier
     classify:
        not applicable        → 'n/a'
        applicable & !installed → 'not_installed'   // §14.1 missing coverage
        installed & decisions==0 → 'installed_no_events'  // installed but silent (DT-33)
        installed & decisions>0  → 'enforced'
  for each cell classified not_installed OR (installed_no_events AND workloads_exist(ns)):
     emit Finding{ type: coverage_gap, severity: high if applicable else low,
       detail: { namespace: ns, control: c, classification, workloads_observed,
                 expected_admissions, evaluated, coverage_pct } }
```
DT-33 nuance: `payments-staging` shows `installed` with an `excludedNamespaces` selector — so it's *effectively* not_installed for that namespace despite a constraint existing. The detector must evaluate the *match selector*, not just constraint existence (D-C3-coverage-selector). DT-80 produces the full (namespace × control) matrix with the four classifications.
**Adversarial guard (from C2 A12):** the expectation set carries a source+timestamp; if the inventory is stale, emit a separate `coverage_gap` of subtype `stale_inventory` rather than silently classifying missing namespaces as `n/a`.

### 2.6 D-CORRELATION — Correlation gap between Gatekeeper and OPA (§13.3; DT-28)
**Trigger:** rolling 15-min window (DT-28: ">1% of denies unpaired in 15 min").
**Inputs:** `policy.decision` Gatekeeper denies; their correlation-members.
**Algorithm:**
```
denies = C2.query(policy_engine=gatekeeper, decision=deny, window=15m)
unpaired = [d for d in denies if 'opa' ∉ correlations(d.correlation_id).present]
rate = |unpaired| / |denies|
if rate > 1%:
   emit Finding{ type: correlation_gap, severity: medium,
     detail: { unpaired_rate: rate, sample: unpaired[:10],
               likely_cause: 'OPA decision log not echoing admission UID (§9.3/DT-28)' } }
```
This is the detector DT-28 prescribes; it fires on a synthetic config-revert test and clears within one window after restoration (PLAN test). It guards the correlation anchor that everything else depends on.

### 2.7 D-RUNTIME-VS-BUILD — Correlate runtime and build-time outcomes (§14.1)
**Trigger:** on new runtime admission for an image previously scanned at build (Conftest, §10.3).
**Inputs:** runtime `policy.decision` for a `resource_id`/image; the build-time Conftest C2 event for the same image/control.
**Algorithm:** join runtime↔build by image digest + `control_id`; if build-time **passed** but runtime **denies** the same control (or vice versa) → emit `inconsistent_enforcement` subtype `runtime_build_divergence` (a policy changed between build and deploy, or an external-data value changed — links to D-DRIFT/external-data drift).

### 2.8 D-CHAIN — Chain integrity (from C2 §7.3; adversarial D6)
**Trigger:** continuous.
**Inputs:** C2 verify endpoint (§10.6) over each source chain.
**Algorithm:** on a `prev_hash` mismatch or `chain_seq` gap or failed checkpoint signature → emit `chain_integrity` finding, severity **critical** (tamper indicator). This is the detector that makes C2's tamper-evidence *actionable* rather than theater (C2 adversarial R4).

### 2.9 D-LATENCY / D-UNGOVERNED (supporting)
- **D-LATENCY:** large `ingest_timestamp − timestamp` gaps → `audit_latency` finding (evidence arriving too late to act on).
- **D-UNGOVERNED:** `policy.decision` events with no `control_id` → `ungoverned_enforcement` finding (enforcement happening outside the governance model — a G1 traceability gap).

---

## 3. Cross-cutting detector requirements
- **N-C3-2 Scope-filtered:** detectors run within the caller's/tenant's authorized scope; federated rollups (HL-20) dedup on the cluster-scoped `correlation_id` (per C2 adversarial D5), never the bare UID.
- **N-C3-3 Confidence honesty:** a detection that rests on `best_effort`/`insufficient` C2 events carries reduced `confidence` and discloses it — C3 never asserts `high` confidence over `insufficient` inputs.
- **N-C3-4 Determinism:** each detector is versioned; re-running a detector version over the same window + same evidence reproduces the same findings (replay-determinism applies to detections too — PLAN T-DET).
- **N-C3-5 No reconstruction/replay here:** when a detection needs a reconstructed input or a replay (bypass → "what would the verdict have been?"), C3 *requests* it from C4/E1 and ingests the result; C3 does not embed a replay engine. (Ownership: C3 detects, C4 reconstructs over a window, E1 replays.)

## 4. Failure modes
- **C2 query lag/unavailable:** detectors degrade gracefully (skip-and-log a missed interval, then backfill); a missed interval is itself recorded so coverage of *detection* is auditable.
- **Threshold tuning false positives:** thresholds (omit_rate, unpaired_rate) are configurable per scope; findings carry the threshold that fired so an operator can recalibrate.
- **Detector vs detector overlap:** D-INCONSISTENT and D-DRIFT can both fire for one root cause; they are *linked* (shared `detail.root_finding_id`) not deduped away, because symptom and cause are both useful.

## 5. Decisions
| ID | Decision | Rationale |
|---|---|---|
| D-C3-01 | Detect per-`correlation_id`, not per-`resource_id`, for bypass | A resource with many changes can have one bypassed change among enforced ones (DT-30 edge). |
| D-C3-02 | Coverage evaluates the **match selector**, not just constraint existence | DT-33: an `excludedNamespaces` selector makes an "installed" constraint effectively absent. |
| D-C3-03 | Emit symptom (`inconsistent_enforcement`) **and** cause (`policy_drift`) linked | Operators need both; DT-32/HL-09. |
| D-C3-04 | C3 detects; C4 reconstructs/replays; C5 reports; C1 ingests | Single ownership per concern; no duplicated detection/replay (N-C3-5). |
| D-C3-05 | Findings reduced-confidence + disclosed over `best_effort`/`insufficient` evidence | Inherits C2 honesty (N-C3-3). |
| D-C3-06 | Chain-integrity is a detector (D-CHAIN), not just a C2 internal | Makes C2 tamper-evidence actionable (C2 adversarial R4). |

## 6. Open questions (defaults)
- **OQ-1:** Streaming vs. batch detectors? *Default:* batch reconciliation on a configurable interval (15m default) for correctness/determinism; a streaming fast-path may be added for `enforcement_bypass`/`chain_integrity` where latency matters, but the batch pass is authoritative.
- **OQ-2:** Where does the source-of-truth deployed-version come from for D-DRIFT? *Default:* A2/B1 bundle-distribution record; if unavailable, fall back to the modal observed version and flag lower confidence.

## 7. Dependencies
- **Consumes:** C2 (events, correlation-members §10.3, coverage feed §10.4, verify §10.6); A1 (in-scope controls per namespace tier); A2/B1 (source-of-truth deployed versions); D1 (required-claim lists); operational webhook-health signal (for bypass root-cause).
- **Produces (consumed by):** C5 (findings → reports, esp. §17E.3 and coverage/drift reports); C1 (findings as evidence for verdicts); C4 (bypass findings trigger reconstruction); operators/alerts.
- **Requests from:** C4 (reconstruction over a window), E1 (replay of a reconstructed input).
- **Blocks on:** C2 schema freeze (M-FREEZE).
