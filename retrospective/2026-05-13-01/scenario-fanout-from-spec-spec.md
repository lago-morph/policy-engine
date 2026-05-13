# Spec: `scenario-fanout-from-spec`

## Intent

Given a product specification and a personas document, produce N testable "after" scenarios — short, structured
markdown files with flowcharts — describing how the product changes specific business flows. The scenarios
become the bridge between an abstract spec and concrete validation of an implementation. This skill exists
because parallelized scenario authoring without a shared contract devolves into generic prose ("the platform
shows a report"); a disciplined fan-out with a pre-committed index and a verbatim file skeleton produces
specific, groundable scenarios at scale.

## Trigger

**Direct user phrases.** "Generate N scenarios from this spec", "write testable scenarios for [product]",
"map personas to flows and produce flowcharts", "I want both high-level and detailed scenarios for [X]".

**Proactive triggers.** When the user has just produced a spec + personas pair and is asking how to validate
the resulting product. When acceptance criteria are being defined for a feature set ≥ ~10 capabilities.

**Negative triggers.** Don't invoke for single-flow walkthroughs (just write it). Don't invoke when the spec
hasn't stabilized — premature scenarios churn. Don't invoke when the user wants prose narrative rather than
discrete files.

## Inputs

- Path to a product specification (markdown).
- Path to a personas document (markdown) describing 3–10 distinct personas with their daily work and pain
  points.
- Optional: target counts of high-level vs detailed scenarios. Default 20 / 80.
- Optional: target output directory. Default `analysis/scenarios/`.

## Outputs

- `analysis/persona-spec-mapping.md` — heatmap and per-persona deep dive (the `persona-spec-heatmap` skill
  handles this if separated; bundled here for one-shot use).
- `analysis/scenarios-index.md` — master index listing every scenario ID, title, persona list, spec section
  list. Committed **before** any subagent runs.
- `analysis/scenarios/high-level/HL-NN-<slug>.md` × N_high (typically 20).
- `analysis/scenarios/detailed/DT-NN-<slug>.md` × N_detail (typically 80).
- One commit per major milestone (mapping doc, index, each fan-out wave).

## Workflow

1. Read the spec and personas in full. Do not delegate this read.
2. Write `analysis/persona-spec-mapping.md` with a persona × spec-section heatmap, per-persona deep dives,
   and a cross-persona handoff diagram. Commit.
3. Plan all N scenarios. For each scenario, fix: ID (`HL-NN` or `DT-NN`), title, slug filename, primary
   personas, primary spec sections, and a 1-line intent. Group detailed scenarios by spec section so a
   subagent's 5 scenarios share context.
4. Write `analysis/scenarios-index.md` containing the full plan as a table per group. Commit.
5. Dispatch one subagent per group of ~5 scenarios. Each brief contains:
   - The 4 file paths above (spec, personas, mapping, index) for context.
   - A 5-line slice of the index listing this subagent's 5 scenarios with title + personas + sections +
     1-line intent for each.
   - The verbatim file skeleton (header block + Steps + Success criteria + Flowchart + Notes).
   - A list of real spec artifacts to cite (control IDs, CRD names, claim names, field names, view names).
   - Constraints: ≤1 page; do not commit; do not run `git`; report file paths and word counts.
   - Dispatch in waves of ~10 in parallel.
6. After each wave, verify with one bash pass:
   - `ls <dir> | wc -l` matches expected count.
   - `grep -L '```mermaid' <dir>/*.md` is empty.
   - `git status` shows the expected new files.
7. Commit each wave's output.
8. After the final wave, run the verification pass once more across both directories.

## Concrete examples

### Example 1 — the original session

Input: `openssf_opa_unified_governance_platform_spec v1.md` (~57KB), `policy engine personas.md` (5 personas
spanning intent → authoring → operation → consumption → assurance), target 20 high-level + 80 detailed.

Plan: 4 high-level subagents (5 scenarios each: HL-01..HL-20) + 16 detailed subagents (5 scenarios each:
DT-01..DT-80), grouped by spec section: §6 Governance Model, §7 Lifecycle, §8 OPA, §9 Gatekeeper, §10
Conftest, §11 Privateer, §12-13 Audit Schema, §14 Analytics, §15 Keycloak, §16 GUI, §17 Simulation, §17A
Roles, §17B Approval, §17C Engines/CRDs, §17D Product Libraries, §17E Reporting.

Each brief inlined the real spec artifacts to use: control IDs like `SC-IMG-001`, CRDs like
`PolicyApprovalRequest`, claim names like `tenant`, fields like `replay_completeness` and `correlation_id`,
views like `Audit Correlation View`, modes like `dry-run → warn → enforce`.

Result: 100 scenarios, 100 Mermaid flowcharts, two waves of 10 parallel subagents, three commits, zero
collisions.

### Example 2 — a smaller fan-out

Input: a 12-page payments-API spec + 3 personas (developer, fraud analyst, treasury ops), target 5
high-level + 15 detailed.

Plan: 1 high-level subagent (HL-01..HL-05) + 3 detailed subagents (DT-01..DT-05, DT-06..DT-10,
DT-11..DT-15), one wave of 4 in parallel. Each subagent brief inlines real endpoint paths
(`POST /v1/transfers`), webhook event names (`transfer.completed`), and idempotency-key semantics.

Result: 20 scenario files, one commit, one verification pass.

## Anti-patterns

- **Skipping the index.** Without `scenarios-index.md` committed first, each subagent brief becomes bespoke
  and the user has no entry point. Always pre-commit the index.
- **Reading subagent outputs into your own context.** Verify with bash (`wc -l`, `grep -L`), not with `Read`.
  100 file Reads is ~400KB of context burned to no end.
- **Generic briefs.** "Write scenarios about the audit subsystem" produces generic prose. "Write scenarios
  citing `correlation_id`, `Audit Correlation View`, `bundle:v12`, `SC-IMG-001`" produces specific prose.
- **Worktree isolation when filenames are unique.** Spawn subagents in the shared tree if they only write
  new files and never call `git` — saves a manual merge step.
- **Re-rolling subagent outputs.** If an agent reports the file was "pre-existing" but its structure is
  correct, trust it; concurrent waves can race. Spot-check, don't re-author.

## Acceptance criteria

1. The persona-spec mapping is committed and links control IDs to personas concretely (not via vague
   "consumer of evidence").
2. The scenarios index lists every scenario with ID, title, personas, and spec sections — and is committed
   before any subagent dispatch.
3. Every scenario file matches the 5-section skeleton (header block + Steps + Success criteria + Flowchart
   + Notes) and contains a valid `mermaid` flowchart.
4. Verification pass (`ls | wc -l` + `grep -L mermaid`) confirms 100% completeness without reading individual
   files.
5. Three or fewer commits land the entire deliverable.

## Files this skill creates / modifies

- `analysis/persona-spec-mapping.md` — heatmap and persona deep dives.
- `analysis/scenarios-index.md` — pre-committed master plan.
- `analysis/scenarios/high-level/HL-NN-*.md` — high-level scenarios.
- `analysis/scenarios/detailed/DT-NN-*.md` — detailed scenarios.
- No code is touched; this is a documentation skill.
