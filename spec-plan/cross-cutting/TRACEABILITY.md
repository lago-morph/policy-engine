# Traceability Matrix — Component ↔ Spec § ↔ Personas ↔ Scenarios

Seeded by the primary orchestrator from `analysis/scenarios-index.md` and
`analysis/persona-spec-mapping.md` while the domain leads run. Cross-cut agents
and the master-index pass may refine it. Personas: **P**riya (GRC), **M**arcus
(Platform Sec), **J**ess (SRE), **Sa**m (Dev), **D**aniel (Auditor).

| Component | Spec § | Primary personas | High-level scenarios | Detailed scenarios |
|---|---|---|---|---|
| A1 Governance Model & Gemara hierarchy | §6 | P, D | HL-01, HL-07 | DT-01, DT-02, DT-03, DT-04 |
| A2 Policy Lifecycle | §7 | M, Sa | HL-02, HL-04, HL-07 | DT-05, DT-06, DT-07, DT-08, DT-09 |
| B1 OPA/Rego & signed bundles | §8 | M | HL-02 | DT-10, DT-11, DT-12, DT-13 |
| B2 Gatekeeper | §9 | M, J | HL-02, HL-03, HL-09 | DT-13, DT-14, DT-15, DT-16, DT-17 |
| B3 Conftest | §10 | Sa, M | HL-02, HL-04 | DT-18, DT-19, DT-20, DT-21 |
| B4 Engine selection, actions & CRDs | §17C | M | HL-10, HL-14, HL-19 | DT-63, DT-64, DT-65, DT-66, DT-67 |
| B5 Real-time enforcement flow | §18 | M, J | HL-03 | DT-41 |
| C1 Privateer | §11 | P, D | HL-01, HL-05 | DT-22, DT-23, DT-24 |
| C2 Audit schema & event schema | §12–13 | M, J, D | HL-03, HL-18 | DT-25, DT-26, DT-27, DT-28, DT-29 |
| C3 Compliance analytics | §14 | P, J | HL-06, HL-09, HL-12, HL-13, HL-15, HL-20 | DT-30, DT-31, DT-32, DT-33, DT-34 |
| C4 Retrospective audit detection | §19 | P, J, D | HL-01, HL-05, HL-06, HL-12 | DT-30, DT-78 |
| C5 Reporting | §17E | P, D | HL-01, HL-05, HL-07, HL-10, HL-12, HL-15, HL-17, HL-20 | DT-77, DT-78, DT-79, DT-80 |
| D1 Keycloak/JWT & mapping layer | §15 | M | HL-13, HL-16 | DT-35, DT-36, DT-37, DT-38 |
| D2 Scoped RBAC & storage authz | §17A | P, M, Sa, D | HL-04, HL-05, HL-08, HL-18 | DT-53, DT-54, DT-55, DT-56, DT-57 |
| D3 Approval-gated decisions | §17B | Sa, M, P | HL-04, HL-10, HL-15, HL-19 | DT-58, DT-59, DT-60, DT-61, DT-62 |
| D4 Security requirements | §23 | M, D | HL-05, HL-18 | (cross-cuts; integrity props in DT-24, DT-57) |
| E1 Simulation & dry-run | §17 | M, Sa, D | HL-02, HL-05, HL-11, HL-12, HL-17, HL-18 | DT-45..DT-52 |
| E2 Governance console / Headlamp | §16 | all | HL-03, HL-08, HL-20 | DT-39, DT-40, DT-41, DT-42, DT-43, DT-44 |
| E3 Per-product PDP libraries | §17D | M, Sa | HL-10, HL-14 | DT-68..DT-76 |
| F1 API requirements | §21 | M | (cross-cuts all) | (cross-cuts) |
| F2 Deployment & extensibility | §24–25 | M, J | HL-14 | (cross-cuts) |
| F3 POC scale, MVP & sequencing | §22, §26, §27 | M, P | (scoping) | — |
| F4 AI / agent governance extension | reframed-for-ai | P, M, Sa | HL-11 | (forward-looking) |

## Persona coverage check
- **Priya (GRC):** A1, C1, C3, C4, C5, D2, E1 — intent→evidence path. ✓
- **Marcus (Platform Sec):** A2, B1–B5, C2, D1, D3, E1–E3, F1–F4 — broadest footprint. ✓
- **Jess (SRE):** B2, B5, C2, C3, C4, E2 — operate/triage path. ✓
- **Sam (Dev):** A2, B3, D2, D3, E1, E3 — consume/exception path. ✓
- **Daniel (Auditor):** A1, C1, C2, C4, C5, D2, D4, E1 — assurance/replay path. ✓

## Notes
- Every HL/DT scenario maps to at least one component; the master-index pass
  will verify no scenario is orphaned and no component lacks a scenario.
- F1/F4 are deliberately cross-cutting; their scenario coverage is indirect.
