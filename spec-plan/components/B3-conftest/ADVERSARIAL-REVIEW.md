# B3 — Conftest Integration — ADVERSARIAL REVIEW

**Reviewer persona:** hostile principal engineer who has watched "shift-left" gates get
disabled the week before a release deadline. Mandate: find where the build-time story is fiction.

---

## A. The "same control, shifted left" claim

- **AR-1 (HIGH) — Build-time and runtime evaluate fundamentally different things, so "the same
  control evaluated earlier" is partly an illusion.** A Helm chart or Terraform plan at CI time
  does NOT contain the information a runtime admission request has: no resolved image digest (the
  tag hasn't been built/pushed yet, or resolves differently at deploy), no injected sidecars, no
  defaulted fields, no admission-mutation, no live JWT identity (the CI identity ≠ the deploying
  user). SC-IMG-001 ("image must be signed") literally cannot be fully evaluated at CI time if the
  image isn't built yet. So Conftest can check *structure* ("references an allowed registry") but
  not *the actual runtime fact* ("this exact digest is signed"). The SPEC's correlation_id story
  (R-B3-7) papers over this: a build-time "pass" and a runtime "deny" for the "same" control are
  not contradictory — they checked different things. Without explicitly documenting *which facet*
  of a control each class checks, users will wrongly believe a green CI means a green admission, and
  be blindsided at deploy (the opposite of shift-left's promise).

- **AR-2 (HIGH) — The input-normalization shim (R-B3-10) wrapping Conftest input in a fake `review`
  envelope with a fake `subject` is a correctness landmine.** To make one package work in both
  engines, B3 fabricates `{review:{object:...}, subject: <ci identity>}`. But a governance package
  that reads `input.subject.groups` to make an identity-aware decision will read the *CI runner's*
  identity at build-time, not the deploying user's — producing a *different and wrong* decision
  that nonetheless "conforms" (same code path, different data). The conformance suite (R-B3-11) will
  pass because it feeds identical synthetic input, but production won't. Identity-aware controls
  (a large class per §15/§17A) are fundamentally not build-time-evaluable, and the shim hides that
  by inventing an identity. The SPEC needs to *exclude* identity-dependent controls from the
  build-time class explicitly, not wrap them in a fake subject.

## B. Conformance theater

- **AR-3 (MED) — Choosing the admission envelope as canonical (R-B3-10, OQ2) optimizes for
  Gatekeeper at Conftest's expense, and the rationale ("minimizes shims") is engine-count, not
  correctness.** Conftest's natural, ergonomic input is the bare document; forcing every Conftest
  policy author to think in `input.review.object` for a Terraform plan is unnatural and error-prone.
  Worse, it means policies are written in admission-shaped Rego that's awkward for the build-time
  case — undermining Conftest's main virtue (Rego-native, simple). The decision is defensible but
  the SPEC undersells the ergonomic cost; expect author confusion and shim bugs.

- **AR-4 (MED) — The conformance suite proves agreement on *synthetic* input, which is the easy
  case.** Real conformance failures come from input-shape edge cases: a Helm chart that renders
  multiple documents, a TF plan with computed/unknown values, an SBOM with no image ref, a list vs
  single object. The golden corpus (B1-R30) likely uses clean admission-style objects. If it doesn't
  include the messy build-time input shapes, "conformance green" (M6) is a weak guarantee for B3.

## C. Operational reality of CI gates

- **AR-5 (HIGH) — "Fail loudly when the bundle server is unreachable" (F2, OQ4) will get the gate
  disabled.** The first time the bundle/registry has a blip and CI fails *every* build org-wide with
  a governance error unrelated to the developer's change, the platform team will be told to make it
  non-blocking — and then it's advisory forever. The SPEC's instinct (no silent skip) is right but
  the failure UX is wrong: a transient infra failure should not look like a policy violation, and
  should not block unrelated merges. Need: distinguish "policy says no" (block, it's the dev's fault)
  from "governance infra is down" (page the platform team, time-boxed soft-fail, loud banner) — not
  one undifferentiated "fail the run."

- **AR-6 (MED) — Pre-commit being "non-authoritative" (R-B3-17) means it's pure friction with no
  teeth, and friction-without-teeth gets uninstalled.** A local hook that's slow, occasionally
  offline-stale, and can be skipped with `--no-verify` while CI is the real gate gives developers
  every incentive to disable it and let CI catch things. Then you've shipped a "shift-left" feature
  nobody runs. Either invest in making local *fast and worth it* (instant, incremental, great UX) or
  don't claim pre-commit as a deliverable — a checked-box hook that's universally bypassed is worse
  than none (false sense of coverage).

- **AR-7 (MED) — Evidence emission best-effort-async (R-B3-9) creates an evidence-completeness gap
  that §19/C4 will misread.** If CI runs pass but the async evidence emission silently fails (store
  blip), the platform has *no record* that the check ran — which looks identical to "the check was
  skipped/bypassed." So the §19 bypass detector either ignores missing build-time evidence (hole) or
  flags every transient emission failure (noise). "Best-effort" evidence and "evidence-based bypass
  detection" are in tension. Need durable buffering of evidence (like B1-R28) even for CI, or accept
  that build-time evidence is unreliable for compliance claims.

## D. Scope and parsing

- **AR-8 (MED) — Terraform plan-JSON evaluation requires running `terraform plan` in CI with
  provider credentials.** R-B3-3 mandates plan JSON, which means CI must authenticate to the cloud
  to produce a real plan — a significant security and setup burden the SPEC doesn't acknowledge.
  Plans on PRs from forks can't safely have credentials. Many orgs gate on HCL static analysis
  precisely to avoid this. The SPEC's "evaluate the plan" is correct for accuracy but operationally
  heavy and sometimes impossible (untrusted PRs).

- **AR-9 (LOW/MED) — Helm "render then evaluate" (R-B3-2) ignores that charts render differently per
  environment values.** A chart that's compliant with prod values may be non-compliant with dev
  values (or vice versa). Which values does CI use? If CI renders with placeholder/default values,
  it evaluates a manifest that will never actually deploy. The SPEC captures `values_hash` but
  doesn't say CI must render with the *target environment's* values — so build-time results may not
  correspond to any real deployment.

- **AR-10 (LOW) — "Reject ad-hoc Rego with no control_id" (F7) breaks the common case of teams
  having their own non-governance lint checks in Conftest.** Many teams already run Conftest for
  local style/policy checks unrelated to Gemara. Hard-rejecting any policy without a control_id makes
  the platform hostile to coexistence. Should warn/segregate, not reject.

- **AR-11 (LOW) — D-B3-01 quietly concedes the whole layer's value.** "Conftest doesn't replace a
  dedicated IaC scanner; ingest Trivy/Checkov output separately." If most real misconfig coverage
  comes from an external scanner whose output is normalized in the C-domain, then B3 (the Conftest
  integration) is a thin governance-traceability wrapper around a minority of checks. Fine, but the
  SPEC should be honest that B3 is *not* the org's IaC security gate — it's the governance-traceable
  subset — to avoid the platform being mis-sold as a Checkov replacement.

---

## Prioritized defect list

| ID | Sev | Defect | Required resolution |
|---|---|---|---|
| AR-1 | **HIGH** | "Same control shifted left" conflates structure-check (CI) with runtime-fact-check | Document which facet each enforcement class verifies; green CI ≠ green admission |
| AR-2 | **HIGH** | Fake `subject` shim makes identity-aware controls silently wrong at build-time | Exclude identity-dependent controls from build-time class; don't fabricate identity |
| AR-5 | **HIGH** | "Fail loudly on infra-down" gets the gate disabled org-wide | Distinguish policy-deny (block) from infra-down (page + time-boxed soft-fail) |
| AR-7 | MED | Best-effort evidence vs evidence-based §19 detection are in tension | Durable evidence buffering in CI; or mark build-time evidence non-authoritative for §19 |
| AR-3 | MED | Admission-envelope-as-canonical is ergonomically costly for Conftest authors | Acknowledge cost; provide author tooling/macros to hide the envelope |
| AR-4 | MED | Conformance corpus likely tests clean input, not messy build-time shapes | Add multi-doc/computed-value/no-image edge cases to the corpus |
| AR-6 | MED | Non-authoritative pre-commit = bypassed friction | Invest in fast incremental UX or drop the claim |
| AR-8 | MED | TF plan-JSON needs cloud creds in CI; impossible for untrusted PRs | Document the credential requirement; offer HCL-static fallback for fork PRs |
| AR-9 | LOW/MED | Helm render uses which env's values? | Require rendering with target-env values; record env in evidence |
| AR-10 | LOW | Hard-reject of non-governance Rego is hostile to coexistence | Warn/segregate non-control Rego, don't reject |
| AR-11 | LOW | D-B3-01 concedes B3 isn't the real IaC gate | Position honestly as governance-traceable subset, not Checkov replacement |

**Verdict:** The normalized-evidence schema (§10.3 superset) and CI/pre-commit packaging are
sound and buildable. The two claims that will mislead users are **AR-1** ("a green build means a
green deploy" — it doesn't, they check different facets) and **AR-2** (identity-aware controls are
not honestly build-time-evaluable, and the shim hides it). And **AR-5** is the operational reality
that decides whether this gate survives its first infra blip. Fix the framing of AR-1/AR-2 in the
SPEC and design the AR-5 failure UX before claiming build-time enforcement.
