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

Domain leads and subagents append their own decisions to their docs; material
cross-component decisions get promoted here by the primary during Wave 3.
