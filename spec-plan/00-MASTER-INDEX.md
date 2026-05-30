# Master Index — Detailed Spec + Plan Corpus

**Entry point** for the detailed spec/plan produced for every piece of the
policy-engine design corpus. Built by a maximally-parallel multi-agent workflow
(orchestrator → 6 domain leads → cooperative/adversarial/alt authors), then a
cross-cutting reconciliation wave.

- **How this was produced:** [`00-ORCHESTRATION-PLAN.md`](00-ORCHESTRATION-PLAN.md)
- **Every unattended decision:** [`DECISIONS.md`](DECISIONS.md)
- **23 components · 6 domains · cooperative + adversarial + alt-architecture trees.**

Each component directory contains: `SPEC.md` (exhaustive engineering spec),
`PLAN.md` (parallelism-aware implementation plan), `ADVERSARIAL-REVIEW.md`
(red-team defect register), and — on high-value pieces — one or more `ALT-*.md`
alternative-architecture trees.

---

## Domains and components

### A · Governance Core — [domain index](domains/A-governance-core/DOMAIN-INDEX.md)
| Component | Spec § | Docs |
|---|---|---|
| A1 Governance Model & Gemara hierarchy | §6 | [SPEC](components/A1-governance-model/SPEC.md) · [PLAN](components/A1-governance-model/PLAN.md) · [ADV](components/A1-governance-model/ADVERSARIAL-REVIEW.md) · [ALT: event-sourced lineage](components/A1-governance-model/ALT-event-sourced-lineage-log.md) |
| A2 Policy Lifecycle | §7 | [SPEC](components/A2-policy-lifecycle/SPEC.md) · [PLAN](components/A2-policy-lifecycle/PLAN.md) · [ADV](components/A2-policy-lifecycle/ADVERSARIAL-REVIEW.md) |

### B · Policy Engines & Enforcement — [domain index](domains/B-policy-engines/DOMAIN-INDEX.md)
| Component | Spec § | Docs |
|---|---|---|
| B1 OPA/Rego & signed bundles | §8 | [SPEC](components/B1-opa-rego/SPEC.md) · [PLAN](components/B1-opa-rego/PLAN.md) · [ADV](components/B1-opa-rego/ADVERSARIAL-REVIEW.md) |
| B2 Gatekeeper | §9 | [SPEC](components/B2-gatekeeper/SPEC.md) · [PLAN](components/B2-gatekeeper/PLAN.md) · [ADV](components/B2-gatekeeper/ADVERSARIAL-REVIEW.md) |
| B3 Conftest | §10 | [SPEC](components/B3-conftest/SPEC.md) · [PLAN](components/B3-conftest/PLAN.md) · [ADV](components/B3-conftest/ADVERSARIAL-REVIEW.md) |
| B4 Engine selection, actions & CRDs | §17C | [SPEC](components/B4-engine-selection-crds/SPEC.md) · [PLAN](components/B4-engine-selection-crds/PLAN.md) · [ADV](components/B4-engine-selection-crds/ADVERSARIAL-REVIEW.md) · [ALT: OCP substrate](components/B4-engine-selection-crds/ALT-ocp-substrate.md) · [ALT: Kyverno-first](components/B4-engine-selection-crds/ALT-kyverno-first.md) |
| B5 Real-time enforcement flow | §18 | [SPEC](components/B5-realtime-enforcement/SPEC.md) · [PLAN](components/B5-realtime-enforcement/PLAN.md) · [ADV](components/B5-realtime-enforcement/ADVERSARIAL-REVIEW.md) |

### C · Evidence, Audit & Analytics — [domain index](domains/C-evidence-audit/DOMAIN-INDEX.md)
| Component | Spec § | Docs |
|---|---|---|
| C1 Privateer | §11 | [SPEC](components/C1-privateer/SPEC.md) · [PLAN](components/C1-privateer/PLAN.md) · [ADV](components/C1-privateer/ADVERSARIAL-REVIEW.md) |
| C2 Audit schema & event schema **(keystone)** | §12–13 | [SPEC](components/C2-audit-schema/SPEC.md) · [PLAN](components/C2-audit-schema/PLAN.md) · [ADV](components/C2-audit-schema/ADVERSARIAL-REVIEW.md) · [ALT: OCSF/event-log/CloudEvents](components/C2-audit-schema/ALT-ocsf-eventlog-cloudevents.md) |
| C3 Compliance analytics | §14 | [SPEC](components/C3-compliance-analytics/SPEC.md) · [PLAN](components/C3-compliance-analytics/PLAN.md) · [ADV](components/C3-compliance-analytics/ADVERSARIAL-REVIEW.md) |
| C4 Retrospective audit detection | §19 | [SPEC](components/C4-retrospective-detection/SPEC.md) · [PLAN](components/C4-retrospective-detection/PLAN.md) · [ADV](components/C4-retrospective-detection/ADVERSARIAL-REVIEW.md) |
| C5 Reporting | §17E | [SPEC](components/C5-reporting/SPEC.md) · [PLAN](components/C5-reporting/PLAN.md) · [ADV](components/C5-reporting/ADVERSARIAL-REVIEW.md) |

### D · Identity, Authz & Security — [domain index](domains/D-identity-authz/DOMAIN-INDEX.md)
| Component | Spec § | Docs |
|---|---|---|
| D1 Keycloak/JWT & mapping layer | §15 | [SPEC](components/D1-keycloak-jwt/SPEC.md) · [PLAN](components/D1-keycloak-jwt/PLAN.md) · [ADV](components/D1-keycloak-jwt/ADVERSARIAL-REVIEW.md) |
| D2 Scoped RBAC & storage authz | §17A | [SPEC](components/D2-scoped-rbac-storage/SPEC.md) · [PLAN](components/D2-scoped-rbac-storage/PLAN.md) · [ADV](components/D2-scoped-rbac-storage/ADVERSARIAL-REVIEW.md) · [ALT: OPA/RLS/SpiceDB](components/D2-scoped-rbac-storage/ALT-opa-rls-spicedb.md) |
| D3 Approval-gated decisions | §17B | [SPEC](components/D3-approval-gated/SPEC.md) · [PLAN](components/D3-approval-gated/PLAN.md) · [ADV](components/D3-approval-gated/ADVERSARIAL-REVIEW.md) |
| D4 Security requirements | §23 | [SPEC](components/D4-security/SPEC.md) · [PLAN](components/D4-security/PLAN.md) · [ADV](components/D4-security/ADVERSARIAL-REVIEW.md) |

### E · Simulation & Console — [domain index](domains/E-simulation-console/DOMAIN-INDEX.md)
| Component | Spec § | Docs |
|---|---|---|
| E1 Simulation & dry-run | §17 | [SPEC](components/E1-simulation/SPEC.md) · [PLAN](components/E1-simulation/PLAN.md) · [ADV](components/E1-simulation/ADVERSARIAL-REVIEW.md) · [ALT: batch-vs-shadow](components/E1-simulation/ALT-replay-batch-vs-shadow.md) · [ALT: decision-log reuse](components/E1-simulation/ALT-decisionlog-reuse.md) |
| E2 Governance console / Headlamp | §16 | [SPEC](components/E2-governance-console/SPEC.md) · [PLAN](components/E2-governance-console/PLAN.md) · [ADV](components/E2-governance-console/ADVERSARIAL-REVIEW.md) |
| E3 Per-product PDP libraries | §17D | [SPEC](components/E3-pdp-libraries/SPEC.md) · [PLAN](components/E3-pdp-libraries/PLAN.md) · [ADV](components/E3-pdp-libraries/ADVERSARIAL-REVIEW.md) |

### F · Platform & Cross-cutting — [domain index](domains/F-platform-crosscutting/DOMAIN-INDEX.md)
| Component | Spec § | Docs |
|---|---|---|
| F1 API requirements | §21 | [SPEC](components/F1-api/SPEC.md) · [PLAN](components/F1-api/PLAN.md) · [ADV](components/F1-api/ADVERSARIAL-REVIEW.md) |
| F2 Deployment & extensibility | §24–25 | [SPEC](components/F2-deployment-extensibility/SPEC.md) · [PLAN](components/F2-deployment-extensibility/PLAN.md) · [ADV](components/F2-deployment-extensibility/ADVERSARIAL-REVIEW.md) |
| F3 POC scale, MVP & sequencing | §22,26,27 | [SPEC](components/F3-mvp-sequencing/SPEC.md) · [PLAN](components/F3-mvp-sequencing/PLAN.md) · [ADV](components/F3-mvp-sequencing/ADVERSARIAL-REVIEW.md) |
| F4 AI / agent governance extension | reframed-for-ai | [SPEC](components/F4-ai-agent-extension/SPEC.md) · [PLAN](components/F4-ai-agent-extension/PLAN.md) · [ADV](components/F4-ai-agent-extension/ADVERSARIAL-REVIEW.md) · [ALT: separate product / async tier](components/F4-ai-agent-extension/ALT-ai-as-separate-product-and-async-tier.md) |

---

## Cross-cutting reconciliation (Wave 2)

| Doc | Purpose |
|---|---|
| [`MASTER-PLAN.md`](cross-cutting/MASTER-PLAN.md) | Authoritative parallelism-maximizing whole-platform build DAG, critical path, MVP→GA |
| [`MASTER-PLAN-ALT.md`](cross-cutting/MASTER-PLAN-ALT.md) | Alternative wedge-first sequencing (go-to-market led) |
| [`CROSSCUT-ADVERSARIAL.md`](cross-cutting/CROSSCUT-ADVERSARIAL.md) | Ranked cross-domain contradiction register + resolutions |
| [`DATA-MODEL.md`](cross-cutting/DATA-MODEL.md) | Unified entity/relationship model + shared join keys |
| [`INTER-DOMAIN-CONTRACTS.md`](cross-cutting/INTER-DOMAIN-CONTRACTS.md) | The 6 frozen inter-domain contracts |
| [`TRACEABILITY.md`](cross-cutting/TRACEABILITY.md) | Component ↔ §§ ↔ personas ↔ 100 scenarios |

---

## Source corpus (inputs)

- `openssf_opa_unified_governance_platform_spec v1.md` — authoritative spec (28 §, +17A–E)
- `policy engine personas.md` · `analysis/persona-spec-mapping.md` — 5 personas
- `policy engine market research.md` · `policy engine reframed market position.md` — market + wedges
- `policy engine reframed for ai.md` — AI/agent reframe
- `analysis/scenarios-index.md` + `analysis/scenarios/` — 100 scenarios (20 HL + 80 DT)
- `INDEX.md` — original source-doc guide

---

## Corpus statistics

| Metric | Value |
|---|---|
| Total spec-plan files | 104 |
| Total lines | ~15,600 |
| Components (SPEC + PLAN + ADVERSARIAL each) | 23 |
| Alternative-architecture trees | 8 (A1, B4×2, C2, D2, E1×2, F4) |
| Domains (INDEX + SUMMARY + ADVERSARIAL each) | 6 |
| Cross-cutting reconciliation docs | 6 |
| Unified data-model entities | 51 |
| Consolidated cross-domain defects | 22 (XD-1..XD-22) |
| Agents used | 6 domain leads + 5 cross-cutting (+ orchestrator) |

## Headline reconciliation flags (read before building)

These are decisions the cross-cutting wave surfaced that **override** the
parallel-authored component docs. They are the first things to action.

1. **🔴 Re-open C2 before freezing it (`v1.0` → `v1.0-rc`).** `CROSSCUT-ADVERSARIAL.md`
   (XD-3, XD-1, XD-11) found the "frozen" 36-field audit schema already baked in the
   action-model conflation (`mutate` as a sibling of `deny`; three incompatible closed
   action enums) and a self-contradictory external-data-value capture rule. Land the
   9 build-blocking fixes, then re-freeze. The "frozen" framing in
   `components/C2-audit-schema/SPEC.md`, `INTER-DOMAIN-CONTRACTS.md`, and `DATA-MODEL.md`
   is therefore **provisional** pending the rc pass.
2. **🟠 `correlation_id` is unresolved across two cross-cutting docs (intentional, open).**
   `DATA-MODEL.md` keeps `correlation_id = K8s AdmissionReview UID`; `INTER-DOMAIN-CONTRACTS.md`
   **OV-4** overrides it to a retry-stable *logical-flow* id anchored to the
   `PolicyApprovalRequest` CR name, because an approval retry mints a new UID (XD-8). The
   logical-flow id is the recommended resolution; the per-admission UID demotes to
   `engine_context`. Action this in the C2 rc pass.
3. **🟢 `replay_completeness` middle state renamed `partial` → `best_effort`** (DATA-MODEL R1,
   contracts OV-1), kept distinct from `jwt_claims_completeness=partial`. Convergently
   reached by two independent agents.
4. **🟠 Storage-scope authz (XD-5):** analytics/reporting aggregate reads (C3/C5/E1) must
   pass D2's `ScopePredicate`; today they likely bypass it. Correctness fix, not hardening.
5. **🟠 Additive roles break separation-of-duties (XD-4):** enforce mutual-exclusion on
   SoD role pairs at grant time, or D3/D4 approval controls are defeated.

**Top cross-domain defects:** see `cross-cutting/CROSSCUT-ADVERSARIAL.md` §-ranked
register (XD-1..XD-22) and its build-blocking subset (§4) — the 9 fixes that must land
before the foundation contracts (C2, B4, D2, replay-completeness) re-freeze.

**Where to start building:** `cross-cutting/MASTER-PLAN.md` (platform-first, 5 waves,
critical path `C2→B1→E1→E2` co-equal with `D1→D2`) or `cross-cutting/MASTER-PLAN-ALT.md`
(wedge-first, lead with the Compliance Digital Twin: C2→B1→E1). Both freeze the same 5
load-bearing contracts first.
