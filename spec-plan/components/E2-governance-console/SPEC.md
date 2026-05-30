# E2 — Governance Console / Headlamp GUI — SPEC

**Component:** E2 · **Domain:** E — Simulation & Console
**Authoritative spec:** §16 (§16.1–§16.3), with dependencies on §17A.4/§17A.5 (auth/scope), §13 (audit), §17 (simulation), §8.3 (Rego metadata), §6 (governance hierarchy).
**Author persona:** Jess (SRE/Operator) + Priya (Compliance) — cooperative author.
**Status:** DRAFT v1 · **Date:** 2026-05-30

> Market context (research §8): Headlamp is now the **official Kubernetes SIG UI** (post-Dashboard archival, 2025). A **governance-objective → control → Rego → enforcement-point → audit-record lineage graph is an open market gap** — no open-source product offers it. The Governance Graph View is E2's flagship differentiator.

---

## 1. Scope

E2 is the **graphical governance console**: a lightweight, Kubernetes-native web UI that visualizes governance, Rego, lineage, runtime enforcement, audit, drift, and namespace-scoped authoring, and drives Conftest/simulation/approval workflows (§16.1). Default implementation = **Headlamp plugin** (§16.2) for Kubernetes-first deployments; Backstage/OpenLens-compatible APIs are secondary targets.

**In scope**

- The 5 required views (§16.3): Governance Graph, Rego Explorer, Runtime Enforcement, Audit Correlation, Namespace Authoring.
- The **lineage graph data model** (control → Rego → enforcement-point → decision/audit) and its query API.
- Headlamp-plugin packaging (signed OCI artifact, manifest, WASM Rego runtime) and Keycloak **OIDC** authentication (§16.2, §17A.4).
- **Kubernetes-native RBAC discovery** (§16.2) and how **D2 RBAC scope (§17A.5)** constrains every view at the storage layer (not GUI-only).
- Data-source bindings per view (which backend each view reads/writes).

**Out of scope (owned elsewhere)**

- The simulation engine — **E1** (E2 launches runs + renders results).
- The audit schema/store — **C2**; analytics/drift detection — **C3/C4**; reports — **C5** (E2 renders them).
- Identity/claims/RBAC enforcement — **D1/D2** (E2 *consumes* tokens + scope decisions).
- The engines & bundles — **B1–B5**.

---

## 2. Framework & packaging

### 2.1 Headlamp plugin model (§16.2)

- **Default deployment:** a Headlamp **plugin** distributed as a **signed OCI artifact** (`oci://registry/platform/headlamp-governance:vX`). Install: `headlamp-plugin install oci://...` (DT-44).
- **Plugin manifest declares:** OPA REST endpoints, **WebAssembly Rego runtime** (for in-browser policy eval/preview), OIDC client (`headlamp-governance`), required scopes, and the backend governance-API base URL.
- **React frontend** (§16.2). Inherits Headlamp's Kubernetes context, cluster visibility, and namespace-scoped access patterns (§16.2 rationale for Headlamp default).
- **Why Headlamp default:** it can inherit cluster context + namespace-scoped user access patterns (§16.2); it is the official SIG UI (research §8), de-risking the GUI base choice.
- **Secondary targets:** Backstage plugin + OpenLens-compatible APIs share the same backend governance API so the views are portable.

### 2.2 Authentication — Keycloak OIDC (§16.2, §17A.4)

- On first open, the plugin runs an **OIDC authorization-code flow** against Keycloak realm `platform`, client `headlamp-governance`, scopes `openid groups roles namespaces policy_domains tenants` (DT-44).
- The resulting JWT carries the §17A.4 claims: `groups`, `realm_access.roles`, `namespaces`, `policy_domains`, `tenants`. These drive **both** UI affordances **and** server-side scope (§3 below).
- Token is presented as a bearer to the governance backend API on every call; the backend re-validates and derives scope server-side (never trusts the client's claim of scope).

### 2.3 Kubernetes-native RBAC discovery (§16.2)

- On launch the plugin auto-discovers connected clusters via each kubeconfig context and probes `SelfSubjectAccessReview` per context (DT-44) — no static cluster list. Clusters the user cannot access are not shown.

---

## 3. RBAC scope (Domain D) constrains every view — normative

**Principle (§17A.1):** authorization is enforced in **both** the GUI/API layer **and** the underlying storage/retrieval layer. **GUI-only filtering is insufficient.** Every view's data query passes the subject's scope token to the storage layer (C2/policy store), which strips out-of-scope rows **before** they reach the GUI (DT-43 step 1, §17A.5).

| Role (§17A.2) | Graph | Rego Explorer | Runtime Enforcement | Audit Correlation | Namespace Authoring |
|---|---|---|---|---|---|
| Platform Governance Admin | full | full RW | full | full | full |
| Namespace Policy Author | scoped to owned ns | author/test owned ns | owned-ns decisions | owned-ns violations | **author + simulate, hard-locked to claim namespaces** |
| Namespace Policy Approver | scoped | read | read | owned-ns | approve promotions (owned ns) |
| Compliance Analyst | read (domain/global) | read | read | read evidence | read |
| Security Reviewer | read | read | read + risky sims | read + FP/FN | read |
| Developer | read relevant | run local tests | relevant decisions | relevant | propose (if delegated) |
| Auditor | read-only | read-only | read-only | read-only immutable | read-only |

- **Namespace Authoring View** hard-locks the namespace selector to the subject's `namespaces` claim and **refuses any `clusterScope` field** (DT-43 step 2). Storage filters strip non-matching policies/sims/violations/approvals before render (DT-43 step 1).
- Scope dimensions = `{cluster, namespace, tenant, policy_domain}` from the JWT (§17A.4). The backend computes an effective scope set; every list/graph/query API intersects with it.

---

## 4. The lineage graph data model (flagship — normative)

The Governance Graph View renders a typed, directed graph connecting **organizational objective → control → Rego package → enforcement point → decision/audit record** (research §8 gap). This is E2's core differentiator.

### 4.1 Node types

| Node | Source component | Key fields |
|---|---|---|
| `Objective` | A1 / §6 Gemara hierarchy | `objective_id`, `title`, `domain` |
| `Control` | A1 / §6 | `control_id` (e.g. SC-IMG-001), `severity`, `governance_domain`, `exception_state` |
| `RegoPackage` | B1 / §8.3 | `package`, `bundle_version`, `signing_status`, `promotion_stage` (§7.2), `__control_id__`, `__required_claims__`, `__required_audit_fields__`, test_coverage |
| `EnforcementPoint` | B2/B3/B4/B5/§17D | `kind` (Gatekeeper constraint / Kyverno policy / Conftest test / app-PDP / scanner gate), `engine`, `mode` (warn/dryrun/enforce), `product`, `namespace_scope` |
| `PolicyActionLibrary` | E3 / §17D | `product`, `library_version` |
| `AuditEvidence` | C2 / §13 | `event_id`, `decision`, `replay_completeness`, `correlation_id`, `control_id` |
| `ApprovalGate` | D3 / §17B | `approval_request_ref`, `state` |
| `Exception` | B4/D3 / §17C.6 | `exception_ref`, `expiresAt`, `scope` |

### 4.2 Edge types (directed)

```
Objective  --decomposes-->        Control
Control    --implemented_by-->    RegoPackage          (via §8.3 __control_id__)
Control    --has_exception-->     Exception
RegoPackage--backed_by-->         PolicyActionLibrary  (§17D)
RegoPackage--enforced_at-->       EnforcementPoint
EnforcementPoint --gated_by-->    ApprovalGate         (§17B suspend-pending-approval)
EnforcementPoint --produced-->    AuditEvidence        (each decision)
AuditEvidence --correlates-->     AuditEvidence        (cross-product via correlation_id)
```

### 4.3 Lineage queries

- **Forward (control → evidence):** "show me every decision SC-IMG-001 produced last quarter" — Priya's SOC2 workpaper (DT-39, HL-01). Traverses Control → RegoPackage → EnforcementPoint → AuditEvidence.
- **Backward (decision → objective):** "this deny — which control/objective justifies it?" — trace a runtime deny up to the governing objective (G1 traceability, DT-13).
- **Coverage gap:** Controls with **no** `enforced_by` edge = unenforced controls (§17E coverage gaps); namespaces with no enforcement point for a control (DT-80).
- **Drift:** RegoPackage `promotion_stage` vs EnforcementPoint `mode` mismatch (declared enforce, running dryrun) = drift indicator (C3/C4, §16.3 Runtime Enforcement View).

The graph is materialized as a queryable backend (assembled from A1 controls, B1 bundle metadata, B-engine enforcement registry, C2 audit index). E2 owns the **graph assembly/query service + render**; it does not own the source data.

### 4.4 Node side-panel (DT-39 step 3)

Clicking a RegoPackage node renders §8.3 metadata (`__control_id__`, `__severity__`, `__governance_domain__`, `__required_claims__`), bundle version, signing status, promotion stage. Each node type has a defined detail panel and deep-links into the owning view (e.g., RegoPackage → Rego Explorer; AuditEvidence → Audit Correlation; EnforcementPoint → Runtime Enforcement).

---

## 5. The five views — data sources, interactions, render (normative)

### V1 — Governance Graph View (§16.3)
- **Renders:** objectives, controls, Rego packages, enforcement points, audit evidence, approval gates, policy action libraries (the §4 lineage graph).
- **Data sources:** A1 (objectives/controls), B1 (bundle metadata §8.3), B-engines (enforcement registry), E3 (action libraries §17D), C2 (audit index), D3/B4 (approval gates/exceptions).
- **Interactions:** search by control_id; center/expand/collapse nodes; forward/backward lineage traversal; node side-panels; deep-link to other views; coverage-gap + drift overlays.
- **Scope:** intersect with subject effective scope (out-of-scope nodes hidden server-side).
- **Scenario:** DT-39, HL-01.

### V2 — Rego Explorer (§16.3)
- **Renders:** Rego packages, rule dependencies, control mappings, policy test coverage, required JWT claims, required audit fields.
- **Data sources:** B1 bundle store + `opa inspect`/§8.3 metadata; test results (CI/B1); E1 (links to fixtures/coverage).
- **Interactions:** browse packages; view rule dependency graph; author/edit policy (WASM in-browser preview via §16.2 WASM runtime); see test coverage %; see `__required_claims__`/`__required_audit_fields__`; launch a simulation (hands to E1); link a saved fixture to a control + bundle version (§17.5).
- **Scope:** packages restricted to subject's `policy_domains`/namespaces.
- **Scenario:** DT-40 (coverage), HL-17 (author v13 here).

### V3 — Runtime Enforcement View (§16.3)
- **Renders:** active Gatekeeper constraints, OPA policy bundles, Kyverno policies, decision statistics, recent denies, suspend-pending-approval events, drift indicators.
- **Data sources:** B2 (Gatekeeper), B1 (OPA bundles), B4/Kyverno, C2 (decision stats/recent denies), C3/C4 (drift), D3 (suspend-pending-approval).
- **Interactions:** filter by namespace/engine/control; drill into a recent deny → Audit Correlation; see live mode (warn/dryrun/enforce); drift badges (declared vs actual mode); jump to E1 live-shadow (M3) for a candidate bundle.
- **Scope:** decisions/constraints restricted to subject scope.
- **Scenario:** DT-41 (recent denies), §17E.2 real-time report consumer.

### V4 — Audit Correlation View (§16.3)
- **Renders:** enforcement decisions, missing evaluations, compliance gaps, violation timelines, false-positive/false-negative analysis, **policies that would newly allow previously blocked behavior**, **policies that would newly block previously allowed behavior**.
- **Data sources:** C2 (audit events), C3 (analytics: bypass/drift/coverage), C4 (retrospective detection), E1 (differential newly-allowed/newly-blocked, FP/FN).
- **Interactions:** select an audit event/violation → **create test case** (§17.5, hands fixture to E1); view violation timeline; review FP/FN; view differential newly-allowed/newly-blocked buckets and **tag** them (writes E1 SimulationTag); jump to the §17E.4 report.
- **Scope:** events restricted to subject scope (storage-enforced).
- **Scenario:** DT-42 (gap), DT-49/HL-17 (tagging surface), §17.5 workflow.

### V5 — Namespace Authoring View (§16.3)
- **Renders & enforces:** namespace-scoped policy authoring permissions, namespace-scoped simulation permissions, namespace-scoped violation visibility, namespace-scoped approval state, underlying storage authorization boundaries.
- **Data sources:** policy store (scoped), E1 (scoped sims, M6), C2 (scoped violations), D3 (scoped approval state), D2 (scope enforcement §17A.5).
- **Interactions:** list only policies/sims/violations/approvals whose `namespaces` intersect the subject claim (storage strips the rest, DT-43 step 1); "New namespace policy" form **hard-locked** to claim namespaces, refuses `clusterScope` (DT-43 step 2); "Simulate" creates a `PolicySimulationRun` with `visibility=namespace-scoped` (DT-43 step 3, M6); submit for approval to a `namespace-policy-approver`.
- **Scope:** the canonical demonstration that scope is enforced at storage, not GUI.
- **Scenario:** DT-43, HL-08.

---

## 6. Interfaces / APIs (backend governance API)

| API | View | Notes |
|---|---|---|
| `GET /v1/graph?center={control_id}&depth=N` | V1 | typed lineage subgraph, scope-intersected |
| `GET /v1/graph/coverage-gaps?scope=...` | V1 | controls/namespaces with no enforcement edge |
| `GET /v1/rego/packages` , `/rego/{pkg}` | V2 | metadata, deps, coverage, required claims/audit fields |
| `POST /v1/rego/preview` | V2 | WASM in-browser eval (no enforcement) |
| `GET /v1/enforcement?ns=&engine=` | V3 | constraints/bundles/policies + stats + drift |
| `GET /v1/audit/correlate?...` | V4 | decisions, gaps, timelines, FP/FN, newly-allowed/blocked |
| `POST /v1/fixtures` (→E1) | V4 | create test case from event (§17.5) |
| `POST /v1/simulations/{id}/tags` (→E1) | V4 | tag differential result |
| `GET /v1/ns/{ns}/policies|sims|violations|approvals` | V5 | hard-scoped to claim |
| `POST /v1/ns/{ns}/policies` , `/simulate` | V5 | clusterScope refused; sim visibility=namespace-scoped |

All APIs: bearer JWT (§17A.4 claims) → server-derived effective scope → storage-layer filter (§17A.5). The client cannot widen scope.

---

## 7. Failure modes

| Failure | Handling |
|---|---|
| Token expired mid-session | Silent OIDC refresh; on hard-fail, re-auth, preserve view state |
| Cluster unreachable (RBAC discovery) | Show cluster as degraded; other clusters still usable |
| Graph too large to render | Server-side depth cap + lazy expand; coverage/drift run as queries not full render |
| Source component down (e.g. C2 audit) | View renders partial with "evidence unavailable" banner; never fabricates lineage |
| Client attempts out-of-scope query | Backend rejects (403) + storage returns empty; logged as access-violation |
| WASM Rego preview diverges from server eval | Label preview "advisory"; authoritative eval is server-side via E1 |
| Stale graph (source changed) | Graph carries `assembled_at`; refresh/invalidate on source events |

---

## 8. Security / authz notes

- **Dual-layer authz** (§17A.1): GUI affordances + storage filter. Pen-test must confirm a crafted API call cannot read out-of-scope data even if the UI hides it (DT-43).
- Plugin distributed as **signed OCI**; verify signature on install (§23, DT-44).
- OIDC client confidential where possible; PKCE for the SPA flow.
- Every authoring/tagging/approval action is itself an audit event.
- Auditor role is strictly read-only against immutable evidence (§17A.2).

---

## 9. Open questions — decided defaults

| # | Question | Decided default | Rationale |
|---|---|---|---|
| OQ-1 | Headlamp plugin vs standalone SPA vs Backstage | **Headlamp plugin default**; shared backend API enables Backstage/OpenLens reuse | §16.2 + research §8 (official SIG UI) |
| OQ-2 | Graph backend: build a graph DB or assemble on query? | **Assemble-on-query** over existing stores (A1/B1/C2 indexes) for MVP; materialize/cache hot subgraphs | Avoids a new stateful store; sources are authoritative |
| OQ-3 | WASM in-browser eval authoritative? | **No** — preview only; authoritative eval is E1 server-side | Browser lacks external data + scope enforcement |
| OQ-4 | Scope enforcement location | **Storage layer** (D2 §17A.5) + GUI | §17A.1 GUI-only insufficient |
| OQ-5 | Real-time vs polled decision stats | **Polled** (cache + refresh) for MVP; stream for live-shadow | Simplicity; matches batch-first E1 |

---

## 10. Dependencies

- **D1/D2 (§15, §17A.4/§17A.5)** — OIDC claims + storage-layer scope — **HARD** (every view).
- **A1 (§6)** — objectives/controls (graph nodes).
- **B1 (§8.3)** — Rego package metadata (graph + Rego Explorer).
- **B2/B3/B4/B5** — enforcement registry (Runtime Enforcement + graph edges).
- **C2 (§13)** — audit events (Audit Correlation + graph evidence nodes).
- **C3/C4 (§14, §19)** — drift/analytics overlays.
- **C5 (§17E)** — reports rendered in-console.
- **E1 (§17)** — launch sims, render diffs, tag results, create fixtures.
- **E3 (§17D)** — policy action library nodes.
- **D3/B4 (§17B, §17C.6)** — approval gates / exceptions.

---

## 11. Traceability

Spec: §16.1–§16.3, §17A.4, §17A.5, §8.3, §6, §17, §17E, §13.
Scenarios: DT-39, DT-40, DT-41, DT-42, DT-43, DT-44, HL-01, HL-08, HL-17.
Personas: Jess (operator), Priya (compliance/workpaper), Marcus (admin/author), Sam (namespace author), Daniel (auditor read-only).
