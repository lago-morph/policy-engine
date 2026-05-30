# AGENTS.md suggestions — 2026-05-30-5

Proposed additions to the project's agents file (`AGENTS.md` at the repo root). Each
section has the exact text to paste and the argument for it, grounded in this session.
Decide each on its own merits.

---

## Suggestion 1: Subagents may not be able to spawn subagents

### Proposed addition

> **Don't assume subagents can fan out.** Before designing a 3-level agent hierarchy
> (orchestrator → leads → authors), assume subagents may have **no Agent/Task tool** in
> this environment. Always pre-commit a graceful-degradation rule in the orchestration plan:
> "if a subagent can't be spawned or fails, the lead authors the doc itself, writing files
> incrementally." Verify tool availability rather than assuming the planned topology.

### Why this earns its place in your agents file

This session planned a three-level hierarchy and discovered at Wave 1 that subagents had no
Agent tool — the entire fan-out had to collapse to two levels. The only reason this was a
shrug rather than a re-plan was a robustness rule committed *before* dispatch (D-007). The
marginal cost is one sentence in the plan; the cost of omitting it is a stalled unattended
run with half-empty output directories and no fallback author.

---

## Suggestion 2: Run a serial reconciliation gate after any parallel synthesis

### Proposed addition

> **Reconcile after fan-out.** Whenever ≥3 agents author, in parallel, documents that cite a
> shared contract or each other, run a dedicated **serial** reconciliation pass before
> declaring the work done. Diff every doc's statement of each shared contract; where they
> disagree, produce ONE canonical artifact and mark the rest superseded. Never run the
> reconciliation itself as another parallel fan-out.

### Why this earns its place in your agents file

The exact failure this prevents happened twice in one session. Five Wave-2 agents dispatched
to *reconcile* the corpus instead disagreed with each other about whether the keystone
schema was "frozen"; the same drift then reproduced one level down in the NFR wave. Each
time, a build team could not have known which doc to trust. The fix — a single
canonical artifact — costs one extra serial pass; the contradiction, left in, ships to
whoever builds from the corpus.

---

## Suggestion 3: A "frozen" contract is provisional until the reconciliation gate clears it

### Proposed addition

> **No contract is "frozen" mid-fan-out.** A schema/interface/contract declared frozen while
> parallel work is still in flight is **provisional** until a reconciliation gate confirms no
> consumer has an open change request against it. Mark such contracts `-rc` / provisional in
> the index until cleared.

### Why this earns its place in your agents file

C2's audit schema was declared `v1.0` "frozen" while it still contained the action-model bug
its own consumers flagged, and then eleven components asked it to change. Calling it frozen
created false confidence that three later docs propagated. The cost of the rule is a label;
the benefit is that downstream agents don't build against a contract that's about to move.

---

## Suggestion 4: Cooperative authors and adversarial reviewers must be distinct personas

### Proposed addition

> **Separate the author from the critic.** For any non-trivial spec/design, the cooperative
> author and the adversarial reviewer must be **distinct personas** (ideally distinct
> agents). Additionally, review the *synthesis/reconciliation* docs hardest with a
> fresh-context agent that did not author them — that is where contradictions hide.

### Why this earns its place in your agents file

Every high-value finding this session came from an adversarial layer, not the cooperative
authoring: the per-component reviews caught correctness bugs, and a fresh meta-adversarial
agent caught the cross-doc "is it frozen?" contradiction the five authoring/synthesis agents
all missed. An author reviewing their own work rationalizes; a fresh critic does not.

---

## Suggestion 5: Don't read a background agent's output file via shell

### Proposed addition

> **Never `cat`/`tail` a background agent's `output_file`.** It is the full JSONL transcript
> and will overflow context. Rely on the completion-notification summary, and instruct each
> agent to "report back the top N decisions/risks" so you integrate from the summary, not the
> transcript.

### Why this earns its place in your agents file

The harness explicitly warns that reading the transcript overflows context, and the
"report back top 5" convention was what let the orchestrator integrate 25 agents' work
without ever opening a transcript. One fabricated `cat` of a multi-thousand-line JSONL file
would have blown the orchestrator's context in the middle of an unattended run.

---

## Suggestion 6: Commit + push after every wave in long unattended runs

### Proposed addition

> **Persist continuously when running unattended.** The sandbox filesystem is ephemeral —
> commit and push after every wave / agent-landing, not at the end. WIP commits are fine;
> approximate commit messages are fine. Losing an hour of agent output to a reclaimed
> container is not.

### Why this earns its place in your agents file

This session committed after nearly every one of 25 agent landings; the stop-hook's
git-check nudges reinforced it. Across a 6-wave unattended run producing 142 files, no work
was ever at risk. The cost is a commit per landing; the alternative is re-running expensive
parallel waves after a container reclaim.

---

## Suggestion 7: `git add -A` over concurrent agents gives approximate attribution

### Proposed addition

> **Expect approximate commit attribution under concurrency.** When committing the outputs of
> many concurrent background agents with `git add -A`, you capture whatever is on disk at that
> instant — files from a still-running agent get swept into another wave's commit, and "nothing
> to commit" appears when a prior `add -A` already grabbed them. If precise per-component
> attribution matters, `git add <specific-paths>`; otherwise accept that commit messages are
> approximate and rely on file content, not commit boundaries, as the source of truth.

### Why this earns its place in your agents file

Several times this session a commit reported "nothing to commit" because a previous `add -A`
had already swept a concurrently-written file (Domain C landed inside Domain B's commit;
DATA-MODEL + CROSSCUT-ADVERSARIAL inside the contracts commit). No data was lost, but the
commit log mis-attributes which wave produced which file. Knowing this up front prevents a
confused debugging detour chasing a phantom "missing" file.

---

## Suggestion 8: Don't sleep-poll for background agents

### Proposed addition

> **Dispatch and yield; never sleep-poll.** After launching background agents, end the turn
> and act on completion notifications as they arrive. Do not use `sleep` loops or repeated
> status checks to wait for external/agent events.

### Why this earns its place in your agents file

The whole session ran on the dispatch-then-yield model — 25 agents across 6 waves, each
committed on its completion notification. A `sleep`-based wait would have burned wall-clock
and tokens doing nothing while blocking the orchestrator from handling other landings.

---

## Suggestion 9: Keep a running DECISIONS.md for unattended decisions

### Proposed addition

> **Log every unattended decision.** In any "run unattended, decide when in doubt" task,
> maintain an append-only `DECISIONS.md` with id, decision, rationale, and reversibility.
> Promote cross-cutting decisions into it during finalization so the human reviewer can audit
> and override.

### Why this earns its place in your agents file

This session made 23 logged decisions (D-001..D-023) with nobody watching. Because each was
recorded with rationale and a reversible flag, the final review surface was a single
auditable table rather than an archaeology dig through 27 commits. The cost is a table row
per decision; the benefit is a reviewable audit trail for autonomous work.

---

## Suggestion 10: Index heavily and verify intra-doc links before finalizing

### Proposed addition

> **Index, then link-check.** For multi-file deliverables, maintain a master index + per-group
> index + a `HANDOFF.md`, and run a relative-link sweep (every `](path)` resolves to a file)
> before declaring done. Keep intermediate docs — don't consolidate them away — but make them
> findable.

### Why this earns its place in your agents file

The user's explicit instruction was "keep intermediate docs, indexed." A scripted link sweep
caught dangling pointers to not-yet-written cross-cutting docs before each finalization
commit. For a 142-file corpus, the index + handoff are the only way a reviewer enters the
work at all; a broken link in the index strands them.

---

## Suggestion 11: Ask before burning a big autonomous budget

### Proposed addition

> **Clarify forks before a large autonomous run.** When given a high-ceiling open-ended brief
> ("spare no expense, run unattended"), restate your interpretation and resolve the 2–3
> genuine forks via a question **before** dispatching expensive parallel work — then proceed
> autonomously on everything else.

### Why this earns its place in your agents file

One `AskUserQuestion` at the start returned the load-bearing constraint ("don't consolidate;
orchestrator → indexers → authors; keep intermediate docs") that shaped all 142 files. Had
the run started without it, the entire corpus would have been built toward the wrong
(consolidated) deliverable — hours of parallel work pointed the wrong way. One question up
front versus a full re-run is the asymmetry.
