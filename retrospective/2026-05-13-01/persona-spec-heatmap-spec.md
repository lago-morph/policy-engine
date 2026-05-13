# Spec: `persona-spec-heatmap`

## Intent

Given a product spec and a personas document, produce a single markdown document that maps each persona to
the spec sections where they spend their working time, with a heatmap, per-persona deep dive, and
cross-persona handoff diagram. This artifact exists to bridge the gap between "the spec lists 30 sections"
and "the spec is going to change five different people's daily work in five different ways." It's the
foundation other scenario / acceptance-test / training artifacts build on, and standing alone it's already
useful to product managers and engineering leads.

## Trigger

**Direct user phrases.** "Map personas to the spec", "which sections does each persona care about?", "produce
a persona-spec heatmap", "summarize how this spec affects our users".

**Proactive triggers.** When a spec lands with named personas and the user is about to start scenario
authoring, acceptance-test planning, or stakeholder rollout. When a spec has ≥10 sections and ≥3 personas
the mapping is otherwise tribal knowledge.

**Negative triggers.** Don't invoke when the spec is one page (just write a paragraph). Don't invoke when
the personas are placeholders ("user", "admin") — that's a personas problem to fix first.

## Inputs

- Path to a product specification (markdown).
- Path to a personas document (markdown) describing distinct personas with daily work and pain points.
- Optional: output path (default `analysis/persona-spec-mapping.md`).

## Outputs

- A single markdown document containing:
  - At-a-glance persona table.
  - Persona × spec-section heatmap (●/◐/· legend).
  - Per-persona deep dive: primary surfaces, pain points resolved, before/after summary.
  - Cross-persona handoff diagram (Mermaid).
  - One-paragraph scenario-design implications.

## Workflow

1. Read the spec and personas documents in full. Do not skim.
2. List every distinct persona with one-sentence role focus and lifecycle "moment" they occupy.
3. List every spec section that contains substantive product behavior (not boilerplate sections like
   "Executive Summary" or "Non-Goals").
4. Build the heatmap: for each (persona, section) pair, assess primary (●) vs secondary (◐) vs indirect (·)
   surface based on the persona's daily work and the section's content. Bias toward fewer primary marks —
   if every persona has primary access to every section, the heatmap conveys nothing.
5. Per persona, write a 6-bullet deep dive: primary surfaces (list spec sections), pain points resolved
   (verbatim from personas doc where possible), before/after one-sentence summary.
6. Build the handoff diagram: a Mermaid `flowchart LR` showing the primary work artifacts that pass between
   personas (e.g., "control ID + evidence schema", "bundle + audit schema", "exception request", "signed
   evidence package"). The arrows are the testable contracts.
7. Close with a paragraph naming the scenario classes that fall out of the mapping (end-to-end multi-persona,
   single-section deep, single-feature micro). This is the bridge to any follow-on scenario authoring.

## Concrete examples

### Example 1 — five personas × 30+ spec sections

Input: a unified governance platform spec with 30 numbered sections, 5 personas (compliance lead, platform
security engineer, SRE, app developer, auditor).

Output structure:
- At-a-glance table with role focus + lifecycle moment + which spec roles each maps to.
- Heatmap: 30 rows × 5 persona columns. ~150 cells; typically ~60 primary, ~50 secondary, ~40 indirect.
- Per-persona deep dive: 5 sections, each listing primary surfaces (5–10 §§), pain points (4–6 bullets),
  pre/post summary (1 sentence).
- Handoff diagram: `Priya → Marcus (control_id)`, `Marcus → Jess (bundle + audit schema)`, `Sam → Marcus
  (exception request)`, `Jess → Marcus (audit fixture)`, `Priya → Daniel (signed package)`, `Daniel → Marcus
  (independent replay)`.

The artifact is 200–300 lines of markdown and stands alone as a product-management deliverable.

### Example 2 — three personas × ~15 sections (smaller spec)

Input: a payments-API spec, 3 personas (developer, fraud analyst, treasury ops).

Output: ~100 lines. Per-persona deep dives are tighter (3-4 bullets); the heatmap is 15 × 3 = 45 cells;
the handoff diagram is a single triangle.

## Anti-patterns

- **Heatmap with everything marked primary.** If every persona touches every section primarily, the heatmap
  conveys no information. Be willing to mark sections as `·` (indirect) — most personas use most of any
  spec indirectly at best.
- **Per-persona deep dives written from a template.** Each persona has a different lifecycle moment and a
  different vocabulary of pain. Echo the personas doc's voice; don't homogenize.
- **Skipping the handoff diagram.** The cross-persona arrows are the most testable artifact in the mapping —
  each arrow names a contract (e.g., `control_id`) that must hold between two systems.
- **Trying to be exhaustive on sections.** Skip Executive Summary, Non-Goals, Strategic Value, MVP — they
  don't drive daily work for any persona.

## Acceptance criteria

1. Heatmap covers every substantive spec section and every persona.
2. Each persona's deep dive lists ≥5 specific spec sections by number.
3. The handoff diagram has ≥1 arrow per persona-pair that interacts; each arrow is labeled with the artifact
   that passes.
4. Pre/post summaries cite specific artifacts from the personas doc (timing, ticket counts, page latency).
5. The document stands alone: a reader who hasn't read the spec gets enough context to know what the spec
   does and who it affects.

## Files this skill creates / modifies

- `analysis/persona-spec-mapping.md` — the heatmap + deep dives + handoff diagram. Single file output.
