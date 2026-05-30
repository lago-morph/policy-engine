# Spec: `adversarial-corpus-review`

## Intent

A design corpus authored cooperatively — even by careful agents — tends to be internally
agreeable and externally untested: every author wants their piece to work, so weaknesses,
unowned gaps, and "should this even exist?" questions go unasked. This session showed that
the **highest-value findings came from deliberately adversarial layers stacked on top of
the cooperative authoring**, not from the authoring itself. Per-component adversarial
reviews caught correctness bugs; a *meta-adversarial* pass (an agent reviewing the
synthesis it did not write) caught that the "frozen" keystone schema was internally
contradictory; a *thesis devil's-advocate* (a kill-the-deal VC/CTO) caught that three
reviewers + the market data favored a wedge-first path the default plan ignored; and an
*NFR devil's-advocate* caught that the corpus paired wedge-first sequencing with
platform-first NFRs. This skill operationalizes that persona stack so scrutiny is a
designed-in layer with named roles, not an afterthought.

## Trigger

- **Direct**: "red-team this corpus", "adversarial review", "what did everyone miss?",
  "is this over-engineered?", "should we even build this?", "stress-test the synthesis".
- **Proactive**: after any large cooperatively-authored spec/plan corpus (≥~10 docs), and
  especially after a reconciliation gate — review the *reconciliation* itself, since it had
  the least independent scrutiny.
- **Negative**: skip for a single small doc, or where a cooperative review already covers
  the need (one bug-focused pass on a 200-line change does not need five personas).

## Inputs

- The corpus to review (components, cross-cutting docs, the index/handoff).
- The source/market context needed for the thesis review (e.g. competitive landscape docs).
- Which layers to run (per-component / meta / thesis / NFR-pragmatic) — default: all four.

## Outputs

- Per-piece `ADVERSARIAL-REVIEW.md` (prioritized defect list, correctness vs hardening).
- A **meta-adversarial** doc reviewing the synthesis + the process itself (groupthink,
  unchallenged assumptions, missing layers, internal contradictions across synthesis docs).
- A **thesis devil's-advocate** doc (strongest case against existence / build-vs-buy /
  narrow-to-wedge, ending in a decisive go/no-go test).
- A **pragmatic cut-line** doc (what's genuinely required for the next milestone vs
  gold-plating; one fund-first pick).

## Workflow

1. **Per-piece (cooperative author + adversarial reviewer, distinct personas).** For each
   component, one persona authors; a *hostile* persona ("guilty until proven innocent")
   writes the defect list. On high-value pieces add an alt-architecture persona.
2. **Meta-adversarial.** Dispatch a *fresh* agent that did **not** author the corpus, framed
   as an external review board. Mandate: attack the synthesis hardest (it had the least
   review); find groupthink (claims every author accepted without proof), internal
   contradictions *across* synthesis docs, and whole missing layers (NFRs, ops, human
   factors). This pass found the cross-doc "is it frozen?" contradiction.
3. **Thesis devil's-advocate.** A kill-the-deal persona (skeptical VC + build-vs-buy CTO).
   For each "novel" claim, argue an incumbent absorbs it or customers won't pay; per major
   component, argue buy/adopt over build; end with the single decisive question that would
   change the verdict.
4. **Pragmatic cut-line.** A POC/first-customer engineering director: for each piece, the
   one-line "minimum genuinely required now" vs "defer until a customer pays"; flag the
   *inverted* item (under-specified but more urgent than the corpus implies); one fund-first pick.
5. Feed all four layers into the reconciliation gate / index so findings propagate into the
   owning docs (avoid the "fix lives in one doc, owning docs unchanged" failure).

## Concrete examples

**Example 1 — meta-adversarial catching the synthesis contradiction.** A fresh agent given
`00-MASTER-INDEX.md` + the six cross-cutting docs (but told it authored none of them) found
that `CROSSCUT-ADVERSARIAL` (un-freeze C2), `MASTER-PLAN` (FROZEN), and
`INTER-DOMAIN-CONTRACTS §5.5` ("no new field") were mutually inconsistent — the single most
important finding of the run — plus a missing-NFR layer (scale/DR/keys/tenancy/human-factors)
that no component owned, which became Domain G.

**Example 2 — thesis devil's-advocate forcing the strategic question.** Given the market
research + both build plans, the persona returned "narrow to Wedge 5 (OPA Control Plane
successor) or kill it; adopt ~20 components, build ~2," and the decisive test: *name three
design partners who'll sign a 90-day paid pilot for one named wedge, and the exact sentence
each says about why OPA Control Plane / Wiz / their Vanta+Kyverno stack can't do it today.*
That single test reframed the whole roadmap discussion.

## Anti-patterns

- **Letting the author also be the adversary.** Cooperative authors rationalize; the
  hostile review must be a distinct persona (ideally a distinct agent).
- **Only reviewing components, never the synthesis.** The synthesis is where contradictions
  hide *because* it had the least independent review — review it hardest.
- **Adversarial findings that stay in their own doc.** Propagate into owning docs via the
  reconciliation gate, or they are decorative (the meta-review's own M-1 warning).
- **Polite adversaries.** "Looks good, minor nits" is a failed review. Demand a ranked
  defect list and a blunt verdict.
- **Skipping the pragmatic cut-line.** Without it, an adversarial wave produces a
  gold-plated corpus nobody can afford to build for the actual next milestone.

## Acceptance criteria

1. Every component has a prioritized defect list with correctness-vs-hardening tags.
2. At least one fresh-context agent reviewed the synthesis it did not author.
3. The thesis review ends in a decisive, falsifiable go/no-go test.
4. The pragmatic pass yields a concrete cut-line and one fund-first pick.
5. Findings are propagated into owning docs / the index, not stranded.

## Files this skill creates / modifies

- `components/<id>/ADVERSARIAL-REVIEW.md` — per-piece defect lists.
- `cross-cutting/META-ADVERSARIAL-SECOND-OPINION.md` — synthesis + process review.
- `cross-cutting/THESIS-DEVILS-ADVOCATE.md` — existence / build-vs-buy review.
- `cross-cutting/<X>-DEVILS-ADVOCATE.md` — pragmatic cut-line for a sub-area (e.g. NFR).
- The master index / handoff — folds the verdicts into the headline findings.
