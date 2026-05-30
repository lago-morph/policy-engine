# Domain A — Governance Core — INDEX

**Domain lead:** Domain A parent agent · **Date:** 2026-05-30 · **Status:** Wave-1 complete (SPEC+PLAN+ADVERSARIAL for both components; ALT for A1).

Domain A is the **root of the platform**: it defines governance intent (A1) and the safe path from intent to
running enforcement (A2). Every other component traces into Domain A via the `control_id`.

---

## Components

| ID | Component | Spec § | Purpose (one line) | Files | Status |
|---|---|---|---|---|---|
| **A1** | Governance Model & Gemara hierarchy | §6 (+§5.3, §7.1, §8.3, §13.3) | System of record for governance intent: Objective→Domain→Control→Enforcement/Evaluation/Evidence/Exception requirements, framework cross-refs, lineage, `control_id` authority | [SPEC](../../components/A1-governance-model/SPEC.md) · [PLAN](../../components/A1-governance-model/PLAN.md) · [ADVERSARIAL](../../components/A1-governance-model/ADVERSARIAL-REVIEW.md) · [ALT](../../components/A1-governance-model/ALT-event-sourced-lineage-log.md) | DRAFT v1 |
| **A2** | Policy Lifecycle (author→simulate→promote) | §7 (+§9.2, §17, §17B, §26.1) | Gated, audited, reversible promotion state machine (draft→dry-run→warn→enforce + rollback); §26.1 generation; A1↔A2 reconciler | [SPEC](../../components/A2-policy-lifecycle/SPEC.md) · [PLAN](../../components/A2-policy-lifecycle/PLAN.md) · [ADVERSARIAL](../../components/A2-policy-lifecycle/ADVERSARIAL-REVIEW.md) | DRAFT v1 |

A1 is the designated **high-value** component and carries an alternative-architecture tree
(`ALT-event-sourced-lineage-log.md`).

---

## Spec-section coverage

| Spec § | Title | Owned by |
|---|---|---|
| §6 / §6.1 | Governance Model & Hierarchy | A1 (primary) |
| §7 / §7.1 | Policy Authoring & metadata | A1 (authoring fields) + A2 (lifecycle) |
| §7.2 | Enforcement Classes | A1 (declares) + A2 (gate selection) |
| §5.3 | Policy Lifecycle Diagram | A2 (the state machine) + A1 (origin) |
| §26.1 | Gemara-to-Rego generation guidance | A2 (GenerationDecision) |

(Adjacent sections §8.3, §9.2, §13.3, §17, §17B, §17C.6, §23 are *consumed* by Domain A but *owned* by Domains B/C/D/E.)

---

## Scenario cross-reference (HL / DT)

| Scenario | Title | A1 | A2 |
|---|---|:--:|:--:|
| DT-01 | Author Gemara objective → controls | ● | ◐ |
| DT-02 | Map SOC 2 CC6.1 to a control | ● | · |
| DT-03 | Exception requirement on a control | ● | ◐ |
| DT-04 | Deprecate a Gemara control | ● | ● |
| DT-05 | Promote dry-run → warn → enforce | · | ● |
| DT-06 | Roll back a constraint promotion | · | ● |
| DT-07 | Build-time-only (Conftest) policy | ◐ | ● |
| DT-08 | Detective-only (audit-derived) policy | ◐ | ● |
| DT-09 | Rego template when not generatable | ◐ | ● |
| HL-01 | Quarterly SOC 2 evidence cycle | ● | · |
| HL-02 | Image-signing rollout end-to-end | ◐ | ● |
| HL-07 | HIPAA framework adoption | ● | ● |
| HL-17 | Differential simulation prevents rollback | · | ● |

● primary · ◐ secondary · · indirect

---

## Personas (per `analysis/persona-spec-mapping.md`)

- **Priya (GRC Lead)** — primary user of A1 (§6 authoring, framework mapping, deprecation).
- **Marcus (Platform Security Eng)** — primary user of A2 (§7 lifecycle, generation, promotion/rollback).
- **Daniel (Auditor)** — consumer of A1 lineage/export (G1 traceability) and A2 promotion audit trail (§23).
- **Sam / Jess** — secondary (Sam: Build-Time DT-07; Jess: rollback DT-06).

---

## Reading order

1. `A1/SPEC.md` (entity model + `control_id` authority + lifecycle) — everything else references it.
2. `A2/SPEC.md` (mode state machine + gates + reconciler).
3. `A1/ALT-event-sourced-lineage-log.md` (the tamper-evidence alternative).
4. The two `ADVERSARIAL-REVIEW.md` then `DOMAIN-ADVERSARIAL.md` (reconciled findings).
5. `DOMAIN-SUMMARY.md` (shared model, hardest decisions, open questions).
