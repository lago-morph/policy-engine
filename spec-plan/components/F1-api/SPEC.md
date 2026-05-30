# F1 — API Requirements — SPEC

**Component:** F1 · **Domain:** F (Platform & Cross-cutting) · **Spec source:** §21 (+ cross-cuts §13, §15, §17A–E)
**Status:** Authored (domain-lead fallback author) · **Date:** 2026-05-30
**Persona lens (cooperative author):** Platform/API engineer building the governance control-plane surface.

---

## 1. Scope

F1 specifies the **complete external HTTP/gRPC API surface** of the unified governance platform: the resource model, endpoint catalog, authentication/authorization binding, request/response envelopes, pagination, filtering, versioning, error model, idempotency, rate limiting, and audit/observability behavior of the API tier itself.

The spec §21.1 lists only **8 seed endpoints** (`GET /controls`, `GET /controls/{id}`, `GET /rego-packages`, `GET /evidence/{controlId}`, `POST /simulate`, `POST /conftest/run`, `GET /audit/events`, `GET /violations`). Those are treated as the **MVP-required subset**; this spec defines the full surface they imply, and explicitly marks each endpoint as **MVP** or **DEFERRED** so F3 can draw the cut line.

**In scope:** the API gateway/contract that fronts every other component (A–E). **Out of scope:** the internal implementation of each backing service (owned by its component), the storage engine (Domain D / out-of-scope-for-POC per §26.1), and external workflow execution (§17B.3).

### 1.1 Design principles (normative framing)

- **P1 — One contract, many backends.** The API is a thin, uniform facade; each resource is served by its owning component. Consistency of envelope, auth, pagination, and errors is mandatory across all resources.
- **P2 — Authorization is server-side and storage-consistent.** Every read/write is filtered by the §17A authorization subject. The API MUST NOT return objects the subject's scope (namespace/tenant/policy-domain/control) excludes. GUI-only authz is explicitly insufficient (§17A.1); the API enforces the same predicate the storage layer (Domain D2) enforces.
- **P3 — Everything mutating is auditable.** Policy edits, simulations, approvals, promotions (§23 Auditability) emit a §13 replay-capable audit event with the API `correlation_id`.
- **P4 — Read-optimized for the console.** §16 console views (graph, replay, simulation, scoped authoring) drive the read API shape; list endpoints support cursor pagination, scope filters, and field projection.
- **P5 — Async where the work is unbounded.** Simulation, conftest runs, replay materialization, and report generation are **job-oriented** (submit → poll/stream), per §22.2 ("short background job").

---

## 2. Resource model (entities exposed by the API)

| Resource | Owning component | Collection path | Mutable via API? | Key scope attrs |
|---|---|---|---|---|
| Control | A1 | `/controls` | No (read) — authored via GitOps/A2 | control_id, policy_domain, framework |
| Governance graph / lineage | A1 | `/lineage` | No (read) | control_id, namespace, tenant |
| Policy package (Rego/bundle) | B1/B4 | `/policies`, `/rego-packages` | Promote-only (state transitions) | policy_ref, version, namespace, domain |
| Policy lifecycle state | A2 | `/policies/{ref}/lifecycle` | Yes (promote/demote) | mode (draft→dry-run→warn→enforce) |
| Engine binding / CRD | B4 | `/engine-bindings` | Yes (admin) | engine, action, scope |
| Simulation run | E1 | `/simulations` | Yes (submit) | dataset scope, policy diff |
| Conftest run | B3 | `/conftest/runs` | Yes (submit) | repo/ruleset |
| Evidence set | C1 | `/evidence/{controlId}` | No (read) | control_id, scope |
| Audit event | C2 | `/audit/events` | Append-only ingest (internal) + read | full §13 schema |
| Violation | C3 | `/violations` | Acknowledge/triage | control_id, severity, scope |
| Report | C5 | `/reports` | Yes (generate) | category, scope, period |
| Approval request | D3 | `/approvals` | Yes (request/decide) | control_id, approver, status |
| Exception | C/D | `/exceptions` | Yes (create/approve) | control_id, scope, expiry |
| Authorization subject (me) | D1/D2 | `/whoami` | No (read) | full subject (§17A.4) |
| Plugin / adapter registry | F2 | `/plugins` | Yes (admin) | kind, version, status |

Each resource object carries the §17A.5 authorization metadata block (`cluster`, `namespaces`, `policy_domains`, `control_ids`, `tenant`, `created_by`, `visibility`) so the storage-layer predicate and the API predicate are identical.

---

## 3. Endpoint catalog (full surface)

Conventions: `{...}` path params; all list endpoints accept the common query params in §5. **[MVP]** = in the §21.1 seed set or directly required to make it usable; **[DEF]** = deferrable to post-MVP.

### 3.1 Controls & lineage (A1)
- `GET /v1/controls` **[MVP]** — list controls; filters: `policy_domain`, `framework`, `cluster`, `namespace`, `has_policy`, `q`.
- `GET /v1/controls/{id}` **[MVP]** — control detail incl. mapped policies, evidence summary, coverage status.
- `GET /v1/controls/{id}/policies` **[DEF]** — policies implementing a control.
- `GET /v1/lineage` **[DEF]** — graph query: `?from=control:SC-IMG-001&depth=2&edge=implemented_by`. Returns nodes+edges (the §16 graph view, Wedge-2 lineage).
- `GET /v1/lineage/{nodeId}` **[DEF]** — single node + adjacent edges.

### 3.2 Policies & engine bindings (B1/B4/A2)
- `GET /v1/policies` **[MVP]**, `GET /v1/policies/{ref}` **[MVP]**.
- `GET /v1/rego-packages` **[MVP]** — alias/projection of policies where `engine=opa`; kept for §21.1 literal compatibility.
- `GET /v1/policies/{ref}/versions` **[DEF]**, `GET /v1/policies/{ref}/bundle` **[DEF]** — returns signed bundle digest + signature metadata (§23 policy integrity; signing impl unspecified).
- `POST /v1/policies/{ref}/lifecycle:promote` **[MVP-ish]** — body `{target_mode, justification, approval_ref?}`; transitions draft→dry-run→warn→enforce (§9.2). Requires `policy:promote-dry-run` / `policy:promote-enforce`.
- `POST /v1/policies/{ref}/lifecycle:demote` **[DEF]** — roll back a mode (e.g. behavioral re-tighten in F4).
- `GET /v1/engine-bindings` **[DEF]**, `PUT /v1/engine-bindings/{id}` **[DEF]** — the §17C action-to-engine matrix as live config.

### 3.3 Simulation (E1) — async job
- `POST /v1/simulate` **[MVP]** — submit. Body: `{policy_diff | policy_ref+version, dataset_scope:{cluster,namespace[],control_ids[],time_window}, mode}`. Returns `202` + `{job_id, status:"queued", _links:{status, result}}`.
- `GET /v1/simulations/{job_id}` **[MVP]** — status + result-when-ready. `GET /v1/simulations/{job_id}/result` **[DEF]** — full differential classification (newly_blocked / newly_allowed / unchanged, §17.4).
- `GET /v1/simulations/{job_id}/events` **[DEF]** — SSE stream of progress for the console.
- `GET /v1/simulations` **[DEF]** — list prior runs (scoped).

### 3.4 Conftest (B3) — async job
- `POST /v1/conftest/run` **[MVP]** — `{source_ref|inline_files, ruleset_ref}` → `202 {job_id}`.
- `GET /v1/conftest/runs/{job_id}` **[MVP]**.

### 3.5 Evidence, audit, violations (C1/C2/C3)
- `GET /v1/evidence/{controlId}` **[MVP]** — evidence set for a control over an optional `?period=`.
- `GET /v1/audit/events` **[MVP]** — query the replay-capable store; filters in §6. Cursor-paginated; bounded windows per §22.
- `GET /v1/audit/events/{event_id}` **[DEF]**, `GET /v1/audit/events:export` **[DEF]** — bounded export job (§22.2 "bounded datasets").
- `GET /v1/violations` **[MVP]**, `GET /v1/violations/{id}` **[DEF]**, `POST /v1/violations/{id}:acknowledge` **[DEF]**.

### 3.6 Reports (C5) — async job
- `POST /v1/reports` **[DEF]** — `{category, scope, period}` → `202 {report_id}`.
- `GET /v1/reports/{id}` **[DEF]** — status/metadata; `GET /v1/reports/{id}/content` **[DEF]** — rendered (HTML/PDF/JSON).

### 3.7 Approvals & exceptions (D3)
- `POST /v1/approvals` **[DEF]** — create `PolicyApprovalRequest` (§17C.6); body mirrors §17B.3 webhook schema.
- `GET /v1/approvals` **[DEF]**, `GET /v1/approvals/{id}` **[DEF]**, `POST /v1/approvals/{id}:decide` **[DEF]** — `{decision:approve|reject, justification}` (requires `approval:approve`).
- `POST /v1/exceptions` **[DEF]**, `POST /v1/exceptions/{id}:approve` **[DEF]**.

### 3.8 Identity, plugins, system (D1/F2)
- `GET /v1/whoami` **[MVP]** — resolved §17A.4 authorization subject for the bearer token (lets the console scope its UI).
- `GET /v1/plugins` **[DEF]**, `POST /v1/plugins` **[DEF]** — register an extensibility plugin (§25); admin-only.
- `GET /v1/healthz`, `GET /v1/readyz`, `GET /v1/version` **[MVP]** — unauthenticated liveness/readiness/build info.
- `GET /v1/openapi.json` **[MVP]** — machine-readable contract.

---

## 4. Authentication & authorization

### 4.1 AuthN
- **R-F1-AUTHN-1 (MUST):** All `/v1/*` endpoints except `/healthz`, `/readyz`, `/version`, `/openapi.json` require a valid OIDC/JWT bearer token (§23 Identity; Keycloak as initial IdP, but the JWT-mapping layer §15.4 is the real contract — Okta/Entra/Auth0/Cognito/SPIFFE accepted per the positioning memo's "Keycloak is one of many" pivot).
- **R-F1-AUTHN-2 (MUST):** Validate `iss`, `aud`, `exp`, `iat` (§15.2). Reject with `401` + `WWW-Authenticate: Bearer error="invalid_token"`.
- **R-F1-AUTHN-3 (SHOULD):** Service-to-service (e.g. console BFF → API) MAY use mTLS + workload-identity JWT (§15.3 `workload_identity`); TLS required in deployed environments (§23 Transport).

### 4.2 AuthZ — the binding to D2 storage authz
- **R-F1-AUTHZ-1 (MUST):** The API resolves the JWT into the internal authorization subject (§17A.4: `roles, groups, namespaces, policy_domains, tenants`) via the §15.4 mapping layer before any handler runs.
- **R-F1-AUTHZ-2 (MUST):** Every endpoint declares the required **permission primitive** (§17A.3: `policy:view`, `policy:promote-enforce`, `audit:replay`, `approval:approve`, …). Missing permission → `403`.
- **R-F1-AUTHZ-3 (MUST — scope predicate):** List/read endpoints push the subject scope into the storage query (D2), not into post-filtering. A namespace-scoped user MUST NOT retrieve out-of-scope policies/violations/simulations/audit fixtures (§17A.5). The API and storage apply the **same** predicate; the API never widens it.
- **R-F1-AUTHZ-4 (MUST):** Global-admin cross-tenant/cross-namespace reads are permitted **but audited** (§17A.5 "global admins must be auditable when crossing boundaries"). The audit event records `boundary_crossed: true`.
- **R-F1-AUTHZ-5 (SHOULD):** `403` vs `404` — for objects the subject may not even know exist (cross-tenant), return `404` to avoid scope enumeration; for in-scope-but-insufficient-permission, return `403`.

### 4.3 Permission ↔ endpoint matrix (excerpt)

| Endpoint | Permission | Scope check |
|---|---|---|
| `GET /controls` | `policy:view` | domain/namespace filter |
| `POST /simulate` | `policy:simulate` | dataset must be in subject scope; out-of-scope dataset → 403 |
| `POST /policies/{ref}/lifecycle:promote` (enforce) | `policy:promote-enforce` | namespace + domain |
| `GET /audit/events` | `audit:replay` | namespace/tenant; bounded window |
| `POST /approvals/{id}:decide` | `approval:approve` | approver role/group match |
| `POST /reports` | `report:view` (+ generate) | scope of report |
| `POST /plugins` | platform-admin | global |

---

## 5. Common request/response envelope, pagination, projection

- **R-F1-ENV-1 (MUST):** JSON over HTTPS; `Content-Type: application/json`. gRPC is an optional parallel transport (DEFERRED) for high-volume audit ingest.
- **Collection envelope:**
  ```json
  { "items": [ ... ], "page": { "next_cursor": "opaque", "limit": 50 }, "scope_applied": {"namespaces":["payments-prod"], "tenant":"payments"} }
  ```
  `scope_applied` echoes the effective authz predicate so the console can show "you are seeing a filtered view."
- **R-F1-PAGE-1 (MUST — cursor pagination):** Opaque, signed cursors; `?limit=` (default 50, max 500). No offset pagination on audit/violations (unbounded). Cursors encode the scope so they cannot be replayed to widen scope.
- **R-F1-PROJ-1 (SHOULD):** `?fields=` projection and `?expand=` (e.g. `expand=policies` on a control) to serve §16 views without N+1 calls.
- **R-F1-SORT-1 (SHOULD):** `?sort=` whitelisted per resource (e.g. `audit/events` by `timestamp`).

---

## 6. Filtering model (consistent across list endpoints)

Common filters, all AND-combined, all **intersected with the authz scope** (a filter can never widen scope):

`cluster`, `namespace` (repeatable), `tenant`, `policy_domain`, `control_id`, `policy_ref`, `policy_engine`, `decision` (allow/deny/warn/mutate/suspend_pending_approval), `event_type`, `from`/`to` (RFC3339 time window), `severity`, `q` (free-text where supported).

- **R-F1-FILTER-1 (MUST):** `/audit/events` requires a bounded time window (`from`/`to`); if absent, default to the configured replay window (7–30 days, §22.1) — never unbounded.
- **R-F1-FILTER-2 (MUST):** Unknown filter keys → `400` (fail closed, prevents silent scope mistakes).

---

## 7. Versioning, idempotency, rate limiting, errors

- **R-F1-VER-1 (MUST):** URL-path major version `/v1`. Breaking changes → `/v2`. Additive fields are non-breaking; clients MUST ignore unknown fields.
- **R-F1-VER-2 (SHOULD):** Deprecations surfaced via `Deprecation` + `Sunset` headers and in `/openapi.json`.
- **R-F1-IDEM-1 (MUST):** All `POST` that create jobs/objects accept an `Idempotency-Key` header; replays within a TTL return the original `job_id`/object (critical for simulate/conftest/report submit and approval decisions).
- **R-F1-RATE-1 (SHOULD):** Per-subject token-bucket rate limiting; `429` + `Retry-After`. POC targets (§22: 5–50 GUI users, 100–1000 evals/sec) are modest; limits protect the simulation/report job queue, not the eval hot path.
- **R-F1-ERR-1 (MUST — problem+json, RFC 9457):**
  ```json
  { "type":"https://docs/errors/scope-violation", "title":"Out of scope", "status":403,
    "detail":"namespace 'payments-prod' not in subject scope", "correlation_id":"uuid" }
  ```
  Every error carries `correlation_id` linking to the audit/observability trace.

---

## 8. Async job lifecycle (simulate / conftest / report / export)

State machine: `queued → running → (succeeded | failed | partial | cancelled)`.
- **R-F1-JOB-1 (MUST):** Submit returns `202` + `job_id` + `_links{status,result,events,cancel}`.
- **R-F1-JOB-2 (MUST):** `partial` is a first-class terminal state for simulation over large datasets that exceed interactive bounds (§22.2 "supported only for small POC datasets") — result includes `replay_completeness` propagated from §13.
- **R-F1-JOB-3 (SHOULD):** SSE/long-poll for progress; jobs are scope-tagged objects subject to the same authz as any other resource (you can only see your jobs / in-scope jobs).
- **R-F1-JOB-4 (MUST):** `DELETE /v1/{collection}/{job_id}` cancels; cancellation is auditable.

---

## 9. Audit & observability of the API tier itself

- **R-F1-OBS-1 (MUST):** Every mutating call and every approval/promotion emits a §13 replay-capable event with `event_type=api.<verb>`, `subject`, `jwt_claims` (subset), `correlation_id`, `before_state`/`after_state` where applicable (§23 Auditability).
- **R-F1-OBS-2 (SHOULD):** Distributed tracing (OTel); `correlation_id` == trace id where possible. Reuses the same OTel pipeline F4 leans on for GenAI traces.
- **R-F1-OBS-3 (MUST):** Auth failures and scope violations are logged (security event) but MUST NOT leak existence of out-of-scope objects in the message.

---

## 10. Failure modes

| Failure | API behavior |
|---|---|
| Backing component (e.g. E1 simulator) down | `503` problem+json, `Retry-After`; job submit may still `202` and queue |
| JWT mapping layer can't resolve required claim (e.g. missing `tenant`, the §14.2 drift case) | `403` + `outcome_reason="required claim 'tenant' absent"`; emit JWT-drift audit event for C3 |
| Storage scope predicate fails open risk | MUST fail closed: deny the read, `503`, alert |
| Simulation dataset exceeds POC bounds | `202` then terminal `partial` with `replay_completeness:"partial"` |
| Signed-bundle signature missing/invalid on read | bundle metadata returns `signature_status:"unverified"`; never silently treat as verified (§23 integrity) |

---

## 11. Dependencies on other components

- **D1/D2 (identity & storage authz):** the §17A subject, the §15.4 mapping layer, and the storage-side scope predicate. F1 is a thin enforcement facade over D2; **the API MUST NOT be the only enforcement point.**
- **C2 (audit schema):** every list/read of `/audit/events` returns the §13 schema verbatim; F4 deltas (evaluator_results, trace context) surface here once C2 adds them.
- **C3/C5 (analytics/reports):** violations, reports.
- **A1/A2 (governance/lifecycle):** controls, lineage, promotion.
- **B1/B3/B4 (engines):** policies, rego-packages, conftest, engine bindings.
- **E1 (simulation):** the async simulate job.
- **F2 (plugins):** `/plugins` registry; new PDPs/adapters surface new sub-resources via the plugin contract.
- **F4 (AI):** adds agent resources (sessions, traces, evaluator results, capability tokens) as new sub-collections under the same envelope/authz — see F4 SPEC §"API deltas."

---

## 12. Open questions — decided defaults

| # | Question | Decided default | Rationale |
|---|---|---|---|
| OQ-F1-1 | REST vs GraphQL for lineage | **REST for MVP**, GraphQL/Cypher-ish `/lineage` query param later | §21.1 is REST; lineage graph (Wedge-2) is the one place GraphQL earns its keep — defer. |
| OQ-F1-2 | `/rego-packages` separate vs projection of `/policies` | **Projection** (`engine=opa`) with the literal `/rego-packages` path kept for §21.1 compatibility | Avoids two sources of truth. |
| OQ-F1-3 | Sync vs async simulate | **Async (202+job)** with interactive fast-path when dataset is small | §22.2 explicitly allows "short background job"; large replay must not block. |
| OQ-F1-4 | 403 vs 404 for out-of-scope | **404 cross-tenant, 403 in-scope-no-perm** | Prevents scope enumeration (§17A.5). |
| OQ-F1-5 | API the sole authz point? | **No — storage co-enforces** | §17A.1 "GUI-only authorization is insufficient"; same predicate both layers. |
| OQ-F1-6 | gRPC | **Deferred**, REST first | POC scale is modest; gRPC only for audit ingest if needed. |
| OQ-F1-7 | Write controls via API? | **No — GitOps/A2 authoring**, API is promote/read | Keeps governance-as-code; controls are versioned artifacts. |

---

## 13. Normative requirements summary (numbered)

R-F1-AUTHN-1..3, R-F1-AUTHZ-1..5, R-F1-ENV-1, R-F1-PAGE-1, R-F1-PROJ-1, R-F1-SORT-1, R-F1-FILTER-1..2, R-F1-VER-1..2, R-F1-IDEM-1, R-F1-RATE-1, R-F1-ERR-1, R-F1-JOB-1..4, R-F1-OBS-1..3. (See inline.) The MUST set is the conformance floor for the API tier.
