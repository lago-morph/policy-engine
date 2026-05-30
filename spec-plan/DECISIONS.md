# Decision Log (unattended)

Every decision made without the user present, with rationale. Append-only.

| ID | Decision | Rationale | Reversible? |
|---|---|---|---|
| D-001 | Exclude `.claude/skills/**` and `retrospective/**` from components to be spec'd | They are tooling/meta (how we work), not the product. Still indexed. | Yes |
| D-002 | 100 scenario files treated as traceability targets, not re-spec'd | They already specify behavior; we link them from TRACEABILITY + component SPECs | Yes |
| D-003 | Three-level agent hierarchy (orchestrator→domain leads→authors) | Max parallelism; matches user's requested structure | Yes |
| D-004 | Keep all intermediate docs, heavily indexed | User instruction; reuse later | Yes |
| D-005 | 23 components in 6 domains as the decomposition unit | Maps cleanly to spec sections; balanced domain sizes for parallel leads | Yes |
| D-006 | ALT trees on A1, B4, C2, D2, E1, F4 + a cross-cut alt sequencing | Highest-leverage / most-contested architecture choices | Yes |
| D-007 | Domain leads must guarantee SPEC.md+PLAN.md per component even if a subagent fails | Robustness against nested-agent flakiness over a long unattended run | Yes |

## Wave 2 cross-cutting decisions (promoted by the primary in Wave 3)

| ID | Decision | Rationale | Reversible? |
|---|---|---|---|
| D-008 | Environment had no Agent/Task tool inside subagents → domain leads authored all three personas (cooperative/adversarial/alt) directly | Robustness rule D-007 fired; quality preserved, hierarchy degraded gracefully to 2 levels | n/a |
| D-009 | Promote **B4 action model** to a foundation contract alongside C2/A1/D1/D2 | The action enum is baked into C2's `decision` field; must resolve before C2 freezes (MASTER-PLAN, XD-3) | Yes |
| D-010 | Rename `replay_completeness` middle state `partial` → **`best_effort`**; keep `jwt_claims_completeness=partial` distinct | Two `partial` meanings were the root cause of consumer confusion (DATA-MODEL R1, contracts OV-1) | Yes |
| D-011 | **Re-open C2 `v1.0` → `v1.0-rc`** before treating it as frozen | CROSSCUT XD-3/XD-1/XD-11: the "frozen" schema baked in the action conflation + self-contradictory external-data capture the domains said to fix first | Yes |
| D-012 | `correlation_id` = retry-stable **logical-flow id** (anchored to `PolicyApprovalRequest` CR name), per-admission UID demoted to `engine_context` | Approval retry mints a new AdmissionReview UID, fragmenting the governance transaction (XD-8, contracts OV-4) — supersedes DATA-MODEL's literal UID rule, to be settled in the C2 rc pass | Yes |
| D-013 | CRD ownership split: **B4** owns the 6 §17C.6 schemas, **F2** owns controllers + 3 new CRDs, **F1** owns REST projection | Resolves the B4/F2/F1 collision (contracts OV-2) | Yes |
| D-014 | Storage `ScopePredicate` MUST be traversed by analytics/reporting aggregate reads (C3/C5/E1), not just CRUD | XD-5: the richest queries are the most likely scope-escape surface | Yes |
| D-015 | Mutual-exclusion enforced on separation-of-duties role pairs at grant time | XD-4: additive/union roles otherwise let one subject be author+approver, defeating D3/D4 | Yes |

These cross-cutting decisions are **provisional** where they touch the C2 rc pass
(D-011/D-012); they are the agenda for the foundation-contract re-freeze, documented
in `cross-cutting/CROSSCUT-ADVERSARIAL.md` §4 (build-blocking subset).
