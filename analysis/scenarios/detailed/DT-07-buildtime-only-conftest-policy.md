# DT-07 — Author a build-time-only (Conftest) policy

**Personas:** Marcus (Policy Library Maintainer), Sam (Application Developer)
**Spec sections:** §7.1 Policy Authoring, §7.2 Enforcement Classes (Build-Time), §10 Conftest, §10.3 Evidence Output
**Type:** Low-level
**Pre-condition:** A signed shared Rego bundle is published as an OCI artifact (§8.2) and consumed by Conftest in CI. Gemara control `CFG-RES-007` ("Helm charts intended for production must declare CPU and memory `resources.limits` on every container") exists with no Rego implementation yet. There is no admission-time equivalent.
**Trigger:** Marcus is asked to enforce `CFG-RES-007` in CI only, because admission cannot reliably reason about Helm-templated values before render and the team wants the failure surfaced to developers at PR time, not at deploy time.

## Steps
1. Marcus authors a new Rego package `governance.helm.resourcelimits` in the shared bundle (§8). He adds §8.3 metadata: `__control_id__ := "CFG-RES-007"`, `__severity__ := "high"`, `__governance_domain__ := "platform-config"`. The rule iterates rendered Deployments/StatefulSets/DaemonSets and denies when any container lacks `resources.limits.cpu` or `resources.limits.memory`.
2. In the control's §7.1 authoring metadata, Marcus sets `enforcement_class: build-time` and `enforcement_targets: [conftest]`. He explicitly omits Gatekeeper and OPA admission targets so the platform does not attempt to wire admission enforcement.
3. Marcus writes the human-readable `outcome_reason` template: `"Container <name> in <kind>/<resource> is missing resources.limits.<cpu|memory> (control CFG-RES-007)"`. The same template renders identically whether invoked locally or in CI.
4. Marcus adds Conftest test fixtures: a passing Helm chart (limits set), a failing chart (limits absent), and a partially failing chart (only memory set). Tests run in the bundle's CI per §7 lifecycle.
5. Marcus publishes the bundle as `bundle:v31`, signed and pushed to the OCI registry. CI pipelines using the shared Conftest step pick up the new version on their next build.
6. Sam pushes a PR adding a new microservice's Helm chart. His pipeline runs `conftest test` against the rendered chart and fails with the templated message above plus the `control_id`, the chart path, and a link to the control's authoring page (§16.3 Rego Explorer).
7. Sam edits the chart to add `resources.limits`, pushes again, Conftest passes. The Conftest run emits the §10.3 normalized evidence object with `evidence_type=build-time`, `pipeline=github-actions`, `decision=allow` for the corrected chart.
8. The Compliance Analytics Engine (§14) ingests both events and credits coverage of `CFG-RES-007` to the build-time enforcement point only; no admission gap is flagged because the control's `enforcement_class` is `build-time`.

## Success criteria (testable)
- The control record persists `enforcement_class=build-time` and `enforcement_targets=[conftest]`; the platform does not generate a Gatekeeper constraint or OPA admission policy for `CFG-RES-007`.
- Conftest output for both Sam's failing and passing runs conforms to §10.3, carrying `control_id=CFG-RES-007`, `policy_package=governance.helm.resourcelimits`, the bundle `policy_version`, and `evidence_type=build-time`.
- The denial message Sam sees in CI is byte-identical to the message emitted from a local `conftest test` run against the same chart (same bundle, same `policy_version`).
- The §17E coverage-gap report does not flag `CFG-RES-007` as missing admission coverage; analytics treats build-time as the declared enforcement class for this control.
- All bundle changes for this package are traceable via signed OCI tags and Rego metadata to `CFG-RES-007`.

## Flowchart

```mermaid
flowchart TD
  M1[Marcus: author Rego\ngovernance.helm.resourcelimits §8.3] --> M2[Set enforcement_class=build-time §7.2\ntargets=[conftest] §7.1]
  M2 --> M3[Write Conftest fixtures + tests §10]
  M3 --> PUB[Publish signed bundle:v31 §8.2]
  PUB --> CI[Sam's PR CI: conftest test §10.1]
  CI -->|chart missing limits| FAIL[Deny + outcome_reason\n+ control_id link to §16.3]
  FAIL --> FIX[Sam adds resources.limits]
  FIX --> CI2[CI re-run: conftest pass]
  CI2 --> EV[§10.3 normalized evidence\nevidence_type=build-time]
  EV --> AN[§14 analytics: coverage credited\nas build-time only]
```

## Notes
The policy lives only in the Conftest enforcement point. It is intentionally absent from §9 Gatekeeper because Helm chart shape is not reliably reconstructible at admission time. Related: DT-19, DT-21.
