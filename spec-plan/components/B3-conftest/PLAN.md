# B3 — Conftest Integration — PLAN

**Component:** B3 · **Pairs with:** SPEC.md · **Date:** 2026-05-30

---

## 1. Dependency DAG

```
[B1 signed bundle + decision + conformance] ──► W1 Bundle pull/verify in CI & local
[input-normalization contract (B1/B2/B3)] ─────► W2 Input adapters (k8s/helm/tf/json/oci/sbom/tekton/gha)
                                                     │
                                                     ▼
                                          W3 Result→NormalizedEvidence mapper ──► [C2]
                                                     │
                              ┌──────────────────────┼───────────────────────┐
                              ▼                       ▼                        ▼
                    W4 CI integrations       W5 Pre-commit hook        W6 Conformance (Conftest leg)
                    (GHA/GitLab/Jenkins/Tekton)  (local CLI)            (extends B1-R30)
```

Critical path: **W2 (input adapters) → W3 (evidence) → W4 (CI)**.

---

## 2. Parallelizable workstreams

| WS | Title | Deps | Parallel with |
|---|---|---|---|
| W1 | Bundle pull + signature verify (CI + cached local) | B1 publish/verify | W2 |
| W2 | Input adapters per §10.2 type + admission-envelope wrap (R-B3-2/3/4/10) | input-norm contract | W1 |
| W3 | Result→NormalizedEvidence (R-B3-6/7) + correlation_id propagation | W2 | W4, W5 |
| W4 | CI integrations (GHA/GitLab/Jenkins/Tekton) + gate (R-B3-12/13) | W1, W3 | W5 |
| W5 | Pre-commit hook + CLI (R-B3-15/16/17) | W1, W3 | W4 |
| W6 | Conformance: Conftest as 4th engine in B1 corpus (R-B3-11) | W2 shim | W3, W4 |

---

## 3. Critical path & milestones

- **M1 — Verified bundle eval (W1):** Conftest pulls + verifies a signed B1 bundle and evaluates a
  K8s manifest, producing the canonical `decision`. (DT-07.)
- **M2 — All input types (W2):** Helm rendered, TF plan JSON, SBOM/OCI, Tekton, GHA all parse →
  normalized input. (DT-19/20.)
- **M3 — Normalized evidence (W3):** every result emits §10.3-superset evidence with provenance +
  correlation_id. (DT-21.)
- **M4 — CI gates live (W4):** build fails on deny, warns on warn, evidence emitted, on 4 platforms.
- **M5 — Pre-commit (W5):** local hook gives same decisions; offline-degraded; marked non-authoritative.
- **M6 — Conformance green (W6):** Conftest agrees with OPA REST/Wasm/Gatekeeper on the golden corpus
  via the input shim. **Gate for the cross-engine claim on the build-time leg.**

---

## 4. Test strategy

1. **Per-input-type parsing:** golden artifacts for each §10.2 type → correct normalized input;
   Helm rendered (not raw), TF from plan JSON (not HCL).
2. **Evidence normalization (DT-21):** result → §10.3-superset; all MUST fields present; correlation_id
   present and stable; control_id traces to a real Gemara control (reject ad-hoc Rego, F7).
3. **Conformance (M6):** the B1 golden corpus run through the Conftest shim equals the other engines.
   This is the highest-value B3 test — it proves "same control, shifted left."
4. **CI gate behavior:** deny → non-zero exit/build fail; warn → annotation, build passes; evidence
   emitted in all cases; bundle-unreachable → cached-or-fail-loud (F2), never silent skip.
5. **Pre-commit:** same decision locally as CI; offline uses cached verified bundle + staleness warn;
   bypass → CI still catches (F6, R-B3-17).
6. **Provenance/correlation:** CI deny on an image → later B2 admission deny on the same image →
   correlation_id links them (DT-13, HL-02 end-to-end).
7. **Security:** poisoned bundle in CI rejected (F1); evidence redaction before leaving CI; anonymous
   evidence injection rejected by the store.
8. **Differential/simulate:** `--simulate` against a candidate bundle surfaces decision flips pre-PR
   (overlaps E1; DT-45).

---

## 5. What blocks what

- **Blocked by B1:** signed bundles, decision contract, and the input-normalization contract (jointly
  owned with B2 — this is the cross-cut to settle early).
- **Blocked by B4:** the build-time subset of the action taxonomy.
- **Blocks C2/C3:** build-time evidence feed and coverage analytics.
- **Parallel with B2:** B3 (build-time) and B2 (runtime) are independent engines sharing B1; they only
  meet at the input-normalization contract and at correlation_id.

---

## 6. Risks & mitigations

| Risk | Mitigation |
|---|---|
| Input-normalization contract slips (cross-cut) | Settle the admission-envelope-as-canonical decision (R-B3-10) in week 1 jointly with B1/B2 |
| Helm/TF rendering complexity (values, workspaces) | Render/plan as a required pre-step; capture values_hash/workspace in evidence |
| Conftest breadth gap vs Checkov/Trivy | Scope: governance-traceable checks only; external scanner ingestion is a C-domain follow-on (D-B3-01) |
| Local-hook bypass undermines compliance | CI is authoritative; local is advisory; §19 detective backstop |
| Evidence-store coupling slows dev | Best-effort async emission, gate is local exit code (R-B3-9) |

---

## 7. Estimated sequencing (relative)

Week 1: input-norm contract + W1 + W2 (k8s/helm/tf first). Week 2: W2 remainder + W3 evidence.
Week 3: W4 CI (GHA+GitLab first) + W6 conformance. Week 4: W4 (Jenkins/Tekton) + W5 pre-commit + hardening.
