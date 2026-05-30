# E1 — Policy Simulation & Dry-Run Framework — SPEC

**Component:** E1 · **Domain:** E — Simulation & Console
**Authoritative spec:** §17 (§17.1–§17.6), with hard dependencies on §13 (audit schema), §17C.6 (CRDs), §17E.4 (simulation report), §17A.5 (storage authz).
**Author persona:** Marcus (Platform Security Engineer) — cooperative author.
**Status:** DRAFT v1 · **Date:** 2026-05-30

> Primary authoring workflow this component exists to serve (§17.1):
> *"This event happened and should not happen again. I wrote or modified a policy. I want to verify the policy catches the intended behavior, does not block supported behavior, and does not unintentionally allow behavior that was previously blocked."*

---

## 1. Scope

E1 is the **simulation engine + dry-run framework**. It is a *first-class product capability* (§17.1), not a debugging convenience. It evaluates one or more policy bundles against a defined **evidence set** (manifests, historical audit events, live traffic mirror, or cluster snapshot) and produces classified, taggable, reportable results without mutating production state.

**In scope**

- The 9 simulation modes (§17.2) — each with a defined input contract, data source, determinism class, and output.
- The differential-simulation algorithm (§17.4): compare previous vs new policy outcomes over an identical evidence set and classify into the four-quadrant matrix.
- The tagging workflow (§17.4) for newly-blocked and newly-allowed results (intended vs regression/false-positive).
- The audit-driven test-case creation workflow (§17.5) and example end-to-end workflow (§17.6).
- The `replay_completeness` gating model (§17.3): which modes/results are *authoritative* vs *advisory*.
- The `PolicySimulationRun` CRD lifecycle (§17C.6) and its reconciliation.
- The §17E.4 Simulation Report contract (E1 is the *producer*; C5 Reporting is the consumer/renderer).

**Out of scope (owned elsewhere)**

- The audit event schema and its storage/retention — **C2** (§12–13). E1 *consumes* it.
- Rendering reports / dashboards — **C5** (§17E) and **E2** (Console).
- The enforcement engines themselves (OPA, Gatekeeper, Conftest, Kyverno) — **B1–B5**. E1 *invokes* them in evaluate-only mode.
- Approval workflow / exception CRDs — **D3** (§17B), **B4** (`PolicyException`). E1 *reads* exceptions as input and *links to* exception requests as output.
- RBAC scope enforcement at storage layer — **D2** (§17A.5). E1 *honors* the namespace/scope boundary on every evidence query.

---

## 2. Data model & entities

### 2.1 Core entities

| Entity | Description | Key fields |
|---|---|---|
| `SimulationRun` | One execution of a mode over an evidence set | `run_id`, `mode`, `previous_bundle?`, `new_bundle`, `evidence_set_ref`, `scope`, `requested_by`, `status`, `started_at`, `finished_at`, `report_ref`, `authoritative` |
| `EvidenceSet` | A materialized, immutable snapshot of inputs to evaluate | `evidence_set_ref`, `source_kind`, `selector`, `event_count`, `replay_completeness_histogram`, `materialized_at`, `scope`, `digest` |
| `EvaluatedEvent` | One policy input fixture + per-bundle decision(s) | `event_id`, `policy_input`, `replay_completeness`, `decision_previous?`, `decision_new`, `classification?`, `tag?`, `tag_rationale?`, `engine`, `bundle_versions` |
| `DiffResult` | Aggregate of a differential run | `run_id`, `events_evaluated`, `newly_blocked`, `newly_allowed`, `unchanged_allowed`, `unchanged_denied`, `incomplete_count`, `tagged_intentional`, `untagged_risky`, `fp_candidates`, `fn_candidates` |
| `RegressionFixture` | A saved test case derived from an audit event (§17.5) | `fixture_id`, `policy_input`, `desired_outcome`, `control_id`, `policy_ref`, `bundle_version`, `source_event_id`, `created_by` |
| `SimulationTag` | A reviewer's classification of a changed decision | `event_id`, `run_id`, `tag`, `rationale`, `tagged_by`, `tagged_at`, `exception_ref?` |

### 2.2 Policy input fixture

The atomic unit E1 evaluates. It is reconstructed from a §13 audit event (replay) or extracted from a manifest/snapshot/live request. It MUST be byte-stable so re-runs are reproducible.

```jsonc
{
  "fixture_id": "uuid",
  "source": { "kind": "audit_event", "event_id": "uuid", "correlation_id": "uuid" },
  "policy_input": { /* normalized §13 input: request_object, subject, jwt_claims, before/after, operation, ... */ },
  "external_data_snapshot": [ { "name": "image-signature-status", "version": "2026-05-12T00:00:00Z", "value": {} } ],
  "replay_completeness": "complete | partial | insufficient",
  "missing_fields": ["signer"],         // populated when not complete
  "control_id": "SC-IMG-001",
  "scope": { "cluster": "cluster-a", "namespace": "payments-prod", "tenant": "payments" }
}
```

**Key principle (§13.1, §17.3):** a replay record that lacks information the original engine used cannot produce an *authoritative* result. E1 carries `external_data_snapshot` so external-data-dependent policies replay deterministically against the *version that was live at decision time* — not against today's external data.

---

## 3. The 9 simulation modes (normative)

Each mode is specified by **{input contract, data source, determinism class, output, authoritative-when}**. Determinism classes:

- **D-FULL** — pure function of immutable fixtures; identical inputs ⇒ identical output forever.
- **D-SNAPSHOT** — deterministic against a captured snapshot, but the snapshot is a point-in-time view of mutable state.
- **D-LIVE** — non-deterministic; depends on live traffic at run time; not reproducible.

> Mode IDs E1-M1..M9 map to the §17.2 table rows in order.

### M1 — Manifest Simulation
- **Input:** one or more proposed Kubernetes manifests / config files (YAML/JSON), a target bundle, optional namespace scope.
- **Data source:** user-supplied files (PR diff, local files, console paste). No audit dependency.
- **Determinism:** **D-FULL** (manifests are static).
- **Output:** per-manifest decision (allow/deny/warn/mutate + mutation diff), violation messages, matched control IDs.
- **Authoritative-when:** always (no replay_completeness concern — external data must be supplied or stubbed; if a policy needs external data not supplied, result is `partial`).
- **Engines:** Conftest, OPA `eval`, Gatekeeper `gator`, Kyverno CLI `test`.
- **Scenario:** DT-45 (pre-PR manifest sim).

### M2 — Historical Replay
- **Input:** a §13 audit-event selector (time range, control_id, namespace, product, event_type) + a bundle.
- **Data source:** **C2 Audit Schema Service** (replay-capable events).
- **Determinism:** **D-FULL** *iff* every selected event is `replay_completeness=complete` and `external_data_snapshot` is preserved; otherwise the incomplete subset is **D-SNAPSHOT/advisory**.
- **Output:** per-event re-decision under the new bundle; count of changed vs unchanged vs incomplete.
- **Authoritative-when:** the report is marked authoritative only over the `complete` subset; the `partial`/`insufficient` subset is reported separately and flagged non-authoritative (§17.3).
- **Scenario:** DT-46 (30-day replay).

### M3 — Live Shadow Mode
- **Input:** a live admission/decision stream tee; a candidate bundle to evaluate in parallel with the enforcing bundle; a time window.
- **Data source:** production traffic mirror (e.g., Gatekeeper `dryrun` action, Kyverno `Audit` action, or an OPA decision-log sidecar). **No enforcement** — decisions are observed only.
- **Determinism:** **D-LIVE** (not reproducible; each run sees different traffic).
- **Output:** streaming/accumulating decision stats, recent would-be-denies, drift indicators feeding E2's Runtime Enforcement View.
- **Authoritative-when:** authoritative as a *forward-looking* signal for the window observed; cannot be re-run for the same window. Results SHOULD be archived as a (now-immutable) EvidenceSet so a later differential run can replay them.
- **Scenario:** DT-47.

### M4 — Cluster Snapshot Simulation
- **Input:** a target bundle; a scope (cluster/namespace); optionally a captured snapshot digest.
- **Data source:** current live cluster state read via Kubernetes API (all in-scope resources), materialized to an immutable snapshot.
- **Determinism:** **D-SNAPSHOT** (deterministic against the captured snapshot; the snapshot itself is point-in-time).
- **Output:** which currently-deployed resources *would* violate the bundle if it were enforced now ("how many existing pods are non-compliant?"). Used for upgrade/impact analysis.
- **Authoritative-when:** authoritative against the captured snapshot digest; results reference the snapshot timestamp.
- **Scenario:** DT-48 (snapshot sim before upgrade).

### M5 — Differential Policy Simulation  *(the flagship — see §4)*
- **Input:** `previous_bundle`, `new_bundle`, an identical EvidenceSet (typically from M2 historical replay or M3 archived shadow).
- **Data source:** C2 audit events (or any materialized EvidenceSet).
- **Determinism:** inherits the EvidenceSet's class (D-FULL over the complete subset).
- **Output:** the §17.4 four-quadrant matrix + tagging state + §17E.4 report.
- **Authoritative-when:** over the `complete` subset only.
- **Scenarios:** DT-49, HL-17.

### M6 — Namespace Simulation
- **Input:** any other mode's input, **constrained to authorized namespaces** for the requesting subject.
- **Data source:** the underlying mode's source, **filtered at the storage layer** (D2 §17A.5) so a Namespace Policy Author cannot read events outside their namespaces — GUI filtering is insufficient (§17A.1).
- **Determinism:** inherits underlying mode.
- **Output:** results restricted to in-scope objects; effects (e.g., regression fixtures saved) restricted to in-scope namespaces.
- **Authoritative-when:** inherits; additionally the scope boundary MUST be enforced server-side and recorded on the run.
- **Scenarios:** DT-50, HL-08.

### M7 — Previously Blocked Replay  *(Deny→? subset)*
- **Input:** a §13 selector limited to historical `decision=deny` events + a new bundle.
- **Data source:** C2 audit events filtered to prior denials.
- **Determinism:** D-FULL over complete subset.
- **Output:** which previously-denied actions the **new policy would now ALLOW** (Deny→Allow), i.e., relaxations/regressions to scrutinize. This is the safety-critical "did I accidentally open a hole?" check.
- **Authoritative-when:** over complete subset; each Deny→Allow MUST be tagged before promotion.
- **Scenario:** subset of DT-49 / HL-17.

### M8 — Intended Behavior Test
- **Input:** one or more target audit events that represent behavior that *should* be caught + the new bundle.
- **Data source:** specific C2 events (selected by the user as "this is the thing that should be blocked").
- **Determinism:** D-FULL.
- **Output:** PASS/FAIL — does the new policy produce the desired outcome (typically deny/suspend) for the target behavior? Becomes a regression fixture (§17.5).
- **Authoritative-when:** target event must be `complete`; else the test is inconclusive.
- **Scenario:** DT-51 (regression test from audit).

### M9 — False Positive Test
- **Input:** one or more audit events representing *supported, legitimate* historical behavior + the new bundle.
- **Data source:** specific C2 events (selected as "this must keep working").
- **Determinism:** D-FULL.
- **Output:** PASS/FAIL — do supported historical behaviors **remain allowed** under the new policy? A FAIL is a false-positive candidate.
- **Authoritative-when:** target events `complete`.
- **Scenario:** DT-52 (false positive test).

### Mode summary table

| ID | Mode | Source | Determinism | Authoritative gate | Scenario |
|---|---|---|---|---|---|
| M1 | Manifest | user files | D-FULL | external data supplied | DT-45 |
| M2 | Historical Replay | C2 audit | D-FULL over complete | replay_completeness=complete | DT-46 |
| M3 | Live Shadow | traffic mirror | D-LIVE | window-only, not re-runnable | DT-47 |
| M4 | Cluster Snapshot | K8s API snapshot | D-SNAPSHOT | snapshot digest | DT-48 |
| M5 | Differential | C2 / EvidenceSet | inherits | complete subset | DT-49, HL-17 |
| M6 | Namespace | any, scoped | inherits | + server-side scope | DT-50, HL-08 |
| M7 | Previously-Blocked Replay | C2 (denies) | D-FULL | complete subset | DT-49/HL-17 |
| M8 | Intended Behavior Test | C2 (targets) | D-FULL | targets complete | DT-51 |
| M9 | False Positive Test | C2 (supported) | D-FULL | targets complete | DT-52 |

---

## 4. The differential-simulation algorithm (normative)

Given `previous_bundle P`, `new_bundle N`, and an `EvidenceSet S = {e_1..e_n}` of policy-input fixtures.

```
function differential_simulate(P, N, S):
    materialize S immutably; compute digest; record replay_completeness histogram
    result = { newly_blocked:[], newly_allowed:[], unchanged_allowed:[],
               unchanged_denied:[], incomplete:[] }
    for e in S:
        if e.replay_completeness == "insufficient":
            result.incomplete.append(e); continue        # cannot decide authoritatively
        # Evaluate against BOTH bundles using the SAME reconstructed input + the
        # external_data_snapshot captured at decision time (NOT live external data).
        d_prev = evaluate(P, e.policy_input, e.external_data_snapshot)
        d_new  = evaluate(N, e.policy_input, e.external_data_snapshot)
        prev_allow = is_allow(d_prev)   # normalize warn/mutate/allow -> "allow-class";
        new_allow  = is_allow(d_new)    # deny/suspend_pending_approval -> "deny-class"
        bucket = classify(prev_allow, new_allow)   # §17.4 matrix
        if e.replay_completeness == "partial":
            mark e as advisory (counts toward bucket but flagged non-authoritative)
        result[bucket].append(e)
    return aggregate(result)
```

### 4.1 §17.4 classification matrix

| Previous | New | Bucket |
|---|---|---|
| Allow-class | Deny-class | **Newly blocked** |
| Deny-class  | Allow-class | **Newly allowed** |
| Allow-class | Allow-class | **No enforcement change** (unchanged allowed) |
| Deny-class  | Deny-class  | **Continued block** (unchanged denied) |

**Allow-class vs deny-class normalization** (decided default): `allow`, `warn` (permit+log), `mutate` (permit+modify) are **allow-class** for differential purposes; `deny` and `suspend_pending_approval` are **deny-class**. A separate *secondary* diff tracks `mutate` diffs and `warn↔allow` transitions because a warn→deny tightening within allow-class would otherwise be invisible. **Decision D-E1-01:** surface a sub-classification (`effect_changed_within_class`) so a relaxation from deny→warn (still allow-class) is still reviewable.

### 4.2 Reconciliation invariant (testable)

`newly_blocked + newly_allowed + unchanged_allowed + unchanged_denied + incomplete == events_evaluated`. The §17E.4 report MUST show this reconciliation (DT-49 success criterion).

### 4.3 External-data determinism

Differential runs MUST replay against `external_data_snapshot` (the version/digest live at the original decision time, per §13 `external_data_refs`). Replaying against *current* external data would conflate policy change with data change and produce non-reproducible diffs. **Decision D-E1-02:** if `external_data_snapshot` is absent for an event whose policy reads external data, that event is downgraded to `replay_completeness=partial` and reported as advisory.

---

## 5. Tagging workflow (§17.4, normative)

After a differential run, every **changed** result (newly blocked / newly allowed) MUST be tagged before the policy can be promoted (gate enforced by A2 Policy Lifecycle / D3 approval).

**Newly allowed** tag set: `Intended relaxation`, `Potential regression`, `Requires review`, `Approved exception`.
**Newly blocked** tag set: `Intended enforcement`, `Potential false positive`, `Requires review`, `Emergency block`.

```
state machine per changed event:
  UNTAGGED --tag--> TAGGED(tag, rationale, tagged_by)
  TAGGED --retag--> TAGGED            # audit-logged, prior tag retained in history
  TAGGED(Potential false positive | Potential regression) --file_exception-->
        links exception_ref (D3 §17B PolicyException request); event becomes Approved-exception when exception approved
```

- **Promotion gate:** `untagged_risky == 0` is REQUIRED before promote (DT-49 SC). `untagged_risky = count(changed events with tag ∈ {null, "Requires review"})`.
- **Regression vs intended-relaxation distinction (the core of §17.1):** `Potential regression` (newly allowed) and `Potential false positive` (newly blocked) are the two "this is a bug in my policy" tags. `Intended relaxation` / `Intended enforcement` are "this is what I meant." The report counts `tagged_intentional` vs `untagged_risky` and `fp_candidates`/`fn_candidates` separately (§17E.4).
- Tags are recorded as `SimulationTag` with rationale (non-empty required for intended-relaxation per DT-49 SC) and link to the `PolicyException` CRD when an exception is filed.

---

## 6. Audit-driven test-case creation (§17.5 workflow, normative)

1. User selects an audit event/violation in E2 (Audit Correlation View).
2. User marks desired outcome: `allow | deny | warn | mutate | suspend_pending_approval`.
3. E1 extracts the policy-input fixture from the event (`replay_completeness` carried through).
4. User authors/modifies policy.
5. E1 runs the engine-appropriate evaluation (Conftest / OPA / `gator` Gatekeeper dry-run / Kyverno `test`).
6. E1 reports match/mismatch vs the desired outcome.
7. Fixture saved as a `RegressionFixture` (regression test).
8. Fixture **linked** to governance control ID and Rego/Kyverno policy version.

Saved fixtures are re-run automatically on the next bundle build (CI hook, owned by A2/B1) — HL-17 SC: "re-run automatically on the next bundle build."

---

## 7. Hard dependency on C2 audit schema replay fields

E1 is **blocked on Domain C / C2** for all replay-based modes (M2, M5–M9). The contract E1 requires from C2 (§13.3):

**Required for authoritative replay:** `event_id`, `timestamp`, `event_type`, `decision`, `policy_engine`, `policy_version`, `rego_package/policy_ref`, `control_id`, `resource_type`, `resource_id`, `cluster/namespace/tenant`, `subject`, `jwt_claims`, `operation`, `before_state`, `after_state`, `request_object`, `external_data_refs` (with version/digest), `correlation_id`, `outcome_reason`, and **`replay_completeness ∈ {complete, partial, insufficient}`**.

**`replay_completeness` is the authoritativeness gate (§17.3, §13.1):**

| `replay_completeness` | E1 behavior |
|---|---|
| `complete` | Result is **authoritative**; counts toward the headline matrix. |
| `partial` | Result is **advisory**; counted in a separate non-authoritative bucket; flagged in report; missing_fields enumerated. |
| `insufficient` | Event is **not evaluated** for a decision; counted in `incomplete`; surfaced as "audit coverage gap" feeding §17E "missing audit fields" report. |

**Normative E1 requirement (mirrors §17.3 last sentence):** *If a runtime policy depends on a field not present in the reconstructed input, the simulation result MUST be marked incomplete rather than authoritative.* E1 determines this by introspecting the bundle's referenced input paths (via Rego metadata / `opa inspect`) and intersecting with the fixture's present fields; any referenced-but-absent field downgrades the event.

**Soft-contract note (parallel-build reality):** C2's component dir was empty at E1 authoring time (other domain leads in flight). E1 pins to the **spec §13.3 field list** as the contract of record. If C2's final schema renames/omits any field above, that is a cross-domain reconciliation item (see DOMAIN-ADVERSARIAL / cross-cut). **Decision D-E1-03:** E1 codes against a thin `ReplayEventV1` adapter interface so a C2 schema rename is absorbed in one mapping layer, not throughout the engine.

---

## 8. CRD: `PolicySimulationRun` (§17C.6)

```yaml
apiVersion: governance.example.io/v1alpha1
kind: PolicySimulationRun
metadata: { name: diff-sc-img-001-v12-v13, namespace: payments-prod }
spec:
  mode: differential                 # M1..M9 enum
  previousBundle: bundle:v12
  newBundle: bundle:v13
  evidenceSet:
    sourceKind: audit_replay          # audit_replay | manifest | snapshot | live_shadow
    selector: { controlId: SC-IMG-001, window: "30d", eventType: kubernetes.admission.request }
    scope: { namespaces: [payments-prod] }
  requireComplete: true               # if true, run fails authoritative gate unless all events complete
status:
  phase: Succeeded                    # Pending | Materializing | Evaluating | Succeeded | Failed | PartiallyAuthoritative
  evidenceSetRef: sc-img-001-diff-v12-v13
  reportRef: report/...
  diff: { eventsEvaluated: 24108, newlyBlocked: 47, newlyAllowed: 12,
          unchangedAllowed: 19902, unchangedDenied: 4147, incomplete: 0,
          untaggedRisky: 0, fpCandidates: 3, fnCandidates: 0 }
```

A controller (E1-owned) reconciles: materialize evidence (honoring D2 scope) → evaluate (call B-engines in eval mode) → classify → produce §17E.4 report → set status. Run is **immutable** once `Succeeded` (re-runs create a new run; tagging mutates `SimulationTag` records, not the run's decision facts).

---

## 9. Interfaces / APIs

| API | Purpose | Authz (D2) |
|---|---|---|
| `POST /v1/simulations` | Create a run (any mode); returns `run_id` | requires `policy:simulate` in scope |
| `GET /v1/simulations/{id}` | Status + DiffResult | scoped read |
| `GET /v1/simulations/{id}/events?bucket=newly_allowed` | Paginated EvaluatedEvents | scoped read |
| `POST /v1/simulations/{id}/tags` | Tag a changed event (`event_id`, `tag`, `rationale`, `exception_ref?`) | requires `simulation:tag` |
| `GET /v1/simulations/{id}/report` | §17E.4 Simulation Report (JSON; C5 renders) | scoped read |
| `POST /v1/fixtures` | Create RegressionFixture from event (§17.5) | `policy:author` in scope |
| `GET /v1/evidence-sets/{ref}` | EvidenceSet metadata + digest | scoped read |

All evidence queries pass the subject's scope token to C2's **storage layer**, which enforces namespace boundaries (D2 §17A.5) — E1 never relies on filtering results after the fact for security (§17A.1).

---

## 10. §17E.4 Simulation Report contract (E1 produces)

Fields (all REQUIRED): policy version before, policy version after, audit dataset used (`evidence_set_ref` + digest + window), events evaluated, newly blocked count, newly allowed count, unchanged allowed count, unchanged denied count, tagged intentional changes, untagged risky changes, false-positive candidates, false-negative candidates. Plus E1 additions: `incomplete_count`, `replay_completeness_histogram`, `authoritative: bool`, per-bucket drill-down links, exception linkages, fixture linkages. Report is **signable** (§23) so it can be attached to a promotion decision (DT-49 SC, HL-17 SC).

---

## 11. State machine (SimulationRun)

```
Pending -> Materializing -> Evaluating -> { Succeeded | PartiallyAuthoritative | Failed }
Succeeded/PartiallyAuthoritative: immutable decision facts; tags mutable until promotion freeze
Failed: retryable (idempotent on evidence_set digest)
```

`PartiallyAuthoritative` is entered when `incomplete > 0` or any `partial` events exist and `requireComplete=false`. If `requireComplete=true` and not all complete → `Failed` with reason `audit-coverage-insufficient`.

---

## 12. Failure modes

| Failure | Handling |
|---|---|
| Audit events incomplete for a referenced field | Downgrade event; report advisory; never silently treat as authoritative (§17.3) |
| External data snapshot missing | Event → `partial`; advisory (D-E1-02) |
| C2 schema field renamed/missing | Adapter layer maps; if unmappable, fail run with explicit `schema-mismatch` (D-E1-03) |
| Engine eval crashes on a fixture | Mark event `eval_error`; exclude from buckets; surface count; do not abort whole run |
| Evidence set too large (memory) | Stream + chunk; persist intermediate EvaluatedEvents; resumable run |
| Live shadow mode non-reproducible | Archive observed decisions as immutable EvidenceSet so later differential is reproducible |
| Scope escalation attempt (read out-of-namespace events) | Storage layer denies (D2); run records the scope it ran under |
| Two bundles use different external-data versions | Pin BOTH evaluations to the *event's* snapshot, not each bundle's current data |

---

## 13. Security / authz notes

- Every evidence query is scope-enforced at the storage layer (D2 §17A.5). GUI-only filtering is explicitly insufficient (§17A.1).
- Simulation runs are themselves auditable events (who ran what diff over which evidence).
- Reports are signable; tags carry actor + timestamp; tag history is immutable (retag appends).
- Namespace Policy Author can only simulate over and save fixtures into their namespaces (M6, HL-08).
- A "risky simulation" (large newly-allowed count) is visible to the Security Reviewer role (§17A.2).

---

## 14. Open questions — decided defaults

| # | Question | Decided default | Rationale |
|---|---|---|---|
| OQ-1 | Reuse OPA decision logs vs re-execute bundles? | **Re-execute bundles** against reconstructed input; decision logs are only a cross-check oracle | Decision logs record the *old* outcome, not the *new* bundle's; differential needs both bundles run on the same input. (See ALT-decisionlog-reuse.) |
| OQ-2 | Always-on shadow service vs on-demand batch replay? | **On-demand batch `PolicySimulationRun`** is the default/MVP; shadow (M3) is an opt-in add-on | Batch is reproducible + cheaper; shadow is D-LIVE and operationally heavier. (See ALT-replay-batch-vs-shadow.) |
| OQ-3 | Allow-class normalization of `warn`/`mutate` | warn/mutate = allow-class, with secondary within-class diff | Matches §17.4 (only allow/deny in the matrix) but doesn't hide effect changes (D-E1-01) |
| OQ-4 | Replay against current vs historical external data | **Historical snapshot** (D-E1-02) | Reproducibility; isolates policy change from data change |
| OQ-5 | Gate promotion on `untagged_risky==0`? | **Yes** (enforced by A2/D3) | DT-49 / HL-17 success criteria require it |
| OQ-6 | Tie E1 to C2 schema directly or via adapter? | **Adapter `ReplayEventV1`** (D-E1-03) | C2 built in parallel; absorb schema drift in one layer |

---

## 15. Dependencies

- **C2 (§13)** — audit schema + replay_completeness — **HARD BLOCKER** for M2, M5–M9.
- **B1–B5** — engines invoked in eval/dry-run mode (OPA `eval`, `gator`, Kyverno `test`, Conftest).
- **D2 (§17A.5)** — storage-layer scope enforcement on every evidence query.
- **D3 / B4 (§17B, PolicyException)** — exception filing from tags; exception input to re-runs.
- **A2 (§7)** — promotion gate consumes `untagged_risky==0` + signed report.
- **C5 / E2 (§17E, §16)** — consume/render the §17E.4 report and tagging UI.
- **B4 (§17C.6)** — `PolicySimulationRun` CRD definition is shared with the CRD catalog.

---

## 16. Traceability

Spec: §17.1–§17.6, §13, §17C.6, §17E.4, §17A.5, §17.3.
Scenarios: DT-45, DT-46, DT-47, DT-48, DT-49, DT-50, DT-51, DT-52, HL-08, HL-17.
Personas: Marcus (primary author/operator), Priya (report consumer/approver), Daniel (auditor of signed reports), Sam (developer running M1/M9 locally).
