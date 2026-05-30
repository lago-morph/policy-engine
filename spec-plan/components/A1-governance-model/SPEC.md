# A1 — Governance Model & Gemara Hierarchy — SPEC

**Component:** A1 · **Domain:** A — Governance Core · **Spec source:** §6 (with §5.3, §7.1, §8.2/§8.3, §13.3, §17B/§17C.6, §26.1)
**Status:** DRAFT v1 · **Persona for this doc:** meticulous staff engineer (make it real and buildable)
**Scenarios exercised:** DT-01, DT-02, DT-03, DT-04, HL-01, HL-07 (and indirectly every scenario that resolves a `control_id`).

---

## 0. Reading guide & normative language

Requirement IDs are `A1-MUST-NNN` / `A1-SHOULD-NNN` / `A1-MAY-NNN`. "MUST" = required for the
component to be considered correct; "SHOULD" = strongly recommended, deviations must be recorded;
"MAY" = optional. Decisions taken unattended in this spec are tagged **[DECISION Dn]** with rationale
and collected in §13.

---

## 1. Scope & purpose

A1 is the **system of record for governance intent**. It owns the Gemara-aligned governance hierarchy —
the durable, engine-independent description of *what the organization wants to be true* and *how each
desire is enforced, evaluated, evidenced, and excepted*. Every downstream artifact in the platform
(Rego package, Gatekeeper constraint, Conftest rule, Privateer evaluation, audit event, simulation run,
exception CRD, report row) **traces back to a Control owned by A1 via a stable `control_id`**.

A1 explicitly **owns**:
1. The entity model for the 7-layer governance hierarchy (Objective → Domain → Control → Enforcement /
   Evaluation / Evidence / Exception Requirements) plus framework cross-references and lineage.
2. The persistence contract (storage-engine-neutral; graph-shaped lineage **records**, not a graph DB —
   per spec §26.1 "Policy lineage").
3. The Governance API read/write surface for these entities (the §21 API's governance slice).
4. The identifier scheme (`control_id`, `objective_id`, `domain_id`, requirement IDs) and the validation
   rules that keep the hierarchy referentially intact.
5. The control **lifecycle status** model (`draft → in_review → active → deprecated → retired`) — distinct
   from, but feeding, A2's *enforcement* lifecycle (dry-run → warn → enforce).

A1 explicitly **does not own** (delegated, with named owner):
- Authoring/promotion *workflow* of executable policy (→ A2 Policy Lifecycle).
- Rego packaging/signing/OCI (→ B1).
- Admission enforcement mechanics (→ B2 Gatekeeper, B3 Conftest, B4 engine selection).
- The audit event *transport/store* (→ C2); A1 only **declares** which audit fields a Control's Evidence
  Requirement demands.
- The exception *runtime* (CRD admission, approval webhooks) (→ D3 / B4 §17C.6); A1 only **declares** the
  Exception Requirement that those mechanisms enforce.
- Identity/claims (→ D1); A1 only **references** required JWT claim names on Controls.

---

## 2. Domain model (entities)

All entities are versioned, immutable-by-revision (append-only revisions; see §6). IDs are ULIDs internally
but every externally meaningful entity also carries a human-authored stable business key (`*_id`).

### 2.1 Entity catalogue & relationships

```
Objective (1) ──owns──> (N) Control            [an Objective decomposes into Controls]
Domain   (1) ──contains──> (N) Objective         [a Domain groups Objectives]
Domain   (1) ──contains──> (N) Control            [a Control names exactly one owning Domain]
Control  (1) ──has──> (1) EnforcementRequirement  [1:1, may be "none" for advisory]
Control  (1) ──has──> (0..1) EvaluationRequirement
Control  (1) ──has──> (0..1) EvidenceRequirement
Control  (1) ──has──> (0..1) ExceptionRequirement
Control  (N) <──satisfied_by──> (N) FrameworkRequirement   [bidirectional, with coverage qualifier]
Control  (N) ──supersededBy──> (0..1) Control               [deprecation chain]
ControlRevision (N) ──revision_of──> (1) Control            [append-only history]
LineageEdge (N) ──connects──> (2) any nodes                 [graph-shaped lineage records]
```

> **[DECISION D1]** Enforcement/Evaluation/Evidence/Exception requirements are modeled as **owned child
> entities of a Control (1:1 / 1:0..1)**, NOT as free-floating reusable objects. Rationale: spec §6.1 lists
> them as *layers of a single control's decomposition*; the example shows one of each per control. Reuse is
> achieved by *templates* (§2.8), not by sharing live instances, which keeps deprecation/versioning local to
> a control and avoids action-at-a-distance when one control's requirement changes.

### 2.2 `Domain` (Policy Domain — §6.1 layer 2)

| Field | Type | Req | Notes |
|---|---|---|---|
| `domain_id` | slug (`^[a-z][a-z0-9-]{1,62}$`) | MUST | business key, e.g. `supply-chain`, `data-governance`, `identity`, `hipaa` |
| `title` | string | MUST | |
| `description` | markdown | SHOULD | |
| `owner` | RoleRef (§17A.2 role name) + subject | MUST | the accountable role, e.g. `Platform Governance Admin` |
| `parent_domain_id` | slug \| null | MAY | domains MAY nest one level (D2 below) for org hierarchy |
| `status` | `active \| archived` | MUST | |
| `created_at`/`created_by` | ts / JWT subject | MUST | |

> **[DECISION D2]** Domains MAY nest **one level deep** (parent/child) but the hierarchy is otherwise flat;
> we do not model arbitrary domain trees. Rationale: spec shows a single Domain layer; one level of nesting
> covers real org structure (e.g. `security` → `supply-chain`) without inviting unbounded taxonomy bikeshedding.

### 2.3 `Objective` (Governance Objective — §6.1 layer 1)

| Field | Type | Req | Notes |
|---|---|---|---|
| `objective_id` | slug (`^OBJ-[A-Z0-9-]{3,40}$`) | MUST | e.g. `OBJ-EXFIL-001` (DT-01) |
| `title` | string | MUST | e.g. "Prevent unauthorized data exfiltration" |
| `rationale` | markdown | MUST | the risk/why |
| `owning_domain_id` | FK→Domain | MUST | |
| `framework_refs` | list\<FrameworkRequirement ref\> | MAY | may be empty at create, populated later (DT-01 step 1, DT-02) |
| `status` | LifecycleStatus (§4) | MUST | |
| `risk_rating` | `low\|medium\|high\|critical` | SHOULD | |
| audit fields | | MUST | created/updated/by |

### 2.4 `Control` (§6.1 layer 3, §7.1 authoring fields)

The central entity. Carries both §6.1 structural fields and §7.1 authoring metadata.

| Field | Type | Req | Source | Notes |
|---|---|---|---|---|
| `control_id` | string (`^[A-Z][A-Z0-9]+(-[A-Z0-9]+)*-\d{3,}$`) | MUST | §7.1, §8.3 | **globally unique**; the universal join key. e.g. `SC-IMG-001`, `DG-EXFIL-001`, `HIPAA-AC-001` |
| `title` | string | MUST | §6.1 | "All container images must be signed" |
| `statement` | markdown | MUST | §6.1 | normative control text |
| `owning_objective_id` | FK→Objective | MUST | §6.1 | |
| `owning_domain_id` | FK→Domain | MUST | §6.1 | denormalized for query (DT-01 step "search by domain") |
| `severity` | `low\|medium\|high\|critical` | MUST | §7.1 | |
| `applicability` | Applicability obj (§2.9) | MUST | §7.1 | namespaces/clusters/tenants/environments selector |
| `enforcement_class` | enum (§2.10) | MUST | §7.1, §7.2 | Runtime / Build-Time / Detective / Manual / Advisory |
| `enforcement_targets` | list\<EngineTarget\> | MUST | §7.1 | e.g. `[gatekeeper]`, `[conftest]`, `[analytics]`, `[opa]`, `[kyverno]`, `[manual]` |
| `related_rego_package` | string \| null | SHOULD | §7.1, §8.3 | e.g. `governance.kubernetes.imagesigning`; null until A2 authors it |
| `required_jwt_claims` | list\<string\> | SHOULD | §7.1, §8.3, §15.2/.3 | e.g. `["groups","tenant","environment"]` |
| `evidence_schema_ref` | FK→EvidenceRequirement | MUST | §7.1 | |
| `exception_workflow_ref` | FK→ExceptionRequirement \| null | SHOULD | §7.1 | |
| `generated_from` | `full \| template \| manual \| null` | SHOULD | §26.1, DT-09 | provenance of the Rego implementation |
| `status` | LifecycleStatus (§4) | MUST | §6/§7 | |
| `superseded_by` | FK→Control \| null | MAY | DT-04 | |
| `current_policy_version` | string \| null | SHOULD | §8.2 | last known signed bundle version implementing this control (mirror from A2/B1; see §10) |
| `tags` | map\<string,string\> | MAY | | e.g. `framework=SOC2:CC6.1` |
| audit fields | | MUST | | |

> **[DECISION D3]** `control_id` is the single global namespace and the canonical foreign key across the
> *entire platform* (Rego `__control_id__`, Gatekeeper constraint label, Conftest evidence, audit
> `control_id`, exception `spec.controlId`, report filters). A1 is the **allocator and uniqueness authority**.
> No other component may mint a `control_id`. Rationale: the persona-mapping calls the control ID "the shared
> anchor" (Priya↔Marcus). A single authority prevents collisions and dangling references.

### 2.5 `EnforcementRequirement` (§6.1 layer 4)

*What must happen at the point of control.*

| Field | Type | Req | Notes |
|---|---|---|---|
| `requirement_id` | slug | MUST | `ENF-<control_id>` |
| `mode_intent` | `deny \| warn \| dry-run \| audit \| suspend_pending_approval \| mutate \| generate \| none` | MUST | intended terminal mode (the *target* of A2's lifecycle, §17B.2, §9.2) |
| `enforcement_point` | enum | MUST | `k8s-admission`, `ci-pipeline`, `pdp-library`, `analytics-stream`, `manual` |
| `engine` | EngineTarget | MUST | resolved per §17C engine matrix (B4) |
| `narrative` | markdown | MUST | e.g. "Reject unsigned image admission" (§6.1 example) |
| `deterministic` | bool | MUST | drives §26.1 generation (full vs template) — see DT-09 |
| `target_selector` | Applicability override \| null | MAY | narrow enforcement to subset of control applicability |

### 2.6 `EvaluationRequirement` (§6.1 layer 5)

*The detective signal that proves the enforcement is (or isn't) working.*

| Field | Type | Req | Notes |
|---|---|---|---|
| `requirement_id` | slug | MUST | `EVAL-<control_id>` |
| `method` | `opa-replay \| gatekeeper-audit \| analytics-detection \| privateer-evaluation \| manual-review` | MUST | DT-01 step 5 |
| `signal_description` | markdown | MUST | e.g. "Detect workloads bypassing admission" (§6.1 example) |
| `source_audit_query` | structured query ref | SHOULD | what audit slice feeds the evaluation (links to C2 schema) |
| `cadence` | `continuous \| periodic:<cron> \| on-demand` | MUST | |
| `privateer_evaluation_id` | string \| null | MAY | link to §11 evaluation (C1) |

### 2.7 `EvidenceRequirement` (§6.1 layer 6)

*Which fields every decision/evaluation about this control MUST record so it is replay-capable.*

| Field | Type | Req | Notes |
|---|---|---|---|
| `requirement_id` | slug | MUST | `EVID-<control_id>` |
| `required_core_fields` | list\<string\> | MUST | MUST be a superset of §13.3 core fields (timestamp, cluster, namespace, JWT subject, JWT groups, control_id, decision outcome, policy_version, correlation_id) |
| `control_specific_fields` | list\<FieldSpec\> | MAY | e.g. `destination_cidr` for DG-EXFIL-001 (DT-01 step 6) |
| `external_data_refs` | list\<FieldSpec\> | MAY | classification tables etc. (DT-08) |
| `retention_min_days` | int | SHOULD | feeds C2 retention; default 400 (>13mo for annual audits) |
| `redaction_policy_ref` | string \| null | MAY | for §17A.5/§23 redacted exports (DT-57) |

> **[DECISION D4]** A1 **validates** that `required_core_fields ⊇ §13.3 mandatory set` at save time and
> **rejects** an EvidenceRequirement that drops a §13.3 core field. Rationale: §13.3 is the floor that makes
> replay possible; allowing a control author to omit `correlation_id` or `policy_version` would silently break
> §17.4 differential simulation and §19 retrospective detection downstream. A1 is the right enforcement point
> because it is where the requirement is declared. The canonical §13.3 list is owned by C2 and imported as a
> versioned constant (`AUDIT_CORE_FIELDS@<schema_version>`); see dependency in §10.

### 2.8 `ExceptionRequirement` (§6.1 layer 7)

*The governance rule the runtime exception mechanism (§17B / §17C.6 `PolicyException`) must enforce.*

| Field | Type | Req | Notes |
|---|---|---|---|
| `requirement_id` | slug | MUST | `EXC-<control_id>` |
| `approver_role` | RoleRef (§17A.2) | MUST | e.g. `Security Reviewer` (DT-03) |
| `max_duration_days` | int (>0) | MUST | e.g. 90 (DT-03) |
| `required_linked_artifacts` | list\<ArtifactType\> | MUST | e.g. `[jira_ticket]`; each with a validation pattern |
| `scope` | `namespace \| cluster \| tenant \| global` | MUST | |
| `reauth_on_expiry` | bool | SHOULD | drives HL-19 / DT-62 |
| `requires_separation_of_duties` | bool | MAY | approver ≠ requester |

A1 **publishes** this requirement on the Control via the Governance API so the `PolicyException` CRD
admission validator (D3/B4) reads it at `spec.controlId` resolution time (DT-03 step 3). A1 does **not**
run the validator.

### 2.9 `Applicability`

```jsonc
{
  "clusters":     ["hipaa-prod", "hipaa-dev"],   // glob ok: "prod-*"
  "namespaces":   ["regulated-*"],
  "tenants":      ["regulated-data"],
  "environments": ["production"],
  "match":        "all"   // all | any  — how the selectors combine
}
```

### 2.10 Enumerations

- **`enforcement_class`** (§7.2): `Runtime | Build-Time | Detective | Manual | Advisory`. A control MAY carry
  a *set* (DT-01 `DG-EXFIL-003` is "Runtime + Detective"); modeled as `primary_class` + `additional_classes[]`.
- **`EngineTarget`**: `opa | gatekeeper | kyverno | conftest | privateer | analytics | pdp-library | manual`.
- **`LifecycleStatus`** (control/objective): see §4.

### 2.11 `FrameworkRequirement` & cross-reference (DT-02)

```jsonc
FrameworkRequirement { framework: "SOC2", requirement: "CC6.1", text: "...", domains:["identity","data-governance"] }
CoverageLink { framework_ref, control_id, coverage: "full"|"partial", rationale: "...", bidirectional: true }
```
A1 stores both the framework reference node and the `satisfied_by` links, and serves the
coverage badge aggregate (`covered`, `partial`, `gaps`) consumed by the §17E coverage-gap report (DT-02 step 7).
**[DECISION D5]** *Gap* computation (which framework requirements have no `full` link) is an A1 read-side
aggregate, but the *report rendering* and backlog item creation belong to C5 (§17E) / the console (E2). A1
exposes the raw coverage matrix; it does not render reports.

### 2.12 `LineageEdge` (graph-shaped lineage records — §26.1)

Append-only typed edges enabling the Governance Graph View (§16.3) and traceability (G1) **without** a graph DB.

| Field | Type | Notes |
|---|---|---|
| `edge_id` | ULID | |
| `from_ref` / `to_ref` | typed node ref (`control:SC-IMG-001`, `rego:governance.kubernetes.imagesigning`, `framework:SOC2:CC6.1`, `bundle:v24`, `constraint:imagesigning-required`, `decision:<correlation_id>`) | |
| `edge_type` | `owns \| satisfied_by \| implemented_by \| enforced_at \| produced_evidence \| superseded_by \| derived_from` | |
| `valid_from` / `valid_to` | ts | edges are **temporal**: closing an edge (set `valid_to`) records deprecation without deletion (DT-04) |
| `created_by` | JWT subject | |

> **[DECISION D6]** Lineage is stored as **temporal edge records in the primary relational store**, queried
> with recursive CTEs, not in a dedicated graph database (spec §26.1 forbids requiring a graph DB at spec
> phase). Edges are append-only and temporal so historical traceability (G1) survives deprecation. The Graph
> View (E2) materializes a subgraph by bounded-depth traversal from a seed node. See ALT-event-sourced for an
> alternative that makes the edge log the *source of truth*.

---

## 3. Identifier & uniqueness rules

- **A1-MUST-001** `control_id` MUST be globally unique across all domains and all lifecycle states (including
  `retired`); A1 MUST reject creation of a duplicate even if the prior holder is deprecated.
- **A1-MUST-002** `control_id` MUST match the regex in §2.4 and MUST embed a domain-meaningful prefix
  (`SC-`, `DG-`, `HIPAA-AC-`…). A1 SHOULD offer a suggested next ID per prefix but MUST allow author override.
- **A1-MUST-003** Deleting a Control is forbidden once it has emitted any decision/evidence. The only removal
  path is `deprecated → retired` (status), preserving the ID forever (DT-04 "historical decisions remain
  queryable"). IDs are never reused.
- **A1-SHOULD-001** `objective_id` and `domain_id` SHOULD follow the same prefix discipline.

---

## 4. Control lifecycle state machine

A1 owns the **governance** lifecycle of a Control (distinct from A2's *enforcement-mode* lifecycle).

```
            ┌──────────── reject ──────────┐
            ▼                              │
        ┌───────┐  submit   ┌───────────┐  approve   ┌────────┐
  new ─►│ draft │ ────────► │ in_review │ ─────────► │ active │
        └───────┘           └───────────┘            └────────┘
            ▲                                            │  │
            │            (edit returns to draft)         │  │ deprecate(supersededBy?, grace_window)
            └────────────────────────────────────────────┘  ▼
                                                       ┌────────────┐  grace+dryrun elapsed,
                                                       │ deprecated │  enforcement removed
                                                       └────────────┘
                                                              │ seal
                                                              ▼
                                                       ┌──────────┐
                                                       │ retired  │ (immutable; ID locked forever)
                                                       └──────────┘
```

| Transition | Guard (MUST) | Side effects |
|---|---|---|
| `draft→in_review` | all MUST fields populated; EvidenceRequirement passes §13.3 superset check (A1-MUST-010) | snapshot revision |
| `in_review→active` | approver holds the owning Domain's accountable role; no dangling FKs | publishes control via Governance API; opens lineage `owns` edges; A2 may now author |
| `in_review→draft` | reviewer rejects | records reason |
| `active→deprecated` | actor holds Domain role; `effective_deprecation_at` set; `superseded_by` set OR rationale "no replacement" recorded; **coverage-gap check passes** (no framework requirement loses its only `full` link without acknowledgement) | emits governance-change event (DT-04 step 3); notifies A2 to wind down enforcement |
| `deprecated→retired` | grace window elapsed AND A2/B-layer reports **zero active enforcement points** referencing the control (DT-04 step 6) AND final bundle hash recorded | seals control; closes lineage edges (`valid_to`); locks all fields |
| `deprecated→active` | re-activation within grace window only | reverses wind-down |

- **A1-MUST-010** A1 MUST NOT allow `draft→in_review` unless the EvidenceRequirement's
  `required_core_fields` is a superset of the current `AUDIT_CORE_FIELDS@schema_version`.
- **A1-MUST-011** `active→deprecated` MUST persist `deprecated_at`, deprecating JWT subject, `superseded_by`,
  rationale, and `effective_deprecation_at` (DT-04 success criteria).
- **A1-MUST-012** `deprecated→retired` MUST record a hash of the final pre-deprecation policy bundle version
  (`current_policy_version`) for tamper-evident sealing (DT-04 step 8, §23).
- **A1-SHOULD-010** A1 SHOULD expose a *blocking check* endpoint that A2/B layers call to confirm "is this
  control safe to remove enforcement?" returning the live enforcement-point inventory.

> **[DECISION D7]** Two lifecycles are deliberately separate: A1's **governance status**
> (`draft/in_review/active/deprecated/retired`) and A2's **enforcement mode**
> (`dry-run/warn/enforce/deprecated`). A control can be `active` (governance) while its enforcement is still
> `dry-run`. Conflating them would force a control into "active" before it is safely enforcing, or block
> governance edits during enforcement tuning. The two are linked by events, not by a shared field. Rationale:
> DT-05 promotes *enforcement* without changing governance status; DT-04 deprecates *governance* which then
> *drives* enforcement wind-down.

---

## 5. Interfaces / API (the §21 governance slice)

REST/JSON (gRPC optional later). All mutating endpoints require an OIDC/JWT bearer (D1) and are
authorized server-side against §17A roles (D2). All responses are stable, paginated, and ETag'd.

```
# Domains / Objectives / Controls (CRUD + lifecycle)
POST   /v1/domains
GET    /v1/domains/{domain_id}
POST   /v1/objectives
GET    /v1/objectives/{objective_id}
POST   /v1/controls
GET    /v1/controls/{control_id}            # full control incl. 4 requirements (DT-01 success criterion)
PATCH  /v1/controls/{control_id}            # edits create a revision (append-only)
POST   /v1/controls/{control_id}:transition # body: {to: in_review|active|deprecated|retired, ...guards}
GET    /v1/controls?domain={d}&framework={f:req}&enforcement_class={c}&status={s}   # query (DT-01/DT-02)

# Requirements (child of control)
PUT    /v1/controls/{control_id}/enforcement-requirement
PUT    /v1/controls/{control_id}/evaluation-requirement
PUT    /v1/controls/{control_id}/evidence-requirement      # runs §13.3 superset validation
PUT    /v1/controls/{control_id}/exception-requirement     # read by PolicyException validator (DT-03)

# Framework cross-references & coverage
POST   /v1/frameworks/{framework}/requirements
POST   /v1/coverage-links                                   # {framework_ref, control_id, coverage, rationale}
GET    /v1/coverage?framework={f}[&requirement={r}]         # coverage matrix + badge aggregate (DT-02)

# Lineage / traceability (G1)
GET    /v1/lineage?seed={node_ref}&depth={n}&at={ts}        # temporal subgraph for Graph View (§16.3)
GET    /v1/controls/{control_id}/enforcement-points         # live inventory for safe-deprecation (DT-04)
GET    /v1/controls/{control_id}/revisions                  # append-only history

# Bulk / export
GET    /v1/export?domain={d}&format=gemara|json             # signed export feeds C5 evidence packages
```

- **A1-MUST-020** `GET /v1/controls/{id}` MUST return the control with all four requirement objects inlined
  (DT-01 success criterion: "retrievable with all §6.1 layers populated").
- **A1-MUST-021** The API MUST expose the ExceptionRequirement on the control object so the §17C.6 validator
  can read it (DT-03). This field MUST be readable by the `PolicyException` admission controller's service identity.
- **A1-MUST-022** Every mutating call MUST emit a governance-change event onto the change feed (§10 dep on C2),
  carrying actor JWT subject, entity ref, before/after revision IDs, and a `correlation_id`.
- **A1-SHOULD-020** A1 SHOULD support a webhook/subscription so A2 and C-layer learn of `active` and
  `deprecated` transitions without polling (DT-04 step 3 "governance-change feed").

### 5.1 Gemara import/export representation

> **[DECISION D8]** The on-disk / API canonical representation is **Gemara-aligned YAML** (the governance
> definitions per §5.3 "Policies originate in Gemara governance definitions"), serialized 1:1 with the entity
> model above. A1 MUST round-trip Gemara YAML ↔ entity model without loss. This makes the governance corpus
> GitOps-friendly (DT-04/DT-05 commit via GitOps) and lets the OpenSSF Gemara toolchain consume it. Where the
> OpenSSF Gemara schema and this model diverge, A1 stores a `x-platform-*` extension block rather than dropping
> data. Rationale: §5 names Gemara as the governance layer; treating Gemara YAML as the source format avoids a
> proprietary lock-in and keeps controls reviewable in PRs.

---

## 6. Persistence & versioning

- **A1-MUST-030** All entity mutations MUST be **append-only revisions**: a `ControlRevision` row per change,
  the live row pointing at the head revision. No in-place destructive update of historical fields.
- **A1-MUST-031** Lineage edges MUST be temporal (`valid_from`/`valid_to`), never hard-deleted (D6).
- **A1-SHOULD-030** Storage MUST be relational-portable (Postgres reference); graph queries via recursive CTE.
  No hard dependency on a graph database (§26.1).
- **A1-MUST-032** A signed/sealable export MUST be reproducible: exporting the same domain at the same `at=`
  timestamp MUST yield byte-identical canonical output (deterministic field ordering) so C5/§23 can sign it.

---

## 7. Failure modes & handling

| # | Failure | Detection | Handling (MUST/SHOULD) |
|---|---|---|---|
| F1 | Duplicate `control_id` submitted | uniqueness index | MUST reject 409 with existing holder ref |
| F2 | EvidenceRequirement drops a §13.3 core field | save-time validation (A1-MUST-010) | MUST reject 422 listing missing fields |
| F3 | Deprecate a control still actively enforced | enforcement-points inventory non-empty at `→retired` | MUST block retire; allow deprecate-with-grace only (DT-04) |
| F4 | Dangling FK (objective points at missing domain) | referential check on transition-to-active | MUST block `→active` |
| F5 | `AUDIT_CORE_FIELDS` schema_version bumped by C2, existing controls now under-spec'd | nightly conformance scan | SHOULD flag affected controls as `evidence_drift` (not auto-edit); surface in coverage report |
| F6 | Framework requirement loses its only `full` link on deprecation | coverage-gap guard on `→deprecated` | MUST require explicit acknowledgement (rewire or accept gap) (DT-04 step 2) |
| F7 | Concurrent edits to same control | optimistic concurrency (ETag/revision) | MUST 412 on stale write |
| F8 | Governance store unavailable | health probe | Admission/enforcement MUST fail per its own component's policy using *cached* exception/requirement data (see §9 cache contract); A1 outage MUST NOT silently disable enforcement |
| F9 | `control_id` referenced by a Rego bundle but absent in A1 | lineage reconciliation job | SHOULD raise `orphan_implementation` finding (a Rego claims a control that doesn't exist) |

> **[DECISION D9]** A1 is a **control-plane, not a data-plane** dependency. Enforcement engines (B2/B3/B4)
> MUST cache the small slices they need (ExceptionRequirement, required claims) and continue enforcing if A1
> is down (F8). A1 unavailability degrades *authoring/reporting*, never *enforcement*. Rationale: putting the
> governance store on the admission hot path would make every Pod admission depend on A1 uptime — unacceptable.

---

## 8. Security & authz notes

- **A1-MUST-040** All writes authorized server-side (§23, §17A) — never trust client-asserted role; resolve
  from JWT (D1) and the §17A.2 role model (D2). The owning Domain's role gates control mutations.
- **A1-MUST-041** Deprecation/retire transitions MUST capture deprecating subject identity and be tamper-evident
  (revision hash chain) for §23/audit.
- **A1-MUST-042** Read scoping: Auditor (read-only) and Developer (own-namespace) roles see filtered slices;
  redaction of `external_data_refs`/control-specific fields per `redaction_policy_ref` (DT-57, §17A.5).
- **A1-SHOULD-040** Export signing keys and the revision hash chain SHOULD let an auditor independently verify
  that a governance export was not altered (Daniel's audit-committee question).

---

## 9. Cache contract for data-plane consumers

A1 publishes a **read-optimized projection** (`GovernanceProjection`) — for each `active` control: `control_id`,
`enforcement_class`, `enforcement_targets`, `required_jwt_claims`, the ExceptionRequirement, and
`current_policy_version`. Consumers (B2/B3/B4/D3) subscribe and cache it.
- **A1-MUST-050** The projection MUST be versioned and monotonic; consumers MUST be able to detect staleness.
- **A1-SHOULD-050** Projection propagation SHOULD be < 60s p99 from an `active`/`deprecated` transition.

---

## 10. Dependencies on other components

| Depends on | What A1 needs | Direction |
|---|---|---|
| **C2 Audit schema** | the authoritative §13.3 `AUDIT_CORE_FIELDS@version` constant (validation in A1-MUST-010/F2) | A1 imports |
| **D1 Keycloak/JWT** | claim *names* to reference in `required_jwt_claims`; subject identity for audit | A1 references |
| **D2 Scoped roles** | the §17A.2 role catalogue for `owner`/`approver_role` (RoleRef) and server-side authz | A1 references |
| **A2 Policy Lifecycle** | consumes A1 `active` controls; reports back `current_policy_version`, enforcement-point inventory, deprecation wind-down | bidirectional |
| **B1 OPA bundles** | `related_rego_package` ↔ `__control_id__` (§8.3) join; bundle signing/version | A1 ↔ B1 via control_id |
| **B2/B3/B4 engines** | consume GovernanceProjection cache; report active enforcement points | A1 → engines |
| **D3 / §17C.6 PolicyException** | reads ExceptionRequirement to validate exception CRDs (DT-03) | engines → A1 |
| **C1 Privateer** | EvaluationRequirement may reference a Privateer evaluation id (§11) | A1 references |
| **C5 / §17E Reporting** | consumes coverage matrix + signed export for coverage-gap & evidence packages | A1 → C5 |
| **E2 Console / §16.3** | Graph View, framework-mapping panel, Rego Explorer link consume lineage + control API | A1 → E2 |

---

## 11. Acceptance criteria (component-level, traceable)

- **AC-1 (DT-01)** Creating `OBJ-EXFIL-001` + 3 controls yields, via `GET /v1/controls/{id}`, all four
  requirement objects populated; query by `domain=data-governance` returns all three; each evidence
  requirement enumerates §13.3 core fields plus control-specific fields.
- **AC-2 (DT-02)** A `CoverageLink` set + coverage query returns `{covered, partial, gaps}` and the gap list
  matches the missing-`full`-link computation; bidirectional lookup works from either side.
- **AC-3 (DT-03)** `GET /v1/controls/SC-IMG-001` exposes an ExceptionRequirement with `approver_role=Security
  Reviewer`, `max_duration_days=90`, `required_linked_artifacts=[jira_ticket]`, readable by the CRD validator identity.
- **AC-4 (DT-04)** A `deprecate` transition persists `deprecated_at`/subject/`supersededBy`; `retire` is blocked
  while enforcement-points inventory is non-empty; final bundle hash is recorded; historical revisions remain queryable.
- **AC-5 (HL-07)** A new `hipaa` domain with objectives/controls supports filtering all governance reads by
  `domain=hipaa` with no cross-domain leakage in export.
- **AC-6 (G1 traceability)** `GET /v1/lineage?seed=control:SC-IMG-001` returns the path control→rego→
  constraint→decision at a given `at=` timestamp.

---

## 12. Open questions — each with a DECIDED default

| # | Question | Decided default | Rationale |
|---|---|---|---|
| OQ-1 | Reusable requirement templates across controls? | **Templates only, not shared instances (D1).** Add a `RequirementTemplate` library that *clones* into a control. | Avoids action-at-a-distance; keeps versioning local. |
| OQ-2 | Who renders coverage-gap reports? | **C5/§17E renders; A1 exposes raw matrix (D5).** | Separation: A1 = data, C5 = presentation. |
| OQ-3 | Graph DB vs relational? | **Relational + temporal edges + CTE (D6).** Re-evaluate at GA if traversal cost dominates. | §26.1 forbids requiring a graph DB. |
| OQ-4 | Multi-tenant isolation of the governance store itself? | **Single store, row-level domain/tenant scoping via §17A.5 storage authz (D2).** | POC scale (§26.1); avoids premature sharding. |
| OQ-5 | Can a control belong to >1 objective? | **No — one owning objective; use framework cross-refs for many-to-many semantics.** | Keeps the tree a tree; coverage handled by CoverageLink. |
| OQ-6 | Versioning of the governance *schema* itself (entity shape)? | **Semver on the Gemara YAML schema; A1 supports N and N-1 on read.** | GitOps corpora outlive code deploys. |

---

## 13. Decisions log (this component)

D1 child-entity requirements; D2 one-level domain nesting; D3 control_id single global authority; D4 §13.3
superset validation enforced at A1; D5 A1 exposes coverage matrix, C5 renders; D6 relational temporal lineage
(no graph DB); D7 governance status vs enforcement mode are separate lifecycles; D8 Gemara-YAML canonical
round-trip representation; D9 A1 is control-plane (cacheable, off the enforcement hot path).
