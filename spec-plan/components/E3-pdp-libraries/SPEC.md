# E3 — Per-Product PDP Libraries — SPEC

**Component:** E3 · **Domain:** E — Simulation & Console
**Authoritative spec:** §17D (§17D.1–§17D.11), with §17C (PDP model, action taxonomy, CRDs) and §13 (replay schema).
**Author persona:** Marcus (Platform Security Engineer) + Priya (Compliance) — cooperative author.
**Status:** DRAFT v1 · **Date:** 2026-05-30

> Market context (research §12): "There is **no product on the market** that defines these uniformly. The spec's §17D explicit per-product decision-point catalog is **the most unusual contribution** of the spec — there's nothing comparable in the open ecosystem." E3 is therefore a **key differentiator**: a reusable catalog of decision points, hooks, audit/replay schemas, supported actions, example policies, and limitations for 9 products.

---

## 1. Scope

E3 is the **Product Decision Point (PDP) and Action Library catalog** (§17D.1). For each supported product it defines: decision points, real-time hooks, audit/replay sources, replay input schemas, supported actions, example controls, and limitations — so users build enforcement and audit-replay policies from **reusable** building blocks.

**In scope**

- A **library entry TEMPLATE** (§4) that every product library MUST conform to (§17C.5 + §17D.1).
- Per-product library entries for all **9 products** (§5): Kubernetes, Keycloak, Jenkins, GitLab, Trivy, OWASP, SonarQube, Grafana/Prometheus, Elasticsearch/Kibana.
- The cross-product modeling pattern (§17D.11): event → hook? → PDP/audit-only → action → decision log → replay schema → simulation.
- How each library feeds the platform: subject mapping, resource mapping, replay schema (to C2/§13), example controls (to A1/§6), action types (to §17C.3 taxonomy), CRD usage (§17C.6).
- A library distribution model (`PolicyActionLibrary` CRD, §17C.6) + versioning.

**Out of scope (owned elsewhere)**

- The engines that *execute* the policies (OPA/Gatekeeper/Kyverno/Conftest) — **B1–B5**.
- The audit schema/store — **C2** (E3 defines per-product *replay input schemas* that map *into* §13).
- The simulation engine — **E1** (E3 libraries are *replay-able* by E1).
- The console rendering of libraries — **E2** (graph `PolicyActionLibrary` nodes).
- The approval/exception machinery — **D3/B4**.

---

## 2. Why a catalog (design principle)

Each product's policy hook is currently ad-hoc (research §12: Jenkins Audit Trail plugin, GitLab compliance frameworks, Trivy report-driven, Grafana Alertmanager webhooks, Elasticsearch own-RBAC). E3's contribution is to **normalize** all of them onto one model so:

1. A control authored once (§6) maps to *any* product's decision points.
2. Real-time enforceability and audit-only observability are made **explicit per decision point** (some products can block; many can only observe — §17D.11 `hook available?` branch).
3. Replay schemas are uniform enough that E1 can run **cross-product differential simulation** over normalized audit logs (§17C.4 retrospective replay).

---

## 3. Library element model (§17D.1)

Every product library defines these elements (the §17D.1 table):

| Element | Description |
|---|---|
| Decision points | Product events where policy may apply |
| Real-time hook | Whether/where blocking or modification is possible |
| Audit source | Logs/events needed for retrospective analysis |
| Replay input schema | Fields required to reconstruct policy input (maps to §13) |
| Supported actions | allow, deny, warn, require approval, mutate, etc. (§17C.3) |
| Example controls | Ready-to-adapt governance/policy examples |
| Limitations | What cannot be enforced directly |

Plus the §17C.5 per-product required definitions: event taxonomy, enforcement location, audit source, replay schema, **subject mapping**, **resource mapping**, decision outcomes, missing-capability notes.

---

## 4. PDP Library Entry TEMPLATE (normative, reusable)

> Every per-product library (§5) and every future product MUST be expressed in this template. This is the catalog's reuse contract.

```yaml
# PolicyActionLibrary entry — TEMPLATE
product: <name>                       # e.g. kubernetes, keycloak, ...
library_version: <semver>
pdp_class: <Admission|Application|CI/CD|Identity|Scanner|Observability|Data/API|Approval>  # §17C.4
realtime_enforcement: <Yes|Sometimes|Usually-no|No>     # §17C.4
retrospective_replay: <Yes|Yes-if-logged|No>            # §17C.4

subject_mapping:                      # §17C.5 — how identity normalizes to governance subject
  source: <jwt|service-account|product-user|api-key|...>
  normalized_to: { sub, groups, roles, tenant, namespaces }   # §13 subject + §17A.4 claims

resource_mapping:                     # §17C.5 — product object -> governance resource
  product_object: <...>
  governance_resource_type: <...>
  resource_id_format: <...>           # stable identity for §13 resource_id

decision_points:                      # the core catalog rows (§17D)
  - id: <slug>
    event: <product event>
    realtime_hook:
      available: <true|false>
      location: <where the decision can be made in real time, §17C.5>
    audit_source: <log/event stream for retrospective, §17C.5>
    replay_input_schema:              # minimum §13 fields to reconstruct this decision
      required: [<field>, ...]
      external_data_refs: [<name/version>, ...]
      replay_completeness_notes: <what is typically missing -> partial/insufficient>
    supported_actions: [<from §17C.3 taxonomy>]
    example_controls:
      - control_id: <CTRL-...>
        statement: <ready-to-adapt example>
        recommended_action: <from supported_actions>
        recommended_engine: <OPA|Gatekeeper|Kyverno|Conftest|tool-gate|app-PDP>
    limitations: <what cannot be enforced directly, §17D.1>

crds_used: [PolicyActionLibrary, PolicyApprovalRequest?, PolicyException?, ...]  # §17C.6
```

**Catalog conventions**

- `realtime_hook.available=false` ⇒ decision point is **audit-only** (§17D.11 "no hook" branch); supported_actions limited to `detect|alert|require review|notify` and recommended_engine is "audit replay (OPA over §13)".
- `recommended_action` per example MUST be in the decision point's `supported_actions`.
- `replay_input_schema.replay_completeness_notes` is the per-product input to E1's authoritativeness gate (§17.3): it tells E1/C2 which fields are typically missing for this product, hence whether replay is `complete|partial|insufficient`.
- `subject_mapping.normalized_to` MUST land in the §13 `subject` + §17A.4 claim shape so cross-product correlation (`correlation_id`) and identity-aware policy work uniformly.

---

## 5. Per-product libraries (the catalog)

Each entry below distills §17D.2–§17D.10 into the template's decision-point rows with **hook location, inputs available, example policies, recommended action**, plus subject/resource mapping and limitations. (Full YAML expansions are generated from these tables at build time.)

### 5.1 Kubernetes (§17D.2) — `pdp_class: Admission` · realtime: Yes · replay: Yes
- **Subject mapping:** AdmissionReview `userInfo` (+ Keycloak JWT via OIDC) → `{sub, groups, roles, tenant, namespaces}`. **Resource mapping:** GVK + namespace/name → `resource_id = cluster/ns/kind/name`.

| Decision point | Hook (location) | Inputs available (replay) | Recommended action | Example policy |
|---|---|---|---|---|
| Create/update/delete resource | Admission webhook | AdmissionReview, K8s audit log, GK/Kyverno reports | deny / mutate / require approval | Block privileged pods; require labels; restrict hostPath |
| Deploy image | Admission webhook | Admission request, image metadata, scan/signature evidence (external_data) | deny / require approval | Require signed image; block critical CVEs |
| Exec into pod | API authz/audit (**detect-only**) | K8s audit log | detect / alert / require review | Exec into prod pods requires break-glass role |
| Port-forward | API authz/audit (**detect-only**) | K8s audit log | detect / alert / require review | Port-forward to regulated ns requires approval |
| Role/RoleBinding change | Admission webhook | K8s audit log | deny / require approval | Prevent cross-tenant role grants |
| Secret create/read/update | Admission/audit (product-dependent) | K8s audit log | deny / warn / detect | Prevent plaintext secret; alert on secret reads |
| Namespace creation | Admission webhook | K8s audit log | mutate / generate / require approval | Generate default NetworkPolicy + ResourceQuota |
| Ingress change | Admission webhook | K8s audit log | deny / mutate / require approval | External ingress requires approved domain |

- **Limitations:** exec/port-forward/secret-read are typically **detect-only** (API authz can't always block at policy-engine layer); generate/cleanup need Kyverno/controller. **Scenario:** DT-68 (privileged pods), DT-64 (generate NetworkPolicy).

### 5.2 Keycloak (§17D.3) — `pdp_class: Identity` · realtime: Sometimes · replay: Yes
- **Subject mapping:** Keycloak user/client/service-account → `{sub, groups, roles, tenant}`. **Resource mapping:** realm/client/role/group object.

| Decision point | Hook | Inputs | Action | Example |
|---|---|---|---|---|
| User login | Auth flow / event listener | Login events | allow / deny / require MFA / detect | Admin login from unusual network requires MFA |
| Token issuance | Protocol mapper / client policy / extension | Token/event logs | deny / require claim / detect | Deployment tokens must include namespace claims |
| Token exchange | Policy/extension | Admin/user events | deny / require approval / detect | Cross-tenant token exchange prohibited |
| Role assignment | Admin event listener / API wrapper | Admin events | deny / require approval / detect | Privileged role grant requires approval |
| Group membership change | Admin event listener / API wrapper | Admin events | deny / require approval / detect | Adding user to prod-deployer group requires approval |
| Client creation/update | Admin event listener | Admin events | deny / require approval / detect | Public clients cannot request privileged scopes |
| IdP change | Admin event listener | Admin events | require approval / detect | IdP trust changes require platform approval |
| Service-account credential rotation | Admin event listener | Admin events | require approval / detect | Prod service-account changes require review |

- **Limitations:** many identity events are **event-listener (post-hoc)** not pre-decision; real-time block needs custom extensions. **Scenario:** DT-69 (MFA on admin login).

### 5.3 Jenkins (§17D.4) — `pdp_class: CI/CD` · realtime: Yes · replay: Yes
- **Subject mapping:** Jenkins user/credential/job identity. **Resource mapping:** job/stage/artifact.

| Decision point | Hook | Inputs | Action | Example |
|---|---|---|---|---|
| Job started | Shared library/plugin/webhook | Build logs, job metadata | allow / deny / warn | Untrusted branch cannot use deploy job |
| Stage started | Pipeline step wrapper | Pipeline logs | pause / deny / require approval | Production stage requires approver role |
| Artifact produced | Pipeline policy step | Artifact metadata, SBOM, scan result | deny / require scan / attach evidence | Artifact must have SBOM and provenance |
| Credentials accessed | Plugin/audit logs | Jenkins audit/security logs | deny if hookable / detect | Prod credential use restricted to release jobs |
| Deployment requested | Pipeline gate | Build/deploy logs | pause / require approval / deny | Deployment requires ticket + passing checks |
| Job config changed | Audit log / config-as-code review | Jenkins config history | require approval / detect | Disabling security checks requires approval |

- **Limitations:** credential-access blocking depends on hookability; config-change is often detect-only. **Scenario:** DT-70 (artifact gate).

### 5.4 GitLab (§17D.5) — `pdp_class: CI/CD` · realtime: Yes · replay: Yes
- **Subject mapping:** GitLab user/CI identity → governance subject. **Resource mapping:** MR/pipeline/branch/environment/runner/variable.

| Decision point | Hook | Inputs | Action | Example |
|---|---|---|---|---|
| MR opened/updated | MR approval rules, CI policy job | GitLab audit/events | block merge / require approval | Policy-file change requires security review |
| MR approved | Approval rules | MR events | allow / deny / require extra approval | Production code requires code-owner approval |
| Pipeline started | CI job | Pipeline logs | deny / require scan | Untrusted source cannot run privileged runner |
| Pipeline completed | CI job/status check | Pipeline logs | block merge/deploy | Failed security scan blocks merge |
| Protected branch push | Branch protection | Audit events | deny / detect | Direct push to protected branch prohibited |
| Environment promotion | Deployment approvals | Deployment events | require approval / deny | Prod promotion requires release approver |
| Runner registration/change | Admin/audit events | Audit events | require approval / detect | Shared runner registration requires admin approval |
| Secret/variable change | GitLab audit events | Audit events | require approval / detect | Protected variable changes require two-person review |

- **Limitations:** GitLab's built-in compliance frameworks (research §12) cover some natively; E3 normalizes them. **Scenario:** DT-71 (code-owner approval).

### 5.5 Trivy (§17D.6) — `pdp_class: Scanner` · realtime: build-time · replay: Yes
- **Subject mapping:** pipeline identity. **Resource mapping:** image/filesystem/SBOM/manifest.

| Decision point | Hook | Inputs | Action | Example |
|---|---|---|---|---|
| Image scan completed | CI/CD gate | Scan report | fail build / block deploy / require exception | Critical vuln blocks production |
| Filesystem scan completed | CI/CD gate | Scan report | fail build / warn | Secrets in repo fail pipeline |
| SBOM generated | CI/CD gate | SBOM artifact | require evidence | Production artifact must have SBOM |
| Misconfiguration detected | CI/CD gate | Scan report | fail build / warn | Manifest with privileged pod fails |
| License finding | CI/CD gate | Scan report | require review | Prohibited license requires legal approval |

- **Limitations:** report-driven (research §12) — enforcement happens at a gate consuming the report, not in Trivy itself. **Scenario:** DT-72 (critical CVE).

### 5.6 OWASP (§17D.7) — `pdp_class: Scanner` · realtime: build-time/manual · replay: Yes

| Decision point | Hook | Inputs | Action | Example |
|---|---|---|---|---|
| ZAP scan completed | CI/CD gate | ZAP report | fail build / require remediation | High-risk auth finding blocks release |
| ASVS control result | CI/CD / manual evidence | Assessment result | require approval / attach evidence | ASVS L2 failure requires security approval |
| Dependency risk finding | CI/CD scanner | Dependency report | fail build / require exception | Known-exploited dependency blocks release |
| Threat model update | Workflow gate | Threat model metadata | require approval | High-risk feature requires threat model approval |

- **Limitations:** ASVS/threat-model are partly manual evidence; replay reconstructs from assessment artifacts. **Scenario:** DT-73 (ASVS L2).

### 5.7 SonarQube (§17D.8) — `pdp_class: Scanner` · realtime: build-time · replay: Yes

| Decision point | Hook | Inputs | Action | Example |
|---|---|---|---|---|
| Quality gate result | CI/CD status check | SonarQube analysis result | block merge / fail build | Production release requires passing quality gate |
| Vulnerability detected | CI/CD status check | Finding report | block merge / require review | New critical vulnerability blocks merge |
| Security hotspot created | Review workflow | Hotspot status | require review | Hotspots reviewed before release |
| Coverage threshold failed | CI/CD gate | Analysis result | warn / fail build | New-code coverage must exceed threshold |
| Quality profile changed | Admin/audit event | Audit/config history | require approval / detect | Security rules cannot be weakened without approval |

- **Limitations:** report-driven; profile-change detect is audit-based. **Scenario:** DT-74 (quality gate).

### 5.8 Grafana/Prometheus (§17D.9) — `pdp_class: Observability` · realtime: usually-no direct block · replay: Yes

| Decision point | Hook | Inputs | Action | Example |
|---|---|---|---|---|
| Alert fired | Alertmanager webhook | Alert history | notify / block promotion through CI/CD | Active critical alert blocks deployment promotion |
| Alert resolved | Alertmanager webhook | Alert history | clear hold / attach evidence | Deployment hold removed after alert resolves |
| SLO breach | Metrics query/gate | Metrics history | require approval / block promotion | Service below SLO cannot receive risky deploy |
| Recording/alert rule changed | GitOps/config review or Grafana API wrapper | Config history/audit | require approval / detect | Critical alert rule change requires approval |
| Dashboard changed | Grafana audit/plugin | Audit logs | detect / require review | Compliance dashboard changes require review |
| Data source changed | Grafana admin/audit | Audit logs | require approval / detect | Production data source change requires approval |

- **Limitations:** no native policy hook (research §12) — enforcement is *indirect* via CI/CD promotion gates fed by Alertmanager webhooks; most actions are detect/notify/block-downstream, not block-in-place. **Scenario:** DT-75 (SLO breach blocks promotion).

### 5.9 Elasticsearch/Kibana (§17D.10) — `pdp_class: Data/API` · realtime: product-dependent · replay: Yes
- Treated as **both data systems and administrative surfaces** (§17D.10): UI actions, API calls, authn/authz, and ACL changes are policy-relevant.

| Decision point | Hook | Inputs | Action | Example |
|---|---|---|---|---|
| Kibana web UI login | Kibana/IdP integration, reverse proxy, custom plugin | Kibana audit logs, IdP logs | allow / deny / require MFA / detect | Privileged Kibana login requires approved group + MFA |
| Elasticsearch API call | Reverse proxy/API gateway, app PDP, ES security model | ES audit logs | allow / deny if proxied / detect | Bulk delete on prod index requires approval |
| Index read/search | ES security model / proxy | ES audit logs | allow / deny if proxied / detect | Sensitive index search restricted to approved role |
| Document write/delete | Proxy / app PDP | ES audit logs | allow / deny if proxied / detect | Delete on regulated index requires approval |
| Role / role-mapping changed | Admin API wrapper, proxy, workflow gate | ES audit logs | require approval / detect | Granting access to sensitive index requires approval |
| User / API key created | Admin API wrapper / proxy | ES audit logs | require approval / detect | Write-privileged API key requires expiration + approval |
| Index template / ILM changed | Admin API wrapper / proxy | ES audit logs | require approval / detect | Retention-policy reduction requires compliance approval |
| Kibana saved object changed | Kibana audit logs / plugin | Kibana audit logs | require review / detect | Compliance dashboard requires approval |
| Security config changed | Admin API wrapper / proxy | ES audit logs | require approval / detect | Disabling audit logging is prohibited |

- **Limitations:** real-time block requires **proxying** (reverse proxy/API gateway/app PDP) since ES has its own RBAC (research §12); un-proxied calls are detect-only. **Scenario:** DT-76 (bulk delete).

---

## 6. Cross-product pattern (§17D.11, normative)

Every product library MUST conform to the shared modeling flow:

```
Product event (login/API/admin/scan/deploy)
   → real-time hook available?
        Yes → PDP (OPA/Gatekeeper/Kyverno/tool gate) → {Allow|Deny|Warn|Mutate|Suspend} + Decision log
        No  → Audit-only observation → Decision log
   → Decision log → Replay schema (§13) → E1 simulation + differential → C5 reports
```

This is what lets E1 run cross-product differential simulation and E2 render cross-product lineage: a Keycloak login deny and a Kubernetes admission deny share the §13 replay shape and the §17C.3 action taxonomy.

---

## 7. Distribution & versioning (`PolicyActionLibrary` CRD, §17C.6)

- Each product library ships as a versioned `PolicyActionLibrary` CRD instance (signed, §23) referenced by Rego packages (`RegoPackage --backed_by--> PolicyActionLibrary` in E2's graph).
- Libraries carry `library_version` (semver); controls/examples pin to a version so example policies remain reproducible.
- Maintained by the **Policy Library Maintainer** role (§17A.2).
- New products are added by authoring a new template (§4) instance — the catalog is **extensible by construction** (§24–25 extensibility).

---

## 8. Failure modes / limitations (catalog-level)

| Issue | Handling |
|---|---|
| Product has no real-time hook | Mark `realtime_hook.available=false`; restrict to detect/alert/notify; document in `limitations` (§17D.11 no-hook branch) |
| Product audit log lacks replay fields | `replay_completeness_notes` warns E1; replay marked partial/insufficient (§17.3) |
| Product RBAC bypasses platform (e.g. ES native) | Require proxy/app-PDP for enforcement; else detect-only |
| Example control drift across library versions | Pin examples to `library_version`; regression via E1 fixtures |
| Subject identity not normalizable | `subject_mapping` flagged incomplete → identity-aware policies degrade; document |

---

## 9. Open questions — decided defaults

| # | Question | Decided default | Rationale |
|---|---|---|---|
| OQ-1 | One mega-library or per-product libraries? | **Per-product** `PolicyActionLibrary` instances on a shared template | Embarrassingly parallel authoring; independent versioning |
| OQ-2 | Ship example policies as Rego or product-native? | **Rego/OPA primary** (cross-product consistency, §17C.1) + product-native where the action is product-native (Kyverno generate, GitLab approval rules) | §17C.1/§17C.2 recommendation |
| OQ-3 | Detect-only decision points worth cataloging? | **Yes** — explicit `realtime_hook.available=false` is a feature (sets correct expectations) | §17D.11 honesty about enforceability |
| OQ-4 | Replay schema per product vs one §13 schema | **One §13 schema**, with per-product `replay_input_schema.required` subset + completeness notes | Uniform replay enables cross-product differential |
| OQ-5 | Who owns the proxy for ES/Grafana enforcement? | Out of E3 scope; E3 *documents the requirement*; deployment owns it (F2) | Keep catalog declarative |

---

## 10. Dependencies

- **§13 / C2** — replay schema each library maps into.
- **§17C.3** — action taxonomy each library draws from.
- **§17C.6 / B4** — `PolicyActionLibrary` (+ `PolicyApprovalRequest`, `PolicyException`) CRDs.
- **A1 (§6)** — example controls map to governance controls.
- **B1–B5** — engines that execute the example policies.
- **E1 (§17)** — replays libraries' decision points.
- **E2 (§16)** — renders `PolicyActionLibrary` graph nodes.
- **D1 (§17A.4)** — subject mapping normalizes to claims.

---

## 11. Traceability

Spec: §17D.1–§17D.11, §17C.3–§17C.6, §13.
Scenarios: DT-63 (OPA vs Kyverno), DT-64 (Kyverno generate), DT-65/66/67 (CRD lifecycles), DT-68 (K8s), DT-69 (Keycloak), DT-70 (Jenkins), DT-71 (GitLab), DT-72 (Trivy), DT-73 (OWASP), DT-74 (SonarQube), DT-75 (Grafana), DT-76 (Elastic).
Personas: Marcus (library author/operator), Priya (control mapping), Sam (developer adapting examples), Daniel (auditor of evidence).
