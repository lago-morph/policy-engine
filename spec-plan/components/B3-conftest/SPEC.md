# B3 — Conftest Integration — SPEC

**Component ID:** B3 · **Domain:** B · **Spec source:** §10 (with §8/B1, §7.2, §17C deps)
**Status:** DRAFT v1 · **Date:** 2026-05-30 · **Author persona:** cooperative engineering author

---

## 1. Scope

B3 makes **Conftest** the **build-time** enforcement class (§7.2) — the place governance Rego
runs in CI/CD and on developers' machines (pre-commit) against IaC and config artifacts,
*before* anything reaches the cluster. Conftest is the "shift-left" twin of Gatekeeper: same
governance controls, evaluated earlier on declarative artifacts. B3 owns:

1. Running governance Rego (from B1 bundles) over the supported input types (§10.2).
2. CI integration (pipeline gates) and **pre-commit** (local developer) integration.
3. **Normalized evidence output** (§10.3) so build-time decisions become first-class platform
   evidence, correlatable with runtime decisions (B2) for the same control.

**In scope:** input loading/parsing for all §10.2 types; Conftest bundle consumption (pulling
signed B1 bundles); the normalized evidence schema and emission; CI gate behavior (fail the
build / warn); pre-commit hook packaging; mapping Conftest's `deny`/`warn`/`violation` results
onto the platform's action taxonomy and the canonical control_id.

**Out of scope:** Rego authoring/signing (B1); Kubernetes admission (B2); other IaC scanners
(Checkov/Trivy/KICS — noted as alternatives in §2 below, not implemented here); CRDs (B4);
audit normalization downstream (C2 consumes B3 evidence).

---

## 2. Background (market context, §4 market research)

- Conftest is "a thin Rego wrapper for IaC" and **the only Rego-native scanner**. The platform's
  governance model emits Rego, so Conftest is the natural CI side — **even though** Checkov has
  more built-in checks, Trivy absorbed tfsec, and KICS has the broadest format support.
  - **D-B3-01 (decided):** Use Conftest as the Rego-native gate, but **do not claim Conftest
    replaces a dedicated IaC scanner.** The platform's value is *governance-traceable* checks
    (control_id → Rego → CI evidence), not breadth of generic misconfig rules. Where breadth is
    needed, the platform SHOULD ingest external scanner output (Trivy/Checkov) as additional
    evidence via the same normalized schema (§5) — but that ingestion is a C-domain concern; B3
    only guarantees the Conftest path. (Terrascan is archived as of Nov 2025 — not a target.)
- Workflow-layer gates (GitLab Compliance frameworks, GitHub Environments/Required Reviewers)
  complement Conftest; B3's evidence MUST be consumable by those gates (exit codes + report files).

---

## 3. Data model / entities

| Entity | Description | Key fields |
|---|---|---|
| **ConftestRun** | One invocation (CI job or pre-commit) | `run_id`, `trigger` (ci/pre-commit), `pipeline`, `commit`, `bundle_revision`, `inputs[]` |
| **InputArtifact** | A parsed artifact under test | `type` (k8s/helm/terraform/json/oci/sbom/tekton/gha), `path`, `parsed_doc` |
| **ConftestResult** | Per-(artifact,policy) result | `control_id`, `policy_package`, `resource`, `decision` (deny/warn/allow), `messages[]` |
| **NormalizedEvidence** | The §10.3 evidence record emitted to the platform | see §5 |

**Relationships:** `ConftestRun 1→* InputArtifact`; each `ConftestResult → Gemara control` via the
package's `__control_id__` (B1); `ConftestRun → B1 PolicyBundle` via `bundle_revision`.

---

## 4. Supported inputs (normative — §10.2)

- **R-B3-1 (MUST):** B3 MUST support evaluating: Kubernetes YAML, Helm charts (rendered),
  Terraform plans (JSON plan), generic JSON, OCI metadata, SBOMs, Tekton pipelines, GitHub Actions
  workflows.
- **R-B3-2 (MUST):** Helm charts MUST be **rendered** (`helm template`) before evaluation so the
  policy sees the actual manifests (DT-19), and the rendered values/release context MUST be
  recorded in evidence (so the same chart with different values is distinguishable).
- **R-B3-3 (MUST):** Terraform MUST be evaluated against the **plan JSON** (`terraform show -json`),
  not raw HCL, so resource attributes are resolved (DT-20). The plan's input variables/workspace
  MUST be captured in evidence.
- **R-B3-4 (MUST):** Each input type MUST have a defined **parser → normalized input** mapping so
  the same governance package can evaluate it. This is where B3 reconciles Conftest's bare-`input`
  convention with the cross-engine input-normalization contract (see §6 / cross-cut with B1/B2).
- **R-B3-5 (SHOULD):** SBOM and OCI-metadata inputs SHOULD map image references and digests into
  the same `evidence.image_digest` field used by the runtime path (B2), so build-time and runtime
  signing decisions for the same image are correlatable (HL-02 supply-chain end-to-end).

---

## 5. Normalized evidence output (normative — §10.3)

The §10.3 example is the floor; B3 emits a superset that aligns with the OPA decision log (B1)
and the platform audit schema (C2), so build-time evidence sits in the same store as runtime.

```jsonc
{
  "control_id": "SC-IMG-001",
  "policy_package": "governance.kubernetes.imagesigning",
  "resource": "deployment/api-server",
  "decision": "deny",                  // mapped to taxonomy: deny|warn|allow (build-time subset)
  "action": "deny",                    // §17C.3 taxonomy (B4) — build-time actions are a subset
  "evidence_type": "build-time",       // §7.2 enforcement class
  "pipeline": "github-actions",        // or gitlab-ci, jenkins, tekton, pre-commit
  "timestamp": "2026-05-12T00:00:00Z",
  "bundle_revision": "1.4.0+ab12cd3",  // B1 provenance — which policy version decided
  "commit": "deadbeef",                // VCS provenance
  "repository": "acme/api",
  "input_type": "helm",                // §10.2
  "resource_ref": { "kind": "Deployment", "name": "api-server", "namespace": "payments" },
  "messages": ["Unsigned image prohibited: ghcr.io/acme/api@<no-sig>"],
  "evidence": { "image_digest": "sha256:...", "chart": "api@1.2.0", "values_hash": "..." },
  "correlation_id": "uuid",            // ties this CI check to the eventual admission (B2/B5)
  "severity": "critical"
}
```

- **R-B3-6 (MUST):** Every ConftestResult MUST normalize into this schema and include
  `control_id`, `policy_package`, `decision`, `evidence_type=build-time`, `pipeline`, `timestamp`,
  `bundle_revision`, and VCS provenance (`commit`, `repository`).
- **R-B3-7 (MUST):** `correlation_id` MUST be generatable and propagatable so a build-time deny and
  a later runtime deny for the same artifact/control can be correlated (DT-13 trace, HL-02). When a
  CI run produces an artifact that is later deployed, the correlation_id SHOULD flow via image
  annotation / attestation so B2 can pick it up.
- **R-B3-8 (MUST):** Evidence MUST be emitted in a machine-readable file (JSON) **and** Conftest's
  native exit code MUST drive the CI gate (non-zero on deny). The two are independent: evidence is
  for the platform store (C2), exit code is for the pipeline.
- **R-B3-9 (SHOULD):** Evidence emission to the platform store SHOULD be best-effort-async and MUST
  NOT block the developer's pipeline if the store is unreachable; failure to emit evidence is logged
  but does not change the gate decision (the gate is the exit code, which is local).

---

## 6. Input normalization contract (cross-cut, normative)

This is the resolution of the cross-engine concern raised in B1-AR-2 / B2-AR-6: the *same*
governance package must evaluate Gatekeeper's `input.review.object`, Conftest's bare parsed
document, and an app PDP's input.

- **R-B3-10 (MUST):** Governance packages MUST read input through a **normalization shim** so the
  policy logic operates on a canonical resource shape, regardless of caller. For Conftest, B3
  provides an adapter that wraps each parsed artifact into the canonical shape:
  `{ "review": { "object": <parsed doc> }, "subject": <ci identity>, "context": {...} }` — i.e.
  Conftest input is shaped to *match the admission envelope* so one package works for both.
  Rationale: choosing the admission envelope as canonical means Gatekeeper needs no shim and only
  Conftest/app-PDP wrap; the alternative (canonical = bare object) would require a Gatekeeper shim.
- **R-B3-11 (MUST):** The cross-engine conformance suite (B1-R30) MUST include Conftest as one of
  the four engines and MUST feed it via this shim, proving identical `decision` output for the same
  logical resource. If the shim cannot make Conftest agree, the package is non-conformant.

---

## 7. CI & pre-commit integration (normative)

### 7.1 CI

- **R-B3-12 (MUST):** B3 MUST provide ready-to-use CI integrations for at least GitHub Actions,
  GitLab CI, Jenkins, and Tekton (the §10.2 pipeline types + §17D product libs). Each pulls the
  **signed B1 bundle** (verifying signature, B1-R19) before evaluating.
- **R-B3-13 (MUST):** The CI gate MUST fail the build on any `deny`, surface `warn` as a non-fatal
  annotation, and emit normalized evidence for all results (deny/warn/allow-with-note). The set of
  controls enforced at build-time is governance-configured per repo/pipeline (not every control
  applies at build-time; §7.2 class membership decides).
- **R-B3-14 (SHOULD):** CI integration SHOULD post results as PR/MR annotations (DT-45 pre-PR
  simulation overlaps E1) and SHOULD support a `--simulate` mode that evaluates against a candidate
  bundle for differential testing before promotion.

### 7.2 Pre-commit (local)

- **R-B3-15 (MUST):** B3 MUST ship a **pre-commit hook** (and a CLI wrapper) so developers get the
  same governance decisions locally that they'd get in CI (DT-18). It MUST pull/verify the same
  signed bundle (or use a locally cached, signature-verified copy).
- **R-B3-16 (SHOULD):** The local hook SHOULD be fast (incremental — only changed files) and SHOULD
  degrade gracefully offline (use last-verified cached bundle; warn that it may be stale).
- **R-B3-17 (MUST):** Local pre-commit evidence MUST be marked `trigger: pre-commit` and MUST NOT be
  treated as authoritative compliance evidence on its own (a developer can bypass a local hook); the
  **CI gate is the enforcement point of record** (build-time). Local is advisory acceleration.

---

## 8. Failure modes

| # | Failure | Required behavior |
|---|---|---|
| F1 | Bundle signature invalid (B1-R19) | Fail the run (fail-closed for build-time enforcement); do not evaluate with an unverified bundle |
| F2 | Bundle server unreachable in CI | Use last-verified cached bundle if present (record staleness); else fail the run loudly (no silent skip) |
| F3 | Parser failure (malformed Helm/TF/YAML) | Treat as a failed run (cannot prove compliance ⇒ do not pass); emit `parse_error` evidence |
| F4 | Evidence store unreachable | Best-effort async; log; do NOT change the gate (R-B3-9) |
| F5 | Helm/TF not rendered/planned | Fail with actionable error (R-B3-2/3): policy can't evaluate raw templates correctly |
| F6 | Developer bypasses local hook | Acceptable; CI gate is authoritative (R-B3-17); §19/detective catches what slips |
| F7 | Conftest result has no control_id (ad-hoc Rego) | Reject/flag: governance evidence MUST be control-traceable (B1-R2) |

---

## 9. Security / authz notes

- CI runners pull signed bundles; signature verification in CI is mandatory (F1) — a poisoned
  bundle in CI is as dangerous as in the cluster.
- Pre-commit runs on developer machines (untrusted); local evidence is non-authoritative (R-B3-17).
- Evidence may contain config snippets with secrets — redaction policy (B1-R27 analog) MUST apply
  before evidence leaves CI (D4).
- The CI identity that emits evidence must be authenticated to the platform store (no anonymous
  evidence injection — otherwise §19 detection can be spoofed with fake "passed" evidence).

---

## 10. Dependencies

| Depends on | For |
|---|---|
| B1 | signed bundles, canonical `decision`, control_id metadata, conformance suite membership |
| B4 | action taxonomy (build-time subset); engine selection (Conftest is build-time leg) |
| C2 | normalized-evidence sink; correlation across build-time + runtime |
| A2 | which controls are build-time class; lifecycle/promotion (build-time can run in simulate) |
| D1/D4 | CI identity, redaction |
| E1 | `--simulate`/differential overlaps the simulation framework (DT-45) |

**Consumed by:** C2 (evidence), C3 (analytics — coverage of build-time vs runtime), HL-02 supply chain.

---

## 11. Open questions (decided defaults)

| # | Question | Default | Rationale |
|---|---|---|---|
| OQ1 | Conftest only, or bundle external scanners? | **Conftest is the guaranteed Rego-native path; external scanner ingestion is a C-domain add-on** | Governance traceability > generic breadth (D-B3-01) |
| OQ2 | Canonical input = admission envelope or bare object? | **Admission envelope (Conftest wraps to match)** | Minimizes shims; Gatekeeper needs none (R-B3-10) |
| OQ3 | Is pre-commit authoritative evidence? | **No; CI is the build-time enforcement of record** | Local hooks are bypassable (R-B3-17) |
| OQ4 | Fail or warn when bundle unreachable in CI? | **Fail loudly (no silent skip), unless valid cached bundle** | Silent skip = invisible compliance gap |
| OQ5 | Block pipeline if evidence store down? | **No — gate is local exit code; evidence is best-effort async** | Don't couple dev velocity to evidence-store uptime (R-B3-9) |

---

## 12. Traceability

- **Spec:** §10 (10.1–10.3), §7.2 (build-time class), §17C.3/4 (CI/CD PDP), §20.1 (supply chain).
- **Scenarios:** DT-07 (build-time-only Conftest policy), DT-18 (pre-commit local), DT-19 (Helm in CI),
  DT-20 (Terraform plan), DT-21 (normalize evidence), DT-45 (manifest simulation pre-PR), DT-13 (trace),
  HL-02 (image-signing end-to-end), HL-04 (developer onboarding through gates).
- **Personas:** Sam (developer), Marcus (platform), Priya (compliance — build-time evidence).
