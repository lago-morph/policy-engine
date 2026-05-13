# Document Index

This repository holds the design corpus for a unified governance and policy-enforcement platform built around OpenSSF Gemara, OPA/Rego, Conftest, Gatekeeper, Kyverno, Privateer, and Keycloak. The five source documents below cover the technical specification plus three strategic / user-facing companion documents (market landscape, personas, AI reframe, and market positioning).

Read them in roughly this order:

1. **`openssf_opa_unified_governance_platform_spec v1.md`** — the authoritative technical spec.
2. **`policy engine personas.md`** — who the spec is for.
3. **`policy engine market research.md`** — what already exists in the market.
4. **`policy engine reframed market position.md`** — how to position against that market.
5. **`policy engine reframed for ai.md`** — how the same platform extends to AI-agent governance.

---

## `openssf_opa_unified_governance_platform_spec v1.md`

**Purpose.** Defines a comprehensive product specification for a unified governance, policy enforcement, audit, and simulation platform combining OpenSSF Gemara, OPA/Rego, Conftest, Gatekeeper, Kyverno, Privateer, and Keycloak. It provides an executable, traceable bridge between governance intent and runtime enforcement across Kubernetes, CI/CD, identity, and supply-chain systems.

**Type.** Technical product specification (architecture + requirements).
**Length.** ~1,720 lines, 28 numbered top-level sections (plus 17A–17E subsections).

**Top-level structure.**

- 1. Executive Summary
- 2. Problem Statement
- 3. Product Goals
- 4. Non-Goals
- 5. High-Level Architecture
- 6. Governance Model
- 7. Policy Lifecycle
- 8. OPA Integration
- 9. Gatekeeper Integration
- 10. Conftest Integration
- 11. Privateer Integration
- 12. Audit Schema Framework
- 13. Standardized Audit Event Schema
- 14. Compliance Analytics Engine
- 15. Keycloak and JWT Integration
- 16. Graphical Governance Console
- 17. Policy Simulation and Dry-Run Framework
- 17A. Scoped Roles, Permissions, and Storage Authorization
- 17B. Approval-Gated Policy Decisions
- 17C. Policy Actions, Engine Gaps, and Kyverno/OPA Extensions
- 17D. Product Decision Point and Action Libraries
- 17E. Reporting Requirements
- 18. Real-Time Enforcement Flow
- 19. Retrospective Audit Detection Scenario
- 20. Publicly Derived Use Cases
- 21. API Requirements
- 22. Proof-of-Concept Scale Requirements
- 23. Security Requirements
- 24. Deployment Model
- 25. Extensibility Model
- 26. Resolved Design Guidance and Remaining Open Questions
- 27. Recommended Initial MVP
- 28. Strategic Value

**Key information.**

- Core component layering: Gemara (governance) → OPA/Rego, Kyverno, Conftest, Privateer (policy) → Gatekeeper (enforcement) → Keycloak (identity) → Audit Schema + Analytics + Governance Console.
- Seven-layer governance hierarchy (Objectives → Domains → Controls → Enforcement / Evaluation / Evidence / Exception Requirements) and five enforcement classes (Runtime, Build-Time, Detective, Manual, Advisory).
- Replay-capable standardized audit event schema with required fields (`jwt_claims`, before/after state, `external_data_refs`, `replay_completeness`, etc.) designed so simulations are authoritative.
- Differential simulation semantics comparing previous vs. new policy outcomes (newly blocked, newly allowed, unchanged), with tagging workflows.
- Scoped RBAC model (§17A): nine roles, permission primitives, Keycloak claim mapping, and a hard requirement that authorization is enforced at the storage layer — not just in the GUI.
- Approval-gated decisions (§17B): `suspend_pending_approval` and `require_async_check` actions, webhook schema, and the Kubernetes-admission constraint that long-running approvals must use deny-with-approval-required + CRDs.
- Engine selection guidance (§17C): explicit matrix for OPA/Gatekeeper vs. Kyverno, a 13-action taxonomy, and a PDP typology.
- Per-product decision-point libraries (§17D) for Kubernetes, Keycloak, Jenkins, GitLab, Trivy, OWASP, SonarQube, Grafana/Prometheus, Elasticsearch/Kibana.
- POC scale targets (1–5 clusters, 100–1000 evals/sec, 10k–500k audit events/day, 25–100 policy controls) and an explicit MVP scope with deferred features.
- Resolved design decisions: Gemara-to-Rego generation, OCSF optional, Wasm non-required, signed bundles, storage out of scope for POC.

**Direct references worth citing.**

- `Lines 137–149 (§5.1 "Core Components")` — canonical component-to-purpose table for the platform's architecture.
- `Lines 446–535 (§13 "Standardized Audit Event Schema")` — replay-capable audit schema fields and full JSON example; essential for any audit/replay implementation.
- `Lines 738–847 (§17 "Policy Simulation and Dry-Run Framework")` — first-class simulation requirements, nine simulation modes, differential semantics, and audit-driven test workflow.
- `Lines 850–985 (§17A "Scoped Roles, Permissions, and Storage Authorization")` — role model, permission primitives, Keycloak claim mapping, storage-level enforcement requirement.
- `Lines 1062–1183 (§17C "Policy Actions, Engine Gaps, and Kyverno/OPA Extensions")` — OPA vs. Gatekeeper vs. Kyverno decision matrix, action taxonomy, PDP typology.
- `Lines 1642–1696 (§26 "Resolved Design Guidance" + §27 "Recommended Initial MVP")` — locked-in design decisions vs. open questions plus the explicit MVP build list.

**When to consult.** Open this file when designing or scoping any component of the platform — especially when choosing between OPA, Gatekeeper, and Kyverno; defining audit/replay schemas; implementing RBAC or storage authorization; building simulation / differential analysis features; integrating Keycloak/JWT claims; modeling per-product decision points; or determining MVP/POC scope. It is the authoritative source of truth.

---

## `policy engine market research.md`

**Purpose.** A market survey assessing what already exists in the ecosystem comparable to the unified governance/policy spec. Maps spec capabilities layer-by-layer to commercial, open-source, cloud, and GRC alternatives as of May 2026, and identifies genuine differentiators.

**Type.** Market analysis / competitive landscape report.
**Length.** ~245 lines, 13 major sections.

**Top-level structure.**

- How I broke the spec into evaluable pieces
- 1. The single most direct architectural equivalent: OSCAL Compass + C2P
- 2. The OPA control plane has changed dramatically in the last year
- 3. Kubernetes admission enforcement (Gatekeeper + Kyverno layer)
- 4. CI/CD validation and IaC scanning (Conftest layer)
- 5. Continuous compliance & evidence collection (Privateer + audit schema layer)
- 6. Enterprise IRM / GRC suites (overall governance layer)
- 7. Cloud-Native Application Protection Platforms (CNAPP)
- 8. Visualization, simulation, and policy lineage (the Governance Console layer)
- 9. Identity-aware enforcement (Keycloak layer)
- 10. Supply chain governance (orthogonal layer the spec covers via §20.1)
- 11. AI governance (the spec's §20.3)
- 12. Per-product PDPs (the spec's §17D libraries)
- Overall verdict and what's genuinely novel

**Key information.**

- Decomposes the spec into 10 capability layers (governance model, runtime PDP, K8s admission, CI/CD policy, evidence collection, audit replay, identity, visualization, simulation, per-product PDPs).
- Identifies **OSCAL Compass + C2P** (CNCF sandbox, IBM/Red Hat) as the closest architectural equivalent to the Gemara + Privateer + OPA loop.
- Documents the **August 2025 Apple acqui-hire of Styra**, the sunset of Styra DAS, and open-sourcing of EOPA / OPA Control Plane (OCP) / Regal into CNCF.
- Surveys Kubernetes admission: Gatekeeper, Kyverno, Nirmata's AI Platform Engineering Assistant (Nov 2025), Azure Policy, Anthos Policy Controller, AWS Cedar/AVP, Red Hat ACS/RHACM.
- IaC scanning landscape: Checkov, Trivy (absorbed tfsec), KICS, Snyk IaC, Terrascan archived Nov 2025.
- Tabulates continuous-compliance vendors (Vanta, Drata, Secureframe, Sprinto, Hyperproof, Optro / ex-AuditBoard) and enterprise GRC suites (ServiceNow, Archer, MetricStream, OpenPages, OneTrust, LogicGate, Diligent).
- Compares CNAPP vendors (Wiz, Prisma Cloud, Defender for Cloud, SentinelOne, Sysdig, Lacework, Aqua, Red Hat ACS) with mindshare data.
- Notes FINOS Common Cloud Controls / AIGF v2.0 / CC4AI alignment with Gemara Layer 2.
- Headlamp's promotion to official Kubernetes SIG UI (2025); Backstage Soundcheck as a developer-facing analogue.
- Concludes with five "largely uncovered" areas representing genuine differentiation: end-to-end governance-to-enforcement lineage graph, replay-capable audit schema, differential policy simulation, per-product PDP catalog, approval-gated CRD admission pattern.

**Direct references worth citing.**

- `Lines 24–34 (§1 "OSCAL Compass + C2P")` — nearest existing competitor architecture and Gemara's relative maturity gap.
- `Lines 36–46 (§2 "The OPA control plane has changed dramatically")` — Apple/Styra/OCP shift that should reshape build-vs-adopt decisions.
- `Lines 82–96 (§5 "Continuous compliance & evidence collection")` — vendor table plus the key observation that no vendor emits governance-traceable evidence from enforcement decisions (the Privateer gap).
- `Lines 138–150 (§8 "Visualization, simulation, and policy lineage")` — confirms Headlamp choice and identifies policy-lineage graphs as an open market gap.
- `Lines 194–205 (§12 "Per-product PDPs")` — argues §17D's PDP catalog is the most unusual/novel contribution of the spec.
- `Lines 209–244 ("Overall verdict and what's genuinely novel")` — condensed bottom line: covered, partially covered, uncovered, and the major May 2026 industry shifts.

**When to consult.** Open when scoping competitive positioning, deciding build-vs-adopt for any spec layer, writing pitch/strategy material, evaluating whether to build on OPA Control Plane / OSCAL Compass / Kyverno rather than parallel to them, or identifying which spec sections (lineage graph, replay schema, differential simulation, per-product PDPs, approval CRDs) carry genuine differentiation worth investing in.

---

## `policy engine personas.md`

**Purpose.** A persona analysis derived from the spec, profiling five archetypal users across the governance-intent-to-runtime-evidence lifecycle. Explains who each persona is, where current tooling fails them, and how the spec's proposed capabilities close those gaps.

**Type.** Persona doc / user-research narrative (companion to the spec).
**Length.** ~97 lines, 5 persona sections plus framing preamble.

**Top-level structure.**

- Framing preamble (persona-selection rationale)
- Persona 1: Priya — Compliance & GRC Lead
- Persona 2: Marcus — Platform Security Engineer
- Persona 3: Jess — SRE / Cluster Operator
- Persona 4: Sam — Application Developer
- Persona 5: Daniel — Internal / External Auditor
- Each persona follows the same sub-structure: **Who they are**, **Day-to-day**, **Gaps with current tooling**, **How the spec helps**, **Before / after**.

**Key information.**

- Rationale for five personas rather than expanding the spec's §17A.2 role list — which roles were intentionally collapsed (Auditor + Compliance Analyst; Policy Library Maintainer + Namespace Policy Author; Identity/IAM admin folded into Platform Security Engineer).
- Priya (GRC Lead): owns SOC 2 / ISO 27001 / HIPAA story; key gap is continuous evidence vs. point-in-time sampling; mapped to §6, §11, §14, §17E.3, §19.
- Marcus (Platform Security Engineer): writes Rego/Gatekeeper/Kyverno; key gaps are seam-drift between CI and admission, no Rego simulation, bespoke Keycloak claim mapping; mapped to §7, §15.4, §16.3, §17, §17C.
- Jess (SRE / Cluster Operator): 8-cluster fleet across two clouds; key gaps are policy lineage during incidents, bypass detection, multi-cluster visibility; mapped to §13, §14.2, §16.2, §16.3, §19.
- Sam (Application Developer): payments-team Go developer who consumes policy; key gaps are unhelpful CI failure messages, CI-vs-admission divergence, painful exception requests; mapped to §7, §10, §16.3, §17.5, §17A.5, §17B.
- Daniel (External Auditor): SOC 2 Type II engagement; key gaps are sample-based testing, inability to prove absence of bypass, log integrity; mapped to §11, §14.2, §17.4, §17A.2, §17E.3, §19, §23.
- Each persona includes a "Before / after" narrative with measurable improvements (e.g., 2-day blocker → 20 minutes; 90-minute incident → 12 minutes; week of audit testing → one hour).
- Recurring spec themes: Goal G1 (Governance-to-Enforcement Traceability), Governance Console, Privateer evaluation logs, Differential Simulation, tamper-evident evidence.
- Distinguishes attestation/workflow tools (Vanta, Drata, OneTrust, ServiceNow GRC) from runtime-policy enforcement; notes cloud-native (AWS Audit Manager, Azure Policy Compliance, GCP SCC) and OSS (Comply, Eramba) limitations.

**Direct references worth citing.**

- `Lines 1–5 (framing preamble)` — persona-selection methodology and merged-role justification.
- `Lines 13–17 (Priya "Gaps with current tooling")` — concise competitive view of GRC tooling and the continuous-evidence gap.
- `Lines 31–36 (Marcus "Gaps with current tooling")` — best inventory of OPA/Gatekeeper/Kyverno/Conftest seam problems and missing simulation story.
- `Lines 50–55 (Jess "Gaps with current tooling")` — incident-triage friction and bypass-detection blind spots.
- `Lines 69–74 (Sam "Gaps with current tooling")` — developer-experience pain around exceptions and CI/admission divergence.
- `Lines 88–94 (Daniel "Gaps with current tooling")` — auditor-specific limitations around sampling, bypass proof, and log tamper-evidence.

**When to consult.** Open when you need user-centered framing for the spec — product messaging, UX flows in the Governance Console / Headlamp plugin, prioritization of spec sections by which persona they serve, or before/after narratives for stakeholder reviews. Also useful as a cross-reference from personas back into specific spec sections (§6, §7, §10, §11, §13, §14, §15.4, §16, §17, §17A, §17B, §17C, §17E, §19, §23).

---

## `policy engine reframed for ai.md`

**Purpose.** Argues that the existing cross-product policy-engine spec is already ~70% "agent-shaped" and can be reframed to govern AI agents by relabeling and extending — not redesigning — its layers. Walks layer by layer (resource, subject, enforcement, audit, PDPs, approvals, visualization, standards) explaining what translates directly versus what needs new primitives.

**Type.** Architectural reframing / design commentary on the spec.
**Length.** ~149 lines, 10 top-level sections plus intro and conclusion.

**Top-level structure.**

- Reframing the platform around AI agents (intro)
- What is "the resource" being governed?
- What is "the subject"?
- The trust gradient is the lifecycle the spec already describes
- What is "enforcement"? Adding a behavioral tier
- The audit schema additions
- The per-component PDP catalog, rewritten for agents
- What approval gates apply to (the prompt example)
- Visualization implications
- High-level recommendations on standards and protocols
- What stays unchanged

**Key information.**

- Six new agent "resources" become PDP decision points: prompt, assembled context window, model invocation, tool/MCP call, output, resource-budget events.
- Subject model expands from a single user to a chain: originating user, agent identity, model identity, tool catalog, capability token, trust grade, delegation chain.
- Trust is reframed as a graded property of the (agent, model, tools, context) tuple — not of the originating user alone.
- The "trust gradient" (restrictive → loosened) maps directly onto the existing §9.2 enforcement-mode lifecycle (Deny → Warn → Dry Run → Audit) plus §17.4 differential simulation.
- Proposes a new **behavioral-evaluation enforcement tier** between Runtime and Detective, with evaluators for hallucination/groundedness, drift, looping, resource consumption, output quality, and sensitive-action detection.
- Audit schema extensions: `request_object` carries full agent state; `external_data_refs` gains RAG/MCP IDs; new `evaluator_results` sub-record; `replay_completeness` distinguishes policy-decision replay (achievable) from exact model-output replay (best-effort).
- New PDP catalog (model gateway, MCP gateway, RAG/vector retrieval, agent memory, agent runtime, output sink, eval gate, resource accounting) mirrors the §17D library structure.
- Three approval-gate patterns: pre-prompt approval, mid-session approval (CRD-based), capability-token approval at session start (OAuth-scope-like).
- Standards alignment recommended: MCP, OpenTelemetry GenAI semantic conventions, Sigstore-style attestation, FINOS AIGF v2.0, NIST AI Agent Standards Initiative, NIST AI RMF GenAI profile, EU AI Act Annex III.
- Six-item delta list of what actually needs to change in the original spec (in "What stays unchanged").

**Direct references worth citing.**

- `Lines 11–22 ("What is 'the resource' being governed?")` — canonical six-resource table mapping each agent decision point to a real-time hook and example policies.
- `Lines 28–38 ("What is 'the subject'?")` — agent subject chain and the conceptual shift that trust is a tuple property, not a user property.
- `Lines 44–51 ("The trust gradient is the lifecycle the spec already describes")` — explicit mapping of trust gradient onto §9.2 enforcement-mode lifecycle and §17.4 differential simulation; central thesis.
- `Lines 59–70 ("What is 'enforcement'? Adding a behavioral tier")` — behavioral-evaluator table with concerns, evaluation patterns, and decision outcomes; the new enforcement tier.
- `Lines 76–84 ("The audit schema additions")` — concrete field-level additions to §13.3 and the replay-completeness note (policy decisions vs. model outputs).
- `Lines 92–103 ("The per-component PDP catalog, rewritten for agents")` — new agent-specific PDP catalog; confirms §17C.3 action taxonomy (including `suspend_pending_approval`) applies unchanged.
- `Lines 141–149 ("What stays unchanged")` — numbered six-item delta list; the most actionable summary of required work.

**When to consult.** Open when bridging the original (pre-AI) spec to AI-agent governance: extending the spec's resource/subject/audit/PDP models for agents, designing trust-gradient or behavioral-evaluator features, mapping approval-gate patterns to agent workflows, or aligning the platform to external AI governance standards (MCP, OTel GenAI, FINOS AIGF, NIST AI RMF, EU AI Act). The orienting document for any contributor whose work touches both the legacy spec and AI-agent functionality.

---

## `policy engine reframed market position.md`

**Purpose.** A strategic positioning memo that reframes the all-in-one platform into a set of narrower "wedge" strategies, each anchored on the connective tissue between existing GRC, runtime-enforcement, and CNAPP incumbents. Argues against build-everything and enumerates viable go-to-market angles, partner integrations, and architectural pivots.

**Type.** Strategic market-positioning memo (with light architectural recommendations).
**Length.** ~180 lines, ~14 top-level sections (8 wedges + stacking patterns + integration matrix + architecture pivots + intro + final recommendation).

**Top-level structure.**

- Reframing the strategic position (intro)
- Wedge 1: The "Compliance Digital Twin" — replay and simulation as a standalone service
- Wedge 2: The "Governance Lineage Graph" — be the spine, not the muscles
- Wedge 3: The "Backstage / Soundcheck for Compliance" — developer-surface play
- Wedge 4: The "Vanta-Compatible Runtime Evidence Engine"
- Wedge 5: Build on OPA Control Plane, don't compete with it
- Wedge 6: The "Approval Gate Mesh" — solving the one thing no one else does well
- Wedge 7: The AI / Agent governance layer
- Wedge 8: The PDP Catalog as an open standard
- Stacking patterns — how the wedges combine (Stacks A, B, C)
- Integration matrix — what to be "complementary to," concretely
- Pivots in the base architecture itself
- What I'd weigh hardest if forced to pick one path

**Key information.**

- Central thesis: incumbents own every horizontal layer (GRC, runtime, CNAPP, compliance-as-code) but no one owns the *connective tissue* between them — that is the strategic opening.
- Eight discrete wedge strategies, from a standalone replay/simulation SaaS to a community-owned PDP catalog, each with ingestion sources, export targets, and a "why this works in 2026" rationale.
- Three stacking combinations: Stack A "Connective Tissue", Stack B "OPA-and-Friends Successor", Stack C "FINOS / Regulated Vertical Play" — each bundles wedges into a coherent company shape.
- Partner-by-partner integration matrix giving the exact sentence each potential partner (Vanta, ServiceNow IRM, OSCAL Compass, Nirmata, Red Hat ACS, Wiz, Sigstore, Backstage, Headlamp, FINOS, OCP, Credo AI) would say about the product.
- Six concrete architectural pivots to the original spec: schema-as-product (Sigstore-style), ingestion-only simulation, lineage-as-graph-DB, plugin-everywhere GUI, external-tool export adapters as first-class, JWT/identity-source pluggability replacing the Keycloak mandate.
- Market context cues: Styra DAS sunsetting after the Apple acqui-hire, OPA Control Plane open-sourcing, NIST CAISI AI Agent Standards Initiative (Feb 2026), FINOS AIGF v2.0, EU AI Act enforcement timeline, Vanta/Drata "hourly snapshot" weakness.
- Cross-references to specific spec sections (§3.1 G1, §5.1, §8.3, §13, §15.4, §16, §17, §17B, §17D).
- Final recommendation: lead with Wedge 5 + Wedge 1 for near-term revenue, layer Wedges 2 + 8 for long-term defensibility, use Wedge 4 as compliance-buyer wedge, treat Wedge 6 as tactical add-on, watch Wedge 7 for six months.

**Direct references worth citing.**

- `Lines 3–5 ("Reframing the strategic position" intro)` — the core "connective tissue" thesis in one paragraph; foundational.
- `Lines 9–21 (Wedge 1: Compliance Digital Twin)` — replay+simulation SaaS, smallest-viable-version pitch, and the Vanta/Drata depth-layer argument.
- `Lines 68–78 (Wedge 5: Build on OPA Control Plane)` — the Styra-DAS-orphan opportunity and the "Chainguard for OCP" framing; key for revenue-path discussions.
- `Lines 117–125 (Stacking patterns A/B/C)` — canonical three-stack framing for comparing go-to-market shapes.
- `Lines 133–148 (Integration matrix table)` — partner-by-partner positioning sentences; directly reusable in pitch and BD material.
- `Lines 154–166 (Pivots in the base architecture itself)` — six concrete spec-level changes; the bridge from positioning back to engineering.
- `Lines 170–180 ("What I'd weigh hardest if forced to pick one path")` — explicit recommended sequencing and trade-offs between stacks.

**When to consult.** Read when making product-scope, go-to-market, partnership, or roadmap-prioritization decisions — particularly when deciding *what not to build*, evaluating whether a feature should be standalone vs. embedded, drafting partner messaging, or revisiting the base spec to align architecture with a chosen wedge. Less relevant for day-to-day implementation work.

---

## Quick cross-reference

| If you need to… | Start with |
| --- | --- |
| Look up a normative requirement, schema field, or §-number | `openssf_opa_unified_governance_platform_spec v1.md` |
| Know who suffers without the platform and how to message them | `policy engine personas.md` |
| Decide build-vs-adopt for a given layer | `policy engine market research.md` |
| Choose a go-to-market wedge or partnership angle | `policy engine reframed market position.md` |
| Extend the platform to govern AI agents | `policy engine reframed for ai.md` |
