# A1 — ALTERNATIVE ARCHITECTURE — Event-Sourced Governance Ledger

**Persona:** alt-architecture author. A genuinely different approach to the *same* §6 requirement, with an
honest trade-off analysis vs. the primary `SPEC.md`.
**One-line thesis:** make the **append-only event log the source of truth** (event sourcing + CQRS), with the
relational entity model from the primary SPEC demoted to a *rebuildable read projection*. The governance
hierarchy, lineage, and the audit trail become **the same artifact** rather than three synchronized copies.

---

## 1. Why consider an alternative at all

The primary SPEC stores three things that must agree: (a) the live entity rows, (b) append-only
`ControlRevision` history, and (c) temporal `LineageEdge` records. The adversarial review's DEF-09 (which
revision was active at time X?), DEF-03 (lifecycle drift), and DEF-12 (lineage growth) are all symptoms of
**maintaining three representations of one truth**. The auditor's core question — *"what was the governance
state, exactly, at time X, and prove it wasn't altered"* — is naturally answered by an **immutable, ordered,
hash-chained event log**, not by reconstructing it from mutable tables.

This alternative says: don't store state and derive history; **store history and derive state.**

---

## 2. The architecture

### 2.1 Core idea

A single **append-only, hash-chained Governance Event Ledger** is the authority. Every change is an event:

```
GovernanceEvent {
  seq:            monotonic int (gapless),
  event_id:       ULID,
  prev_hash:      sha256(prev event canonical bytes),   // hash chain (tamper-evident, §23)
  ts:             timestamp,
  actor:          JWT subject + resolved role,
  correlation_id: string,
  type:           ObjectiveCreated | ControlDrafted | ControlSubmitted | ControlActivated |
                  EnforcementRequirementSet | EvidenceRequirementSet | ExceptionRequirementSet |
                  FrameworkRefLinked | CoverageClaimed | ControlDeprecated | ControlRetired |
                  ImplementationLinked | EnforcementPointObserved | DecisionObserved | ...,
  payload:        type-specific (the field deltas),
  schema_version: gemara schema semver
}
```

State (the entities in primary SPEC §2) is a **materialized view** produced by replaying events through a
deterministic reducer. Lineage is **not a separate store** — it is a projection that folds
`ImplementationLinked` / `EnforcementPointObserved` / `DecisionObserved` events into a graph view. The audit
trail *is* the ledger.

### 2.2 CQRS read side

Multiple independently-rebuildable projections, each owned by a consumer need:
- `ControlView` — current entity shape (what primary SPEC's tables hold) for the Governance API.
- `LineageView` — the graph for §16.3 (bounded-depth, materialized).
- `CoverageView` — framework coverage matrix + badges (DT-02).
- `GovernanceProjection` — the data-plane cache for B2/B3/B4 (primary SPEC §9).
- `AsOfView(T)` — *any historical state* by replaying the ledger up to `seq` at time T. This is free in event
  sourcing and is exactly DEF-09's "active at X" + the auditor's as-of query.

Projections are disposable: corrupt one, delete and replay. The ledger is the only thing that must be durable.

### 2.3 How the same requirements are met

| §6 / scenario need | Event-sourced realization |
|---|---|
| 7-layer hierarchy (§6.1) | reducer builds the Objective→Domain→Control→4-requirements tree from `*Set`/`*Created` events |
| Append-only revisions | **native** — the ledger *is* the revision history; "revision N" = ledger replayed to event N |
| Temporal lineage (D6) | `ImplementationLinked`/`...Observed` events; LineageView folds them; `valid_to` = the closing event's seq |
| Deterministic signed export (A1-MUST-032) | export = `{ledger segment [seq_a, seq_b], head_hash}`; the **hash chain is the signature substrate** — verification is recomputing the chain |
| As-of-date audit query (DEF-09) | replay to `seq@T`; multi-interval "active" falls out naturally from the event order |
| Deprecation/retire (DT-04) | `ControlDeprecated`/`ControlRetired` events; retire reducer checks no open enforcement-point events |
| Coverage (DT-02) | `FrameworkRefLinked`/`CoverageClaimed` events fold into CoverageView |
| Tamper-evidence (§23, Daniel) | the hash chain makes any retro-edit detectable: re-deriving `prev_hash` fails |

---

## 3. Where this is *better* than the primary SPEC

1. **Tamper-evidence is intrinsic, not bolted on.** Primary SPEC adds a "revision hash chain" (A1-MUST-041)
   as a feature; here it's the substrate. Daniel's "prove governance wasn't altered" is answered by chain
   verification, no separate mechanism.
2. **As-of-date queries are free and provably correct** (kills DEF-09). "What did SC-IMG-001 look like on
   2025-11-02?" = replay to that seq. No "which interval was active" ambiguity.
3. **The A1/A2 lifecycle seam (DEF-03) becomes observable.** A2's mode promotions and A1's status changes are
   *both events on the same ledger* (or two ledgers with cross-correlation_id). A reconciler is just another
   projection that flags `ControlActivated` with no subsequent `EnforcementModeEnforced` within N days —
   "governed but never enforced" becomes a query, not a missing feature.
4. **DEF-01 (declared vs emitted) shrinks.** `DecisionObserved`/`EnforcementPointObserved` events let the
   ledger *fold actual emissions* against the EvidenceRequirement: a projection can compute "control X declares
   `correlation_id` but 12% of observed decisions lacked it" directly.
5. **Projections are cheap and many.** The data-plane cache (B-layer), the console graph, and the coverage
   matrix are independent rebuildable views — no contention on a shared mutable table.
6. **Audit and governance unify.** The platform already needs a replay-capable audit log (§13). This makes the
   *governance* changes replay-capable in the same paradigm — conceptual economy.

---

## 4. Where this is *worse* / the costs

1. **Complexity & team familiarity.** Event sourcing + CQRS is a known footgun: eventual consistency between
   write (ledger) and read (projections) confuses developers who expect read-after-write. The primary SPEC's
   plain CRUD-with-revisions is *boring and buildable* — a decisive advantage for a POC (§26.1 says "remain
   suitable for a small POC").
2. **Read-after-write latency.** "Create control, immediately GET it" needs either synchronous projection
   update or a read-your-writes hack. CRUD gives this for free.
3. **Schema/reducer evolution is hard.** Changing the meaning of an old event type requires versioned reducers
   forever (upcasting). The Gemara schema bump problem (DEF-05) becomes a *reducer-versioning* problem — moved,
   not eliminated, and arguably harder to reason about.
4. **Ledger growth & per-decision events.** If `DecisionObserved` events live on the governance ledger, it
   stops being POC-sized (the exact DEF-12 concern). Mitigation: keep `DecisionObserved` in C2's audit store
   and only *reference* it from the governance ledger — but then we're back to two stores and the "single
   artifact" elegance erodes.
5. **Tooling/GitOps friction.** The primary SPEC's Gemara-YAML-as-source (D8) is reviewable in PRs — a human
   reads a control diff. An event ledger is not human-diffable; you'd still need to project to YAML for review,
   so you build the projection *anyway*. The ledger doesn't replace the GitOps story; it sits beneath it.
6. **Query ergonomics.** Ad-hoc "find all controls in domain X with severity high and no enforcement" is a
   trivial SQL `WHERE` in the primary SPEC; here it requires a purpose-built projection or replaying into one.

---

## 5. Honest trade-off table

| Dimension | Primary (CRUD + revisions + temporal edges) | Alt (event-sourced ledger + CQRS) |
|---|---|---|
| Buildability / POC fit (§26.1) | **High** (boring, well-understood) | Medium (powerful but easy to misbuild) |
| Tamper-evidence (§23, Daniel) | Bolt-on hash chain | **Intrinsic** |
| As-of-date correctness (DEF-09) | Needs explicit interval modeling | **Free & provable** |
| Lifecycle-drift detection (DEF-03) | Needs a bespoke reconciler | **A projection/query** |
| Declared-vs-emitted (DEF-01) | Separate conformance loop | **Foldable on the ledger** |
| Read-after-write ergonomics | **Trivial** | Needs care (eventual consistency) |
| Schema evolution | YAML semver, N/N-1 | Reducer upcasting (harder) |
| GitOps human review | **Native (YAML diffs)** | Needs a YAML projection anyway |
| Operational risk for a small team | **Low** | Higher |

---

## 6. Recommendation (a real decision, not a hedge)

**Adopt a hybrid, not the pure alternative.** Concretely:

1. **Keep the primary SPEC's CRUD + Gemara-YAML-source model as the developer/authoring surface** (it wins on
   buildability, GitOps review, and POC fit — which §26.1 prioritizes).
2. **Adopt the event-sourced ledger for the *governance-change record* only** — i.e. make A1-MUST-022's
   "governance-change event" a *hash-chained, gapless, sealed ledger* rather than a fire-and-forget feed.
   This buys intrinsic tamper-evidence (DEF-05/§23), provably-correct as-of-date queries (DEF-09), and a
   natural substrate for the lifecycle-drift reconciler (DEF-03) — the three biggest adversarial wins — **without**
   forcing the whole API onto CQRS eventual consistency.
3. **Keep per-decision lineage in C2** (resolves DEF-12); the governance ledger references audit events by
   `correlation_id`, it does not absorb them.

In other words: **state-first storage for ergonomics, log-first storage for the audit guarantee.** The pure
event-sourced design is the right *mental model* and the right answer for the tamper-evidence/as-of-date
requirements; the pure CRUD design is the right answer for a small team shipping a POC. The hybrid takes the
specific, high-value pieces of event sourcing (the sealed governance ledger) and pays only that complexity,
leaving the authoring path boring. This is the recommended path; the pure ledger is the GA-scale fallback if
governance-corpus tamper-evidence becomes a first-class product requirement (e.g. regulated/FedRAMP customers).
