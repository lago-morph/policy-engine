# Inter-Domain Contracts — Frozen Source of Truth

**Status:** NORMATIVE · **Scope:** all 23 components across domains A–F · **Date:** 2026-05-30
**Owner of this document:** cross-cutting (inter-domain contract editor)

## 0. Purpose & how to read this

The 23 components were spec'd in parallel. They agree on six shared contracts but
expressed them in scattered, occasionally conflicting places. This document **freezes**
each shared contract in one authoritative place so every component builds against a single
source of truth. Where component specs disagree, this document picks the authoritative
version, states it, and records the override. **If a component SPEC and this document
conflict, this document wins** for the inter-domain surface (component-internal detail
still lives in the component SPEC).

Each contract below is stated as: **Owner · Consumers · Frozen interface · Stability rule ·
Open question + decided default · Overrides**.

Global conventions used throughout:
- **Additive-only within a major version** means: no field/enum-value may be removed,
  renamed, or repurposed; new optional fields and new enum values may be added; consumers
  **MUST ignore unknown fields and tolerate unknown enum values** (treat unknown enums as
  their nearest known supertype or as opaque).
- Requirement-code prefixes (`N-C2-*`, `D2-R*`, `R-B4-*`, `B5-R*`, `D1-R*`, `D-E1-*`) are
  the originating component's; this document references them rather than renumbering.

### 0.1 Contract index

| # | Contract | Owner | Primary consumers |
|---|---|---|---|
| 1 | Audit Event Contract (`c2.audit-event/1.0`) | **C2** | C1, C3, C4, C5, E1, F1, F2, F4 |
| 2 | Identity / Claims Contract | **D1** | C2 (`jwt_claims`), D2, D3, every PDP |
| 3 | Scope / Authorization Contract (`ScopePredicate`) | **D2** | C2, C3, C5, E1, F1, every storage query |
| 4 | CRD Contract (schema vs controller vs REST) | **B4** (schema) | F2 (controllers), F1 (REST), E1, D3 |
| 5 | Action-Model Contract (disposition vs obligations) | **B4** | C2, E1, E3, C3, B5 |
| 6 | `correlation_id` / replay Contract | **B5** (mint) + **C2** (state machine) | C2, C4, E1, D3, B4, C3 |

### 0.2 Cross-cutting overrides made by this document (summary)

| OV | Where the disagreement was | Authoritative resolution |
|---|---|---|
| **OV-1** | C2 freezes middle replay state as **`best_effort`**; E1, B4, F-domain text still say **`partial`** | **`best_effort`** is canonical (C2 D-C2-01). `partial` is a **deprecated ingest/query alias only**. E1's `ReplayEventV1` adapter (D-E1-03) maps `partial→best_effort`. |
| **OV-2** | F2 DOMAIN-SUMMARY says "F2 owns the in-cluster CRD API"; B4 defines the CRD schemas | **B4 owns the schema**, **F2 owns the controllers/operator**, **F1 owns the REST projection** (F-domain decision §38, formalized in Contract 4). |
| **OV-3** | B4 §4 conflates *disposition* and *action/obligation* ("action=require_approval, disposition=deny-pending-approval" stated inline but not modeled as separate fields) | Disposition and obligations are **separate dimensions** with separate field homes in C2 (Contract 5). |
| **OV-4** | B5-R1 says `correlation_id = AdmissionReview UID` minted earliest; D3 says "the CR name doubles as correlation_id" because an approval **retry mints a NEW AdmissionReview UID** | The **logical-flow `correlation_id` is stable across the approval retry**; the per-admission UID is recorded separately. Resolved in Contract 6. |
| **OV-5** | E1 §2.2/§7 lists required replay fields against "spec §13.3"; C2 froze 36 named fields | C2's 36-field list (Contract 1) is authoritative; E1 consumes it through its adapter. |

---

## 1. Audit Event Contract — `c2.audit-event/1.0`

**Owner:** **C2** (`spec-plan/components/C2-audit-schema/SPEC.md`). This is the **keystone**
contract; schema freeze is the gating milestone for all of Domain C and for simulation.

**Consumers:** C1 (Privateer), C3 (analytics), C4 (retrospective detection), C5 (reporting),
E1 (simulation), F1 (`/audit/events` REST), F2 (every plugin/CRD emits it), F4 (AI extension,
additive agent fields). D1 and D2 **produce into** it (`jwt_claims`, `authz_denied`/
`boundary_crossing` events).

### 1.1 The frozen interface

A C2 event is a single normalized, immutable JSON document. The contract is the **36
top-level fields** frozen in C2 §3.13 (do not restate them here field-by-field — C2 §3 is the
authoritative field table; this is the binding pointer):

```
schema_version = "c2.audit-event/1.0"

R  (14, present on every event):
   schema_version, event_id, correlation_id, timestamp, ingest_timestamp,
   event_type, producer, subject, scope, replay_completeness,
   source, content_hash, prev_hash, chain_seq
C  (18, conditionally required per field rule):
   decision, policy_engine, policy_version, policy_ref, control_id,
   outcome_reason, mutation_diff, jwt_claims, jwt_claims_completeness,
   operation, resource_type, resource_id, before_state, after_state,
   request_object, external_data_refs, replay_completeness_reasons,
   confidence_level
O  (4):
   parent_correlation_id, action_performed, engine_context, signature
```

(14 R + 18 C + 4 O = **36**.) Authoritative field-by-field semantics, types, and the
conditional rules live in **C2 §3.1–§3.6**; the canonical examples in **C2 §3.8–§3.10**.

### 1.2 Binding invariants (frozen — every producer and consumer MUST honor)

1. **`replay_completeness` enum is `complete | best_effort | insufficient`** (C2 §3.7,
   D-C2-01). This is the record's honesty label and is **required on every event**.
   `partial` is a **deprecated alias** of `best_effort`, accepted on ingest and as a query
   synonym only (see Contract 6 for the state machine and **OV-1**).
2. **`correlation_id` anchor = Kubernetes AdmissionReview UID** where applicable
   (N-C2-200/201/202, D-C2-03). OPA MUST echo the AdmissionReview UID into `correlation_id`
   (keeping its own `decision_id` in `engine_context.opa.decision_id`). See Contract 6 for the
   minting rule and the approval-retry exception (**OV-4**).
3. **Hash-chain integrity** (N-C2-300/301/302): each `source.system` maintains an
   append-only chain — `prev_hash`(N)=`content_hash`(N−1), strictly-monotonic `chain_seq`,
   RFC 8785 JCS canonicalization before `sha256`. A broken link or `chain_seq` gap is
   detectable tampering and is surfaced by C3 as a **critical** `chain_integrity` finding
   (C3 D-C3-06). The store is **append-only**: no update-in-place, no hard delete inside the
   retention window; corrections are new appended events (supersession via side index).
4. **One signing primitive** (N-C2-302/304, D-C2-04): **ed25519** signed Merkle checkpoints
   (default cadence N=10 000 events / T=15 min) and one signed-export envelope format
   (`manifest.json` + per-control NDJSON + `merkle.json` + detached signature). Per-event
   signing is optional; **checkpoint signing is mandatory**. C5/C1 exports MUST use this one
   envelope so every export across the platform is verifiable with only the published public
   key.
5. **Additive-only within v1.x** (N-C2-FWD): no frozen field is removed or repurposed; new
   optional fields and new enum values are additive; the `replay_completeness_reasons`
   vocabulary (C2 §5.5) is additive-only.
6. **Consumers ignore unknown fields and tolerate unknown enum values** (N-C2-FWD).
7. **No silent promotion** (N-C2-NOPROMOTE, N-C2-SYNTH): an `insufficient` event is never
   promoted to a verdict; a `replay.synthetic` event is capped at `best_effort` and MUST NOT
   carry `replay_completeness=complete`.

### 1.3 Stability rule
Schema-**versioned** with **additive-only minor bumps** (`1.0 → 1.1` for new optional
fields). Required-field changes or semantic changes require a **major bump + migration plan**
(C2 §3.13, PLAN M-FREEZE). v1.0 is **FROZEN**.

### 1.4 Open question + decided default
- **OQ:** Physical store backend? **Default:** logical contract is backend-agnostic; PLAN
  recommends an append-only log (object-store segments + index DB) over a single RDBMS so
  tamper-evidence is natural (C2 OQ-1). Revisit at scale.
- **OQ:** OCSF relationship? **Default (D-C2-02):** the custom C2 schema is **authoritative**;
  OCSF is a **one-way export** only. There is **no** OCSF→C2 import that yields `complete`
  (N-C2-501) — ingesting a SIEM does not grant replay capability.

### 1.5 Overrides
- **OV-1** (the `best_effort`/`partial` naming) and **OV-5** (E1 pins to this 36-field list
  via its adapter) apply here. See §6 for the state machine.

---

## 2. Identity / Claims Contract

**Owner:** **D1** (`spec-plan/components/D1-keycloak-jwt/SPEC.md`). D1 is the single identity
ingestion edge: raw OIDC/JWT → one **canonical normalized subject**.

**Consumers:** C2 (`jwt_claims` + `jwt_claims_completeness`), D2 (role/scope derivation and
storage authz), D3 (approval subjects/approvers), every PDP (`input.subject`).

### 2.1 The frozen interface — two distinct outputs

D1 emits **two** related but distinct artifacts; do not conflate them:

**(a) The canonical normalized subject** (`subject/v3`, D1 §2.1) — the *normalized*
authorization subject every downstream authz consumer references. Frozen shape (key fields):

```
subject_id, subject_type(human|service_account|workload), username, email, issuer,
tenants[], namespaces[], policy_domains[], roles[] (expanded, "role:<x>"),
groups[], environment, risk_level, compliance_scope[], data_classification,
workload_identity, deployment_approval (normalized only — D3 owns semantics),
claim_provenance{}, token_ref{jti,iat,exp,aud}, normalization_status(complete|degraded|incomplete)
```

**Load-bearing tenet (D1 §1.3):** no Rego rule, Gatekeeper constraint, or storage query may
reference a **raw upstream claim path** (e.g. `token.org_id`). Everything downstream
references the canonical subject only. CI lint rejects raw-claim references (D1-R8).

**(b) The `jwt_claims` block into the C2 audit event** (D1 §3, feeds Contract 1 fields
#18/#19). This is the **verbatim** claim set used as policy input (NOT the normalized
subject) so replay is faithful (C2 §3.3, §17.3). It populates:
- `jwt_claims` (C2 field #18): the *raw* claims the policy consulted, redaction-aware
  (C2 §7.5 — redacted values become a salted `value_digest`, never silently dropped).
- `jwt_claims_completeness` (C2 field #19): **`full | partial | reconstructed`** — distinct
  from `replay_completeness`. `reconstructed` = claims rebuilt from a Keycloak token-issuance
  event joined by `sub`+time, not captured at decision time; this feeds `replay_completeness`
  (a `reconstructed` claim set caps the event at `best_effort`).

> **Naming clarification (frozen):** `jwt_claims_completeness=partial` is a **different
> enum** from the replay `partial`/`best_effort`. The collision between these two `partial`
> meanings is precisely why C2 renamed the *replay* middle state to `best_effort`
> (D-C2-01 / **OV-1**). `jwt_claims_completeness` keeps `partial` unchanged.

### 2.2 D2 role/scope derivation (the D1→D2 hand-off)
- D1's expanded `roles[]` (`role:<group>`, group-hierarchy expanded at normalization time per
  DT-38) MUST each resolve to a registered role in **D2's role registry**
  (`{role_id, scope_level, granted_permissions[], implied_roles[]}`, D2 §2). This is the
  D1↔D2 contract: every emitted `role:<x>` is registered with scope + permission primitives.
- D1's `{tenants[], namespaces[], policy_domains[]}` (+ `clusters[]`, `*` for global) is the
  **subject scope** D2's `scope_match` consumes (Contract 3).
- **Fail-closed coupling (D1-R5):** a claim required by any deployed policy (per the Rego
  `__required_claims__` index) but absent ⇒ `normalization_status != complete` +
  `missing_required` entry. D2/PDP MUST honor an `incomplete`/`degraded` subject fail-closed
  (deny / empty filtered set) for any policy needing the missing claim.
- **Determinism (D1-R6):** same token + same mapping version ⇒ byte-identical subject (modulo
  `token_ref.iat`). Required for replay and audit faithfulness.

### 2.3 Stability rule
The canonical subject is **explicitly versioned** (`schema_version: subject/v3`): **additive
fields bump minor, removals bump major** (D1 OQ-4). The mapping configuration is a versioned,
audited governance artifact (a mapping edit *is* a privilege change — D1 §8). `jwt_claims` in
the audit event follows the **C2 additive-only** rule (Contract 1).

### 2.4 Open question + decided default
- **OQ:** Group→role expansion at token issuance or normalization? **Default
  (D1 OQ-1):** normalization-time; issuance-time only as a token-size optimization.
- **OQ:** Missing claim → absent or sentinel? **Default (D1 OQ-2):** sentinel `unknown` for
  value-bearing claims; `missing_required` flag for required ones (fail-closed without
  ambiguity).
- **OQ:** Is the `deployment_approval` claim authoritative for approval state? **Default
  (D1 OQ-6 / D3):** **No** — D3's CRD/approval store is authoritative; the claim is a cache
  hint at most (avoids stale-token approval bypass).

### 2.5 Overrides
- **OV-1** governs the two `partial` meanings (see §2.1 clarification box). No other override.

---

## 3. Scope / Authorization Contract — the `ScopePredicate`

**Owner:** **D2** (`spec-plan/components/D2-scoped-rbac-storage/SPEC.md`). D2 owns authorization
end-to-end and **enforces it at the storage layer, not the GUI/API**.

**Consumers:** C2 (scope-filters every query — N-C2-401), C3 (analytics aggregate reads),
C5 (reporting + redaction export), E1 (every evidence query), F1 (REST list/read push the
predicate into storage, not post-filtering — R-F1-AUTHZ-3), D3, and **every storage query in
the platform**.

### 3.1 The load-bearing invariant (frozen)
> **Every read and write of a platform-stored object is authorized server-side against the
> subject's scope, independent of the caller (Console, curl, CI job, custom MCP client). The
> GUI is a convenience, never a control. A scope escape is a P0.** (D2 §1.3.)

### 3.2 The frozen `ScopePredicate` interface
Every storage query MUST pass through the **single authz interceptor choke point** (D2 §5.2,
D2-R3) which derives a scope predicate from the normalized subject (Contract 2) and **rewrites
the query** (mandatory rewrite, never an application-level `if`):

```
authorize(subject, verb, object) :=
      role_grants_verb(subject.roles, verb, object.type)      # role layer (9-role × verb matrix, D2 §4)
  AND scope_match(subject.scope, object.scope)                # scope layer
  AND not denied_by_explicit_deny(subject, verb, object)      # deny overrides

scope_match :=  (object.tenant ∈ subject.tenants OR subject.tenants = ["*"])
            AND (object.namespaces ∩ subject.namespaces ≠ ∅ OR namespace not applicable)
            AND (object.policy_domains ∩ subject.policy_domains ≠ ∅ OR subject domains = ["*"])
```

**`scope_match` is INTERSECTION, not containment-by-string** (D2 §3.4).

**Scope dimensions (frozen):** `cluster, namespace, tenant, policy_domain, control_id`. The
three primary axes called out for storage filtering are **cluster / namespace / tenant**
(C2 `scope` field #23 carries `{cluster, namespace, tenant, region, environment}`; D2 filters
on `tenant ∩, namespaces ∩, policy_domains ∩`).

**Every stored object carries an immutable `authz` block** (D2 §5.1, D2-R1):
`{object_type, cluster, namespaces[], policy_domains[], control_ids[], tenant, created_by,
visibility(namespace-scoped|tenant-scoped|global)}`, set at write time from the authoring
subject's scope and **immutable thereafter**. Writes outside the author's scope are rejected
(D2-R2).

### 3.3 Binding rules (frozen — MUST)
- **D2-R3:** No code path from API handler to storage bypasses the rewrite (architecturally
  enforced; lint/test forbids direct storage access).
- **D2-R4:** Explicit out-of-scope filter (e.g. `?namespace=billing` for a payments subject)
  → **403, no row data, no existence signal**.
- **D2-R5 (the analytics/reporting rule — load-bearing here):** an *unfiltered* query has the
  predicate applied **implicitly**, AND **counts, aggregates, and pagination cursors are
  computed over the FILTERED set**. **Analytics (C3) and reporting (C5) aggregate reads MUST
  also pass through the ScopePredicate** — an aggregate/count/coverage-matrix read is a read,
  and out-of-scope rows must never leak through a total, a histogram bucket, or a cursor.
- **D2-R6:** `/objects/{id}` for an out-of-scope object → **404** (not 403), defeating
  ID-enumeration.
- **D2-R8:** audit-replay datasets are **materialized as scoped datasets before use**
  (Contract 6 / C2 §8.5); Auditor/Analyst replay operates on the materialized snapshot, never
  the live mutable store.
- **D2-R9:** every denied request emits `authz_denied`; every global-subject cross-tenant
  access emits `boundary_crossing` (both are C2 events — *power is logged, not hidden*).

### 3.4 Stability rule
The **predicate is portable** (D2 OQ-1): the POC implements it as an app-layer mandatory
query rewrite at one interceptor; the design keeps the predicate replaceable by DB row-level
security, OPA, or SpiceDB later **without changing the contract surface**. The role × verb
matrix and scope dimensions are **additive** (new roles/verbs/dimensions may be added; none
removed without a major review). Roles are additive within a subject (effective set = union ∩
scope).

### 3.5 Open question + decided default
- **OQ:** Enforcement mechanism — app rewrite / RLS / external authz? **Default (D2 OQ-1):**
  app-layer mandatory rewrite at a single interceptor for the POC; portable predicate.
- **OQ:** 403 vs 404 for out-of-scope by ID? **Default (D2 OQ-2):** **404** (no existence
  signal).
- **OQ:** Counts/cursors over full or filtered set? **Default (D2 OQ-3):** **filtered set**.
- **OQ:** Boundary-crossing audit blocking or best-effort? **Default (D2 OQ-4):** **blocking
  for global subjects** (audit is a precondition for the cross-tenant read).
- **OQ:** Where does scope live? **Default (D2 OQ-6):** server-side grant store is
  authoritative; token scope is an assertion validated against it for privileged ops
  (mitigates stale-token scope).

### 3.6 Overrides
None against other domains. D2-R5 is **emphasized as binding on C3/C5/E1 aggregate reads** —
this was implicit in those specs (E1 §9 honors it; C3/C5 inherit it) and is hereby made
explicit.

---

## 4. CRD Contract — schema vs controller vs REST

**Owner of the schema:** **B4** (`spec-plan/components/B4-engine-selection-crds/SPEC.md`).
**Owner of the controllers/operator:** **F2**. **Owner of the REST projection:** **F1**.

This resolves the **B4 / F2 / F1 ownership collision** (F2 DOMAIN-SUMMARY §38 flagged it: both
B4 and F2 define the §17C.6 CRD surface). **Frozen split (OV-2):**

| Layer | Owner | Authority |
|---|---|---|
| CRD **OpenAPI schema** + state machines | **B4** | The schema is the contract; B4 §5 is authoritative for field shapes & reconcile semantics |
| CRD **controllers / operator** (reconcile loops, leader election, idempotency) | **F2** | One `governance-operator`, many controllers (F2 §, R-F2-OP-1) |
| **REST projection** of CRDs to the external API | **F1** | `/approvals`, `/exceptions`, `/simulations`, `/engine-bindings`, etc. (F1 §3) |

### 4.1 The frozen CRD list (group `governance.example.io/v1alpha1`)

| CRD | Schema owner | Controller (F2) | REST (F1) | Notes |
|---|---|---|---|---|
| `PolicyApprovalRequest` | **B4** §5.1 | Approval controller → webhook | `/approvals` | load-bearing; deny-with-approval-required (Contract 5/6) |
| `PolicyException` | **B4** §5.2 | Exception controller | `/exceptions` | bounded + scoped; expiry flips back to enforce |
| `PolicySimulationRun` | **B4** §5.3 / **E1** §8 (shared) | Simulation controller (E1-owned reconcile logic, F2-operated) | `/simulations` | E1 owns the reconcile semantics; B4 owns the catalog entry |
| `PolicyActionLibrary` | **B4** §5.4 | Library controller | (via `/engine-bindings`) | data CRD; feeds E3/§17D |
| `PolicyEvidenceSchema` | **B4** §5.4 | Schema controller | — | data CRD; declares per-PDP replay schema → C2/C4 |
| `PolicyRemediationAction` | **B4** §5.4 | Remediation controller | — | idempotent; dry-run/propose default for destructive actions |
| `GovernanceBundle` (new) | **B4**-aligned, **F2** §-introduced | Bundle controller | `/policies/{ref}/bundle` (F1) | signed bundle + Gemara/OSCAL control metadata + required JWT claims |
| `PolicyEnginePlugin` (new) | **F2** §-introduced | Plugin controller | `/engine-bindings` | registers an engine/PDP plugin (§25) |
| `ExportAdapter` (new) | **F2** §-introduced | Adapter controller | — | registers a SIEM/GRC export sink |

**Resolution of the three "new" CRDs:** `GovernanceBundle`, `PolicyEnginePlugin`,
`ExportAdapter` were introduced by F2 (deployment/extensibility) and have no B4 §17C.6 entry.
**Frozen:** F2 owns both their schema and controller (they are deployment/extensibility
objects, not action/engine-selection objects); B4's authoritative-schema ownership covers
**only the six §17C.6 CRDs**. If any of the three later needs an action/engine-selection
field, that field's schema is co-owned and reconciled at cross-cut.

### 4.2 Cross-cutting CRD rules (frozen — MUST, apply to all nine)
- **R-F2-CRD-1 / R-B4-24:** every CRD carries the §17A.5 **authz scope metadata** (Contract 3)
  so controllers and storage enforce scope uniformly; CRD writes are RBAC-scoped (D2).
- **R-F2-CRD-2 / R-B4-23:** controllers are **idempotent, leader-elected**, and **emit a C2
  audit event with `correlation_id` on every reconcile** that changes enforcement-relevant
  state.
- **R-B4-22 / R-F2-UPGRADE-1:** all are `v1alpha1`; a conversion/upgrade path (conversion
  webhook `v1alpha1→v1beta1`) MUST exist before any becomes a durable contract — **no
  breaking changes to stored objects without conversion**.

### 4.3 Stability rule
**Versioned** Kubernetes-style (`v1alpha1` → `v1beta1` via conversion webhooks). Within a
version, schema changes are additive; stored-object-breaking changes require a conversion
path. The **schema owner (B4 for the six, F2 for the three) is the single source of truth**;
F1's REST projection and F2's controllers MUST NOT introduce divergent field shapes.

### 4.4 Open question + decided default
- **OQ:** Single CRD-schema owner? **Default (OV-2, F-domain §38):** **B4 owns the six
  §17C.6 schemas; F2 owns controllers + the three new CRDs; F1 owns REST.**
- **OQ (F2 OQ-2):** One operator or many? **Default:** one `governance-operator`, multiple
  controllers, splittable later.

### 4.5 Overrides
**OV-2** (this whole section). Also: F1's `/rego-packages` is a **projection** of `/policies`
where `engine=opa` (F1 OQ-F1-2), not a second source of truth.

---

## 5. Action-Model Contract — disposition vs obligations

**Owner:** **B4** (the 13-action taxonomy, B4 §4). **Consumers:** C2 (records both dimensions),
E1 (allow-class/deny-class normalization for differential sim), E3 (per-product action
libraries), C3 (analytics), B5 (computes disposition at admission).

### 5.1 The problem being fixed (OV-3)
B4 §4 states inline that "the audit records both: **action=require_approval, disposition=
deny-pending-approval**" and defines a **precedence chain** over the 13 actions — but it never
cleanly separates the two dimensions. This conflation means a single `decision`/`action` enum
is forced to carry both "what is the verdict?" and "what side-effects/follow-ups attach?",
which breaks down for `require_approval` (verdict = deny, obligation = approval), `mutate`
(verdict = allow, obligation = mutate), and `exception` (verdict = allow, obligation =
record-waiver). **This contract splits them.**

### 5.2 The frozen two-dimension model

**Dimension A — Disposition (mutually exclusive; exactly one per decision).** The terminal
verdict on the request:

```
allow | deny | warn | suspend_pending_approval | require_approval | unknown
```

- Maps to C2 field #9 `decision` (C2 §3.2). Exactly one disposition per `policy.decision`
  event.
- `warn` is allow-class with a recorded warning; `require_approval` / `suspend_pending_approval`
  are **deny-class** at admission time (the request does not proceed) — see Contract 6.
- `unknown` = source recorded an action but not a verdict (e.g. a bypassed deployment, C4).

**Dimension B — Obligations (co-occurring; zero-or-more per decision).** Side-effects /
follow-ups that attach to a disposition. Drawn from the B4 13-action taxonomy, minus the pure
verdicts:

```
audit (always implied) | mutate | generate | cleanup/delete | quarantine | suspend |
require_approval(obligation form) | require_scan | notify | annotate/label | exception
```

- The **concrete effect** is recorded in C2 field #15 `action_performed`
  (`block | mutate | annotate | route_for_approval | log_only`) and, for mutations, field #16
  `mutation_diff` (RFC 6902 JSON Patch).
- Multiple obligations may co-occur (e.g. `allow` + `annotate` + `notify`).

### 5.3 The canonical reconciliation (frozen)
The B4 §4.2 inline pair maps cleanly onto the two dimensions:

| B4 13-action | Disposition (A) | Obligation(s) (B) |
|---|---|---|
| `allow` | allow | — (audit implied) |
| `deny` | deny | — |
| `warn` | allow | warn/notify |
| `mutate` | allow | mutate (+ `mutation_diff`) |
| `generate` | allow | generate |
| `cleanup`/`delete` | (controller-loop verdict) | cleanup |
| `quarantine` | (controller) | quarantine |
| `suspend` | suspend_pending_approval | suspend |
| `require_approval` | **deny** (at admission) | **require_approval** (creates `PolicyApprovalRequest`) |
| `require_scan` | allow/deny per result | require_scan |
| `notify` | (inherits) | notify |
| `annotate`/`label` | allow | annotate |
| `exception` | allow | exception (record waiver; references unexpired in-scope `PolicyException`) |

- **R-B4-5 (frozen):** the obligation vocabulary (the 13-action set) is **closed**: exactly
  these. New effects require a spec change, not ad-hoc actions.
- **R-B4-6 (frozen, the keystone of this split):** `require_approval` at a Kubernetes admission
  PDP is realized as **disposition = deny + obligation = require_approval +
  `PolicyApprovalRequest` CRD**. The action and the admission disposition are **not
  contradictory** — they are two different dimensions, now modeled as such.
- **R-B4-7 (frozen) precedence** (when multiple controls fire on one request, resolving the
  *disposition*): `deny > require_approval > quarantine > mutate/generate/annotate >
  require_scan > warn > exception > allow`. A hard `deny` from any control wins. `exception`
  only converts a would-deny **within its own control+scope**, never an unrelated deny
  (R-B4-17).

### 5.4 E1 consumption (frozen)
E1's differential algorithm normalizes **dispositions** into allow-class vs deny-class
(E1 §4.1, D-E1-01): `allow, warn, mutate` → **allow-class**; `deny,
suspend_pending_approval` → **deny-class**. A *secondary* within-class diff
(`effect_changed_within_class`) tracks obligation changes (e.g. `mutate` diff changes, a
`warn→deny` tightening, a `deny→warn` relaxation) so an obligation change inside one
disposition class is still reviewable. This is exactly why the two dimensions must be separate
fields: a within-class obligation change would be invisible if disposition and obligation were
one enum.

### 5.5 Stability rule
The disposition enum and the obligation (13-action) set are both **closed** and
**additive-only** (a new value is a spec change, never ad-hoc). C2 carries both via existing
frozen fields (`decision`, `action_performed`, `mutation_diff`) — **no new C2 field is
required**, so this clarification is purely semantic and needs no schema bump.

### 5.6 Open question + decided default
- **OQ (B4 OQ1):** Decision and effector separable, or engine owns both? **Default
  (R-B4-2, D-B4-01):** **separable — OPA decides the disposition, an engine effects the
  obligation.** This is what makes the platform engine-portable.
- **OQ (B4 OQ2):** Closed or open action set? **Default:** **closed 13.**

### 5.7 Overrides
**OV-3:** disposition and obligations are now **separate dimensions**, not a single
conflated enum. B4 §4's inline "action vs disposition" prose is hereby promoted to the
normative two-field model above.

---

## 6. `correlation_id` / replay Contract

**Owner:** **B5** owns minting & propagation (`spec-plan/components/B5-realtime-enforcement/
SPEC.md`); **C2** owns the `replay_completeness` state machine (C2 §5); **E1** recomputes
per-bundle at replay time (E1 §7). **Consumers:** C2, C4, E1, D3, B4, C3.

### 6.1 How `correlation_id` is minted (frozen)
- **B5-R1:** a **single `correlation_id`** is generated **server-side at the earliest point**
  of the request (gateway or admission), and propagated **unchanged** through every downstream
  artifact: OPA decision log, Gatekeeper audit event (field 17), Privateer evidence, analytics
  correlation, and any `PolicyApprovalRequest`/`PolicyException`. A flow that loses or
  re-generates it mid-stream is **non-conformant**.
- **Anchor (C2 N-C2-200/201/202, D-C2-03):** for Kubernetes admission, the canonical anchor is
  the **AdmissionReview UID**. OPA MUST **echo** it (not emit its own `decision_id` as the
  join key). Non-admission anchors: CI = `ci:<provider>:<run-id>`; Keycloak = auth session/flow
  id (issuance events joined by `sub`+time when no shared id ⇒ `jwt_claims_completeness=
  reconstructed`); app-embedded PDP = SDK-supplied request id.
- **Mint-and-record (N-C2-203):** if no upstream anchor exists, the normalizer mints a UUIDv7
  and records `correlation_source="minted"` so consumers know it cannot be joined. A minted id
  on a `policy.decision` that *should* have had an anchor is itself a C3 finding.

### 6.2 Preservation across the approval retry — the B5/C4/D3 bug (OV-4, frozen)
**The conflict:** B5-R1 says `correlation_id = AdmissionReview UID` minted at the earliest
point. But an approval flow has **two** admissions for **one logical request**: (1) the
original admission that returns **deny-with-approval-required**, and (2) the **retry**
admission (e.g. Sam re-runs `kubectl apply`) once the `PolicyApprovalRequest` is approved. The
retry mints a **NEW AdmissionReview UID** (Kubernetes generates a fresh UID per request), so
the naive rule would give the deny and the retry-admit **different** `correlation_id`s and
break the link — exactly the trace gap C4 must not have.

**Frozen resolution:**
1. The **logical-flow `correlation_id` is STABLE across the retry**. It is anchored to the
   `PolicyApprovalRequest` identity, not to either AdmissionReview UID. Per D3, **the CR name
   doubles as the logical `correlation_id`**, derived deterministically from
   `(controlId, resourceRef, requestedBy)`, so the original deny and the retry-admit share one
   id.
2. **Each individual admission's AdmissionReview UID is still recorded**, in
   `engine_context.gatekeeper.admission_review_uid` (per-event), and the original is also
   carried in `PolicyApprovalRequest.spec.correlationId`; the consuming retry records its own
   UID in `status.consumedBy.admissionReviewUID` (B4 §5.1). This preserves both the per-event
   UID *and* the stable logical link.
3. `parent_correlation_id` (C2 field #4) MAY link a retry event back to the original deny when
   a single stable id is not used, but the **default and preferred** mechanism is the stable
   `PolicyApprovalRequest`-anchored id (1).
4. **Single-use idempotency (R-B4-13):** one approved request authorizes exactly one admission,
   matched by `(correlationId, resourceRef, subject)`; a second distinct request cannot ride
   the same approval. `consumed`/`expired` requests cannot be reused.

> **Net rule (frozen):** *Within one logical request — including across an approval
> deny→approve→retry cycle — there is exactly ONE `correlation_id`. Per-admission
> AdmissionReview UIDs are recorded as secondary ids in `engine_context`, never as the join
> key.* This supersedes the literal reading of B5-R1 ("correlation_id = AdmissionReview UID")
> for approval-retry flows.

### 6.3 The single `replay_completeness` state machine (frozen — C2 §5)
Three states, **`complete | best_effort | insufficient`** (D-C2-01; `partial` is a deprecated
alias — **OV-1**). Computed deterministically in the normalizer (live) AND recomputed in
E1/C4 (replay); the two MUST agree on the same inputs (determinism test).

```
complete      : request_object/required state captured; jwt_claims=full (if policy uses claims);
                EVERY consulted external_data_refs entry present w/ resolvable version/digest;
                policy_version known & digest-addressable; produced by a REAL engine evaluation
                (event_type != replay.synthetic).  → replay is AUTHORITATIVE.
best_effort   : a defensible replay is possible but fidelity reduced. ANY of:
                event_type=replay.synthetic (capped here, N-C2-SYNTH);
                jwt_claims_completeness ∈ {partial, reconstructed};
                external-data value unavailable but re-resolvable at known version;
                policy_version inferred not recorded.  → INDICATIVE, carries confidence_level
                + replay_completeness_reasons; never in authoritative totals without disclosure.
insufficient  : faithful replay NOT possible. ANY of: policy uses external data and NO ref
                exists & not re-resolvable; request_object/required state absent & unreconstructable;
                policy uses claims & jwt_claims absent & no issuance event; policy_version unknown
                & not inferable.  → FLAGGED not promoted (N-C2-NOPROMOTE); excluded from
                authoritative counts; feeds the "missing audit fields" report; triggers backfill.
```

**Backfill / re-scoring (C2 §5.4):** because raw events (incl. raw external-data responses) are
retained (N-C2-402), an `insufficient`/`best_effort` event can be **re-scored** after a fix
(DT-25: 312 re-normalized, 309 recover to `complete`, 3 stay `insufficient` because their raw
response was not retained). Recovery is bounded by raw-data retention.

### 6.4 E1 recomputes per-bundle (frozen)
**E1 recomputes `replay_completeness` per evidence event at replay time, against the specific
bundle being replayed** (E1 §7, D-E1-03). A field that is "present" in the audit event may
still be insufficient *for a given bundle* if that bundle references an input path the
reconstructed fixture lacks: E1 introspects the bundle's referenced input paths (Rego
metadata / `opa inspect`) and intersects them with the fixture's present fields; any
referenced-but-absent field **downgrades that event for that run**. Mapping to E1 behavior:

| `replay_completeness` (per-bundle) | E1 behavior |
|---|---|
| `complete` | authoritative; counts toward the headline differential matrix |
| `best_effort` (E1 source text: `partial`) | advisory; separate non-authoritative bucket; `missing_fields` enumerated |
| `insufficient` | not evaluated for a decision; counted in `incomplete`; surfaced as audit-coverage gap |

E1 consumes C2 through the thin **`ReplayEventV1` adapter** (D-E1-03), which is also where the
`partial → best_effort` alias normalization happens (**OV-1/OV-5**).

### 6.5 Stability rule
The three replay states are a **frozen closed enum**; the `replay_completeness_reasons`
vocabulary (C2 §5.5) is **additive-only**. `correlation_id` semantics are frozen; the
anchor-precedence list (admission UID → CI run → session → minted) is additive.

### 6.6 Open question + decided default
- **OQ (B5 OQ1):** Where is `correlation_id` minted? **Default:** server-side at
  gateway/admission, earliest point (integrity + single id).
- **OQ (B5 OQ2):** Sync vs async evidence emission? **Default:** decision synchronous, evidence
  async-buffered — a slow/down audit store never delays or fails an admission; `correlation_id`
  is preserved so backfill correlates later (B5-R3, F4).
- **OQ:** middle replay state name? **Default (D-C2-01):** **`best_effort`**; `partial`
  deprecated alias (**OV-1**).

### 6.7 Overrides
**OV-4** (stable `correlation_id` across approval retry; per-admission UID demoted to a
secondary id) and **OV-1** (replay state naming). E1's source SPEC still uses `partial`; that
is the deprecated alias and is normalized in the `ReplayEventV1` adapter.

---

## 7. Conformance checklist (per consumer)

| Component | MUST conform to |
|---|---|
| C2 | Contract 1 (owner); enforce Contract 3 scope-filter on its query API (N-C2-401); carry Contract 2 `jwt_claims`; record Contract 5 disposition+obligation; own Contract 6 state machine |
| C3 | Contract 1 (consume, ignore-unknown); **Contract 3 D2-R5 on aggregate reads**; Contract 6 (chain-integrity detector) |
| C4 | Contract 1; Contract 6 (correlation-members, reconstruction caps at `best_effort`) |
| C5 | Contract 1 export envelope; **Contract 3 D2-R5 + redaction**; Contract 6 reporting of non-authoritative subsets |
| D1 | Contract 2 (owner) |
| D2 | Contract 3 (owner); consume Contract 2 subject |
| D3 | Contract 6 (stable correlation_id across retry); Contract 5 (`require_approval` realization) |
| B4 | Contract 4 (schema owner of 6 CRDs); Contract 5 (owner) |
| B5 | Contract 6 (mint owner); Contract 1 evidence emission |
| E1 | Contract 1 (via `ReplayEventV1` adapter, `partial→best_effort`); Contract 3 on evidence queries; Contract 5 allow/deny normalization; **Contract 6 per-bundle recompute** |
| E3 | Contract 5 (actions drawn from closed 13-set); Contract 1 replay schema |
| F1 | Contract 3 (R-F1-AUTHZ-3 predicate into storage); Contract 4 (REST projection only) |
| F2 | Contract 4 (controllers + the 3 new CRDs); Contract 1 (every plugin/CRD emits C2 events) |
| F4 | Contracts 1–6 as **additive deltas only** (agent subject chain, evaluator_results — additive C2 fields) |
