# C2 — Audit Schema Framework & Standardized Replay-Capable Audit Event Schema — SPEC

**Component ID:** C2 · **Domain:** C — Evidence, Audit & Analytics
**Spec sources:** §12 (Audit Schema Framework), §13 (Standardized Audit Event Schema), with normative inputs from §9.3 (Gatekeeper required audit fields), §10.3 (Conftest evidence), §11.2 (Privateer correlation), §15.2 (required JWT claims), §17.3 (audit-driven simulation requirements), §17E (reporting), §23 (evidence integrity).
**Status:** **SCHEMA FROZEN v1.0** (see §3.13). This document is the authoritative contract that C3 (analytics), C4 (retrospective detection), C5 (reporting), E1 (simulation) and F4 (AI extension) depend on.
**Scenarios exercised:** DT-16, DT-25, DT-26, DT-27, DT-28, DT-42, HL-18 (and transitively all of Domain C).

---

## 0. Why this is the keystone

Market research §5 (and the platform's overall verdict) identifies the *replay-capable audit event schema* as one of the five genuinely uncovered differentiators: **no vendor emits governance-traceable evidence from the enforcement decision itself; they scrape evidence from logs after the fact.** OCSF normalizes events for a SIEM; it does not preserve enough decision *input* to re-run the policy. C2 is the contract that makes G1 (governance-to-enforcement traceability), G3 (runtime + retrospective compliance), simulation (E1), and audit-derived violation detection (C4) all possible.

The design principle (§13.1) is the load-bearing sentence of the whole platform:

> *Audit records used for replay must preserve the policy decision **input**, not merely the final outcome. A replay record that lacks information used by the original enforcement engine cannot produce an authoritative simulation result.*

Everything in this spec is downstream of that sentence. A record is only as valuable as the fidelity of the input it preserved — which is exactly what `replay_completeness` measures and labels honestly.

---

## 1. Scope

### 1.1 In scope
- The **normalized, replay-capable audit event schema** (the "C2 event"): field-by-field definition, types, required/optional, semantics, examples (§3).
- The **normalization pipeline contract**: how raw product events (Kubernetes audit, Gatekeeper audit, OPA decision logs, Conftest evidence, Keycloak events, service-mesh logs, application logs, scanner output) map into C2 events (§4).
- The **`replay_completeness` state machine** — `complete | best_effort | insufficient` — and the precise conditions that produce each state (§5).
- **Correlation** across Gatekeeper ↔ OPA ↔ Kubernetes-API ↔ Conftest ↔ Privateer via `correlation_id`, including the canonical anchor (Admission Review UID) and the recovery rules (§6).
- **Tamper-evidence**: per-event hashing, per-source hash-chaining (append-only log), periodic Merkle checkpoints, and signing of exports (§7).
- **Storage & retention** contract: append-only event store, raw-event retention requirement for re-normalization, addressing-by-digest (§8).
- **OCSF mapping** (optional compatibility layer) (§9).
- The **consumer API surface** that C3/C4/C5/E1 query against (§10).

### 1.2 Out of scope (delegated)
- The platform does **not** replace logging systems (§12.1). C2 ingests from them; it does not become the system of record for raw product logs (it retains raw events only long enough to re-normalize — §8.3).
- Running the replay itself (executing Rego against a C2 event) is **E1 (simulation)**; C2 defines the input contract and the completeness label, not the evaluator.
- Detections built on C2 events are **C3/C4**; report rendering and export packaging are **C5**. C2 provides the substrate and the export-integrity primitives (§7), not the report layouts.
- Gemara evaluation-log production and supply-chain evidence correlation are **C1 (Privateer)**; C1 consumes C2 events and adds control-evaluation semantics.

### 1.3 Design tenets (normative posture)
- **Honesty over coverage.** A record is never upgraded to `complete` when input fidelity is unknown. Silent promotion is forbidden (DT-25, DT-46). The schema is designed so that "we don't know" is a representable, queryable state.
- **Input-preserving, not outcome-preserving.** The decision outcome is recorded, but the *inputs the engine consulted* are the payload that gives the record its value.
- **Append-only and tamper-evident by construction**, because the records are audit evidence presented to external auditors (HL-18, DT-24).
- **Engine-neutral.** One schema spans OPA, Gatekeeper, Kyverno, Conftest, application-embedded PDPs, and scanners (§17C engine taxonomy). Engine-specific detail lives in a typed `engine_context` extension, never by forking the core schema.

---

## 2. Data model overview

A **C2 event** is a single normalized, immutable JSON document representing one policy-relevant occurrence (a decision, a resource change, a scan result, an approval request, an auth event). Events are grouped and joined by `correlation_id`; one logical request (e.g. one Kubernetes admission) typically produces several C2 events (a Gatekeeper decision event, an OPA decision event, a K8s-API resource-change event) that share a `correlation_id`.

```
                         ┌──────────────────────────────────────────┐
 raw product events ───▶ │  Normalizer (per-source adapters)        │
 (K8s audit, GK audit,   │  - field projection per §4 mapping       │
  OPA decision logs,     │  - JWT-claim capture (§15.2)             │
  Conftest, Keycloak,    │  - external_data_refs capture (§17.3)   │
  mesh, app, scanner)    │  - replay_completeness scoring (§5)     │
                         │  - correlation_id assignment (§6)        │
                         │  - canonicalization + content_hash (§7)  │
                         └───────────────────┬──────────────────────┘
                                             ▼
                         ┌──────────────────────────────────────────┐
                         │  Append-only event store (§8)            │
                         │  - per-source hash chain (prev_hash)     │
                         │  - periodic Merkle checkpoint + sign     │
                         │  - raw-event retention for re-normalize  │
                         └───────────────────┬──────────────────────┘
                                             ▼
            ┌────────────────────────────────────────────────────────────┐
            │  Consumer query API (§10)                                  │
            │   C1 Privateer · C3 analytics · C4 retrospective ·         │
            │   C5 reporting · E1 simulation · F4 AI extension          │
            └────────────────────────────────────────────────────────────┘
```

---

## 3. The frozen event schema — field by field

> **Versioning.** The schema document carries `schema_version` (string, e.g. `"c2.audit-event/1.0"`). v1.0 is **FROZEN** for the POC. Additive optional fields may be introduced under a minor bump (`1.1`) without breaking consumers; required-field changes or semantic changes require a major bump and a migration plan (§3.13, PLAN milestone M-FREEZE). Consumers MUST ignore unknown fields (forward-compat rule **N-C2-FWD**).

Legend: **R** = required (MUST be present on every event); **C** = conditionally required (required when the condition holds, else omitted/null); **O** = optional.

### 3.1 Identity & envelope fields

| # | Field | Type | R/C/O | Semantics |
|---|---|---|---|---|
| 1 | `schema_version` | string | **R** | Schema contract version, e.g. `"c2.audit-event/1.0"`. Enables consumer forward-compat. |
| 2 | `event_id` | string (UUIDv7) | **R** | Globally unique event identifier. UUIDv7 chosen so IDs are time-sortable (aids store locality and chain ordering). |
| 3 | `correlation_id` | string | **R** | Join key across all events for one logical request/flow. Canonical anchor = Kubernetes Admission Review UID where applicable (§6). MUST be present; if no upstream anchor exists the normalizer mints one and records `correlation_source` (§6.4). |
| 4 | `parent_correlation_id` | string | O | For nested/derived flows (e.g. a synthetic replay event derived from a reconstructed input points back to the original via this field). |
| 5 | `timestamp` | string (RFC 3339 UTC, ms precision) | **R** | Time of the original action or enforcement decision (NOT ingestion time). |
| 6 | `ingest_timestamp` | string (RFC 3339 UTC) | **R** | Time the normalizer wrote the event. `ingest_timestamp − timestamp` is the audit latency; large gaps feed C3 lag detection. |
| 7 | `event_type` | enum string | **R** | One of: `policy.decision`, `resource.change`, `scan.result`, `approval.request`, `approval.decision`, `auth.event`, `replay.synthetic`. (Extensible enum; unknown values tolerated per N-C2-FWD.) |
| 8 | `producer` | object | **R** | `{ "component": "audit-normalizer", "version": "x.y.z", "adapter": "<source-adapter-id>" }` — provenance of the normalization. |

### 3.2 Decision fields

| # | Field | Type | R/C/O | Semantics |
|---|---|---|---|---|
| 9 | `decision` | enum string | **C** | `allow \| deny \| warn \| mutate \| suspend_pending_approval \| require_approval \| unknown`. Required when `event_type=policy.decision`. `unknown` is used when the source recorded an action but not a verdict (e.g. a bypassed deployment — C4). |
| 10 | `policy_engine` | enum string | **C** | `opa \| gatekeeper \| kyverno \| conftest \| application \| scanner`. Required for `policy.decision` and `scan.result`. (Mirrors §17C engine taxonomy.) |
| 11 | `policy_version` | string | **C** | Policy or bundle version/digest used for the decision (e.g. `bundle:v12` or an OCI digest `sha256:…`). Required for `policy.decision`. Critical for cross-cluster drift detection (C3, HL-09, DT-32). |
| 12 | `policy_ref` | object | **C** | `{ "rego_package": "...", "rule": "...", "constraint_template": "...", "constraint_name": "..." }` — the executed policy reference. `rego_package` for OPA; `constraint_template`+`constraint_name` for Gatekeeper (§9.3). At least one sub-field required for `policy.decision`. |
| 13 | `control_id` | string | **C** | Governance control mapping (e.g. `SC-IMG-001`). Required when the policy is bound to a Gemara control (the normal case in production); may be absent for ungoverned/ad-hoc policies, in which case C3 raises an "ungoverned enforcement" finding. |
| 14 | `outcome_reason` | string | **C** | Human-readable explanation (e.g. `"Unsigned image prohibited"`). Required when `decision ∈ {deny, warn, suspend_pending_approval, require_approval}`. |
| 15 | `action_performed` | enum string | O | The concrete action taken (`block`, `mutate`, `annotate`, `route_for_approval`, `log_only`). Distinguishes intent (`decision`) from effect; consumed by C5 real-time report mutation/approval columns. |
| 16 | `mutation_diff` | object (RFC 6902 JSON Patch) | C | Present iff `action_performed=mutate`. The applied patch. Lets C5 render the mutation diff and E1 reconstruct post-mutation state. |

### 3.3 Subject & identity fields

| # | Field | Type | R/C/O | Semantics |
|---|---|---|---|---|
| 17 | `subject` | object | **R** | Normalized actor/workload identity: `{ "sub": "...", "groups": [...], "roles": [...], "tenant": "...", "type": "user\|service-account\|workload\|ci-pipeline" }`. The platform's canonical identity; populated from the JWT or from the source's actor field. |
| 18 | `jwt_claims` | object | **C** | The *raw* claims used for policy input, where available (§15.2). MUST include at minimum the claims the policy consulted. Required when a JWT was present at decision time. Distinct from `subject`: `subject` is normalized; `jwt_claims` is verbatim so replay is faithful (§17.3). Sensitive claims are redaction-aware (§7.5). |
| 19 | `jwt_claims_completeness` | enum string | C | `full \| partial \| reconstructed`. `partial` when only a subset of claims was captured; `reconstructed` when claims were rebuilt from a token-issuance (Keycloak) event joined by `sub`+time, not captured at decision time. Feeds `replay_completeness` (§5). |

### 3.4 Resource & operation fields

| # | Field | Type | R/C/O | Semantics |
|---|---|---|---|---|
| 20 | `operation` | enum string | **C** | `create \| update \| delete \| connect \| deploy \| scan \| approve \| login \| logout`. Required for `resource.change` and `policy.decision` over a resource. |
| 21 | `resource_type` | string | **C** | Type of object evaluated (e.g. `Deployment`, `terraform.aws_s3_bucket`, `index-pattern`). Required when a resource is in play. |
| 22 | `resource_id` | string | **C** | Stable resource identity, conventionally `<cluster>/<namespace>/<kind>/<name>` for K8s (e.g. `cluster-a/payments-prod/Deployment/api`). Required when a resource is in play. The join key for C4 "does a workload have a paired decision?" |
| 23 | `scope` | object | **R** | `{ "cluster": "...", "namespace": "...", "tenant": "...", "region": "...", "environment": "..." }`. Any sub-field may be null when inapplicable (e.g. a CI Conftest event has no cluster). The scoping substrate for storage-level authz (§17A.5) and per-scope coverage reports (C5, DT-80). |
| 24 | `before_state` | object | C | Previous object state, if relevant to the decision (update/delete). Stored as a canonical object or a content-addressed reference (§8.4) when large. |
| 25 | `after_state` | object | C | New/resulting object state, if relevant. |
| 26 | `request_object` | object | C | The original requested object exactly as submitted (e.g. the K8s `AdmissionRequest.object`). The single most important replay input for admission decisions (§13.4, DT-30 reconstruction). Stored canonical or by reference. |

### 3.5 External-data & replay-fidelity fields

| # | Field | Type | R/C/O | Semantics |
|---|---|---|---|---|
| 27 | `external_data_refs` | array of object | **C** | Every external datum the policy consulted, with its version so the value can be reproduced. Each element: `{ "name": "...", "provider": "...", "version": "<ts or semver>", "digest": "sha256:…", "value_ref": "<content-addressed pointer, optional>" }`. **Required and non-empty whenever the executed policy is known to consult external data** (e.g. `image-signature-status`). Its absence when external data was used is the root cause of `insufficient` (DT-25, DT-27). |
| 28 | `replay_completeness` | enum string | **R** | `complete \| best_effort \| insufficient` (state machine §5). MUST be present on every event. The honesty label of the whole record. |
| 29 | `replay_completeness_reasons` | array of string | C | Machine-readable reason codes explaining a non-`complete` state (e.g. `["missing_external_data:image-signature-status", "jwt_reconstructed"]`). Required when `replay_completeness != complete`. Drives the §17E.1 "missing audit fields" report. |
| 30 | `confidence_level` | enum string | C | `high \| medium \| low`. For synthetic/reconstructed events (C4) the confidence the reconstructed input matches what the engine would have seen. Required when `event_type=replay.synthetic`. Maps to §17E.3 `confidence_level`. |

### 3.6 Provenance, source & integrity fields

| # | Field | Type | R/C/O | Semantics |
|---|---|---|---|---|
| 31 | `source` | object | **R** | The originating audit source: `{ "system": "kubernetes-audit\|gatekeeper\|opa-decision-log\|conftest\|keycloak\|service-mesh\|application\|scanner", "raw_event_ref": "<pointer to retained raw event>", "raw_event_digest": "sha256:…" }`. `raw_event_ref` makes re-normalization (DT-25 backfill) and round-trip-to-source (DT-78) possible. |
| 32 | `engine_context` | object | O | Typed, engine-specific extension bag, namespaced by engine. E.g. for Gatekeeper: `{ "gatekeeper": { "admission_review_uid": "...", "request_uid": "...", "constraint_action": "deny" } }`; for OPA: `{ "opa": { "decision_id": "...", "bundle_revision": "..." } }`. Never used to carry fields that belong in the core schema. |
| 33 | `content_hash` | string | **R** | `sha256` over the canonical serialization of all fields **except** `content_hash`, `prev_hash`, `chain_seq`, and the signature envelope (§7.2). Identifies the event by content. |
| 34 | `prev_hash` | string | **R** | `content_hash` of the previous event in this source's append-only chain (§7.3). Genesis events use a fixed zero hash. |
| 35 | `chain_seq` | integer | **R** | Monotonic per-source sequence number. Gaps are detectable evidence of deletion/tampering. |
| 36 | `signature` | object | O | Present on checkpoint/anchor events and on exported events: `{ "alg": "ed25519", "key_id": "...", "sig": "base64…", "signed_at": "..." }` (§7.4). Per-event signing is optional; checkpoint signing is mandatory. |

### 3.7 The `replay_completeness` enum — note on naming

The spec table (§13.3) lists the third value as **`partial`**; the spec example (§13.4) and reporting (§17E) use `complete` / `insufficient`. **This SPEC freezes the middle state as `best_effort`** (the orchestration brief's wording) and treats `partial` as a **deprecated alias** that the normalizer MUST accept on ingest and rewrite to `best_effort`, and that the consumer API MUST accept as a query synonym. Rationale: `best_effort` is unambiguous about meaning ("we replayed with the best inputs we had, but fidelity is reduced"), whereas `partial` is overloaded with `jwt_claims_completeness=partial`. **Decision D-C2-01** (see §11). This is the single naming reconciliation other domains must adopt; everywhere the spec text says `partial` for replay completeness, read `best_effort`.

### 3.8 Full JSON example — a `complete` Gatekeeper deny (canonical, mirrors §13.4)

```json
{
  "schema_version": "c2.audit-event/1.0",
  "event_id": "018f7c2a-0000-7aaa-bbbb-0123456789ab",
  "correlation_id": "k8s-admrev:7f3c1e90-2c4d-4a1b-9f10-aa11bb22cc33",
  "timestamp": "2026-05-12T00:00:00.000Z",
  "ingest_timestamp": "2026-05-12T00:00:00.412Z",
  "event_type": "policy.decision",
  "producer": { "component": "audit-normalizer", "version": "1.0.0", "adapter": "gatekeeper-audit-webhook" },
  "decision": "deny",
  "policy_engine": "gatekeeper",
  "policy_version": "bundle:v12",
  "policy_ref": {
    "constraint_template": "K8sUnsignedImage",
    "constraint_name": "require-signed-images",
    "rego_package": "governance.kubernetes.imagesigning",
    "rule": "deny"
  },
  "control_id": "SC-IMG-001",
  "outcome_reason": "Unsigned image prohibited",
  "action_performed": "block",
  "subject": {
    "sub": "user-123",
    "groups": ["team-payments"],
    "roles": ["developer"],
    "tenant": "payments",
    "type": "user"
  },
  "jwt_claims": {
    "iss": "https://keycloak.example/realms/platform",
    "aud": "kubernetes",
    "sub": "user-123",
    "groups": ["team-payments"],
    "namespaces": ["payments-dev", "payments-prod"],
    "tenant": "payments"
  },
  "jwt_claims_completeness": "full",
  "operation": "create",
  "resource_type": "Deployment",
  "resource_id": "cluster-a/payments-prod/Deployment/api",
  "scope": {
    "cluster": "cluster-a",
    "namespace": "payments-prod",
    "tenant": "payments",
    "region": "us-east-1",
    "environment": "prod"
  },
  "request_object": {
    "apiVersion": "apps/v1",
    "kind": "Deployment",
    "metadata": { "name": "api", "namespace": "payments-prod" },
    "spec": { "template": { "spec": { "containers": [ { "image": "registry.example/api:1.4.2" } ] } } }
  },
  "external_data_refs": [
    {
      "name": "image-signature-status",
      "provider": "cosign-verifier",
      "version": "2026-05-12T00:00:00Z",
      "digest": "sha256:9a1f…",
      "value_ref": "cas://blobs/sha256:9a1f…"
    }
  ],
  "replay_completeness": "complete",
  "replay_completeness_reasons": [],
  "source": {
    "system": "gatekeeper",
    "raw_event_ref": "raw://gatekeeper/2026-05-12/00/evt-77231",
    "raw_event_digest": "sha256:c0ffee…"
  },
  "engine_context": {
    "gatekeeper": {
      "admission_review_uid": "7f3c1e90-2c4d-4a1b-9f10-aa11bb22cc33",
      "request_uid": "ad77-9931",
      "constraint_action": "deny"
    }
  },
  "content_hash": "sha256:11aa…",
  "prev_hash": "sha256:00ff…",
  "chain_seq": 980421
}
```

### 3.9 Second example — a synthetic reconstructed event for a bypassed deployment (`best_effort`, C4 / DT-30)

```json
{
  "schema_version": "c2.audit-event/1.0",
  "event_id": "018f7c2a-0000-7aaa-cccc-1111deadbeef",
  "correlation_id": "k8s-audit:7f3c8821-…",
  "parent_correlation_id": "k8s-audit:7f3c8821-…",
  "timestamp": "2026-05-12T14:07:11.000Z",
  "ingest_timestamp": "2026-05-13T02:30:00.000Z",
  "event_type": "replay.synthetic",
  "producer": { "component": "retrospective-detector", "version": "1.0.0", "adapter": "c4-reconstructor" },
  "decision": "deny",
  "policy_engine": "opa",
  "policy_version": "bundle:v12",
  "policy_ref": { "rego_package": "governance.kubernetes.imagesigning", "rule": "deny" },
  "control_id": "SC-IMG-001",
  "outcome_reason": "Unsigned image prohibited (reconstructed)",
  "subject": { "sub": "user-987", "groups": ["team-payments"], "tenant": "payments", "type": "user" },
  "jwt_claims": { "sub": "user-987", "groups": ["team-payments"], "tenant": "payments" },
  "jwt_claims_completeness": "reconstructed",
  "operation": "create",
  "resource_type": "Deployment",
  "resource_id": "cluster-a/payments-prod/Deployment/api-legacy",
  "scope": { "cluster": "cluster-a", "namespace": "payments-prod", "tenant": "payments", "environment": "prod" },
  "request_object": { "apiVersion": "apps/v1", "kind": "Deployment", "metadata": { "name": "api-legacy" } },
  "external_data_refs": [],
  "replay_completeness": "best_effort",
  "replay_completeness_reasons": [
    "no_engine_evaluation:reconstructed_from_k8s_audit",
    "jwt_reconstructed",
    "external_data_unavailable_at_replay_time"
  ],
  "confidence_level": "high",
  "source": {
    "system": "kubernetes-audit",
    "raw_event_ref": "raw://k8s-audit/2026-05-12/14/evt-55012",
    "raw_event_digest": "sha256:beef…"
  },
  "content_hash": "sha256:22bb…",
  "prev_hash": "sha256:11aa…",
  "chain_seq": 12
}
```

> **Invariant N-C2-SYNTH:** an event with `event_type=replay.synthetic` MUST NOT carry `replay_completeness=complete`. Reconstructed inputs are by definition not an authoritative capture of what the engine saw (§13.1). Maximum is `best_effort`. This is asserted across DT-30, DT-42, HL-06.

### 3.10 Conftest / build-time example (`complete`, §10.3)

```json
{
  "schema_version": "c2.audit-event/1.0",
  "event_id": "018f7c2a-…",
  "correlation_id": "ci:github-actions:run-88213",
  "timestamp": "2026-05-12T00:00:00Z",
  "ingest_timestamp": "2026-05-12T00:00:05Z",
  "event_type": "policy.decision",
  "producer": { "component": "audit-normalizer", "version": "1.0.0", "adapter": "conftest-sarif" },
  "decision": "deny",
  "policy_engine": "conftest",
  "policy_version": "policybundle:ci:v8",
  "policy_ref": { "rego_package": "governance.kubernetes.imagesigning" },
  "control_id": "SC-IMG-001",
  "outcome_reason": "image lacks signature metadata",
  "operation": "scan",
  "resource_type": "Deployment",
  "resource_id": "repo:payments/api//deployment/api-server",
  "scope": { "tenant": "payments", "environment": "ci" },
  "external_data_refs": [],
  "replay_completeness": "complete",
  "source": { "system": "conftest", "raw_event_ref": "raw://ci/run-88213/conftest.json", "raw_event_digest": "sha256:…" },
  "engine_context": { "conftest": { "pipeline": "github-actions", "evidence_type": "build-time" } },
  "content_hash": "sha256:…",
  "prev_hash": "sha256:…",
  "chain_seq": 4471
}
```

### 3.11 Field coverage cross-check against §9.3 (Gatekeeper required audit fields)

Every §9.3-mandated Gatekeeper field maps into C2 (proving the schema is a superset, so no source field is lost — DT-16):

| §9.3 field | C2 location |
|---|---|
| Timestamp | `timestamp` |
| Cluster ID | `scope.cluster` |
| Namespace | `scope.namespace` |
| Resource Kind | `resource_type` |
| Resource Name | `resource_id` (last path segment) |
| User Identity | `subject.sub` |
| JWT Subject | `jwt_claims.sub` |
| JWT Groups | `jwt_claims.groups` |
| Control ID | `control_id` |
| Constraint Template | `policy_ref.constraint_template` |
| Constraint Name | `policy_ref.constraint_name` |
| Rego Package | `policy_ref.rego_package` |
| Decision Outcome | `decision` |
| Violation Reason | `outcome_reason` |
| Request UID | `engine_context.gatekeeper.request_uid` |
| Admission Review UID | `engine_context.gatekeeper.admission_review_uid` (= `correlation_id` anchor) |
| Correlation ID | `correlation_id` |

### 3.12 Field coverage cross-check against §15.2 (required JWT claims)

`jwt_claims` MUST faithfully carry whatever §15.2 / §15.3 claims the policy consulted (`iss`, `aud`, `sub`, `groups`, `roles`, `tenant`, `namespaces`, plus recommended custom claims). C3 JWT-drift detection (DT-31) compares the policy's required-claim list against the claims actually present in `jwt_claims` across the observed population.

### 3.13 FROZEN field list (the contract other domains depend on)

The following **36 top-level fields** constitute frozen schema v1.0. Required (R) fields MUST be present on every event; conditional (C) per their rule; optional (O) as available.

```
R:  schema_version, event_id, correlation_id, timestamp, ingest_timestamp,
    event_type, producer, subject, scope, replay_completeness,
    source, content_hash, prev_hash, chain_seq
C:  decision, policy_engine, policy_version, policy_ref, control_id,
    outcome_reason, mutation_diff, jwt_claims, jwt_claims_completeness,
    operation, resource_type, resource_id, before_state, after_state,
    request_object, external_data_refs, replay_completeness_reasons,
    confidence_level
O:  parent_correlation_id, action_performed, engine_context, signature
```

(14 R + 18 C + 4 O = 36.) `replay_completeness` value set is **frozen** as `complete | best_effort | insufficient` (with `partial` as a deprecated ingest/query alias of `best_effort`). Consumers MUST treat the field list as additive-only within v1.x and MUST ignore unknown fields (**N-C2-FWD**).

---

## 4. Normalization pipeline contract

### 4.1 Per-source adapters
One adapter per audit source (§12.2): `kubernetes-audit`, `gatekeeper-audit-webhook`, `opa-decision-log`, `conftest-sarif`, `keycloak-events`, `service-mesh`, `application-sdk`, `scanner`. Each adapter:
- **N-C2-100** MUST project all source fields into the C2 field map and MUST NOT drop a field that the C2 schema has a home for (the DT-25 regression: a normalizer refactor silently dropped `external_data_refs` — the test suite §test must fail if any home-able field is dropped).
- **N-C2-101** MUST retain the raw source event (or a content-addressed pointer to it) and populate `source.raw_event_ref` + `source.raw_event_digest`, so re-normalization (DT-25 backfill) and round-trip (DT-78) are possible.
- **N-C2-102** MUST compute `replay_completeness` per §5 deterministically from the projected fields plus the policy's known external-data dependencies.
- **N-C2-103** MUST assign `correlation_id` per §6.
- **N-C2-104** MUST canonicalize (RFC 8785 JSON Canonicalization Scheme) before hashing, so `content_hash` is reproducible by any party (auditor independence — HL-18).

### 4.2 Policy-dependency catalog (the input that makes completeness scoring possible)
The normalizer consults a **policy-dependency catalog**: for each `(policy_ref, policy_version)`, the set of inputs the policy consults — which JWT claims, which `external_data_refs` providers, whether it reads `before_state`. This catalog is produced from Rego metadata (§8.3 Rego Metadata Extensions, owned by B1) and is the ground truth against which "did we capture everything the engine used?" is judged. **Without it, completeness scoring degrades to `best_effort` by default** (you cannot prove `complete` if you do not know what was needed).

### 4.3 Idempotency & ordering
- **N-C2-105** Normalization is idempotent on `(source.system, source.raw_event_digest)`: re-normalizing the same raw event yields the same `content_hash` (modulo chain fields). Re-normalization replaces nothing in place; it appends a new event whose `parent_correlation_id`/lineage references the superseded one and marks the prior `superseded_by` via a side index (the chain is never rewritten — §7.3).
- Per-source ordering is by `chain_seq`; cross-source ordering is by `timestamp` with `event_id` (UUIDv7) as tiebreak.

---

## 5. The `replay_completeness` state machine

Three states. The classifier runs in the normalizer (live capture) and again in E1/C4 (at replay time), and they MUST agree on the same inputs (determinism test §test).

```
                 ┌─────────── all required replay inputs captured AND verifiable ───────────┐
                 │                                                                          ▼
   raw event ──▶ score() ──▶ ┌────────────┐   missing/unverifiable inputs   ┌──────────────┐
                             │  complete  │◀────── none ───────┐            │  best_effort │
                             └────────────┘                    │            └──────────────┘
                                   ▲                            │                   │
                                   │ backfill restores inputs   │ critical input    │ critical input
                                   │ (DT-25)                    │ degraded but       │ entirely absent
                                   │                            │ a defensible       │ AND not
                                   └────────────────────────────┘ replay is possible │ reconstructable
                                                                                     ▼
                                                                            ┌──────────────┐
                                                                            │ insufficient │
                                                                            └──────────────┘
```

### 5.1 `complete`
All of the following hold:
- `request_object` (or `before_state`/`after_state` as the policy requires) is captured.
- `jwt_claims` present with `jwt_claims_completeness=full` **if** the policy consults JWT claims (per the dependency catalog §4.2).
- **every** `external_data_refs` entry the policy consults is present with a resolvable `version`/`digest` (and, where the value can be large/volatile, a `value_ref` to the captured value).
- `policy_version` is known and the bundle is addressable by digest.
- The event was produced by a real engine evaluation (`event_type != replay.synthetic`).

→ The replay result is **authoritative**: it is exactly what the engine decided / would decide.

### 5.2 `best_effort`
A defensible replay is possible but fidelity is reduced. Triggered by **any** of:
- `event_type=replay.synthetic` — the input was reconstructed (C4), so by construction not an authoritative capture (N-C2-SYNTH). Always at most `best_effort`.
- `jwt_claims_completeness ∈ {partial, reconstructed}` — some claims rebuilt or missing.
- One or more `external_data_refs` versions are known but the captured *value* is unavailable, and the provider can re-resolve the value at that version with bounded risk of drift (a documented `replay_completeness_reasons` code explains it).
- `policy_version` resolved by inference (e.g. "the bundle deployed to this cluster at `timestamp`") rather than recorded directly.

→ The replay result is **indicative, not authoritative**; it carries `confidence_level` and `replay_completeness_reasons`, and C5/C4 disclose it as such. It is **never** counted in authoritative totals without the reason being surfaced (DT-46 step 3-4).

### 5.3 `insufficient`
A faithful replay is **not** possible. Triggered by **any** of:
- The policy consults external data and **no** `external_data_refs` entry exists and the value is not re-resolvable (the DT-25 root cause: the normalizer dropped the field and the raw external-data response was not retained).
- `request_object` / required state is absent and unreconstructable.
- The policy consults JWT claims and `jwt_claims` is absent and no token-issuance event can supply them.
- `policy_version` is unknown and not inferable.

→ The event is **flagged, not promoted**. It appears in the §17E.1 "missing audit fields" report (C5), is **excluded from authoritative replay counts** (DT-46 step 4), and is the trigger for the DT-25 remediation loop (fix the normalizer projection → backfill → re-score). Silent promotion to a verdict is forbidden (N-C2-SYNTH's sibling invariant **N-C2-NOPROMOTE**).

### 5.4 Backfill / re-scoring
Because raw events are retained (N-C2-101), an `insufficient`/`best_effort` event can be **re-scored** after a fix:
- DT-25: the normalizer projection bug is fixed and the 312 raw events are re-normalized; 309 recover to `complete` (their raw external-data response was retained), 3 remain `insufficient` (raw response lost) and stay flagged. **Recovery is bounded by raw-data retention** — this is why §8.3 mandates retaining the raw external-data response, not just the request object.

### 5.5 Reason-code vocabulary (frozen, additive)
`missing_external_data:<provider>`, `external_data_value_unavailable:<provider>`, `external_data_version_drift:<provider>` (DT-27), `jwt_absent`, `jwt_partial`, `jwt_reconstructed`, `request_object_absent`, `policy_version_inferred`, `policy_version_unknown`, `no_engine_evaluation:reconstructed_from_k8s_audit`, `before_state_absent`. C3/C4/C5 switch on these codes; new codes are additive only.

---

## 6. Correlation contract (Gatekeeper ↔ OPA ↔ K8s ↔ Conftest ↔ Privateer)

### 6.1 Canonical anchor
For Kubernetes admission flows the **Admission Review UID** is the canonical `correlation_id` anchor (§9.3 lists both Request UID and Admission Review UID; DT-28 establishes the Admission Review UID as the only identifier both Gatekeeper *and* embedded OPA see). Therefore:
- **N-C2-200** The Gatekeeper adapter sets `correlation_id = admission_review_uid`.
- **N-C2-201** The OPA decision-log adapter MUST echo the inbound Admission Review UID into `correlation_id` (keeping OPA's own `decision_id` in `engine_context.opa.decision_id` as a secondary id). This is the exact fix DT-28 prescribes: OPA by default emits its own `decision_id` and does *not* echo the admission UID, breaking the join.
- **N-C2-202** The Kubernetes-API audit adapter sets `correlation_id` from the same Admission Review UID (available in the K8s audit annotations for admission-reviewed requests) or, for non-admission API calls, from the K8s audit `auditID`.

### 6.2 Non-admission anchors
- CI/Conftest: `correlation_id = ci:<provider>:<run-id>` (e.g. `ci:github-actions:run-88213`), shared across all Conftest evidence in one pipeline run.
- Keycloak: `correlation_id` from the auth session/flow id; token-issuance events are joined to later decision events by `subject.sub` + time window when no shared id exists (yielding `jwt_claims_completeness=reconstructed`).
- Application-embedded PDP: the application supplies a request id via the PDP SDK (E3) and it becomes `correlation_id`.

### 6.3 Correlation-gap detection feed
The store maintains, per `correlation_id`, the set of source systems that contributed events. C3/C4 query "for this admission, which of {k8s-audit, gatekeeper, opa} are present?" — the absence of expected members is the §14.2 bypass signal (DT-30, DT-42) and the DT-28 missing-link signal. C2 surfaces this as a `correlation_members` derived view (§10.3); it does not itself raise the finding (that is C3/C4).

### 6.4 Mint-and-record
- **N-C2-203** If no upstream anchor exists, the normalizer mints a UUIDv7 `correlation_id` and records `engine_context.<sys>.correlation_source = "minted"` so downstream consumers know it cannot be joined to other sources. A minted id on a `policy.decision` that *should* have had an anchor is itself a C3 finding (DT-28: "412 of 412 unpaired").

### 6.5 Recovery flags
When a correlation is repaired retroactively (DT-28 backfill), prior unjoinable events are tagged `correlation_id_recovered=false` in a side index and excluded from the post-fix alert baseline, so they remain visible as a known historical gap rather than being silently rewritten.

---

## 7. Tamper-evidence & evidence integrity (the auditor's trust anchor)

C2 events are presented to external auditors as evidence (HL-18, DT-24); they must be tamper-*evident* (detectable modification) even if not tamper-*proof*.

### 7.1 Threat model
- **Insider deletion/edit** of an embarrassing event (e.g. the bypassed deployment in HL-06). Defended by hash-chaining + sequence gaps + signed checkpoints.
- **Backdating** an event to fit a narrative. Defended by signed checkpoints whose `signed_at` bounds when an event could have been inserted.
- **Forging an export** to an auditor. Defended by signed export manifests with a Merkle root (DT-24).
- **Replay-result forgery** (claiming a `complete` verdict that was actually `insufficient`). Defended by N-C2-NOPROMOTE + signing the completeness label inside `content_hash`.

### 7.2 Canonicalization & content hash
- **N-C2-300** Events are serialized with RFC 8785 JCS before hashing. `content_hash = sha256(canonical_event_without_chain_and_sig_fields)`. Any party can recompute it from the event JSON → independent verification (HL-18).

### 7.3 Per-source hash chain (append-only log)
- **N-C2-301** Each source maintains an append-only chain: `prev_hash` of event N = `content_hash` of event N−1; `chain_seq` is strictly monotonic. The store is append-only; events are **never** edited or deleted in place. Corrections are new appended events (supersession via side index — §4.3).
- A broken link (`prev_hash` mismatch) or a `chain_seq` gap is detectable tampering. C3 runs a continuous chain-verification check (chain-integrity finding).

### 7.4 Signed Merkle checkpoints
- **N-C2-302** At a fixed cadence (default every N events or T minutes, both configurable; default N=10 000 / T=15 min) the store computes a Merkle root over the chain segment and emits a **checkpoint event** carrying `signature` (ed25519, platform signing key, published public key per §23). The checkpoint binds the segment's contents and the wall-clock `signed_at`. An attacker who edits a past event must forge every subsequent hash *and* re-sign every subsequent checkpoint, which requires the signing key.
- Checkpoints are themselves chained (checkpoint chain), so the latest signed checkpoint transitively attests to all history.

### 7.5 Redaction-aware integrity
- **N-C2-303** Sensitive `jwt_claims` (PII, secrets) MAY be redacted, but redaction is done by replacing the value with a salted hash `{"redacted": true, "value_digest": "sha256:…"}` so the field still contributes deterministically to `content_hash` and a holder of the cleartext can prove a match. Redaction MUST NOT silently drop the field (that would corrupt the chain and lose replay fidelity).

### 7.6 Signed export packages (consumed by C5/C1)
- **N-C2-304** The export primitive (used by DT-24, DT-46, DT-78, HL-18) produces: `manifest.json` (controls, period, scope, per-source row counts, query hash, in-scope bundle versions), per-control NDJSON of C2 events, `merkle.json` (per-file SHA-256 leaves + root), and a detached signature over `manifest.json` embedding the Merkle root, `key_id`, and `signed_at`. C5 lays out the report; **C2 owns the integrity envelope** so every export across the platform uses one verifiable format. Verification requires only the published public key (auditor independence).

---

## 8. Storage & retention contract

### 8.1 Append-only event store
- Logical model: an append-only, content-addressed event log partitioned by `(source.system, time)`, with secondary indexes on `correlation_id`, `control_id`, `scope.{cluster,namespace,tenant}`, `resource_id`, `policy_version`, `replay_completeness`, `timestamp`. (Physical backend is a PLAN decision — §PLAN; logical contract is fixed here.)
- **N-C2-400** No update-in-place, no hard delete within the retention window. Deletion only at retention expiry, recorded as a signed tombstone event so the deletion is itself auditable.

### 8.2 Scope-aware access
- **N-C2-401** Every query is scope-filtered by the caller's authorization subject (§17A.5 storage-level access controls, owned by D2). An Auditor scoped to `cluster=prod-east-2, control=SC-IMG-001` cannot read events outside that scope (DT-46 step 8, HL-18). C2 enforces this at the query API (§10), not just the UI.

### 8.3 Raw-event retention (the backfill enabler)
- **N-C2-402** Raw source events — including the **raw external-data provider responses** the policy consulted — are retained for at least the configured re-normalization window (default = the full replay retention, ≥30 days for the POC per §22.1 referenced by HL-12/HL-18). This is what makes DT-25 backfill possible; the 3 events that stayed `insufficient` did so precisely because their raw external-data response was *not* retained. Retention of the raw external-data response is therefore a first-class requirement, not an afterthought.

### 8.4 Large-object handling
- `before_state`, `after_state`, `request_object`, and external-data `value_ref` may be content-addressed blobs (`cas://blobs/sha256:…`) when large; the event carries the digest inline and the blob in a CAS store. The digest is inside `content_hash`, so integrity holds even when the body is stored out-of-line.

### 8.5 Retention & materialized datasets
- A **materialized replay dataset** (DT-46) is an immutable, named, scope-tagged snapshot of a query result (`object_type=simulation_dataset`, `control_ids`, `created_by`, `visibility` per §17A.5), addressable by digest and reusable across consumers (engineering replay + auditor walkthrough) without re-extraction. C2 provides the materialization + digest; E1 consumes it for replay.

---

## 9. OCSF mapping (optional compatibility layer)

§13.5: OCSF MAY be a compatibility/mapping target; it is **not** required for the POC, and the platform-specific replay schema is **authoritative** because OCSF normalizes for SIEM, not for policy replay (confirmed by market research §5: no OCSF profile preserves replay inputs).

- **N-C2-500** A one-way **C2 → OCSF projection** is provided as an *export adapter*, not as the storage format. Mapping: a `policy.decision` event projects to an OCSF *Detection Finding* / *Compliance Finding* class; `decision`→`status`/`disposition`; `control_id`→a compliance reference; `subject`→`actor`; `resource_id`→`resource`. The replay-critical fields (`request_object`, `external_data_refs`, `jwt_claims`, `replay_completeness`) have **no faithful OCSF home** and are carried in OCSF `unmapped`/`enrichments` — which is exactly why C2, not OCSF, is authoritative.
- **N-C2-501** There is **no** OCSF → C2 import that yields `complete`: an OCSF event from a third-party SIEM lacks replay inputs, so any import is at most `best_effort` and usually `insufficient`. This prevents a false belief that ingesting a SIEM gives replay capability (an adversarial trap — see ADVERSARIAL-REVIEW).

---

## 10. Consumer API surface (what C3/C4/C5/E1/C1 depend on)

C2 exposes a read API. The shapes below are the frozen consumer contract.

### 10.1 Event query
`GET /events?` filters: `control_id`, `scope.cluster|namespace|tenant|region|environment`, `resource_id`, `policy_engine`, `policy_version`, `decision`, `replay_completeness` (accepts `partial` as alias), `event_type`, `correlation_id`, `time_from`, `time_to`, `source.system`. Returns paginated C2 events, scope-filtered (N-C2-401). Cursor-stable on `(timestamp, event_id)`.

### 10.2 Single event + lineage
`GET /events/{event_id}` returns the event, its `source.raw_event_ref` (dereferenceable to the raw row for round-trip — DT-78), supersession lineage, and chain-verification status.

### 10.3 Correlation view
`GET /correlations/{correlation_id}` returns all events sharing the id, plus a `correlation_members` summary `{ present: [...], missing_expected: [...] }`. This is the substrate for the §16.3 Audit Correlation View and for C3/C4 bypass detection (DT-28, DT-30, DT-42).

### 10.4 Coverage matrix feed
`GET /coverage?tenant=&window=` returns, per `(scope.namespace × control_id)`, decision counts and the classification feed (`enforced | installed_no_events | not_installed | n/a`) — consumed by C5 coverage reports (DT-33, DT-80) and C3 coverage detection. C2 supplies the observed-decision side; the in-scope expectation set comes from the governance store (A1) and inventory.

### 10.5 Materialize dataset
`POST /datasets` (query + name + scope) → an immutable digest-addressed dataset (§8.5) for E1 replay and auditor reuse (DT-46).

### 10.6 Verify
`POST /verify` (an event, a dataset digest, or an export manifest) → chain/Merkle/signature verification result. The auditor-independence primitive (HL-18).

### 10.7 Stability guarantees
- **N-C2-FWD** Consumers MUST ignore unknown fields. C2 MUST NOT remove or repurpose a frozen field within v1.x. New optional fields and new enum values are additive (consumers must tolerate unknown enum values, treating them as their nearest known supertype or as opaque).

---

## 11. Decisions (decide-document-continue)

| ID | Decision | Rationale |
|---|---|---|
| **D-C2-01** | Freeze the middle completeness state as **`best_effort`**; accept `partial` as deprecated ingest/query alias. | `partial` collides with `jwt_claims_completeness=partial` and is ambiguous; `best_effort` states intent. Single reconciliation point for all of Domain C. |
| **D-C2-02** | **Custom replay schema is authoritative; OCSF is one-way export only.** | Per §13.5 + market research §5; OCSF cannot represent replay inputs. Prevents the "ingest a SIEM = get replay" fallacy (N-C2-501). |
| **D-C2-03** | **Admission Review UID is the canonical `correlation_id` anchor**; OPA MUST echo it. | Per §9.3 + DT-28: it is the only id both Gatekeeper and OPA see. |
| **D-C2-04** | **Append-only log + per-source hash chain + signed Merkle checkpoints** for tamper-evidence (not a mutable relational store as primary). | Audit evidence must be tamper-evident and independently verifiable (HL-18, DT-24, §23). See ALT for the relational alternative. |
| **D-C2-05** | **Retain raw events including raw external-data responses** for the full replay window. | Backfill (DT-25) is bounded by raw-data retention; without the raw external-data response, `insufficient` events can never recover. |
| **D-C2-06** | **Completeness scoring requires a policy-dependency catalog** (from Rego metadata, B1). Absent it, default to `best_effort`, never `complete`. | You cannot prove you captured everything the engine used unless you know what it needed. Honesty tenet. |
| **D-C2-07** | **`engine_context` typed extension bag** instead of forking the schema per engine. | Engine-neutral core (§17C taxonomy) with lossless engine specifics. |
| **D-C2-08** | **Reconstructed/synthetic events capped at `best_effort`** (N-C2-SYNTH). | §13.1: reconstructed input is not an authoritative capture (DT-30, DT-42, HL-06). |

## 12. Open questions (with decided defaults)
- **OQ-1: Physical store backend?** *Default:* logical contract here is backend-agnostic; PLAN recommends an append-only log (e.g. object-store-backed segments + an index DB) over a single RDBMS to make tamper-evidence natural. Revisit at scale (§22). See ALT.
- **OQ-2: Checkpoint cadence?** *Default:* N=10 000 events or T=15 min, configurable per source; tighter cadence narrows the backdating window at higher cost.
- **OQ-3: How long to retain raw external-data values vs. just versions?** *Default:* retain values for the full replay window for providers whose values are volatile/non-re-resolvable (signature status), versions-only for stable providers; the policy-dependency catalog flags which is which.
- **OQ-4: PII handling in `jwt_claims`?** *Default:* redaction-aware hashing (§7.5); a deployment-time claim allowlist marks which claims are captured verbatim vs. hashed.

## 13. Dependencies
- **Consumes:** raw audit sources (§12.2); B1 Rego metadata → policy-dependency catalog (§8.3, §4.2); D1 JWT/Keycloak normalization (`subject`, `jwt_claims`); D2 storage-authz (§17A.5) for scope filtering.
- **Consumed by (blocks):** **C3, C4, C5, E1, F4, C1** all depend on this frozen schema. Schema freeze (PLAN M-FREEZE) is the gating milestone for the whole domain and for simulation.
- **Cross-domain contract:** the §3.13 frozen field list + §5 state machine + §6 correlation rules + §7 integrity envelope + §10 API are the surface other domains code against.
