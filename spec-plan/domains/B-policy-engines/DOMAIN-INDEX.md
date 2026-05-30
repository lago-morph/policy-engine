# Domain B — Policy Engines & Enforcement — INDEX

**Domain lead deliverable.** Date: 2026-05-30. Working dir: `spec-plan/components/`.

Domain B is the **enforcement spine** of the platform: it turns governance controls (Domain A /
Gemara) into actual allow/deny/effect decisions at runtime and build-time, across multiple engines,
with full audit traceability. The unifying thesis (§17C, D-B4-01): **OPA/Rego is the cross-product
decision brain; Gatekeeper / Kyverno / Conftest / CRD-controllers are interchangeable effectors
selected by a rubric.**

---

## Components

| ID | Title | Spec § | Files | Status | Exercised by (scenarios) |
|---|---|---|---|---|---|
| **B1** | OPA/Rego integration & signed bundles | §8 (+§7.2, §17C.3, §18) | [SPEC](../../components/B1-opa-rego/SPEC.md) · [PLAN](../../components/B1-opa-rego/PLAN.md) · [ADV](../../components/B1-opa-rego/ADVERSARIAL-REVIEW.md) | DRAFT v1 | DT-10/11/12/13/25/27, DT-06, HL-02/12/05 |
| **B2** | Gatekeeper integration | §9 (+§17B.4, §17C.6, §18, §19) | [SPEC](../../components/B2-gatekeeper/SPEC.md) · [PLAN](../../components/B2-gatekeeper/PLAN.md) · [ADV](../../components/B2-gatekeeper/ADVERSARIAL-REVIEW.md) | DRAFT v1 | DT-14/15/16/17/30/58/59, HL-02/03/06/09/10 |
| **B3** | Conftest integration | §10 (+§7.2, §17C.4, §20.1) | [SPEC](../../components/B3-conftest/SPEC.md) · [PLAN](../../components/B3-conftest/PLAN.md) · [ADV](../../components/B3-conftest/ADVERSARIAL-REVIEW.md) | DRAFT v1 | DT-07/18/19/20/21/45/13, HL-02/04 |
| **B4** | Engine selection, action taxonomy & CRDs | §17C (+§17B, §9, §8) | [SPEC](../../components/B4-engine-selection-crds/SPEC.md) · [PLAN](../../components/B4-engine-selection-crds/PLAN.md) · [ADV](../../components/B4-engine-selection-crds/ADVERSARIAL-REVIEW.md) · [ALT-ocp](../../components/B4-engine-selection-crds/ALT-ocp-substrate.md) · [ALT-kyverno](../../components/B4-engine-selection-crds/ALT-kyverno-first.md) | DRAFT v1 | DT-58/59/03/49/25/09, HL-10/14/19 |
| **B5** | Real-time enforcement flow | §18 (+§8, §9, §15, §17B/C, §11, §14, §19) | [SPEC](../../components/B5-realtime-enforcement/SPEC.md) · [PLAN](../../components/B5-realtime-enforcement/PLAN.md) · [ADV](../../components/B5-realtime-enforcement/ADVERSARIAL-REVIEW.md) | DRAFT v1 | DT-13/28/41/42/30/58/59, HL-02/03/06/16 |

**B4 is the high-value component** and carries two ALT-architecture trees:
- **ALT-ocp-substrate** — build the lifecycle/distribution/regression substrate on OPA Control Plane (OCP) vs. a parallel custom engine.
- **ALT-kyverno-first** — Kyverno-first (YAML-native) vs. OPA/Gatekeeper-first for the Kubernetes layer.

---

## Spec-section coverage map

| Spec § | Covered by | Notes |
|---|---|---|
| §7.2 Enforcement classes | B1, B2, B3 | runtime (B2), build-time (B3), detective (B2 audit + C4) |
| §8 OPA integration | **B1** | responsibilities, packaging, Rego metadata, signing |
| §9 Gatekeeper | **B2** | modes, 17 audit fields, constraint schemas |
| §10 Conftest | **B3** | inputs, CI/pre-commit, normalized evidence |
| §17B Approval-gated decisions | B2 (admission), B4 (CRD/controller) | the §17B.4 hard constraint resolved here |
| §17C Actions/engine gaps/CRDs | **B4** | rubric, 13-action taxonomy, PDP typology, 6 CRDs |
| §18 Real-time enforcement flow | **B5** | full sequence, correlation_id, latency budget |
| §19 (input side) | B2 (detective), B5 (expected-decision-set), → C4 | absence-of-evidence detection |

---

## Cross-domain references (out of Domain B)

- **→ A1 (Gemara):** control catalog; `control_id` resolution (B1-R2, B2-R2, B4 controlId).
- **→ A2 (Lifecycle):** dryrun→warn→deny promotion gates; rollback; consumes B1 RegressionTest diffs.
- **→ C1 (Privateer):** evaluation evidence keyed by correlation_id (B5).
- **→ C2 (Audit schema):** normalizes B1 decision logs + B2 audit events + B3 evidence; owns replay schema.
- **→ C3 (Analytics):** correlates events; flags exception/approval over-use.
- **→ C4 (Retrospective/§19):** consumes B2 detective events + B5 expected-decision-set for bypass detection.
- **→ D1 (Keycloak/JWT):** subject/groups/tenant mapping for decisions + audit fields 7/8.
- **→ D2 (Scoped roles):** RBAC on CRD writes; namespace-scoped authoring.
- **→ D4 (Security):** signing, redaction, key management, approver security.
- **→ E1 (Simulation):** replays the exact runtime decision; driven by PolicySimulationRun.
- **→ E3/§17D (PDP libraries):** consume B4 PolicyActionLibrary/PolicyEvidenceSchema.
- **→ F4 (AI governance):** will want new actions (B4 closed-enum tension, B4-AR-6).

---

## The shared contract surface (what every B component agrees on)

1. **Canonical `decision` object** (B1 §5) — one entrypoint per package; `action` from the 13-taxonomy.
2. **13-action taxonomy** (B4 §4) — closed enum; B1/B2/B3/C2 all consume it.
3. **correlation_id** (B5-R1) — one id minted server-side, threaded through every artifact.
4. **Input-normalization contract** (B3-R10, admission-envelope canonical) — so one package runs in
   Gatekeeper, Conftest, and app PDPs. *(Cross-cut, flagged unresolved — see DOMAIN-ADVERSARIAL.)*
5. **Signed-bundle provenance** (B1 §6) — cosign + in-toto; verify-before-activate; bundle_revision in every log.
6. **deny-with-approval-required + CRD** (B2-R17, B4-R6, B5-R5) — the §17B.4 hard constraint resolution.
