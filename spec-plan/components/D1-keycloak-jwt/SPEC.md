# D1 — Keycloak / JWT Integration & Mapping Layer — SPEC

**Domain:** D · Identity, Authz & Security
**Spec sources:** §15 (Keycloak and JWT Integration), §17A.4 (Keycloak claim mapping), §25.1 (custom JWT mappers plugin), §23.1 (Identity: OIDC/JWT, Keycloak as initial IdP)
**Scenarios exercised:** DT-35, DT-36, DT-37, DT-38, DT-53 (claim plumbing), HL-13, HL-16
**Status:** AUTHORED (parent-authored)
**Author persona:** Marcus — Platform Security Engineer (owns Keycloak realms, claim plumbing)

---

## 1. Scope

### 1.1 In scope
D1 owns the **identity ingestion edge** of the platform: turning OIDC/JWT tokens from one
or more upstream IdPs (with Keycloak as the initial/brokering IdP, §23.1) into a single
**canonical internal authorization subject** (§17A.4) that every other component consumes.
Concretely:

1. **OIDC/JWT verification** — signature validation, `iss`/`aud`/`exp`/`iat`/`nbf` checks,
   JWKS retrieval and rotation, clock-skew handling.
2. **Required-claim contract** — the §15.2 required claims plus the §17A.4 required
   authorization claims; fail-closed behavior when a policy-required claim is absent.
3. **The JWT-to-Policy Mapping Layer (§15.4)** — a declarative, versioned transformation
   pipeline performing: claim transformation, claim normalization, identity aliasing,
   tenant inheritance, group expansion, and role-hierarchy resolution.
4. **Multi-IdP tenant normalization** — collapsing heterogeneous source claims
   (`tenant`, `org_id`, …) from federated IdPs into one canonical claim set (DT-36, HL-16).
5. **Group→role expansion at token issuance / normalization time** (DT-38).
6. **Normalized-subject schema** — the versioned output contract (§17A.4 JSON subject).
7. **Claim lifecycle management** — add / evolve / decommission a claim safely (DT-35, DT-37),
   with dry-run, usage proof, and rollback.
8. **A dry-run / introspection toolchain** — evaluate the mapping pipeline over a sampled
   token corpus before promotion; surface value distributions and `unknown` cases.
9. **The `jwt_claims` contribution to the audit schema** (feeds Domain C / C2 §13.3).

### 1.2 Out of scope
- **Authorization decisions** themselves (role→permission, scope filtering): that is **D2**.
  D1 produces the *subject*; D2 *enforces* with it.
- **Keycloak deployment/HA/realm provisioning** as infrastructure (Domain F).
- **External IdP administration** (the upstream Okta/AD): only the broker contract matters.
- **Approval-state claims semantics** (`deployment_approval` lifecycle): D3 owns the
  approval state machine; D1 only normalizes the *claim* if present.

### 1.3 Design tenet (load-bearing)
> **No policy, Rego rule, Gatekeeper constraint, or storage query may ever reference a raw
> upstream claim path (e.g. `token.user_attributes.corp_risk_tier`, `org_id`).**
> Everything downstream references the *canonical normalized subject* only. The mapping
> layer is the single indirection point; this is what makes claim evolution (HL-16) a
> config change rather than a fleet-wide policy edit.

---

## 2. Data model / entities

### 2.1 Canonical normalized authorization subject (the output contract)
Superset of §15.4 (policy-facing claims) and §17A.4 (authorization-facing claims). This is
**versioned** (`schema_version`) because adding a canonical field is a contract change.

```json
{
  "schema_version": "subject/v3",
  "subject_id": "user-123",            // from sub
  "subject_type": "human",             // human | service_account | workload
  "username": "alice",                 // preferred_username
  "email": "alice@corp.example",       // human attribution (§15.2)
  "issuer": "https://kc.example/realms/platform",
  "tenants": ["payments"],             // §17A.4 normalized, multi-IdP collapsed
  "namespaces": ["payments-prod","payments-dev"],
  "policy_domains": ["runtime-security"],
  "roles": ["role:namespace-policy-author","role:org-finance"], // expanded (DT-38)
  "groups": ["/teams/payments"],
  "environment": "prod",               // §15.2 required
  "risk_level": "high",                // §15.3 recommended (DT-35)
  "compliance_scope": ["pci","sox"],   // §15.3
  "data_classification": "restricted", // §15.3
  "workload_identity": null,           // §15.3, for workload subjects
  "deployment_approval": null,         // §15.3, normalized only (D3 owns semantics)
  "claim_provenance": {                // see 2.4
    "tenants": {"source":"org_id","transform":"lowercase","idp":"idp-b"},
    "risk_level": {"source":"corp_risk_tier","transform":"lookup","value":"high"}
  },
  "token_ref": {                       // for audit, never the raw token
    "jti": "…", "iat": 1716, "exp": 1716, "aud": "platform"
  },
  "normalization_status": "complete"   // complete | degraded | incomplete
}
```

### 2.2 Mapping configuration (the §15.4 / §17A.4 config artifact)
A versioned governance artifact (lifecycle per §7), one logical document per realm/tenant
scope, composed of ordered **mapping entries**. Conceptual schema:

```yaml
apiVersion: governance.example.io/v1alpha1
kind: ClaimMapping
metadata: { name: platform-mapping, version: 14, realm: platform }
spec:
  schemaVersion: subject/v3
  defaults: { failClosed: true, unknownValue: "unknown" }
  entries:
    - target: tenants
      sources: [ {claim: tenant}, {claim: org_id} ]   # multi-IdP (DT-36)
      transform: lowercase
      onConflict: prefer_first_non_empty
      cardinality: list
      required: true                                   # §15.2 required claim
    - target: roles
      sources: [ {claim: groups} ]
      transform: expand_group_hierarchy                # DT-38
      hierarchy:
        org-finance:  [team-payments, team-billing]
        org-platform: [team-sre, team-admission]
      emitAs: "role:<group>"
    - target: risk_level
      sources: [ {claim: user_attributes.corp_risk_tier} ]
      transform: lookup
      table: { tier1: high, tier2: medium, tier3: low }
      normalize: { onMissing: unknown, onUnrecognized: unknown }
      identityAlias:
        - when: { subject_type: service_account }
          set: service_account_default
```

### 2.3 Transform catalog (normative set of built-in transforms)
| Transform | Purpose | Notes |
|---|---|---|
| `lowercase` / `uppercase` | case normalization | idempotent |
| `lookup(table)` | value remap | `onMissing`/`onUnrecognized` MUST be set |
| `first_prefix(p)` | extract first value with prefix `p` | e.g. `team-` |
| `extract_environment` | parse env from realm roles | §15.4 example |
| `expand_group_hierarchy` | nested group → role set (leaf + ancestors) | DT-38 |
| `prefer_first_non_empty` | multi-source conflict resolution | DT-36 |
| `concat` / `template` | derive a field from several sources | bounded |
| `tenant_inherit` | child tenant inherits parent scope | §15.4 tenant inheritance |
| `alias(map)` | identity aliasing (legacy → canonical sub) | §15.4 |
| `redact` | drop a source claim from the subject | hygiene |

Transforms MUST be **pure, deterministic, side-effect-free, bounded** (no network in the hot
path; external data resolved out-of-band into a snapshot — see 2.5). Custom transforms are a
plugin point (§25.1 "Custom JWT mappers") but MUST satisfy the same purity contract.

### 2.4 Claim provenance record
Every canonical field carries provenance (source claim, transform, source IdP, raw value or
hash). Purpose: HL-16 drift attribution, DT-37 decommission proof, audit defensibility.

### 2.5 External-data snapshot (for transforms needing lookups)
Some transforms (e.g. tenant-inheritance tables, HRIS tier tables) need reference data.
Resolved into an **immutable, versioned snapshot** keyed by digest, referenced by the mapping
version. Never fetched synchronously during token normalization (latency + failure-mode
isolation). Snapshot digest is part of `claim_provenance`.

---

## 3. Interfaces & APIs

| Interface | Direction | Description |
|---|---|---|
| `POST /v1/identity/normalize` | in | Verify a presented JWT; return the canonical subject (used by API gateway / D2). |
| `GET /v1/identity/whoami` | in | Convenience: normalized subject for the bearer token. |
| `POST /v1/mapping/dry-run` | in | Run a candidate mapping version over a sampled token corpus; return value distributions + `unknown` set (DT-35 step 4). |
| `GET /v1/mapping/versions` / `…/{v}` | in | Mapping artifact history (lifecycle §7). |
| `POST /v1/mapping/validate` | in | Static validation of a candidate mapping (cycles, undefined transforms, required-claim coverage). |
| `GET /v1/identity/introspect` | in | Token introspection view (DT-36 step 2): raw claims vs normalized subject side-by-side. |
| `subject` injected into PDP `input.subject` | out | Consumed by Rego/Gatekeeper/Conftest (canonical paths only). |
| `jwt_claims` block | out | Emitted into the §13.3 audit event (Domain C / C2). |
| JWKS client → Keycloak `/protocol/openid-connect/certs` | out | Key retrieval + rotation. |

### 3.1 Normalize response (degraded/incomplete signaling)
```json
{ "subject": { … }, "normalization_status": "incomplete",
  "missing_required": ["tenant"], "warnings": ["org_id present but no mapping entry"] }
```
`incomplete` subjects MUST be honored fail-closed by D2/PDP for any policy that requires the
missing claim (ties to §17.3 "mark simulation incomplete rather than authoritative").

---

## 4. Required-claim contract

### 4.1 §15.2 required (token) claims
`sub, iss, aud, exp, iat, groups, roles, tenant, environment, email, preferred_username`.

### 4.2 §17A.4 required (authorization) claims
`sub, preferred_username, email, groups, realm_access.roles, resource_access.platform.roles,
namespaces, policy_domains, tenants`.

### 4.3 §15.3 recommended custom claims
`risk_level, compliance_scope, workload_identity, deployment_approval, data_classification`.

### 4.4 Normative requirements
- **D1-R1 (MUST)** Verify JWT signature against the issuer's JWKS; reject on failure.
- **D1-R2 (MUST)** Validate `iss` against an allow-list of trusted issuers, `aud` includes
  `platform`, `exp`/`iat`/`nbf` within configured skew (default ±60 s).
- **D1-R3 (MUST)** Reject tokens whose `alg` is `none` or asymmetric-confused (RS/HS confusion
  guard); pin acceptable algorithms per issuer.
- **D1-R4 (MUST)** Produce the canonical subject (§2.1) for every accepted token.
- **D1-R5 (MUST)** A claim required by *any deployed policy* (per the Rego Explorer
  `__required_claims__` index, §8.3) but absent from the normalized subject MUST yield
  `normalization_status != complete` and a `missing_required` entry — never silently absent.
- **D1-R6 (MUST)** Normalization MUST be deterministic: same token + same mapping version ⇒
  byte-identical subject (modulo `token_ref.iat`). Required for replay (§17.3) and audit.
- **D1-R7 (SHOULD)** Missing/unrecognized source values map to a sentinel (`unknown`) rather
  than absence, so policies fail closed without distinguishing "missing" from "unrecognized"
  (DT-35 step 2).
- **D1-R8 (MUST)** No downstream artifact references a raw claim path; CI lint rejects Rego
  referencing `input.token.*` / `input.subject.<non-canonical>` (design tenet §1.3).
- **D1-R9 (MUST)** Mapping changes are versioned governance artifacts with actor JWT,
  timestamp, prior/new value retained for rollback (DT-36 step 5, DT-37 step 7).
- **D1-R10 (MUST)** Normalization MUST NOT perform synchronous network calls in the hot path;
  reference data comes from a versioned snapshot (§2.5).

---

## 5. The mapping-layer design (the heart of D1)

### 5.1 Pipeline stages (ordered, deterministic)
```
verify(token) → extract(raw claims)
  → S1 alias            (identity aliasing; legacy sub → canonical)
  → S2 source-select    (per target: pick source claim(s) across IdPs)
  → S3 transform        (lowercase/lookup/expand_group_hierarchy/…)
  → S4 conflict-resolve (prefer_first_non_empty / merge)
  → S5 inherit          (tenant inheritance; child ⊇ parent scope)
  → S6 normalize        (sentinel for missing/unknown; cardinality coercion)
  → S7 provenance-stamp (record source+transform+idp per field)
  → S8 validate         (required-claim coverage; status = complete|degraded|incomplete)
  → emit(canonical subject @ schema_version)
```
Stages are pure functions; the whole pipeline is a fold over mapping entries. Ordering is
fixed so the output is reproducible (D1-R6).

### 5.2 Multi-IdP tenant normalization (DT-36 / HL-13 / HL-16)
- A single `tenants` target lists **multiple source claims** (`tenant` from IdP-A, `org_id`
  from IdP-B), unified by `transform: lowercase` + `onConflict: prefer_first_non_empty`.
- Source selection MAY be **issuer-scoped**: `when: {iss: "https://okta-b/…"} use org_id`.
  This is how HL-16 onboards a federated IdP with *zero* policy edits — only the mapping
  config changes.
- Canonicalization rule: tenant strings are lowercased, trimmed, and validated against the
  tenant registry; an unknown tenant ⇒ `degraded` (subject usable, but tenant-scoped policies
  fail closed for that subject).

### 5.3 Group → role expansion at issuance (DT-38)
- `expand_group_hierarchy` walks a declared nested-group hierarchy and emits the leaf role
  **plus every ancestor role** (`team-payments` ⇒ `[role:team-payments, role:org-finance]`).
- This moves hierarchy resolution out of Rego (which previously string-matched parent group
  names — fragile, duplicated) into the mapping layer, making the §17A.2 role registry the
  single source of truth. Each emitted `role:<group>` MUST be registered in D2's role
  registry with scope + permission primitives (the D1↔D2 contract).
- Expansion may happen at **token issuance** (Keycloak protocol mapper / client scope) **or**
  at **normalization** (mapping layer). Default: normalization-time, because it keeps Keycloak
  realm config thin and the hierarchy versioned as a governance artifact. Issuance-time is an
  optimization to shrink token size when the hierarchy is stable.

### 5.4 Identity aliasing & tenant inheritance
- **Aliasing:** map a legacy/secondary `sub` to a canonical `subject_id` (mergers, re-issued
  accounts); service-account aliasing sets defaults without populating every machine identity
  (DT-35 step 3).
- **Tenant inheritance:** a child tenant's subject inherits parent tenant scope where the
  registry declares containment (e.g. `payments-eu ⊂ payments`); inheritance is explicit and
  auditable, never implicit substring matching.

---

## 6. Claim lifecycle (add / evolve / decommission)

### 6.1 Add a claim (DT-35)
1. Add a mapping entry (source, transform, normalize, identity-alias) targeting a **new
   canonical field**; bump `schema_version` if the field is new.
2. `POST /v1/mapping/dry-run` over a sampled token corpus (scoped to the actor's domain per
   §17A.5) → value distribution + `unknown` set.
3. Commit (versioned artifact); publish new normalized-subject schema version.
4. Policies reference the **canonical** field; unit-test against the dry-run fixture.
5. Promote bundle via §7 lifecycle. **No Keycloak realm change required** when the source is
   an existing attribute.

### 6.2 Evolve a claim across IdPs (DT-36 / HL-16)
Add a second source to an existing target + issuer-scoped selection; dry-run; replay failing
events through §17.3; promote. Rego unchanged.

### 6.3 Decommission a claim (DT-37) — *prove-then-remove*
1. **Prove zero use:** Rego Explorer `__required_claims__` index + rule-dependency graph
   (incl. disabled/dry-run packages) report zero references; Audit Correlation View shows zero
   decisions in the retention window where the claim was non-null and contributed.
2. Remove the mapping entry **and** the matching Keycloak protocol mapper.
3. Non-prod first: sample tokens confirm absence; §15.2 required claims intact; token size
   drops.
4. **Audit replay** over the retention window: `replay_completeness=complete` preserved for
   every event (no regression).
5. Promote; retain prior version for rollback.

> **Decommission is safe only because the dependency view *and* audit replay both confirm zero
> references.** Either alone is insufficient (Rego could reference it via external data;
> audit could miss a rarely-exercised path).

---

## 7. Failure modes

| Failure | Behavior |
|---|---|
| JWKS unreachable / key rotation lag | Serve from cached JWKS within max-age; if expired, **reject** (fail closed) rather than skip verification. |
| Unknown issuer | Reject token; emit `auth_denied{reason:untrusted_issuer}`. |
| Required claim absent | `normalization_status=incomplete`; PDP fails closed for policies needing it (D1-R5). |
| Source value unrecognized | Sentinel `unknown` (D1-R7); flagged for follow-up, not a hard reject. |
| Mapping config invalid (cycle, undefined transform) | `POST /validate` rejects at author time; deployed config is immutable + pre-validated. |
| Conflicting multi-IdP values | `onConflict` policy decides; `degraded` if unresolved. |
| External-data snapshot stale | Use last-good snapshot; flag `degraded`; alert (never block the hot path). |
| Token replay / `jti` reuse | Out of scope for D1 normalization; transport/session layer (D4) handles. |
| Clock skew beyond bound | Reject (`exp`/`nbf`); configurable skew with a hard ceiling. |

---

## 8. Security / authz notes
- D1 is a **trust boundary**: a bug here (e.g. accepting `alg:none`, RS/HS confusion, skipping
  `aud`) compromises the whole platform. D1-R1–R3 are security-critical (cross-ref D4-§).
- The normalized subject is **integrity-sensitive**: downstream authz trusts it implicitly.
  Within the platform it travels over mTLS (D4); it is never reconstructed from an untrusted
  source.
- Raw tokens are **never logged**; only `token_ref` (jti, iat, exp, aud) + `claim_provenance`.
  `email`/`sub` are PII → redaction profiles (DT-57) apply downstream.
- The mapping config is an **authz-relevant artifact**: editing it can grant scope (group→role
  expansion). Therefore mapping edits require Platform Governance Admin (§17A.2) and are
  themselves audited (§23.1 Auditability) — a mapping change *is* a privilege change.

---

## 9. Dependencies
| Depends on | For |
|---|---|
| Keycloak (§23.1) | OIDC tokens, JWKS, protocol mappers |
| D2 role registry (§17A.2/.3) | Each emitted `role:<group>` must be registered with scope+permissions |
| Domain C / C2 (§13.3) | Consumes the `jwt_claims` block; D1 defines its shape and provenance |
| §7 lifecycle | Mapping configs are versioned governance artifacts |
| Rego Explorer (§16.3) | `__required_claims__` index drives D1-R5 + decommission proof |

| Depended on by |
|---|
| D2 (every authz decision needs the canonical subject) |
| D3 (approval webhooks/CRDs carry normalized `subject`) |
| Every PDP / Rego / Gatekeeper / Conftest (`input.subject`) |
| Audit schema (`jwt_claims`) |

---

## 10. Open questions — decided defaults
| # | Question | Decided default | Rationale |
|---|---|---|---|
| OQ-1 | Group→role expansion at issuance or normalization? | **Normalization-time default**, issuance-time as size optimization. | Keeps Keycloak thin; hierarchy versioned as governance artifact (DT-38). |
| OQ-2 | Missing-claim → absent or sentinel? | **Sentinel (`unknown`)** for value-bearing claims; `missing_required` flag for required ones. | Fail-closed without ambiguity (DT-35). |
| OQ-3 | Where does multi-IdP conflict resolve? | **Mapping layer, `prefer_first_non_empty` default**, issuer-scoped sources. | Rego stays declarative (DT-36, HL-16). |
| OQ-4 | Subject schema versioning? | **Explicit `schema_version`**; additive fields bump minor, removals bump major. | Replay reproducibility (D1-R6). |
| OQ-5 | Hot-path external data? | **Forbidden**; snapshot only. | Latency + failure isolation (D1-R10). |
| OQ-6 | Is `deployment_approval` claim authoritative for approval state? | **No** — D1 normalizes it but D3's CRD/approval store is authoritative; claim is a cache hint at most. | Avoids stale-token approval bypass. |
