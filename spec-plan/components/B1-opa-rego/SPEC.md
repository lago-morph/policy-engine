# B1 — OPA / Rego Integration & Signed Bundles — SPEC

**Component ID:** B1 · **Domain:** B (Policy Engines & Enforcement) · **Spec source:** §8 (with §7.2, §17C, §18 dependencies)
**Status:** DRAFT v1 · **Date:** 2026-05-30 · **Author persona:** cooperative engineering author

---

## 1. Scope

B1 is the **decision substrate** of the platform. OPA/Rego is the single cross-product
policy evaluation engine that all other enforcement points (Gatekeeper §9/B2, Conftest
§10/B3, application PDPs §17D/E3, retrospective replay §19/C4) ultimately share semantics
with. B1 owns:

1. **Rego policy authoring conventions** — package naming, mandatory metadata, entrypoint
   contracts, input/data normalization.
2. **Bundle packaging** — building OPA bundles, versioning them, publishing them as OCI
   artifacts, **signing** them, and recording provenance traceable to Gemara control IDs.
3. **Bundle distribution & activation** — how OPA instances (sidecars, the central
   decision service, Gatekeeper's embedded OPA, Conftest) discover, verify, and load bundles.
4. **Decision logging** — the structured decision log every evaluation emits, the field
   contract, and its routing to the audit pipeline (C2).
5. **The OPA REST / Wasm evaluation surface** consumed by application PDPs and the
   simulation engine (E1).

**In scope:** Rego metadata schema, bundle layout, OCI signing/versioning, decision-log
schema, bundle verification at load time, the bundle-build pipeline, conformance with
shared decision semantics.

**Out of scope (owned elsewhere):** Gatekeeper Constraint/ConstraintTemplate wiring (B2),
Conftest CI wiring (B3), CRD controllers (B4), the realtime admission sequence (B5),
Gemara control authoring (A1), policy lifecycle/promotion gates (A2), audit-schema
normalization downstream of the decision log (C2).

---

## 2. Background decisions driven by 2025–2026 market shifts

Per `policy engine market research.md` §2: **Styra was acqui-hired by Apple (Aug 2025); the
commercial OPA stack (EOPA, OPA Control Plane / OCP, Regal, SDKs) is being open-sourced into
CNCF; Styra DAS is sunsetting.** This is load-bearing for B1.

- **D-B1-01 (decided):** Adopt **OPA Control Plane (OCP)** as the bundle-build and
  distribution substrate rather than building a bespoke bundle pipeline. OCP already builds
  bundles from multiple Git repos, supports external HTTP/push datasources, HA bundle
  distribution to S3/GCS/Azure Blob, build-time injection of global/hierarchical policies via
  label selectors, and regression testing bundles against historical decision logs. Rationale:
  these primitives overlap §7/§8.2/§14/§17 almost line-for-line; reimplementing them is waste.
  **Risk:** OCP is freshly open-sourced and its API surface/maturity is unproven — see ADR
  and ADVERSARIAL-REVIEW. Mitigation: wrap OCP behind a thin `BundleService` interface (§6.4)
  so it is swappable; do not let OCP-specific concepts leak into Rego or CRDs.
- **D-B1-02 (decided):** Standardize on **upstream CNCF OPA + Regal (lint)** for evaluation
  and authoring. Do **not** hard-depend on EOPA proprietary builtins; if EOPA is adopted for
  performance (Wasm, VM), its use MUST be limited to runtime acceleration and MUST NOT change
  decision results (conformance test enforces this — §10).
- **D-B1-03 (decided):** Signing uses **Sigstore cosign over the OCI bundle artifact**, with
  keyless (Fulcio/Rekor) as the default and KMS keys as an enterprise option. Aligns with the
  platform's broader supply-chain posture (§20.1) and Kyverno/Gatekeeper image-verification
  patterns already in the corpus.

---

## 3. Data model / entities

| Entity | Description | Key fields |
|---|---|---|
| **RegoPackage** | One governance policy package | `package` path, metadata block (§4), entrypoints |
| **PolicyBundle** | A built, versioned, signed OPA bundle | `bundle_id`, `version` (semver), `revision`, `git_provenance`, `manifest`, `control_ids[]`, `signature_ref` |
| **BundleManifest** | OPA `.manifest` describing roots, revision, metadata | `roots[]`, `revision`, `metadata{}` |
| **OCIArtifactRef** | The published artifact | `registry`, `repository`, `tag`, `digest` (sha256) |
| **Signature** | cosign signature + attestation | `signature_digest`, `rekor_log_index`, `cert_identity`, `attestation_predicate` |
| **DecisionLog** | One evaluation record | see §7 |
| **Entrypoint** | A named decision rule a consumer queries | `path` (e.g. `governance/kubernetes/imagesigning/decision`), `result_schema` |

**Relationships:** `RegoPackage *→1 Gemara control` (via `__control_id__`); `PolicyBundle 1→* RegoPackage`;
`PolicyBundle 1→1 OCIArtifactRef`; `OCIArtifactRef 1→* Signature`; `DecisionLog *→1 PolicyBundle` (via bundle `revision`).

---

## 4. Rego metadata extensions (normative)

Extends §8.3. Two complementary mechanisms are REQUIRED:

### 4.1 Package-level annotation variables (spec §8.3 baseline)

```rego
package governance.kubernetes.imagesigning

__control_id__        := "SC-IMG-001"
__severity__          := "critical"            # critical|high|medium|low|info
__governance_domain__ := "supply-chain"
__required_claims__   := ["groups", "tenant", "environment"]
```

### 4.2 OPA native metadata annotations (METADATA blocks) — REQUIRED additionally

Plain Go variables are invisible to OPA's metadata/inspection tooling and to OCP build-time
processing. Therefore every package and every entrypoint rule MUST **also** carry an OPA
`# METADATA` block so the data is machine-discoverable via `opa inspect` and the bundle manifest:

```rego
# METADATA
# title: Image signing enforcement
# description: Prevents unsigned workloads (Gemara SC-IMG-001)
# authors:
#   - platform-governance
# related_resources:
#   - https://gemara.example.io/controls/SC-IMG-001
# custom:
#   control_id: SC-IMG-001
#   severity: critical
#   governance_domain: supply-chain
#   required_claims: [groups, tenant, environment]
#   enforcement_classes: [runtime, build-time, detective]   # §7.2
#   engines: [opa, gatekeeper, conftest]
#   replay_schema_ref: SC-IMG-001-replay-v1
package governance.kubernetes.imagesigning
```

### 4.3 Normative requirements

- **B1-R1 (MUST):** Every governance Rego package MUST declare both the `__control_id__`
  variable **and** a `# METADATA` `custom.control_id` with the **same** value. The build
  pipeline MUST reject a bundle where they disagree or either is absent.
- **B1-R2 (MUST):** `custom.control_id` MUST reference a control that exists in the Gemara
  catalog (A1). The build pipeline MUST resolve and fail-closed on unknown control IDs.
- **B1-R3 (MUST):** `custom.severity` MUST be one of `critical|high|medium|low|info`.
- **B1-R4 (MUST):** `custom.enforcement_classes` MUST be a non-empty subset of
  `{runtime, build-time, detective, manual, advisory}` (§7.2).
- **B1-R5 (SHOULD):** `custom.required_claims` SHOULD list every JWT claim the decision reads,
  so the simulation/replay engine (E1/C4) can detect insufficient replay schemas (DT-25).
- **B1-R6 (MUST):** Package path MUST follow `governance.<product>.<capability>` (e.g.
  `governance.kubernetes.imagesigning`, `governance.cicd.iac`, `governance.identity.tokens`).
- **B1-R7 (MUST):** Regal lint MUST pass in CI with the platform ruleset (which enforces
  R1–R6 via custom Regal rules) before a package is eligible for bundling.

---

## 5. Entrypoint / decision contract (normative)

To guarantee cross-engine consistency (a hard requirement — the same Rego must yield the
same decision whether invoked by Gatekeeper, Conftest, an app PDP, or replay), B1 defines a
**uniform decision result shape**. Consumers MUST query a single canonical entrypoint per
package: `<package path>/decision`.

```jsonc
// result of data.governance.kubernetes.imagesigning.decision
{
  "control_id": "SC-IMG-001",
  "action": "deny",                 // one of the 13-action taxonomy (§17C.3 / B4)
  "allowed": false,                 // convenience boolean; allowed == (action in {allow,warn,annotate,notify})
  "severity": "critical",
  "messages": ["Unsigned image prohibited: ghcr.io/acme/api@<no-sig>"],
  "obligations": [                  // structured side-effects for the caller to execute
    { "type": "annotate", "key": "governance/violation", "value": "SC-IMG-001" }
  ],
  "approval": null,                 // populated when action == require_approval (see B4 / §17B)
  "evidence": {                     // minimal replay fields the decision actually used
    "image_digest": "sha256:...",
    "subject": { "sub": "user-123", "groups": ["team-payments"], "tenant": "acme" }
  },
  "policy": { "bundle_revision": "<revision>", "package": "governance.kubernetes.imagesigning" }
}
```

- **B1-R8 (MUST):** Every governance package MUST expose a `decision` rule conforming to this
  schema. Raw `deny[msg]` / `violation[...]` rules MAY exist internally (Gatekeeper-style),
  but the canonical `decision` rule MUST aggregate them.
- **B1-R9 (MUST):** `action` MUST be drawn from the 13-action taxonomy owned by B4. B1 does
  not redefine it; B4 is authoritative. `allowed` is derived, never independently authored.
- **B1-R10 (MUST):** A package MUST be **pure**: no network calls, no unbounded iteration over
  unbounded data. All external facts come via `data` (bundle data or external_data documents),
  never via Rego `http.send` at decision time (R-perf, replayability).
- **B1-R11 (SHOULD):** `evidence` SHOULD contain exactly the fields the decision read from
  `input`, to support DT-25 replay-completeness detection and minimize audit-log bloat.

---

## 6. Bundle packaging, signing, versioning, distribution

### 6.1 Bundle layout (REQUIRED on-disk/OCI structure)

```
bundle.tar.gz
├── .manifest                         # roots, revision, metadata
├── governance/
│   ├── kubernetes/
│   │   └── imagesigning/
│   │       ├── policy.rego
│   │       └── policy_test.rego       # co-located unit tests (stripped from runtime bundle build)
│   ├── cicd/iac/policy.rego
│   └── lib/                           # shared helper packages (governance.lib.*)
└── data/
    ├── controls.json                 # control-id → metadata snapshot (built from Gemara catalog A1)
    └── external/                      # snapshot of external_data documents (e.g. allowed registries)
```

- **B1-R12 (MUST):** `.manifest` MUST set `roots` to the explicit governance roots owned by
  the bundle (no implicit `[""]` root, to allow safe multi-bundle composition / delta bundles).
- **B1-R13 (MUST):** `.manifest.revision` MUST be set to `<semver>+<git-sha>` and MUST be
  globally unique per published artifact. The revision is what decision logs reference.
- **B1-R14 (MUST):** `.manifest.metadata` MUST embed: `bundle_id`, `version` (semver),
  `built_at`, `git_provenance` (repo, commit, dirty=false), `control_ids[]` (sorted), and the
  Gemara catalog snapshot version used.

### 6.2 OCI publishing, versioning, signing (REQUIRED)

- **B1-R15 (MUST):** Bundles MUST be published as **OCI artifacts** (ORAS media type
  `application/vnd.openpolicyagent.bundle.layer.v1+tar.gz`) to the platform registry.
- **B1-R16 (MUST):** Versioning is **semver** on the artifact tag **plus** an immutable
  content `digest`. Consumers in production MUST pin by **digest**, never floating tags.
  Tags (`v1`, `v1.4`, `latest`, `stable`) are convenience aliases only.
- **B1-R17 (MUST):** Every published bundle MUST be **signed with cosign** (keyless Fulcio/Rekor
  default; KMS optional). The signature MUST cover the digest.
- **B1-R18 (MUST):** Every published bundle MUST carry a **provenance attestation**
  (in-toto/SLSA predicate) recording: source repos+commits, builder identity, Gemara catalog
  version, Regal-lint pass, unit-test pass, and the list of `control_ids` included. This makes
  the decision substrate itself auditable (HL-05, DT-10).
- **B1-R19 (MUST):** Consumers (OPA agents, Gatekeeper, Conftest) MUST **verify the signature
  and attestation before activating** a bundle. Verification failure MUST fail-closed: the
  consumer MUST NOT load the unverified bundle and MUST keep serving the last-known-good bundle,
  emitting a `bundle.verification_failed` event to audit (C2). See ADVERSARIAL §"poisoned bundle".
- **B1-R20 (SHOULD):** Cosign verification identity (the allowed signer cert identity /
  OIDC issuer) MUST be configured per environment and SHOULD itself be governance-controlled.

### 6.3 Activation / hot-reload

- **B1-R21 (MUST):** Bundle activation MUST be atomic; a partially downloaded or
  verification-failing bundle MUST never become active.
- **B1-R22 (MUST):** On activation, the agent MUST emit a `bundle.activated` audit event
  carrying old→new revision, so retrospective analysis (C4/§19) can correlate decision changes
  to bundle changes and detect silent regressions (HL-12).
- **B1-R23 (SHOULD):** Activation SHOULD support **staged rollout** (canary a fraction of
  agents) and instant rollback to the previous digest (DT-06 rollback).

### 6.4 BundleService abstraction (insulates from OCP)

A thin internal interface so OCP is swappable (D-B1-01):

```
BuildBundle(sources, gemaraCatalogVersion) -> PolicyBundle
SignAndPublish(PolicyBundle) -> OCIArtifactRef + Signature
ListBundles() / GetBundle(id, version|digest)
RegressionTest(PolicyBundle, decisionLogWindow) -> DiffReport   # backed by OCP regression testing
Promote(bundle, env)  /  Rollback(env, toDigest)
```

- **B1-R24 (MUST):** No OCP-specific type or label-selector syntax may appear in Rego, in CRDs
  (B4), or in the decision-log schema. OCP is an implementation detail behind BundleService.

---

## 7. Decision-log schema (normative)

OPA emits a decision log per evaluation. B1 fixes the **minimum** field contract; C2 normalizes
it into the platform audit schema and the replay schema (§13).

```jsonc
{
  "decision_id": "uuid",
  "timestamp": "2026-05-12T00:00:00Z",
  "bundle_revision": "1.4.0+ab12cd3",
  "path": "governance/kubernetes/imagesigning/decision",
  "input": { /* the evaluated input, subject to redaction policy */ },
  "result": { /* the §5 decision object */ },
  "control_id": "SC-IMG-001",
  "engine": "opa",                  // opa | gatekeeper-opa | conftest | app-pdp
  "pdp_type": "application",        // §17C.4 PDP typology
  "correlation_id": "uuid",         // ties admission/CI/app request → decision → audit (B5)
  "subject": { "sub": "...", "groups": [], "tenant": "..." },
  "nd_builtin_cache": { /* time, rand, etc. captured for deterministic replay */ }
}
```

- **B1-R25 (MUST):** Decision logs MUST include `decision_id`, `timestamp`, `bundle_revision`,
  `path`, `control_id`, `correlation_id`, and `result`.
- **B1-R26 (MUST):** Non-deterministic builtins used by a decision (`time.now_ns`, `rand.*`)
  MUST be captured (OPA's `nd_builtin_cache`) so the decision can be **replayed deterministically**
  (§17.4 / C4). A package that reads wall-clock time without surfacing it for capture violates R10/R26.
- **B1-R27 (MUST):** `input` logging MUST honor a **redaction policy** (configurable) so secrets/PII
  are masked before the log leaves the agent (D4 security). The redaction MUST be applied at source.
- **B1-R28 (MUST):** Decision logs MUST be exportable to the audit pipeline (C2) and, on failure
  to export, MUST be buffered locally with backpressure rather than dropped (evidence integrity).

---

## 8. Interfaces / APIs

| Interface | Direction | Notes |
|---|---|---|
| OPA REST `POST /v1/data/<path>` | inbound (app PDP, E1 sim) | returns §5 decision object |
| OPA Wasm module | inbound (embedded app PDP §17D/E3) | compiled per entrypoint; results MUST match REST (R-conformance) |
| Bundle pull (HTTP/OCI) | inbound (agents) | signature verified before activate (R19) |
| Decision-log sink | outbound | to C2 audit pipeline; correlation_id preserved |
| `external_data` documents | inbound (data/) | versioned, snapshotted into bundle; DT-27 drift detection |
| BundleService API | internal | §6.4 |

- **B1-R29 (MUST):** The REST result and the Wasm result for the same entrypoint+input MUST be
  byte-identical after canonical JSON serialization (conformance test, §10).

---

## 9. Failure modes

| # | Failure | Required behavior |
|---|---|---|
| F1 | Bundle signature/attestation invalid | Fail-closed: do not activate; keep last-good; emit `bundle.verification_failed` (R19) |
| F2 | Bundle server unreachable | Continue on last-good bundle; emit staleness metric; alert if staleness > threshold |
| F3 | No bundle ever loaded (cold start) | PDP MUST fail-closed for runtime enforcement classes (deny) and fail-open ONLY for advisory; configurable per consumer, default deny |
| F4 | Rego eval error / undefined `decision` | Treat as engine failure → fail-closed for enforcement; emit `decision.error`; never silently allow |
| F5 | external_data version drift (DT-27) | Decision MUST embed the external_data snapshot version in `evidence`; mismatch at replay flagged by C4 |
| F6 | Decision-log sink down | Buffer + backpressure (R28); never drop evidence |
| F7 | Gemara control referenced by Rego no longer exists | Build-time failure (R2); runtime decisions for a deprecated control flagged by C4 |
| F8 | Clock skew affecting nd builtins | Capture nd_builtin_cache (R26); replay uses captured values, not live clock |

---

## 10. Conformance & test strategy hooks (detail in PLAN)

- **B1-R30 (MUST):** A **cross-engine conformance suite** MUST exist: a corpus of
  (package, input, expected decision) golden cases run against (a) OPA REST, (b) OPA Wasm,
  (c) Gatekeeper-embedded OPA (B2), (d) Conftest (B3). All four MUST agree. This is the
  guardrail for the platform's central claim of shared semantics (§17C cross-product consistency).
- **B1-R31 (MUST):** Every governance package MUST ship `*_test.rego` unit tests; CI MUST
  enforce a coverage floor (default 80% of decision branches) via `opa test --coverage`.
- **B1-R32 (SHOULD):** Regression testing (BundleService.RegressionTest, backed by OCP) MUST run
  candidate bundles against a window of historical decision logs and surface decision-flip
  diffs before promotion (feeds A2 promotion gates and E1 differential simulation, DT-49).

---

## 11. Security / authz notes

- Bundle build identity, signing keys, and registry write access are privileged (D4); only the
  policy-build CI identity may publish. Author identity is recorded in provenance (R18).
- Decision-log redaction (R27) is a D4 requirement; raw `input` may contain JWTs/PII.
- Cosign verification identity config (R20) is a tamper target — it MUST be managed as
  governance-controlled config, not free-form per-agent flags.
- The platform MUST NOT depend on EOPA proprietary features for *correctness* (D-B1-02), to
  avoid a single-vendor lock that contradicts the open-source posture post-Styra/Apple.

---

## 12. Dependencies on other components

| Depends on | For |
|---|---|
| A1 Gemara model | control catalog; control_id resolution (R2) |
| A2 Policy lifecycle | promotion gates, dry-run→warn→enforce, rollback (consumes RegressionTest) |
| B4 Action taxonomy & CRDs | authoritative 13-action list (R9); `require_approval`/`exception` semantics |
| C2 Audit schema | normalizes decision logs; owns replay schema (§13) |
| C4 Retrospective detection | consumes nd_builtin_cache & bundle.activated events |
| D1 Keycloak/JWT | claim extraction for `subject`/`required_claims` |
| D4 Security | signing, redaction, key management |
| E1 Simulation | uses REST/Wasm + RegressionTest for dry-run/replay |

**Consumed by:** B2 (Gatekeeper embeds OPA), B3 (Conftest runs Rego), B5 (realtime flow), E3 (per-product PDP libs).

---

## 13. Open questions (each with a decided default)

| # | Question | Decided default | Rationale |
|---|---|---|---|
| OQ1 | Build bespoke bundle pipeline or adopt OCP? | **Adopt OCP behind BundleService** | Avoid reimplementing §7/§8.2/§14/§17 primitives; keep swappable (D-B1-01, R24) |
| OQ2 | Keyless cosign vs KMS keys? | **Keyless default, KMS optional** | Lower key-management burden; KMS for air-gapped/enterprise |
| OQ3 | EOPA for performance? | **Allowed for runtime accel only, MUST NOT change results** | Avoid lock-in; conformance test enforces (D-B1-02, R29) |
| OQ4 | One canonical `decision` entrypoint vs many ad-hoc rules? | **One canonical `decision` per package** | Cross-engine consistency (R8) |
| OQ5 | Cold-start fail-open vs fail-closed? | **Fail-closed for runtime; configurable** | Security default; advisory may fail-open (F3) |
| OQ6 | Capture nd builtins always? | **Yes, when used** | Deterministic replay is a platform differentiator (R26) |
| OQ7 | Single mega-bundle vs per-domain bundles? | **Per-governance-domain bundles with explicit roots** | Independent rollout/rollback, blast-radius control (R12) |

---

## 14. Traceability

- **Spec:** §8 (8.1–8.3), §7.2, §17C.3 (actions), §18 (realtime), §20.1 (supply chain).
- **Scenarios:** DT-10 (sign/version OCI), DT-11 (validate metadata), DT-12 (embedded app OPA evidence),
  DT-13 (trace decision→bundle→control), DT-25 (replay completeness), DT-27 (external-data drift),
  DT-06 (rollback), HL-02 (image-signing rollout), HL-12 (silent regression), HL-05 (external audit).
- **Personas:** Marcus (platform), Jess (SRE/on-call), Priya (compliance), Daniel (auditor).
