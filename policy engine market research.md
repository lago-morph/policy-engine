I'll break the spec into its component capabilities and research the market in each area. The platform combines several distinct capability layers, so I'll search systematically. Let me start.I have a thorough picture across all the spec's layers. Here is the report.

# Market Survey — Equivalents to the Unified Governance & Policy Enforcement Platform (May 2026)

## How I broke the spec into evaluable pieces

The platform you supplied is not a single product category. It bundles ten distinct functions, and the market has different leaders for each one. To assess "what already exists" I split it as follows, then mapped the market layer by layer:

1. **Governance model / controls-as-code** (Gemara layer)
2. **Runtime policy decision engine** (OPA / Rego)
3. **Kubernetes admission enforcement** (Gatekeeper / Kyverno)
4. **CI/CD policy validation** (Conftest / IaC scanners)
5. **Evidence collection & evaluation** (Privateer)
6. **Replay-capable audit schema & retrospective detection**
7. **Identity-aware enforcement** (Keycloak JWT mapping)
8. **Visualization, lineage graphs, namespace authoring** (Headlamp plugin)
9. **Policy simulation / differential replay**
10. **Cross-product PDPs and approval-gated workflows** (Jenkins, GitLab, ES/Kibana, etc.)

I will go layer-by-layer with the closest commercial, open-source, cloud-bundled, and GRC equivalents, then end with overall analogs that hit several of these layers at once.

---

## 1. The single most direct architectural equivalent: OSCAL Compass + C2P

If you asked "is there already an open-source project that bridges compliance controls to multiple policy engines and emits assessment results back to the governance layer," the answer is yes, and it is the closest thing the market has to this spec's spine.

OSCAL-Compass is a project by IBM Research and Red Hat that became a CNCF sandbox project, leveraging NIST's OSCAL standard with three working tools: compliance-trestle, C2P (compliance-to-policy), and Agile Authoring. C2P transforms OSCAL compliance-as-code into native formats for policy engines, and aggregates results back into OSCAL Assessment Results, with plugin support for Kyverno, Open Cluster Management, and Auditree. Trestle adds GitOps-based agile authoring, semantic versioning, provenance traceability, change logs, and approval-based release.

What this means for your spec: the **governance-control → policy-engine → assessment-results loop** that Gemara + Privateer + OPA/Conftest are designed to implement is already a functioning open-source pipeline in OSCAL Compass — but using NIST OSCAL as the data model instead of Gemara. The Gemara project is itself young: a Go SDK exists and Privateer automates Gemara's layer-five evaluations, and OpenSSF released Gemara's inaugural white paper in early 2026, framing it as "compliance as code". Gemara and OSCAL Compass are coexisting alternatives addressing nearly the same problem, with OSCAL Compass roughly two to three years more mature.

A second related effort is **FINOS Common Cloud Controls (CCC) + AI Governance Framework (AIGF) + Common Controls for AI Services (CC4AI)**. FINOS launched CC4AI in 2025 with BMO, Citi, Morgan Stanley, RBC, Bank of America, plus AWS, Microsoft, Google Cloud, Red Hat, Sonatype, and ControlPlane, building on the existing CCC project to deliver "regulation-as-code" validation mechanisms. FINOS AIGF v2.0 released in late 2025, expanding to 46 risks and mitigations cross-referenced to OWASP, MITRE, and EU AI Act. CC4AI uses Gemara's Layer 2 model directly, so Gemara and FINOS are already federated.

---

## 2. The OPA control plane has changed dramatically in the last year

Your spec assumes OPA + a control plane around it. The most important market event since the spec was likely drafted: **Apple acqui-hired Styra in August 2025, and the commercial OPA stack has been open-sourced into CNCF.**

Open Policy Agent's creators along with many team members from Styra joined Apple in August 2025. The OPA code continues to be governed by CNCF with the list of maintainers unchanged, and Styra's commercial offerings — including the enterprise distribution EOPA, OPA Control Plane, multiple SDKs, and the Rego linter Regal — are being released as open source. Styra's commercial offering is being sunset, with the original team now at Apple and existing Styra DAS deployments expected to lose their commercial roadmap and enterprise-level support.

This collapses much of what your spec specifies into one freshly-open-sourced CNCF project: **OPA Control Plane (OCP)**. OCP provides Git-based policy management that builds bundles from multiple Git repositories, external HTTP/push datasources, highly-available bundle distribution to S3/GCS/Azure Blob, and global/hierarchical policies injected at build-time via label selectors. OCP also supports regression testing bundles against historical decision logs.

OCP overlaps with these spec sections almost line-for-line: §7 (policy lifecycle), §8.2 (signed/versioned OCI bundles), §14 (compliance analytics for bundle behavior), and §17 (simulation against historical decisions). The headline implication: in May 2026, what was a $50K+/yr commercial product (Styra DAS) for OPA management is becoming a free CNCF project with the same primitives. Anyone building the platform in your spec should treat OCP as a candidate substrate, not a competitor.

Application-layer alternatives to OPA also matured: **Cerbos, Permit.io, Aserto/Topaz, Oso, OpenFGA (CNCF), SpiceDB (Authzed), Permify, Casbin**. Cerbos in 2026 positions itself for fine-grained, contextual, continuous authorization across applications, gateways, workloads, and AI agents, with RBAC/ABAC/ReBAC/PBAC out of the box and audit-ready logs. Aserto's Topaz uniquely combines policy-as-code (via the OPA engine) and policy-as-data (via a Zanzibar-style directory). These are PDPs only — they don't try to bridge governance↔runtime↔audit the way your spec does — but if you are weighing pure authorization layers as building blocks, this is the field.

---

## 3. Kubernetes admission enforcement (Gatekeeper + Kyverno layer)

The spec uses both Gatekeeper and Kyverno, which mirrors the current market reality: they are complementary, not competitive. Three things have shifted here in the last year.

First, **Kyverno's commercial vendor (Nirmata) is now an "AI platform engineering" product on top of Kyverno.** In November 2025 Nirmata launched its AI Platform Engineering Assistant, using a multi-agent architecture to automate policy authoring, detection, and remediation on top of Kyverno, with a copilot interface and remediator AI agent. Nirmata Enterprise for Kyverno orchestrates policy packs, versions, and exceptions on the native Kyverno engine, creates signed pull requests with approver steps and rollback safety, and offers single sign-on, granular roles and tenant separation, tamper-proof audit logs, and evidence exports. This is essentially a competing product to the Visualization Console + Authoring + Approval Workflow in your spec sections 16, 17A, and 17B — but Kyverno-only.

Second, **all three hyperscalers now bundle Gatekeeper-derived enforcement.**
- Microsoft Azure Policy for Kubernetes is a pod that extends open-source Gatekeeper v3, registering as an admission control webhook. Defender for Cloud's AKS security dashboard layers alerts, vulnerabilities, misconfigurations, and compliance results on top.
- GCP's Anthos Policy Controller is built on Gatekeeper, deployed via Config Sync from Git. Binary Authorization adds signed-image admission.
- AWS uses Cedar via Verified Permissions for application authorization, and EKS users typically run Kyverno or Gatekeeper themselves; Amazon Verified Permissions in April 2026 added policy store aliases and named policies for multi-tenant simplification.

Third, **Red Hat ACS (StackRox) has codified policy-as-code natively.** RHACS 4.x ships a SecurityPolicy CR; policies can be applied with kubectl or Argo CD/GitOps, and Central reconciles policy CRs against its database with drift detection. Combined with RHACM's governance framework deploying RHACS Central server and Secured Cluster services via policies, the Red Hat stack covers the spec's enforcement + lineage + multi-cluster scope without OPA/Kyverno at all.

---

## 4. CI/CD validation and IaC scanning (Conftest layer)

Conftest is a thin Rego wrapper for IaC, and the field has consolidated meaningfully:

- **Checkov** (Palo Alto/Prisma Cloud) — broadest coverage, graph-based analysis, 1000+ checks, Python+YAML policies. Most popular open-source IaC scanner.
- **Trivy** (Aqua) — absorbed tfsec; tfsec deprecated in favor of Trivy starting 2023, migration completed by 2024, sharing the same rules. Scans IaC + containers + dependencies + secrets in one tool.
- **KICS** (Checkmarx) — ~1900 queries, broadest IaC format support (Terraform/CFN/Pulumi/Helm/Ansible/Docker).
- **Terrascan** — archived by Tenable on November 20, 2025 and no longer maintained. Worth knowing if you encounter it in existing pipelines.
- **Snyk IaC** — strong IDE/PR developer UX, policy-as-code with Regula.
- **Conftest** itself remains relevant because it's the only Rego-native scanner; if your governance model emits Rego, Conftest is still the natural CI side.

For pure CI/CD policy gates, **GitLab Compliance frameworks** and **GitHub Environments/Required Reviewers** also fill this space at the workflow layer, and most enterprises stack one IaC scanner + one workflow gate rather than using a single unified tool.

---

## 5. Continuous compliance & evidence collection (Privateer + audit schema layer)

This layer has the most crowded market, mostly oriented at SOC 2 / ISO 27001 / HIPAA rather than Kubernetes policy. The leaders in May 2026:

| Vendor | Position | 2026 status |
|---|---|---|
| Vanta | Largest by customer count | Crossed 15,000 customers, shipped Agentic Trust Platform (AI Agent 2.0) in January 2026 with autonomous policy drafting and questionnaire automation |
| Drata | Strong on customization, recurring ops | Positions as AI-native trust management platform, suited to recurring compliance operations and broader assurance |
| Secureframe | Mid-market, automation-heavy | Forrester Q4 2025 GRC Platforms Landscape leader |
| Sprinto | Mid-market, broader scope | Comprehensive compliance/risk/vendor/questionnaires |
| Hyperproof | AI-powered GRC platform with Hypersyncs to automate evidence collection from Azure/AWS/Slack, 100+ frameworks out-of-the-box | Mature mid-enterprise |
| Optro (was AuditBoard) | Audit/SOX/internal controls focus, rebranded from AuditBoard, 2025 Gartner GRC Magic Quadrant Leader | Enterprise audit-centric |
| Anecdotes, Scytale, Thoropass, Cynomi | Various mid-market positions | Active niche players |

These all do "evidence collection + continuous control monitoring + framework mapping." None of them generate runtime enforcement policies the way your spec does — they pull *evidence* from existing systems. The gap is exactly what Privateer + the audit schema service is meant to fill: governance-traceable evidence emitted *by enforcement decisions themselves*, not scraped from logs later.

For the **replay-capable audit event schema** (§13), the relevant comparator is **OCSF (Open Cybersecurity Schema Framework)**, which your spec acknowledges. OCSF is now sponsored by AWS Security Lake, Splunk, Cisco, and others as the de-facto event normalization standard, but as your spec correctly notes, it's an event-normalization schema, not a policy-replay schema. There is no direct competitor for the "preserve enough fields to replay the policy decision" requirement — this is a genuine gap in the market.

---

## 6. Enterprise IRM / GRC suites (overall governance layer)

These are the heavyweight enterprise alternatives that include governance hierarchy, control catalogs, evidence, workflow, and reporting — but no runtime enforcement:

| Platform | Best fit |
|---|---|
| **ServiceNow IRM / GRC** | Enterprises already on ServiceNow; ties GRC to ITSM/ITOM workflows |
| **Archer IRM** (was RSA Archer) | Mature large-enterprise risk programs, complex governance |
| **MetricStream** | Global enterprises, cross-functional GRC |
| **IBM OpenPages** | Very large enterprises, supports thousands of users |
| **OneTrust** | Privacy + AI governance + GRC consolidation |
| **LogicGate Risk Cloud** | No-code, customizable workflows |
| **Diligent** | Board reporting + governance/ESG |
| **AuditBoard / Optro** | Audit + SOX + IT compliance |

These platforms own the "Gemara-like hierarchy" (Objectives → Domains → Controls → Evidence Requirements → Exception Workflow) but treat it as documentation and workflow, not as input to runtime policy generation. The spec is unusual in trying to make that hierarchy *executable*.

---

## 7. Cloud-Native Application Protection Platforms (CNAPP)

These bundle CSPM + KSPM + workload protection + sometimes policy-as-code, hitting your spec's runtime + retrospective layers without governance traceability:

| Vendor | Position (PeerSpot/G2 May 2026) | Notable |
|---|---|---|
| **Wiz** | Highest CSPM rating (8.8) and largest mindshare (14.3%) in March 2026 | Agentless graph-based, just acquired or being acquired by Google |
| **Prisma Cloud** (Palo Alto) | Mindshare 9.3%, includes Bridgecrew/Checkov | Full code-to-cloud |
| **Microsoft Defender for Cloud** | Mindshare 7.3% | Native Azure + AWS/GCP |
| **SentinelOne Singularity Cloud** | Rising | KSPM + CWPP + offensive engine |
| **Sysdig Secure** | Runtime + Falco-based | Best runtime detection |
| **Lacework** (now Fortinet) | Consolidated under Fortinet | Anomaly detection |
| **Aqua Security** | Container/K8s native | Owns Trivy |
| **Red Hat ACS** (StackRox) | OpenShift-native | Policy-as-code, image signature verification |

None of these expose the full governance-control-to-runtime-decision lineage your spec demands; they treat policy as misconfiguration findings, not as machine-readable controls traced to organizational objectives.

---

## 8. Visualization, simulation, and policy lineage (the Governance Console layer)

Your spec assumes a Headlamp plugin for §16. Important context: Headlamp is now officially part of Kubernetes SIG UI as of 2025, making it the official Kubernetes-recommended dashboard after the original Kubernetes Dashboard was archived. Headlamp is a CNCF sandbox project developed by Kinvolk (now part of Microsoft). So your spec's GUI choice is well-aligned with the official Kubernetes UI direction.

For **policy simulation / differential replay** specifically, no fully open-source tool matches the spec's "differential simulation across the same evidence set" requirement. The closest existing implementations:
- Styra DAS provided decision replay capability for impact analysis to validate policy changes before deployment — and that's becoming open source via OCP.
- Gatekeeper's `enforcementAction: dryrun` and Kyverno's `validationFailureAction: Audit` provide live-shadow modes but not historical replay.
- Conftest re-runs against fixtures, but doesn't do differential classification.
- **OCP** explicitly aims at bundle regression testing against historical decision logs (the same primitive).

For **policy lineage graphs**, no open-source product offers an out-of-the-box governance-to-runtime visualization. CNAPP vendors (Wiz, Prisma) show resource-to-finding graphs; GRC tools show control-to-evidence tables; but nothing connects organizational objective → control → Rego package → enforcement event → audit record in a single graph view. This is a real spec differentiator.

For the **developer-facing tech-health surface** (similar to your namespace authoring view), **Spotify's Backstage Soundcheck** is the most mature analogue. Soundcheck visualizes checks for security, testing, reliability, and other development and operational standards via scorecards, actionable feedback, and positive reinforcement. Soundcheck v1.50 (April 2026) added MCP actions, OpenAPI CRUD endpoints, YAML campaign definitions, and audit events for policy creation/update/publication/deletion. Soundcheck is "policy scorecards as developer experience" rather than "policy enforcement," but the lineage and reporting affinity is strong.

---

## 9. Identity-aware enforcement (Keycloak layer)

Keycloak as the JWT issuer is a stable choice. Adjacent products that overlap parts of the spec:

- **Pomerium**, **Teleport**, **Boundary**, **StrongDM**, **Cloudflare Access** — zero-trust proxies that mix authn + policy, but generally don't expose governance-traceable enforcement records.
- **Auth0/Okta** + **Cedar** via Verified Permissions — application-layer authorization, lacking the K8s admission + audit-replay integration.
- Strata Maverics — orchestrates AVP/Cedar across legacy on-prem apps and pulls runtime attributes from directories/databases for fuller authorization context.

Keycloak with claim-mappers handling tenant/namespace/environment is consistent with industry practice; the spec's "claim normalization layer" (§15.4) is unusual in being explicit about the *governance* dimension of those claims, not just authn.

---

## 10. Supply chain governance (orthogonal layer your spec covers via §20.1)

The Sigstore/SLSA stack is now de facto:

- Sigstore reached GA with a 99.5% uptime SLO at SigstoreCon, providing production-grade signing and verification.
- Sigstore's Policy Controller allows creating SLSA-based policies in Kubernetes clusters.
- SLSA reached CNCF graduated status in early 2026.
- **Chainguard**, **Anchore Enterprise**, **JFrog Xray**, **GUAC** (CNCF) handle SBOM/provenance.

Kyverno's image signature verification (built around cosign) is now the most common admission-layer enforcement for this. Your spec correctly delegates supply-chain governance to these primitives and integrates via the audit schema, which is the right architectural choice.

---

## 11. AI governance (your §20.3)

This is the fastest-moving area. Direct competitors to "AI governance with policy-engine enforcement":

- **Credo AI** — automated policy packs aligned to EU AI Act, NIST AI RMF, ISO 42001, and SOC 2; real-time observability and trace-level policy enforcement for autonomous agents.
- **IBM watsonx.governance** — full-stack governance for enterprises on watsonx.
- **Holistic AI** — bias/fairness auditing.
- **Robust Intelligence** (now part of Cisco after 2024 acquisition) — adversarial security and assurance.
- **ModelOp**, **Arthur AI**, **OneTrust AI Governance**, **Fairly AI** — various AI-GRC positions.
- **FINOS AIGF + CC4AI** as the open-standard layer (open-source, regulation-as-code oriented).

These products handle AI governance well but don't connect it to a unified Kubernetes/CI/CD/identity policy fabric the way your spec proposes. The "AI as a first-class policy domain in a broader governance platform" is uncommon — most AI-governance products are standalone.

---

## 12. Per-product PDPs (your §17D libraries)

Your spec defines policy decision points and audit schemas for Jenkins, GitLab, Trivy, SonarQube, Grafana, and Elasticsearch/Kibana. There is *no* product on the market that defines these uniformly. Each tool's policy hook is currently ad-hoc:

- **Jenkins**: Audit Trail plugin + Pipeline policy steps (no PDP abstraction)
- **GitLab**: Compliance frameworks (15.9+) and Policy-as-code (15.6+) — closest to a built-in PDP model
- **GitHub**: Environments + required reviewers + branch protection — also a built-in but limited PDP
- **Trivy / SonarQube**: report-driven; consumed by separate gates
- **Grafana / Prometheus**: no native policy hook; integrations via Alertmanager webhooks
- **Elasticsearch / Kibana**: own RBAC, no governance integration

The spec's §17D explicit per-product decision-point catalog is **the most unusual contribution** of the spec — there's nothing comparable in the open ecosystem. Some commercial GRC platforms (ServiceNow IRM, AuditBoard/Optro) connect to these systems for evidence, but they don't model decision points as a first-class catalog.

---

## Overall verdict and what's genuinely novel

If I aggregate by spec layer, the picture is:

**Already mostly covered by existing products:**
- Runtime policy decision (OPA, Cedar, Cerbos, OPA Control Plane)
- Kubernetes admission (Gatekeeper, Kyverno, Nirmata, Azure Policy, Anthos Policy Controller)
- CI/CD IaC scanning (Checkov, Trivy, KICS)
- Continuous compliance evidence (Vanta, Drata, Hyperproof, Optro)
- Cloud posture management (Wiz, Prisma, Defender, Sysdig)
- Enterprise GRC suites (ServiceNow, Archer, MetricStream)
- AI governance (Credo AI, watsonx.governance, FINOS AIGF)
- Supply chain governance (Sigstore + SLSA + Kyverno)
- Kubernetes UI base (Headlamp now official)

**Partially covered:**
- Governance-control-to-policy-engine bridge → OSCAL Compass + C2P is the closest, FINOS CC4AI is emerging
- Identity-aware policy with JWT-claim mapping → Keycloak, Pomerium, AVP, but not unified
- Multi-tenant namespace-scoped authoring → Nirmata, Styra DAS (sunsetting), partially Red Hat ACS

**Largely uncovered — the spec's genuine contributions:**
1. **End-to-end governance-objective → enforcement-event lineage as a queryable graph** (§3.1 G1). No product offers this; OSCAL Compass produces assessment results but not a navigable lineage graph.
2. **Replay-capable audit schema as a first-class deliverable** (§13). OCSF normalizes events for SIEM, not for policy replay. This is a real gap.
3. **Differential policy simulation classifying "newly allowed / newly blocked"** (§17.4). OCP and Styra DAS approach this for OPA only; no cross-engine equivalent exists.
4. **Per-product PDP catalog** (§17D). Genuinely novel as a uniform abstraction.
5. **Approval-gated decisions with `suspend_pending_approval` + CRD pattern** (§17B/17C.6). Some workflow products (ServiceNow, GitLab) have approval gates, but none integrate cleanly with K8s admission's short request deadlines through a CRD-based pattern.

**Major industry shifts that should inform your design (May 2026):**
- Apple acqui-hired Styra in August 2025; EOPA, OPA Control Plane, and Regal are being open-sourced into CNCF. Build *on* OCP, not parallel to it.
- AuditBoard rebranded to Optro.
- Terrascan archived November 2025 — don't depend on it.
- Headlamp is the official Kubernetes SIG UI dashboard, validating your GUI choice.
- Kyverno is now AI-augmented via Nirmata's platform engineering assistant — a direct competitor to your simulation/authoring UI for the Kyverno half of your spec.
- FINOS Common Controls for AI Services (CC4AI) launched in 2025 with major banks + hyperscalers, and uses Gemara's Layer 2 model — your spec is already aligned with this stream.
- NIST CAISI's AI Agent Standards Initiative (February 2026) signals that agent-layer policy is the next regulatory frontier; your spec's identity-aware enforcement scales here, but the AI agent use case isn't deeply modeled.

The platform as specified is most defensible as an **integration/lineage/simulation layer on top of OPA Control Plane, Kyverno, OSCAL Compass, and the existing CNAPP ecosystem** — not as a replacement for any of them. The five "largely uncovered" items above are where the differentiation lives, and they're each genuinely valuable enough to be products in themselves.