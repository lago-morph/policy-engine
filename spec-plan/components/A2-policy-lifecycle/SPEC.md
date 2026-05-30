# A2 — Policy Lifecycle (Author → Simulate → Promote) — SPEC

**Component:** A2 · **Domain:** A — Governance Core · **Spec source:** §7 (with §5.3, §6, §8.2/§8.3, §9.2,
§10, §17/§17.2/§17.4, §17B, §17C.6, §26.1)
**Status:** DRAFT v1 · **Persona for this doc:** meticulous staff engineer (make it real and buildable)
**Scenarios exercised:** DT-05, DT-06, DT-07, DT-08, DT-09, HL-02, HL-07, HL-17 (+ §5.3 lifecycle diagram).

---

## 0. Normative language & decisions

Requirement IDs `A2-MUST-NNN` / `A2-SHOULD-NNN` / `A2-MAY-NNN`. Unattended decisions tagged **[DECISION Dn]**,
collected in §13.

---

## 1. Scope & purpose

A2 owns the **lifecycle of an executable policy** from a governance Control (A1) to a running enforcement,
including the **promotion state machine** (`draft → dry-run → warn → enforce`, plus `deprecated`/`rolled-back`),
the **simulation/promotion gates**, and the **policy-version / mode-change audit trail**.

Where A1 answers *"what do we want and how is it evidenced?"*, A2 answers *"how does an executable policy for
that control safely move from idea to blocking production traffic, and how do we walk it back?"*

A2 explicitly **owns**:
1. The **PolicyLifecycle** entity: the binding of a Control (A1 `control_id`) to one or more
   **PolicyImplementations** (a Rego package + its enforcement target) and their current **enforcement mode**.
2. The **enforcement-mode state machine** and the **promotion gates** (simulation, soak, approval).
3. The **Gemara-to-Rego generation decision** (§26.1: full vs template) and the artifact provenance
   (`generated_from`).
4. The **policy-version lineage** of an implementation (which signed bundle version is at which mode, on which
   targets) — the authoritative source for A1's mirrored `current_policy_version`.
5. The **mode-change audit events** (DT-05/DT-06) and the binding of promotions to §17B approvals.
6. The **rollback / demotion** path and its durable record.

A2 explicitly **does NOT own** (delegated):
- The Control entity and its requirements (→ A1).
- Rego *authoring UX*, the Rego Explorer, test-running UI (→ E2 console; A2 defines the lifecycle, not the editor).
- Bundle *signing/packaging/OCI mechanics* (→ B1); A2 *triggers* and *records* versions.
- The actual admission *enforcement* at deny/warn/dry-run (→ B2 Gatekeeper, B3 Conftest, B4 engine select);
  A2 sets the *desired mode* and B-layer realizes it.
- The *simulation engine* mechanics (replay, differential) (→ E1 §17); A2 *consumes* simulation results as gates.
- Approval *runtime* (webhooks, CRDs) (→ D3 §17B); A2 *requires* and *records* approvals.

> **[DECISION D1]** A2 is an **orchestrator/state-machine over other components**, not a re-implementation. It
> calls E1 for simulation, B1 for packaging, D3 for approvals, B2/B3/B4 for mode realization, and A1 for the
> control contract. A2's value is the *gated, audited, reversible workflow* connecting them. Rationale: §7 is a
> *lifecycle*, and the engines/simulation/approval pieces are separately specified; re-implementing them in A2
> would duplicate and drift.

---

## 2. Domain model

### 2.1 `PolicyLifecycle`

The aggregate root: one per (`control_id`) governing how that control becomes enforced.

| Field | Type | Req | Notes |
|---|---|---|---|
| `lifecycle_id` | ULID | MUST | |
| `control_id` | FK→A1 Control | MUST | the governance anchor (A1 D3) |
| `enforcement_class` | mirror from A1 | MUST | Runtime/Build-Time/Detective/Manual/Advisory (§7.2) — gates which targets/modes are legal (§4.4) |
| `implementations` | list\<PolicyImplementation\> | MUST | ≥1 |
| `current_mode` | EnforcementMode | MUST | aggregate mode (most-conservative across targets, or per-target if heterogeneous) |
| `status` | LifecycleStatus | MUST | see §4 |
| `current_policy_version` | string | SHOULD | head signed bundle version (authoritative; A1 mirrors this) |
| `last_promotion_event_id` | FK | SHOULD | |
| audit fields | | MUST | |

### 2.2 `PolicyImplementation`

A concrete executable realization of the control at one enforcement target.

| Field | Type | Req | Notes |
|---|---|---|---|
| `impl_id` | ULID | MUST | |
| `rego_package` | string | MUST when engine∈{opa,gatekeeper,conftest,kyverno,analytics} | e.g. `governance.kubernetes.imagesigning` (§8.3) |
| `engine` | EngineTarget | MUST | gatekeeper / conftest / opa / kyverno / analytics / pdp-library / manual |
| `enforcement_point` | enum | MUST | k8s-admission / ci-pipeline / pdp / analytics-stream / manual |
| `generated_from` | `full \| template \| manual` | MUST | §26.1, DT-09 provenance |
| `mode` | EnforcementMode | MUST | this target's mode (DT-07 conftest has no deny/warn — see §4.4) |
| `bundle_ref` | OCI ref + digest | SHOULD | signed bundle this impl ships in (B1) |
| `policy_version` | string | MUST | per-impl version (e.g. `bundle:v24`) |
| `constraint_name` | string \| null | MAY | e.g. Gatekeeper `imagesigning-required` (DT-05) |
| `tests` | list\<TestCaseRef\> | SHOULD | incl. §17.5 audit-derived fixtures |
| `target_selector` | Applicability | SHOULD | clusters/namespaces this impl is rolled out to (HL-07: hipaa-dev then hipaa-prod) |

### 2.3 `EnforcementMode` enum

`draft | dry-run | warn | enforce | suspend_pending_approval | deprecated`
- `draft` — authored, not deployed.
- `dry-run` — evaluated, never blocks (§9.2 Dry Run); produces would-block telemetry.
- `warn` — allows with warning (§9.2 Warn).
- `enforce` — blocks (§9.2 Deny) / hard CI fail / hard PDP deny — the terminal **active** mode.
- `suspend_pending_approval` — §17B.2 outcome (an *implementation* may terminally resolve to this rather than deny).
- `deprecated` — mode after a control is deprecated (DT-04), winding down.

> **[DECISION D2]** `enforce` is the **abstract terminal mode**; B-layer maps it per engine: Gatekeeper `deny`,
> Conftest hard-fail, PDP-library hard-deny, analytics = "violation emitted." A2 speaks in abstract modes; the
> engine adapter (B4) maps to concrete `enforcementAction`. Rationale: §7.2 enforcement classes and §9.2
> Gatekeeper modes are different vocabularies; A2 needs one vocabulary that spans engines.

### 2.4 `PromotionEvent` (the audited transition)

Every mode change is a first-class, replay-capable event (DT-05/DT-06 success criteria).

| Field | Type | Req | Notes |
|---|---|---|---|
| `event_id` | ULID | MUST | |
| `lifecycle_id` / `impl_id` | FK | MUST | |
| `control_id` | string | MUST | |
| `from_mode` / `to_mode` | EnforcementMode | MUST | e.g. dryrun→warn (DT-05) |
| `from_version` / `to_version` | policy_version | MUST | bundle version trail (DT-06: v23→v24→v24-warn) |
| `actor` | JWT subject + role | MUST | §9.3 |
| `reason` | markdown | MUST | promotion rationale or rollback cause |
| `gate_results` | GateResultSet | MUST | simulation/soak/approval outcomes that justified this (§3) |
| `approval_ref` | §17B approval correlation_id \| null | MUST when crossing into `enforce` | separation-of-duties (DT-05 step 5) |
| `simulation_report_ref` | E1 report id \| null | SHOULD | §17E.4 report attached (DT-06 step 6) |
| `gitops_commit` | sha \| null | SHOULD | DT-05: change committed via GitOps |
| `correlation_id` | string | MUST | §13.3 |
| `ts` | timestamp | MUST | ordered, immutable |

### 2.5 `GenerationDecision` (§26.1, DT-09)

Records the Gemara→Rego generation outcome for an implementation.

| Field | Type | Notes |
|---|---|---|
| `mode` | `full \| template` | §26.1: full only when complete, deterministic, safe |
| `reason` | markdown | why template (e.g. "non-deterministic language; env-specific claim mapping") (DT-09 step 2) |
| `template_todos` | list\<TodoMarker\> | unfilled decision points; **block promotion while non-empty** (DT-09 success criterion) |
| `prefilled_metadata` | §8.3 block | `__control_id__`, `__severity__`, `__required_claims__` pre-populated |
| `test_scaffold` | list\<TestCaseRef\> | one fixture per intended outcome (allow/deny/suspend) (DT-09 step 4) |

---

## 3. Promotion gates (the heart of A2)

A promotion `X→Y` is allowed only if its gates pass. Gates are declarative and recorded in the PromotionEvent.

| Gate | When required | Check (MUST) | Source |
|---|---|---|---|
| **G-AUTHOR** | leaving `draft` | impl compiles; §8.3 metadata present & matches `control_id`; tests green; **no `template_todos` remain** | §8.3, DT-09, DT-11 |
| **G-SIM-BASELINE** | `draft→dry-run` (Runtime) | historical replay (§17.2) executed over the EvidenceRequirement-mandated window (≥ N days, default 30) | §17.2/§17.3, DT-05 step 1 |
| **G-DIFF** | `dry-run→warn` (Runtime) | differential simulation (§17.4) run; **zero `Requires review` rows outstanding** (all would-blocks triaged Intended/FalsePositive) | §17.4, DT-05 step 2 |
| **G-SOAK** | `warn→enforce` (Runtime) | warn observed for a soak window; actual warn volume within tolerance of dry-run forecast; no unresolved complaints | §17E.2, DT-05 step 4 |
| **G-APPROVE** | any `*→enforce` (Runtime) AND any explicitly-gated control | §17B approval event present from a *different* admin (separation of duties); approval `correlation_id` recorded | §17B, DT-05 step 5 |
| **G-CLASS** | always | requested target/mode legal for the control's `enforcement_class` (§4.4) | §7.2 |

> **[DECISION D3]** Gates are **per-enforcement-class** (§4.4). Build-Time (Conftest) and Detective (analytics)
> controls do **not** traverse dry-run→warn→enforce — they have no admission modes (DT-07: conftest is "deny in
> CI" with no warn/dry-run mode; DT-08: detective lands "directly as audit-only"). The full dry-run→warn→enforce
> ladder is a **Runtime-class** concern. A2 selects the gate set from the class. Rationale: forcing a Conftest
> rule through "warn mode" is meaningless; HL-07 step 6 explicitly says detective lands directly as audit-only.

### 3.1 Gate evaluation contract

- **A2-MUST-010** A2 MUST NOT record a successful promotion unless every required gate for that transition
  returns `pass` with an attached artifact reference (report id, approval id, test run id).
- **A2-MUST-011** Gate artifacts MUST be **immutably referenced** in the PromotionEvent so DT-06 rollback can
  reconstruct the original decision rationale (DT-05 note: "differential tags persisted with each promotion").
- **A2-SHOULD-010** A2 SHOULD re-run G-DIFF automatically if the underlying audit dataset materially changes
  between gate pass and the actual promotion (avoid stale-simulation promotion).

---

## 4. State machines

### 4.1 PolicyImplementation enforcement-mode machine (Runtime class)

```
        G-AUTHOR        G-SIM-BASELINE       G-DIFF            G-SOAK + G-APPROVE
 ┌───────┐         ┌─────────┐          ┌──────┐          ┌─────────┐
 │ draft │ ──────► │ dry-run │ ───────► │ warn │ ───────► │ enforce │
 └───────┘         └─────────┘          └──────┘          └─────────┘
     ▲                  ▲                   │                  │  │
     │                  │   demote(rollback)│ ◄────────────────┘  │ (DT-06: enforce→warn)
     │                  └───────────────────┴─────────────────────┘
     │  (edit reopens draft; new version)                          │
     └─────────────────────────────────────── deprecate ──────────┘
                                                                   ▼
                                                           ┌────────────┐
                                                           │ deprecated │ (DT-04 wind-down; then removed)
                                                           └────────────┘
```

- **Promotions** move *forward* (draft→dry-run→warn→enforce), gated.
- **Demotions** (rollback) move *backward* any number of steps in one transition (enforce→warn DT-06;
  enforce→dry-run; warn→dry-run). Demotions are **always allowed without the forward gates** (safety: you can
  always make a policy *less* blocking immediately) but MUST still emit a PromotionEvent with cause and the
  triggering simulation/incident reference (DT-06).
- **Re-author** (edit the rule body / new bundle version) reopens `draft` and requires re-traversal.

> **[DECISION D4]** **Demotion is ungated and instantaneous; promotion is gated.** Making a policy *less*
> restrictive is a safety action (DT-06: stop the deny spike in <2 min) and must never be blocked by an
> approval queue. Making it *more* restrictive is a risk action and is gated. Rationale: DT-06 requires deny
> rate → 0 within two minutes; a gate on rollback would cause outages.

### 4.2 PolicyImplementation modes (Build-Time / Detective)

- **Build-Time (Conftest, DT-07):** `draft → enforce(=CI hard-fail)` with G-AUTHOR only; no dry-run/warn/soak/approve.
  (A `warn` analogue MAY be a non-blocking CI annotation, but it is not the admission warn.)
- **Detective (analytics, DT-08):** `draft → enforce(=violation emitted)` with G-AUTHOR only; lands "directly
  as audit-only" (HL-07). No admission ladder.
- **Manual / Advisory:** `draft → enforce(=record only)`; G-AUTHOR (compiles/metadata) only.

### 4.3 PolicyLifecycle status machine

```
 draft ──► active ──► deprecated ──► retired
   ▲         │  ▲          │
   └─ edit ──┘  └──────────┘ (re-activate within grace; mirrors A1 control status)
```

`status` tracks the *governance* binding (mirrors A1's control status per A1 D7), while `mode` tracks the
*enforcement* posture. **A2-MUST-020** A2 MUST keep these reconciled: a `retired` lifecycle MUST have all
implementations in `deprecated`/removed mode (this is the A1-DEF-03 reconciliation owner — see §8).

### 4.4 Class × mode legality matrix (G-CLASS)

| enforcement_class | legal modes | legal targets |
|---|---|---|
| Runtime | draft, dry-run, warn, enforce, suspend_pending_approval, deprecated | gatekeeper, kyverno, opa, pdp-library |
| Build-Time | draft, enforce, deprecated (+ non-blocking annotate) | conftest |
| Detective | draft, enforce(=emit), deprecated | analytics, opa-replay |
| Manual | draft, enforce(=record), deprecated | manual |
| Advisory | draft, enforce(=inform), deprecated | any (non-blocking) |

- **A2-MUST-021** A2 MUST reject a promotion that violates §4.4 (e.g. asking a Conftest impl to enter `warn`
  admission mode), returning a machine-readable reason.

---

## 5. Interfaces / API

```
POST /v1/lifecycles                         # bind a control_id to its first implementation(s)
GET  /v1/lifecycles/{control_id}            # aggregate: impls, modes, current_policy_version, history
POST /v1/lifecycles/{control_id}/implementations
POST /v1/implementations/{impl_id}:generate # run §26.1 Gemara→Rego generation; returns GenerationDecision
POST /v1/implementations/{impl_id}:promote  # body:{to_mode, gate_overrides?, approval_ref?}  → runs gate set
POST /v1/implementations/{impl_id}:demote   # body:{to_mode, cause, sim_ref?}  → ungated rollback (DT-06)
GET  /v1/implementations/{impl_id}/history  # ordered PromotionEvent trail (DT-06 success criterion)
GET  /v1/lifecycles/{control_id}/enforcement-points   # served back to A1 for safe-deprecation (A1 DT-04)
POST /v1/lifecycles/{control_id}:deprecate  # A1 deprecation event drives wind-down (DT-04)
```

- **A2-MUST-030** `:promote` MUST execute the full gate set synchronously-or-saga and refuse on any gate fail,
  returning the failing gate + artifact. It MUST be idempotent on `(impl_id, to_version, to_mode)`.
- **A2-MUST-031** `GET .../enforcement-points` MUST enumerate every live (non-`deprecated`) implementation for a
  control so A1's retire guard (A1-DEF-07) can verify zero active enforcement.
- **A2-MUST-032** A2 MUST emit each PromotionEvent to the audit/change feed (C2) with §13.3 + §9.3 fields.

### 5.1 Events A2 consumes
- A1 `ControlActivated` → A2 may create a lifecycle. A1 `ControlDeprecated` → A2 starts wind-down (DT-04).
- A1 `EnforcementRequirementChanged(mode_intent)` → A2 reconciles target mode (see §8 contradiction handling).
- E1 simulation-complete → satisfies G-SIM-BASELINE/G-DIFF gates.
- D3 approval-granted → satisfies G-APPROVE.

---

## 6. Failure modes & handling

| # | Failure | Handling (MUST/SHOULD) |
|---|---|---|
| F1 | Promotion requested with stale simulation (dataset changed) | SHOULD re-run G-DIFF; MUST refuse if review rows reappear (A2-SHOULD-010) |
| F2 | `template_todos` remain at promotion | MUST block promotion (G-AUTHOR; DT-09 success criterion) |
| F3 | Bundle signing (B1) fails mid-promotion | MUST abort transition, leave prior mode/version intact, emit failed-PromotionEvent (no partial state) |
| F4 | Approval (D3) never returns | MUST time-box; promotion stays at prior mode; no auto-enforce (DT-62 approval expiry) |
| F5 | Demotion requested during an in-flight promotion | MUST preempt: demotion wins (safety), in-flight promotion aborts (D4) |
| F6 | Mode realized by B-layer diverges from A2's intended mode | reconciler (§8) raises `mode_drift`; A2 is source of intent, B-layer self-heals or alarms |
| F7 | A1 deprecates a control mid-promotion | MUST halt forward promotion, begin wind-down |
| F8 | Class×mode violation requested | MUST reject (A2-MUST-021) |
| F9 | GitOps commit and platform record disagree on current version | MUST treat the signed bundle digest as authoritative; alarm on divergence |
| F10 | Per-target heterogeneity (one cluster enforce, another warn) | MUST represent per-impl/per-selector mode; aggregate `current_mode` = most-conservative for safety display |

---

## 7. Security & authz

- **A2-MUST-040** Promotions to `enforce` MUST be separation-of-duties gated (G-APPROVE): the approver subject
  (verified via JWT group, not GUI role — DT-03 pattern) MUST differ from the promoter (DT-05 step 5).
- **A2-MUST-041** Every mode-change MUST be tamper-evidently recorded (ordered, immutable PromotionEvent;
  §23 auditability) — DT-06 requires "an ordered trail with actor, timestamp, reason for each transition."
- **A2-SHOULD-040** Demotion/rollback authority SHOULD be broad (any platform admin / on-call) — see D4; but
  still attributed and audited.

---

## 8. Reconciliation (owns the A1↔A2 seam — adversarial DEF-03 fix)

> **[DECISION D5]** **A2 owns the A1-status × A2-mode reconciler** that the A1 adversarial review (A1-DEF-03)
> says nobody owns. A2 runs a periodic + event-triggered reconciliation defining the legal cross-product and
> raising findings:

| A1 control status | Legal A2 modes | Illegal / flagged |
|---|---|---|
| draft / in_review | draft only | any deployed mode ⇒ `premature_enforcement` |
| active | any mode | `active` + still `dry-run` after `governed_not_enforced_sla` (default 30d) ⇒ **`governed_not_enforced`** finding |
| deprecated | dry-run/warn (winding down), deprecated | `enforce` past grace ⇒ `stale_enforcement` |
| retired | deprecated/removed only | any live mode ⇒ **`zombie_enforcement`** (critical) |

- **A2-MUST-050** A2 MUST expose these findings (to C5 reporting and the console) and MUST block `retire`
  acknowledgement back to A1 until zero live implementations remain (closes A1-DEF-07 from the A2 side too:
  A2's enforcement-points inventory is the authoritative answer to A1's retire guard).
- **A2-MUST-051** On conflicting `mode_intent` (A1) vs realized `mode` (A2): the **most-restrictive of the two
  wins for safety**, and a `mode_intent_conflict` finding is raised for human resolution. (Resolves the
  A1-DEF-03 "who wins" question: safety wins, humans reconcile.)

---

## 9. Dependencies

| Depends on | What A2 needs | Direction |
|---|---|---|
| **A1** | `control_id`, enforcement_class, EnforcementRequirement(`mode_intent`,`deterministic`), EvidenceRequirement(replay window) | A1 → A2 |
| **E1 Simulation (§17)** | historical replay, differential simulation results = G-SIM-BASELINE/G-DIFF gates | E1 → A2 |
| **D3 / §17B approvals** | approval events = G-APPROVE; separation of duties | D3 → A2 |
| **B1 OPA bundles** | sign/package/version bundle on each mode change (policy_version bump) | A2 → B1 |
| **B2/B3/B4 engines** | realize the abstract mode (`enforce`→`deny` etc.); report realized mode + enforcement points | A2 ↔ B-layer |
| **C2 Audit** | emit PromotionEvents with §13.3/§9.3 fields; consume audit for soak/forecast | A2 ↔ C2 |
| **C5 Reporting (§17E)** | Real-Time Enforcement Report (soak), Simulation Report (gates), reconciliation findings | A2 ↔ C5 |
| **E2 Console** | Simulation View, Rego Explorer, promotion-approval workflow surface | E2 → A2 |

---

## 10. Acceptance criteria (traceable)

- **AC-1 (DT-05)** Promoting `SC-IMG-001` produces three distinct signed `policy_version`s (dryrun/warn/deny),
  each with a PromotionEvent carrying actor, from/to mode, correlation_id; warn→enforce blocked without a
  second-admin G-APPROVE; zero `Requires review` rows at G-DIFF before warn.
- **AC-2 (DT-06)** Demotion enforce→warn reflected in subsequent audit within 2 min; ordered history
  `v23(deny)→v24(deny)→v24(warn)` with actor/ts/reason; attached differential report (28 newly_blocked,
  17 FP / 11 intended); no further deny after demotion.
- **AC-3 (DT-07)** A Build-Time impl never enters dry-run/warn; G-CLASS rejects admission modes; promotion is
  `draft→enforce` with G-AUTHOR only; no Gatekeeper constraint generated.
- **AC-4 (DT-08)** A Detective impl lands `enforce`=audit-only with G-AUTHOR only; no admission/CI gate.
- **AC-5 (DT-09)** Generation returns `template` with reason + `template_todos`; promotion blocked while TODOs
  remain; `generated_from=template` recorded; test scaffold has ≥1 fixture per intended outcome.
- **AC-6 (HL-02/HL-07)** A Runtime control traverses dry-run(hipaa-dev)→warn→enforce(hipaa-prod) with the full
  audit trail and per-selector rollout.
- **AC-7 (reconciliation)** A control `active` + `dry-run` past SLA raises `governed_not_enforced`; a `retired`
  control with a live impl raises `zombie_enforcement` (critical).

---

## 11. Open questions — decided defaults

| # | Question | Decided default | Rationale |
|---|---|---|---|
| OQ-1 | Soak window length? | **Configurable; default 7d warn (DT-04 uses 7d); default 30d replay window** | spec says traceable, not fixed (DT-04 note) |
| OQ-2 | Is G-APPROVE required for *all* enforce, or only flagged controls? | **All Runtime `*→enforce` by default; per-control opt-out for low-severity via ExceptionRequirement-style flag** | DT-05 mandates SoD; allow tuning |
| OQ-3 | Can one control have impls at different modes on different clusters? | **Yes (per-impl/per-selector mode); aggregate shown most-conservative** | HL-09 multi-cluster drift; DT-32 |
| OQ-4 | Who is the authority for `current_policy_version`? | **A2 (the lifecycle), mirrored read-only into A1** | resolves A1-DEF cross-component ambiguity |
| OQ-5 | Auto-promote on green gates, or always human-initiated? | **Human-initiated `:promote`; gates *enable*, never *trigger*** | promotion is a risk action; keep a human in the loop |
| OQ-6 | Demotion authority breadth? | **Broad (on-call/admin), fully audited (D4)** | DT-06 needs sub-2-min rollback |

---

## 12. Non-functional

- **A2-MUST-060** Demotion (`:demote`) MUST realize at the B-layer within the platform's admission-propagation
  budget such that observed deny rate → 0 within 2 minutes (DT-06).
- **A2-SHOULD-060** Promotion gate evaluation SHOULD be resumable (saga) so a long replay doesn't lose progress.

---

## 13. Decisions log

D1 A2 orchestrates, doesn't re-implement; D2 abstract `enforce` mode mapped per-engine by B4; D3 gate set
selected by enforcement_class; D4 demotion ungated/instant, promotion gated; D5 A2 owns the A1↔A2 lifecycle
reconciler (most-restrictive-wins on conflict).
