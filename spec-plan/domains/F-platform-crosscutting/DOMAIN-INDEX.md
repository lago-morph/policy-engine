# Domain F — Platform & Cross-cutting — INDEX

**Domain lead:** F · **Date:** 2026-05-30 · **Status:** Complete (all components have SPEC + PLAN + ADVERSARIAL; F4 has ALT)
**Authoring note:** No subagent/Task tool was available in this environment; per the orchestration plan's robustness rule, the domain lead authored all documents directly with incremental writes. Cooperative-author, adversarial-reviewer, and alt-architecture perspectives are realized as distinct documents/personas.

---

## Components

| ID | Component | Purpose (one line) | Spec § / source | Docs | MVP status |
|---|---|---|---|---|---|
| **F1** | API requirements | The full governance control-plane API surface (resources, endpoints, auth, pagination, versioning) consistent with D2 storage authz + C2 audit schema | §21 (+§13,§15,§17A–E) | [SPEC](../../components/F1-api/SPEC.md) · [PLAN](../../components/F1-api/PLAN.md) · [ADVERSARIAL](../../components/F1-api/ADVERSARIAL-REVIEW.md) | MVP (the §21.1 8 endpoints) |
| **F2** | Deployment & extensibility | K8s-native topology (operators, CRDs, HA, POC scaling, hub-spoke) + the plugin/extensibility SPI (new PDPs/engines/IdPs/export-adapters) | §24–25 (+§22,§17C.6) | [SPEC](../../components/F2-deployment-extensibility/SPEC.md) · [PLAN](../../components/F2-deployment-extensibility/PLAN.md) · [ADVERSARIAL](../../components/F2-deployment-extensibility/ADVERSARIAL-REVIEW.md) | MVP (thin: operator + core CRDs) |
| **F3** | POC scale, MVP scope & sequencing | The MVP cut line (in/deferred + rationale), POC sizing math, phased sequence, AND the first-draft whole-platform build sequence across all 23 components | §22,§26,§27 | [SPEC](../../components/F3-mvp-sequencing/SPEC.md) · [PLAN](../../components/F3-mvp-sequencing/PLAN.md) (← whole-platform sequence) · [ADVERSARIAL](../../components/F3-mvp-sequencing/ADVERSARIAL-REVIEW.md) | MVP (meta) |
| **F4** | AI / agent governance extension | Agent governance as DELTAS on the base: 6 agent PDP resources, agent subject chain, behavioral-eval tier, audit deltas, agent PDP catalog, 3 approval patterns, standards alignment | reframed-for-ai.md (+Wedge-7) | [SPEC](../../components/F4-ai-agent-extension/SPEC.md) · [PLAN](../../components/F4-ai-agent-extension/PLAN.md) · [ADVERSARIAL](../../components/F4-ai-agent-extension/ADVERSARIAL-REVIEW.md) · [ALT](../../components/F4-ai-agent-extension/ALT-ai-as-separate-product-and-async-tier.md) | Phase-3 (deltas, base-first) |

---

## Cross-references into spec §-numbers

- **F1** ← §21.1 (the 8 seed endpoints), §13 (audit read shape), §15 (JWT/authz), §17A (scoped authz it enforces), §17B (approval endpoints), §17E (report endpoints), §22 (scale bounds queries).
- **F2** ← §24 (deployment targets + component table), §25 (six plugin categories), §17C.6 (CRD pattern), §22 (sizing), §23 (TLS/signing).
- **F3** ← §22 (POC scale), §26 (resolved decisions + open questions), §27 (MVP list + deferred). Consumes every domain's "thin slice."
- **F4** ← §20.3 (AI governance use case), and as DELTAS on §6 (Gemara), §9.2/§17.4 (lifecycle/differential), §13 (audit), §15/§17A (subject), §17B (approval), §17C.3/.4 (actions/PDP), §17D (catalog), §16 (console).

## Cross-references into scenarios (HL/DT)

See `analysis/scenarios-index.md` and `analysis/persona-spec-mapping.md`. F-domain components are exercised by: AI-governance scenarios (F4), multi-tenant K8s governance + deployment scenarios (F2/F1), and any scenario asserting the API surface or the MVP demo loop (F1/F3). The primary's `TRACEABILITY.md` will bind specific HL/DT IDs.

## Internal status

All four directories satisfy the doc contract (SPEC.md + PLAN.md minimum). F4, the designated high-value component, additionally carries an ALT architecture document covering two axes (separate-product wedge; async behavioral tier).
