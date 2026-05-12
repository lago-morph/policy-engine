# Scenarios Index — Unified Governance & Policy Enforcement Platform

This index lists 100 testable "after" scenarios derived from the persona-spec
mapping (`analysis/persona-spec-mapping.md`) and the platform specification
(`openssf_opa_unified_governance_platform_spec v1.md`).

- **High-level scenarios (HL-01–HL-20)** — multi-persona, multi-section,
  end-to-end flows describing whole business cycles or incident loops.
- **Detailed scenarios (DT-01–DT-80)** — single-section or single-feature
  flows demonstrating concrete behavior the product must support.

Each scenario file is ≤1 page of text, names the personas involved and the
spec section(s) tested, lists ordered steps, gives success criteria, and
contains a Mermaid flowchart of the flow.

---

## High-Level Scenarios (20)

| ID | Title | Primary personas | Primary spec sections |
|---|---|---|---|
| HL-01 | Quarterly SOC 2 evidence collection cycle | Priya, Daniel | §6, §11, §14, §16.3, §17E, §19 |
| HL-02 | Image-signing policy rollout end-to-end | Marcus, Jess, Priya | §6, §7, §8, §9, §10, §17.4, §17E |
| HL-03 | 2 a.m. admission incident → regression fixture loop | Jess, Marcus | §9, §13, §16.2, §16.3, §17.5 |
| HL-04 | Developer onboarding a new service through policy gates | Sam, Marcus | §7, §10, §16.3, §17A.2, §17B |
| HL-05 | Annual SOC 2 Type II external audit engagement | Daniel, Priya, Marcus | §11, §14, §17.4, §17A.2, §17E.3, §19, §23 |
| HL-06 | Gatekeeper bypass retrospectively detected, post-incident | Jess, Priya, Daniel | §14.2, §17E.3, §19 |
| HL-07 | New compliance framework (HIPAA) adoption | Priya, Marcus | §6, §7, §17E |
| HL-08 | Namespace-scoped policy authoring for an app team | Sam, Marcus | §16.3, §17, §17A.2, §17A.5 |
| HL-09 | Multi-cluster policy drift remediation | Marcus, Jess | §9, §14, §16.3 |
| HL-10 | Production deploy approval with break-glass exception | Sam, Priya, Marcus | §17B, §17C.6, §17D, §17E |
| HL-11 | AI model deployment governance lifecycle | Priya, Marcus, Sam | §6, §17, §20.3 |
| HL-12 | Major outage retrospective uncovers silent policy regression | Marcus, Jess, Priya | §14, §17.4, §17E, §19 |
| HL-13 | Cross-tenant access attempt detected and audited | Marcus, Priya | §14, §15, §17A, §20.2 |
| HL-14 | New product / PDP integration onboarding | Marcus, Workflow Integrator | §17C, §17D, §25 |
| HL-15 | Continuous compliance during change-freeze period | Priya, Jess | §14, §17B, §17E |
| HL-16 | Keycloak IdP change drives JWT claim evolution | Marcus | §15, §17A.4 |
| HL-17 | Differential simulation prevents a 2 a.m. rollback | Marcus | §17.2, §17.4, §17E.4 |
| HL-18 | Auditor independent re-execution of historical events | Daniel | §13, §17.4, §17A.2, §23 |
| HL-19 | Policy exception expiry and re-authorization | Sam, Priya | §17B, §17C.6 |
| HL-20 | Multi-cloud / federated compliance reporting | Priya, Marcus | §14, §16, §17E, §20 |

---

## Detailed Scenarios (80)

### Group A — Governance Model (§6)

| ID | Title | Personas |
|---|---|---|
| DT-01 | Author a new Gemara objective and decompose into controls | Priya |
| DT-02 | Map an external framework requirement (SOC 2 CC6.1) to a Gemara control | Priya |
| DT-03 | Define an exception requirement on a Gemara control | Priya, Marcus |
| DT-04 | Deprecate a Gemara control and remove its enforcement | Priya, Marcus |

### Group B — Policy Lifecycle (§7)

| ID | Title | Personas |
|---|---|---|
| DT-05 | Promote a constraint dry-run → warn → enforce | Marcus |
| DT-06 | Roll back a constraint promotion | Marcus, Jess |
| DT-07 | Author a build-time-only (Conftest) policy | Marcus, Sam |
| DT-08 | Author a detective-only (audit-derived) policy | Marcus, Priya |
| DT-09 | Handle a control where Rego cannot be auto-generated; ship template | Marcus, Priya |

### Group C — OPA Integration (§8)

| ID | Title | Personas |
|---|---|---|
| DT-10 | Sign and version a Rego bundle as OCI artifact | Marcus |
| DT-11 | Validate Rego metadata extensions before promotion | Marcus |
| DT-12 | Integrate an embedded application OPA into the evidence pipeline | Marcus, Sam |
| DT-13 | Trace a runtime decision back to bundle version and Gemara control | Jess, Daniel |

### Group D — Gatekeeper (§9)

| ID | Title | Personas |
|---|---|---|
| DT-14 | Switch a constraint from warn to deny in production | Marcus |
| DT-15 | Use Gatekeeper audit mode for periodic compliance scanning | Marcus, Priya |
| DT-16 | Investigate a Gatekeeper event missing required audit fields | Marcus, Jess |
| DT-17 | Reconcile Gatekeeper periodic audit results against admission denies | Priya, Marcus |

### Group E — Conftest (§10)

| ID | Title | Personas |
|---|---|---|
| DT-18 | Run Conftest locally in pre-commit with the shared bundle | Sam |
| DT-19 | Conftest validates Helm chart in CI for production deploy | Sam, Marcus |
| DT-20 | Conftest validates Terraform plan against a Gemara control | Marcus |
| DT-21 | Normalize Conftest output to the canonical evidence schema | Marcus, Priya |

### Group F — Privateer (§11)

| ID | Title | Personas |
|---|---|---|
| DT-22 | Privateer evaluation log for one control over 30 days | Priya, Daniel |
| DT-23 | Correlate SBOM attestation to a Gemara supply-chain control | Marcus, Priya |
| DT-24 | Export a signed Privateer evidence package for an audit | Priya, Daniel |

### Group G — Audit Schema (§12–13)

| ID | Title | Personas |
|---|---|---|
| DT-25 | Diagnose a `replay_completeness = insufficient` event | Marcus, Jess |
| DT-26 | Add a new JWT claim into audit events for a new policy | Marcus |
| DT-27 | Track `external_data_refs` version drift for image-signature checker | Marcus |
| DT-28 | Investigate a missing `correlation_id` between Gatekeeper and OPA | Jess, Marcus |
| DT-29 | Export OCSF-compatible event for SIEM compatibility | Marcus |

### Group H — Compliance Analytics (§14)

| ID | Title | Personas |
|---|---|---|
| DT-30 | Detect Gatekeeper bypass via missing audit event | Jess, Priya |
| DT-31 | Detect JWT policy drift after a Keycloak realm change | Marcus, Priya |
| DT-32 | Identify inconsistent enforcement across two clusters | Marcus, Jess |
| DT-33 | Detect missing enforcement coverage in a namespace | Priya, Marcus |
| DT-34 | Generate a weekly compliance report for executive review | Priya |

### Group I — Keycloak / JWT (§15)

| ID | Title | Personas |
|---|---|---|
| DT-35 | Add a new claim to a Keycloak realm via the mapping layer | Marcus |
| DT-36 | Normalize a tenant claim across two IdPs | Marcus |
| DT-37 | Decommission an obsolete claim required by no policy | Marcus |
| DT-38 | Map group hierarchy to role expansion at token issuance | Marcus |

### Group J — Governance Console / GUI (§16)

| ID | Title | Personas |
|---|---|---|
| DT-39 | Use Governance Graph View to trace control → Rego → enforcement points | Priya, Marcus |
| DT-40 | Use Rego Explorer to view test coverage and required claims | Marcus |
| DT-41 | Use Runtime Enforcement View to investigate recent denies | Jess, Marcus |
| DT-42 | Use Audit Correlation View to find a compliance gap | Priya, Jess |
| DT-43 | Use Namespace Authoring View as a Namespace Policy Author | Sam |
| DT-44 | Install Headlamp plugin and authenticate via Keycloak OIDC | Marcus, Jess |

### Group K — Simulation (§17)

| ID | Title | Personas |
|---|---|---|
| DT-45 | Run manifest simulation before submitting a PR | Sam |
| DT-46 | Historical replay for one control over 30 days | Marcus, Daniel |
| DT-47 | Run live shadow mode for a new policy | Marcus |
| DT-48 | Cluster snapshot simulation for upgrade readiness | Jess, Marcus |
| DT-49 | Differential simulation across two policy versions | Marcus, Priya |
| DT-50 | Namespace-scoped simulation by a Namespace Policy Author | Sam |
| DT-51 | Create regression test from an audit event ("intended behavior test") | Marcus, Jess |
| DT-52 | False positive test confirms a previously-supported pattern still works | Marcus, Sam |

### Group L — Roles, Permissions, Storage (§17A)

| ID | Title | Personas |
|---|---|---|
| DT-53 | Grant Namespace Policy Author role to a new app team via Keycloak | Marcus |
| DT-54 | Audit a global admin's cross-tenant boundary crossing | Priya, Marcus |
| DT-55 | Verify storage-layer scope enforcement against a Namespace Policy Author | Marcus, Sam |
| DT-56 | Onboard an external Auditor with read-only scoped access | Daniel, Priya |
| DT-57 | Compliance Analyst exports an evidence set with redacted JWT subjects | Priya |

### Group M — Approval-Gated Decisions (§17B)

| ID | Title | Personas |
|---|---|---|
| DT-58 | `suspend_pending_approval` for a production deployment | Sam, Marcus |
| DT-59 | Kubernetes admission deny-with-approval-required pattern | Sam, Marcus |
| DT-60 | Jenkins pipeline pauses for security approval at deploy stage | Sam, Marcus |
| DT-61 | GitOps controller suspends sync pending approval | Jess, Marcus |
| DT-62 | Approval expires; re-authorization workflow | Sam, Priya |

### Group N — Engine Selection / CRDs (§17C)

| ID | Title | Personas |
|---|---|---|
| DT-63 | Decide OPA vs Kyverno for a new image-verification policy | Marcus |
| DT-64 | Add Kyverno generate policy for default NetworkPolicy on namespace create | Marcus, Jess |
| DT-65 | `PolicyApprovalRequest` CRD lifecycle for a deploy gate | Marcus, Sam |
| DT-66 | `PolicySimulationRun` CRD for a long-running replay job | Marcus |
| DT-67 | `PolicyException` CRD lifecycle (request → approve → expire) | Sam, Priya |

### Group O — Product Libraries (§17D)

| ID | Title | Personas |
|---|---|---|
| DT-68 | Kubernetes library — block privileged pods with NS-scoped exception | Marcus, Sam |
| DT-69 | Keycloak library — require MFA for admin login from unusual network | Marcus |
| DT-70 | Jenkins library — gate artifact deployment on SBOM + scan evidence | Sam, Marcus |
| DT-71 | GitLab library — require code-owner approval for policy file changes | Marcus |
| DT-72 | Trivy library — block deployment for critical CVE; require exception | Sam, Marcus |
| DT-73 | OWASP library — ASVS L2 failure requires security approval | Marcus, Priya |
| DT-74 | SonarQube library — block merge on failed quality gate | Sam |
| DT-75 | Grafana/Prometheus library — block deploy when SLO breached or alert firing | Jess, Sam |
| DT-76 | Elasticsearch library — require approval for bulk delete on regulated index | Marcus, Priya |

### Group P — Reporting / Retrospective (§17E, §19)

| ID | Title | Personas |
|---|---|---|
| DT-77 | Generate the real-time enforcement report for production tenants | Priya |
| DT-78 | Generate the audit-derived violation report for control SC-IMG-001 | Priya, Daniel |
| DT-79 | Generate a simulation report for stakeholder review | Marcus, Priya |
| DT-80 | Generate a coverage-gap report by namespace and by control | Priya, Marcus |

---

## File layout

```
analysis/
├── persona-spec-mapping.md
├── scenarios-index.md                          (this file)
└── scenarios/
    ├── high-level/
    │   ├── HL-01-soc2-quarterly-evidence.md
    │   ├── HL-02-image-signing-rollout.md
    │   └── ... (HL-03 … HL-20)
    └── detailed/
        ├── DT-01-author-gemara-objective.md
        ├── DT-02-map-soc2-cc61-to-control.md
        └── ... (DT-03 … DT-80)
```

Each scenario file uses this skeleton:

```markdown
# <ID> — <Title>

**Personas:** <primary persona names>
**Spec sections:** <list of section refs>
**Type:** End-to-end | Mid-level | Low-level
**Pre-condition:** <state of the world before the scenario>
**Trigger:** <what kicks off the scenario>

## Steps
1. ...
2. ...

## Success criteria (testable)
- ...
- ...

## Flowchart

```mermaid
flowchart TD
  ...
```

## Notes
<edge cases, related scenarios, follow-ups — optional, keep short>
```
