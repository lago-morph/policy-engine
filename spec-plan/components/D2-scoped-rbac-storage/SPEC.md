# D2 — Scoped Roles, Permissions & Storage Authorization — SPEC

**Domain:** D · Identity, Authz & Security · **HIGH-VALUE (has ALT tree)**
**Spec sources:** §17A (Scoped Roles, Permissions, and Storage Authorization, ~lines 850–985),
§16.3 (Namespace Authoring View — "Underlying storage authorization boundaries"),
§23.1 (Access control + Authorization: server-side, scope-aware)
**Scenarios exercised:** DT-53, DT-54, DT-55, DT-56, DT-57, HL-04, HL-13, HL-18
**Status:** AUTHORED (parent-authored)
**Author persona:** Marcus (Platform Security Engineer) + storage/data engineer
**Alt-architecture:** `ALT-opa-rls-spicedb.md`

---

## 1. Scope

### 1.1 In scope
D2 owns **authorization** end-to-end: deciding *whether a subject may perform an operation on
an object*, and **enforcing it at the storage layer, not just the GUI/API** (§17A.1's hard
requirement: "GUI-only authorization is insufficient"). Specifically:

1. The **9-role model** (§17A.2) and its registry.
2. **Permission primitives** (§17A.3) — the verbs and the resource dimensions they apply over.
3. The **scope model** — cluster / namespace / tenant / policy-domain / control-id, and how a
   subject's scope (from the D1 normalized subject) is matched against object scope.
4. The **complete role × permission matrix** (this document, §4).
5. **Storage-layer enforcement** (§17A.5) — every stored object carries authorization
   metadata; every query is scope-filtered server-side; no out-of-scope retrieval; pagination
   and counts reflect the filtered set; admin boundary-crossings are audited.
6. **Policy-bundle scope encoding** and **scoped audit-replay dataset materialization**.
7. The **threat model for GUI-bypass** and the negative/scope-escape test suite.

### 1.2 Out of scope
- Producing the subject (D1).
- Approval *gating* of an action (D3) — D2 says "may you request approval"; D3 runs the gate.
- The runtime policy decision (allow/deny of a *Kubernetes* action) — that's Gatekeeper/OPA
  (Domain B). D2 is authz for the **governance platform's own** resources (policies,
  violations, simulations, audit fixtures, reports, bundles, approvals, exceptions).

### 1.3 The load-bearing invariant
> **Every read and write of a platform-stored object is authorized server-side against the
> subject's scope, independent of the caller (Console, curl, CI job, custom MCP client). The
> GUI is a convenience, never a control. A scope escape is a P0.** (§17A.1, DT-55.)

---

## 2. Role model (§17A.2) — the 9 roles

| # | Role | Scope level | Capability summary |
|---|---|---|---|
| 1 | **Platform Governance Admin** | Global | Controls, global policy mappings, system config; cross-tenant (audited) |
| 2 | **Policy Library Maintainer** | Global / domain | Reusable policy + action libraries |
| 3 | **Namespace Policy Author** | Namespace | Author/test policies for owned namespaces |
| 4 | **Namespace Policy Approver** | Namespace | Approve namespace-scoped policy promotion |
| 5 | **Compliance Analyst** | Domain / global | View evidence, audit findings, compliance reports |
| 6 | **Security Reviewer** | Domain / global | Review violations, exceptions, risky simulations |
| 7 | **Developer** | Namespace / project | Local tests; view relevant policy feedback |
| 8 | **Auditor** | Read-only scoped / global | View immutable reports + evidence |
| 9 | **Workflow Integrator** | Global / domain | Configure approval webhooks + workflow endpoints |

Roles are **additive** (a subject may hold several); the effective permission set is the union,
**intersected with the subject's scope**. Roles map from D1's expanded `roles[]`
(`role:namespace-policy-author`, etc.). Each role is registered in the **role registry** with:
`{role_id, scope_level, granted_permissions[], implied_roles[]}`.

---

## 3. Permission primitives & scope dimensions (§17A.3)

### 3.1 Verbs (permission primitives)
`policy:view, policy:edit, policy:test, policy:simulate, policy:promote-dry-run,
policy:promote-enforce, violation:view, audit:replay, approval:request, approval:approve,
exception:create, exception:approve, report:view`
(plus derived admin verbs: `mapping:edit, role:grant, library:maintain, workflow:configure,
config:manage` for roles 1/2/9).

### 3.2 Resource types permissions are defined over (§17A.3)
`policy_package, control, violation, evidence_set, simulation_dataset, audit_fixture,
approval_workflow, report, policy_bundle, exception, action_library`.

### 3.3 Scope dimensions
A permission grant is a triple **(verb, resource_type, scope-selector)** where the
scope-selector ranges over:
`cluster, namespace, tenant, policy_domain, control_id`.
A subject's scope comes from the D1 normalized subject:
`{tenants[], namespaces[], policy_domains[]}` (+ `clusters[]`, `*` for global).

### 3.4 Authorization decision function (normative)
```
authorize(subject, verb, object) :=
    role_grants_verb(subject.roles, verb, object.type)         # role layer
  AND scope_match(subject.scope, object.scope)                 # scope layer
  AND not denied_by_explicit_deny(subject, verb, object)       # deny overrides
```
`scope_match` is **intersection, not containment-by-string**:
```
scope_match := (object.tenant ∈ subject.tenants OR subject.tenants = ["*"])
           AND (object.namespaces ∩ subject.namespaces ≠ ∅ OR namespace not applicable)
           AND (object.policy_domains ∩ subject.policy_domains ≠ ∅ OR subject domains = ["*"])
```
A global subject (`tenants=["*"]`) crossing >1 tenant in one query triggers a
**boundary-crossing audit event** (§17A.5, DT-54) even though the access is permitted.

---

## 4. The full 9-role × permission matrix

Legend: **Y** = granted within the role's scope; **Y\*** = granted but emits boundary-crossing
audit when global subject crosses tenants; **N** = denied; **(own)** = only objects the subject
authored / owns; **RO** = read-only/immutable view.

| Permission | 1 Plat-Admin | 2 Lib-Maint | 3 NS-Author | 4 NS-Approver | 5 Compl-Analyst | 6 Sec-Reviewer | 7 Developer | 8 Auditor | 9 Workflow-Integ |
|---|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| `policy:view` | Y* | Y | Y | Y | Y | Y | Y | RO | N |
| `policy:edit` | Y* | Y(libs) | Y(own NS) | N | N | N | N | N | N |
| `policy:test` | Y* | Y | Y(own NS) | N | N | Y | Y(own NS) | N | N |
| `policy:simulate` | Y* | Y | Y(own NS) | N | Y | Y | Y(own NS) | N | N |
| `policy:promote-dry-run` | Y* | Y(libs) | Y(own NS) | Y(own NS) | N | N | N | N | N |
| `policy:promote-enforce` | Y* | N | N | Y(own NS) | N | N | N | N | N |
| `violation:view` | Y* | Y | Y(own NS) | Y(own NS) | Y | Y | Y(own NS) | RO | N |
| `audit:replay` | Y* | N | Y(own NS) | N | Y | Y | N | RO | N |
| `approval:request` | Y* | Y | Y(own NS) | Y(own NS) | N | Y | Y(own NS) | N | N |
| `approval:approve` | Y* | N | N | Y(own NS) | N | Y(domain) | N | N | N |
| `exception:create` | Y* | N | Y(own NS) | N | N | Y | Y(own NS) | N | N |
| `exception:approve` | Y* | N | N | Y(own NS) | N | Y(domain) | N | N | N |
| `report:view` | Y* | Y | Y(own NS) | Y(own NS) | Y | Y | Y(own NS) | RO | N |
| `mapping:edit` (D1) | Y* | N | N | N | N | N | N | N | N |
| `role:grant` | Y* | N | N | N | N | N | N | N | N |
| `library:maintain` | Y* | Y | N | N | N | N | N | N | N |
| `workflow:configure` | Y* | N | N | N | N | N | N | N | Y |
| `config:manage` | Y* | N | N | N | N | N | N | N | N |

**Reading notes / rationale:**
- **Separation of duties:** `policy:promote-enforce` and `approval:approve`/`exception:approve`
  are *approver*-only (roles 4/6), distinct from authoring (role 3). An author cannot
  enforce-promote their own policy — this is the SoD backbone (cross-ref D3).
- **Auditor (8) is strictly RO/immutable** — DT-56/HL-18: no edit, simulate-write, approve, or
  mapping verbs; `audit:replay` is permitted but only against *materialized scoped datasets*
  (§17A.5), never the live mutable store.
- **Workflow Integrator (9)** is deliberately narrow: it configures webhooks/endpoints but has
  **no** policy or evidence read — a webhook misconfig must not become an evidence-exfil path.
- **Compliance Analyst (5)** reads evidence/reports/replay but cannot edit or approve.
- **Plat-Admin (1)** is `Y*` everywhere: powerful, but every cross-tenant action is audited
  (DT-54) — *power is logged, not hidden*.
- **`(own NS)`** is enforced by the scope layer (§3.4), not the role layer; a Namespace Policy
  Author with `namespaces=["payments-*"]` simply cannot name a `billing` object (DT-55).

---

## 5. Storage-layer enforcement design (the core of D2)

### 5.1 Authorization metadata on every object (§17A.5)
Every stored object carries an immutable `authz` block:
```json
{
  "object_type": "simulation_dataset",
  "cluster": "cluster-a",
  "namespaces": ["payments-prod"],
  "policy_domains": ["supply-chain"],
  "control_ids": ["SC-IMG-001"],
  "tenant": "payments",
  "created_by": "alice",
  "visibility": "namespace-scoped"      // namespace-scoped | tenant-scoped | global
}
```
- **D2-R1 (MUST)** Every stored object includes this metadata at write time; it is set from the
  authoring subject's scope and the object's natural scope, and is **immutable** thereafter.
- **D2-R2 (MUST)** Writes are rejected if the authoring subject's scope does not cover the
  object's declared scope (you cannot create an object outside your scope).

### 5.2 Server-side query filtering (the invariant)
Authorization is enforced **in the data-access layer**, below the API handlers, as a
**mandatory query rewrite** — not an application-level `if`:

```
                ┌─────────────────────────────────────────┐
  request ─────►│ API handler (no authz logic of its own)  │
                └───────────────┬─────────────────────────┘
                                ▼
                ┌─────────────────────────────────────────┐
                │ Authz interceptor (single choke point)   │
                │  • normalize subject (D1)                │
                │  • derive scope predicate from subject   │
                │  • REWRITE every query: append           │
                │    WHERE authz.tenant ∈ subject.tenants  │
                │      AND authz.namespaces ∩ subject.ns    │
                │      AND authz.policy_domains ∩ subject.pd │
                │  • on explicit out-of-scope filter → 403 │
                │  • emit authz_denied / boundary_crossing │
                └───────────────┬─────────────────────────┘
                                ▼
                ┌─────────────────────────────────────────┐
                │ Storage (rows already filtered)          │
                └─────────────────────────────────────────┘
```

- **D2-R3 (MUST)** All reads pass through the interceptor; there is **no code path** from an
  API handler to storage that bypasses the scope rewrite (enforced by architecture: storage
  client is only reachable via the interceptor; a lint/test forbids direct storage access).
- **D2-R4 (MUST)** An *explicit* out-of-scope filter (e.g. `?namespace=billing` for a
  payments-only subject) returns **403 with no row data** and no existence signal (DT-55 step 3).
- **D2-R5 (MUST)** An *unfiltered* query has the subject-scope predicate applied **implicitly**;
  it returns only in-scope rows. **Counts, aggregates, and pagination cursors are computed over
  the filtered set** so existence of out-of-scope rows never leaks (DT-55 step 4, HL-13 step 5).
- **D2-R6 (MUST)** ID-guessing is defeated: fetching `/objects/{id}` for an out-of-scope object
  returns **404** (indistinguishable from "does not exist"), not 403 (which would confirm
  existence). (HL-13 step 5: "cannot enumerate even by ID guessing.")
- **D2-R7 (MUST)** Policy bundles encode scope metadata; bundles whose scope names a domain
  outside the subject are excluded from listings (DT-55 step 5, §17A.5).
- **D2-R8 (MUST)** Audit-replay datasets are **materialized as scoped datasets before use**
  (§17A.5); Auditor/Analyst replay operates on the materialized snapshot, never the live store
  (HL-18 step 3).
- **D2-R9 (MUST)** Every denied request emits an `authz_denied` event
  (`subject_id, requested_scope, denying_rule_ref, correlation_id`); every global-subject
  cross-tenant access emits a `boundary_crossing` event (DT-54, DT-55 step 7).

### 5.3 Visibility levels
`namespace-scoped` (default; visible only within owning namespace scope),
`tenant-scoped` (visible to any subject of the tenant), `global` (platform-wide).
Visibility **narrows** the scope predicate but never widens it beyond the object's `tenant`.

### 5.4 Redaction-aware export (DT-57)
Export resolves the candidate set against `visibility` first (only `tenant-scoped`/`global` for
external share), then applies a redaction profile (`subject.sub`, `email`, …). The pre-redaction
`original_hash` is stored in the manifest (§23 tamper-evidence anchor) before redaction.

---

## 6. Threat model — GUI bypass & scope escape

| Threat | Vector | Mitigation |
|---|---|---|
| **Modified/forged client** issues out-of-scope query | curl, custom MCP client, CI job | Storage-layer rewrite (D2-R3/R5); identical denial regardless of caller (DT-55 step 6) |
| **Enumeration by ID guessing** | `/objects/{out-of-scope-id}` | 404 not 403 (D2-R6) |
| **Existence leak via counts/cursors** | unfiltered list, inspect total | counts over filtered set (D2-R5) |
| **Privilege via authoring** | create object claiming a wider scope | write-scope check (D2-R2) |
| **Replay over live store** | Auditor replays mutable rows | materialized scoped dataset (D2-R8) |
| **Cross-tenant admin abuse** | global subject quietly reads all tenants | boundary-crossing audit (D2-R9, DT-54) |
| **Bundle scope leak** | list bundles, read another domain's | bundle scope metadata + exclusion (D2-R7) |
| **Mass export exfil** | analyst exports everything | visibility gate + redaction + manifest hash (DT-57) |
| **Stale-token scope** | deprovisioned role still in token | bounded token lifetime + (D3) revocation; see D1 A5 |
| **Error-message oracle** | 403 body reveals out-of-scope names | error body names only the subject's own scope (DT-55 step 3 shows subject's ns, not target rows) |

**Scope-escape is the single most important negative-test target.** Every storage release
re-runs the DT-55 pen-test suite; a regression is a §17A.1 violation (P0).

---

## 7. Failure modes
| Failure | Behavior |
|---|---|
| Subject unavailable / `incomplete` (D1) | Fail closed: deny; for tenant/ns-scoped reads, empty filtered set + warning |
| Interceptor down | No storage access at all (fail closed; never fall through to raw storage) |
| Object missing `authz` metadata | Treat as `visibility:none` → invisible to all but Plat-Admin (audited); flag as data defect |
| Scope predicate ambiguous (object names tenant not in registry) | Exclude from results; flag |
| Boundary-crossing audit sink down | Block the cross-tenant read (audit is a precondition, not best-effort) for global subjects; in-scope reads proceed |

---

## 8. Dependencies
| Depends on | For |
|---|---|
| D1 | Normalized subject (`tenants, namespaces, policy_domains, roles`) |
| D1 role-name contract | Each `role:<x>` must resolve to a registry role |
| §13.3 audit schema (C2) | `authz_denied` / `boundary_crossing` event shape |
| §16.3 Namespace Authoring View | Surfaces "storage authorization boundaries" |
| D3 | `approval:approve`/`exception:approve` SoD ties into approval gate |

| Depended on by | For |
|---|---|
| Every API endpoint (§21) | Scope-filtered reads/writes |
| §17 Simulation | Namespace-scoped datasets, materialized replay |
| §17E Reporting | report:view scope + redaction export |
| Auditor flows (HL-18) | RO scoped replay |

---

## 9. Open questions — decided defaults
| # | Question | Decided default | Rationale |
|---|---|---|---|
| OQ-1 | Enforcement mechanism: app-layer rewrite, DB row-level security, or external authz (OPA/SpiceDB)? | **App-layer mandatory query rewrite at a single interceptor** for the POC; design keeps the predicate portable so RLS/OPA/ReBAC can replace it later (see ALT). | Single choke point is simplest to prove correct + test; §22 says ordinary storage is acceptable. |
| OQ-2 | 403 vs 404 for out-of-scope by ID? | **404** (no existence signal). | Defeats enumeration (HL-13, D2-R6). |
| OQ-3 | Counts/cursors over full or filtered set? | **Filtered set.** | No existence leak (D2-R5). |
| OQ-4 | Is boundary-crossing audit blocking or best-effort? | **Blocking for global subjects** (audit precondition). | Admin power must be logged, never silently lost (DT-54). |
| OQ-5 | Default visibility? | **`namespace-scoped`** (least exposure). | Safe default; widen explicitly. |
| OQ-6 | Where does scope live — only in token, or also server-side grant store? | **Server-side grant store is authoritative; token scope is an assertion validated against it for privileged ops.** | Mitigates stale-token (D1 A5); cheap reads still trust token. |
| OQ-7 | Object missing authz metadata | **Invisible (visibility:none), Plat-Admin-only, flagged.** | Fail closed on data defects. |
