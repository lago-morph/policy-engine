# DT-80 — Generate a coverage-gap report by namespace and by control

**Personas:** Priya (Compliance & GRC Lead), Marcus (Platform Security Engineer)
**Spec sections:** §17E.1 Report Categories (Coverage gaps by control, Coverage gaps by namespace), §14.1 Detect missing enforcement coverage, §14.2 Example Detections, §17A.2 Compliance Analyst / Policy Library Maintainer roles
**Type:** Mid-level
**Pre-condition:** The platform tracks the full set of in-scope namespaces from the Kubernetes inventory and the full set of active controls from the Gemara governance store (mapped to constraints/policies per §7). The §14 analytics engine has consumed the last 30 days of OPA decision logs, Gatekeeper audit, Kyverno reports, and Conftest CI events. Priya has the Compliance Analyst role; Marcus has the Policy Library Maintainer role (§17A.2). The tenant `payments` covers 14 namespaces.
**Trigger:** Priya opens "Reports → Coverage Gaps," scopes to `tenant=payments`, `window=30d`, and selects both the by-namespace and by-control breakdowns from §17E.1.

## Steps
1. The §14 analytics engine builds a (namespace × control) matrix for `payments`: 14 × 27 = 378 cells. Each cell is classified as `enforced` (≥1 decision event in window with matching control_id), `installed_no_events` (constraint/policy installed but no decisions observed), `not_installed` (no binding for that namespace+control — §14.1 missing coverage), or `n/a` (control inapplicable by scope).
2. Priya inspects the by-namespace view: rows are namespaces, columns are controls, cells coloured by classification. Sorted by `not_installed` count, `payments-batch` tops with 7 missing controls including `SC-IMG-001`.
3. Priya pivots to the by-control view. `SC-IMG-001` shows `not_installed` in 4 namespaces (`payments-batch`, `payments-sandbox`, `payments-ml-dev`, `payments-edge`). Each cell links to the source-of-truth check (constraint list, policy bindings, last decision row).
4. Priya exports the matrix as a signed CSV and files a finding "SC-IMG-001 missing enforcement coverage in 4 namespaces". The export embeds window, scope filter, and per-cell classification reason.
5. Marcus opens the same report under his Policy Library Maintainer scope, clicks the 4 `not_installed` cells, and uses "Generate ConstraintTemplate binding" to scaffold the missing bindings from the library (§7). He commits via GitOps; Argo CD installs them.
6. Priya re-runs the report 24 h later; the 4 cells transition `not_installed → installed_no_events`, then to `enforced` once the next deploys hit admission. The §17E.1 coverage-gap count for `SC-IMG-001` drops to 0.

## Success criteria (testable)
- Every (namespace × control) cell in scope is classified as exactly one of {`enforced`, `installed_no_events`, `not_installed`, `n/a`}; no cell is blank.
- Cells classified `not_installed` correspond to a verifiable absence of a constraint/policy binding for that (namespace, control) — drillable to the source-of-truth check (§14.1).
- The by-namespace and by-control views are pivots of the same underlying matrix — totals reconcile (sum of `not_installed` by namespace == sum by control).
- The exported CSV is signed and includes window, scope, and per-cell classification reason (§23-aligned).
- After Marcus installs the missing bindings and a deploy hits admission, the affected cells transition to `enforced` in the next report run, demonstrating round-trip closure of the gap.

## Flowchart

```mermaid
flowchart TD
  INV[K8s namespace inventory] --> M[(namespace × control)\nmatrix]
  GOV[Gemara controls + bindings §7] --> M
  DL[Decision logs 30d\nOPA/GK/Kyverno/Conftest] --> CLS[§14 classify per cell]
  M --> CLS
  CLS --> R[§17E.1 coverage report\nby-namespace + by-control]
  PRI[Priya: Reports →\nCoverage Gaps] --> R
  R --> FIND[Finding:\nnot_installed cells]
  FIND --> EXP[Signed CSV export]
  FIND --> MAR[Marcus:\nscaffold bindings §7]
  MAR --> GIT[GitOps install]
  GIT --> RE[Re-run report 24h\nnot_installed → enforced]
```

## Notes
Related: DT-77 (enforcement view of `enforced` cells), DT-78 (audit-derived view of bypasses on `enforced` cells), HL-12 (coverage onboarding). The `installed_no_events` state is the spec-implied bridge between §14.1 missing-coverage and dead-policy detection.
