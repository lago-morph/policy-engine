# Spec: `synthesis-reconciliation-gate`

## Intent

When you fan out N agents in parallel to author documents that **reference a shared
contract or each other**, the parallelism that buys you speed also guarantees a specific
failure: the documents drift, and the drift is invisible from inside any single agent.
This session proved it twice. In Wave 2, five cross-cutting agents were dispatched to
*reconcile* the corpus — and they disagreed with each other about whether the keystone
C2 schema was "frozen" (`CROSSCUT-ADVERSARIAL` said re-open; `MASTER-PLAN` said FROZEN;
`INTER-DOMAIN-CONTRACTS §5.5` said "no new field" while `XD-3` said a new field was
required). A build team handed that corpus could not tell whether to freeze or re-open.
In Wave 5 the *same* failure reproduced one level down: six NFR components each
re-architected the C2 hash chain for operational reasons, and none was folded back. The
fix both times was a dedicated, **serial** reconciliation pass that reads all the parallel
outputs together and produces one canonical artifact where they disagree. This skill makes
that pass a required, repeatable step rather than a thing you notice late.

## Trigger

- **Direct**: "reconcile these", "do the docs agree?", "is this corpus internally
  consistent?", "is the contract actually frozen?", after any `parallel-subagent-fanout`
  run that produced cross-referencing documents.
- **Proactive**: activate automatically whenever ≥3 agents authored, in parallel,
  documents that (a) cite a shared schema/contract/interface, or (b) link to each other.
- **Negative**: skip for independent leaf documents that share no contract (e.g. 9
  unrelated product README files), or for a single-author serial run.

## Inputs

- The set of parallel-authored documents (paths or a glob).
- The shared contract(s) they depend on (e.g. a schema SPEC), if one exists.
- Optional: prior reconciliation docs to update rather than recreate.

## Outputs

- **One canonical reconciliation artifact** that, for each shared contract, states the
  single authoritative definition and explicitly lists every place that must change to
  match (an "override/ratification list").
- A **ranked contradiction register** (id, the docs in tension, severity, the conflict,
  the proposed resolution, and whether it is a *correctness* fix or *hardening*).
- A **provisional-status flag** on any contract that a consumer asked to change: it is not
  "frozen" until this gate confirms no open ratification asks remain.

## Workflow

1. Enumerate the parallel-authored docs and the shared contracts they reference.
2. For each shared contract, extract every statement each doc makes about it (the field
   list, the enum, the invariant, the "frozen/provisional" claim). Put them side by side.
3. Diff them. Any field/enum/invariant/status that is not byte-identical across docs is a
   contradiction — record it. Pay special attention to *status* claims ("frozen") and to
   "no change needed" vs "change required" pairs (the highest-signal contradiction shape).
4. For each contradiction, pick the authoritative version, write it once, and list every
   doc that must be edited to match. Mark correctness-vs-hardening and an owner.
5. Produce the canonical artifact + ranked register. Where a contract was called "frozen"
   but has open ratification asks, downgrade it to `*-rc` / provisional in the index.
6. Check whether the proposed changes *compose* — do they fight each other? (In this
   session: do chain-sharding + per-tenant-chain + epochs + restore-markers compose? Answer
   only after this explicit step.) State the composite model or the conflict.
7. Update the master index / handoff to point at the canonical artifact and mark superseded
   docs. Do NOT silently leave the parallel docs as co-equal sources of truth.

## Concrete examples

**Example 1 — the C2 freeze contradiction (Wave 2 → Wave 4).**
Inputs: `cross-cutting/{CROSSCUT-ADVERSARIAL,MASTER-PLAN,INTER-DOMAIN-CONTRACTS,DATA-MODEL}.md`
+ `components/C2-audit-schema/SPEC.md`. Step 3 surfaced that four docs made incompatible
status claims about the same 36-field schema, and that `INTER-DOMAIN-CONTRACTS §5.5`
("no new field") directly contradicted `XD-3` ("needs a new field"). Output:
`C2-v1.0-rc-RECONCILED.md` (41 fields, additive-only) as the single canonical schema, a
9-ticket `BUILD-BLOCKING-FIXES.md`, and a master-index flag marking the other docs
"provisional / superseded on these points." Without the gate, the contradiction shipped.

**Example 2 — the NFR chain-model re-architecture (Wave 5).**
Inputs: 8 `components/G*/SPEC.md`. Step 3 found G1 (shard the chain), G5 (per-tenant
chain), G6 (epoch boundary), G3 (restore-boundary + failover fence), G7 (erased_input),
G4 (key_id/KTL) each changed C2's chain model independently. Step 6 (composition check)
produced the unified model: identity `(tenant, source.system, cluster, epoch)`, monotonic
`chain_seq` within the tuple, plus a global signed roll-up super-checkpoint — and the
finding that they compose *in principle* but the rc still encodes a bare per-source integer,
so they "conflict on paper." Output: `NFR-CROSSCUT-ADVERSARIAL.md` with a consolidated
N-1..N-10 ratification list folded into the single open C2-rc.

## Anti-patterns

- **Treating the parallel docs as co-equal sources of truth.** Once they disagree, pick one
  and mark the rest superseded — don't leave the reader to reconcile.
- **Declaring a contract "frozen" inside a parallel wave.** It is provisional until this
  gate runs; this session's whole C2 saga is the cost of skipping it.
- **Running the reconciliation itself as another parallel fan-out.** The gate must be
  serial (or single-author) — parallel reconciliation reproduces the disease (proven here).
- **Stopping at "list the contradictions."** You must also check whether the fixes compose.

## Acceptance criteria

1. Every shared contract has exactly one canonical definition after the pass.
2. Every contradiction has a resolution, an owner, and a correctness/hardening tag.
3. Any "frozen" status with open ratification asks is downgraded to provisional/`-rc`.
4. The composition question ("do the fixes fight?") is explicitly answered.
5. The master index/handoff points at the canonical artifact; superseded docs are marked.

## Files this skill creates / modifies

- `cross-cutting/<CONTRACT>-RECONCILED.md` (or `-rc`) — the canonical artifact.
- `cross-cutting/<...>-CROSSCUT-ADVERSARIAL.md` — the ranked contradiction register + ratification list.
- The master index / handoff — updated pointers and provisional-status flags.
