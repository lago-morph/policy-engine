# Domain E — Simulation & Console — DOMAIN INDEX

**Domain lead:** E · **Date:** 2026-05-30 · **Status:** Components drafted (SPEC+PLAN+ADVERSARIAL all present; E1 has 2 ALTs)

Domain E is the **product surface**: the three components that turn the governance/enforcement/audit machinery (Domains A–D) into something a human uses and trusts — simulate-before-promote (E1), see-and-author (E2), and reuse-per-product (E3). Per market research, all three contain the platform's **strongest differentiators**: differential simulation (open gap, research §8), governance→runtime lineage graph (open gap, research §8), and the per-product PDP catalog (the spec's "most unusual contribution," research §12).

---

## Components

| ID | Component | Spec § | Purpose (one line) | Docs | ALT |
|---|---|---|---|---|---|
| **E1** | Policy simulation & dry-run framework | §17 | 9 simulation modes + differential algorithm + tagging; "verify my policy before I promote it" | [SPEC](../../components/E1-simulation/SPEC.md) · [PLAN](../../components/E1-simulation/PLAN.md) · [ADVERSARIAL](../../components/E1-simulation/ADVERSARIAL-REVIEW.md) | [batch-vs-shadow](../../components/E1-simulation/ALT-replay-batch-vs-shadow.md) · [decisionlog-reuse](../../components/E1-simulation/ALT-decisionlog-reuse.md) |
| **E2** | Governance console / Headlamp GUI | §16 | 5 views + lineage graph + Headlamp plugin + OIDC; the dual-layer-scoped UI | [SPEC](../../components/E2-governance-console/SPEC.md) · [PLAN](../../components/E2-governance-console/PLAN.md) · [ADVERSARIAL](../../components/E2-governance-console/ADVERSARIAL-REVIEW.md) | — |
| **E3** | Per-product PDP libraries | §17D | Reusable catalog of decision points/hooks/replay schemas/examples for 9 products | [SPEC](../../components/E3-pdp-libraries/SPEC.md) · [PLAN](../../components/E3-pdp-libraries/PLAN.md) · [ADVERSARIAL](../../components/E3-pdp-libraries/ADVERSARIAL-REVIEW.md) | — |

---

## Spec-section coverage

| Spec § | Title | Component |
|---|---|---|
| §16.1–§16.3 | Graphical Governance Console (5 views, Headlamp, OIDC) | E2 |
| §17.1–§17.6 | Policy Simulation & Dry-Run (9 modes, differential, tagging, §17.5/§17.6 workflows) | E1 |
| §17.3 | Audit-driven simulation requirements / replay_completeness gate | E1 (consumes C2 §13) |
| §17.4 | Differential simulation semantics (4-quadrant matrix + tagging) | E1 |
| §17D.1–§17D.11 | Product Decision Point & Action Libraries (9 products + pattern) | E3 |
| §17C (referenced) | PDP model, action taxonomy, CRDs | E3 (uses), E1 (CRD) |
| §17E.4 (produced) | Simulation Report | E1 produces, E2/C5 render |
| §17A.4/§17A.5 (consumed) | OIDC claims + storage-layer scope | E2, E1, E3 honor |

---

## Scenario coverage (HL/DT)

| Scenario | Title | Component |
|---|---|---|
| DT-39 | Governance Graph trace control→Rego→enforcement | E2 |
| DT-40 | Rego Explorer coverage | E2 |
| DT-41 | Runtime Enforcement recent denies | E2 |
| DT-42 | Audit Correlation gap | E2 |
| DT-43 | Namespace Authoring View (storage-scoped) | E2 |
| DT-44 | Headlamp plugin + Keycloak OIDC | E2 |
| DT-45 | Manifest simulation pre-PR (M1) | E1 |
| DT-46 | Historical replay 30 days (M2) | E1 |
| DT-47 | Live shadow mode (M3) | E1 |
| DT-48 | Snapshot simulation pre-upgrade (M4) | E1 |
| DT-49 | Differential simulation across versions (M5) | E1 |
| DT-50 | Namespace-scoped simulation (M6) | E1 |
| DT-51 | Regression test from audit (M8/§17.5) | E1 |
| DT-52 | False positive test (M9) | E1 |
| DT-63 | OPA vs Kyverno decision | E3 |
| DT-64 | Kyverno generate NetworkPolicy | E3 (K8s lib) |
| DT-65/66/67 | CRD lifecycles (ApprovalRequest/SimulationRun/Exception) | E3/E1 |
| DT-68..76 | 9 product libraries (K8s, Keycloak, Jenkins, GitLab, Trivy, OWASP, Sonar, Grafana, Elastic) | E3 |
| DT-77..80 | Reporting (real-time, audit-derived, simulation, coverage-gap) | E1 produces §17E.4; C5 renders |
| HL-08 | Namespace-scoped authoring | E2 + E1 (M6) |
| HL-11 | AI model governance | (cross-domain; E1/E3 replay AI decision points via F4) |
| HL-17 | Differential simulation prevents 2 a.m. rollback | **E1 flagship** + E2 (tagging surface) |

---

## Status

All 3 components: **SPEC.md + PLAN.md + ADVERSARIAL-REVIEW.md present.** E1 (high-value) additionally has **2 ALT architectures**. Robustness rule satisfied (every dir ends with at least SPEC+PLAN). Domain docs: this INDEX, [DOMAIN-SUMMARY](DOMAIN-SUMMARY.md), [DOMAIN-ADVERSARIAL](DOMAIN-ADVERSARIAL.md).
