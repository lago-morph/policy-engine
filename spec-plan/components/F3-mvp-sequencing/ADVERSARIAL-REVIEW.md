# F3 — POC Scale, MVP Scope & Sequencing — ADVERSARIAL REVIEW

**Reviewer persona:** Skeptical delivery lead / investor-grade scope auditor. Mandate: prove the MVP is either under-scoped to be credible or over-scoped to ship.

---

## 1. Headline finding

The cut line quietly **smuggles five "not in §27.1" components into MVP as "thin slices"** (D2, F1, F2, C3, B4) — and at least two of them (D2 storage authz, E1 differential simulation) are NOT thin. The §27.1 list reads like 9 items; the real MVP is **~14 components**, two of which are the hardest in the whole platform. Calling them "thin slices" understates the build and risks a missed date. **DEFECT-1 (critical to planning honesty).**

## 2. Defect list (prioritized)

**DEFECT-1 (critical) — "Thin slice" is doing too much work.** D2 (scope predicate over deferred storage) and E1 (differential simulation, the novel core) are labeled MVP-thin but are each plausibly the longest pole. The plan even admits E1 is the critical-path long pole and D2 is the #1 risk — yet the cut line presents them as enablers, not as the two hardest builds. Re-label them **MVP-core, high-effort**, and size them honestly, or the date is fiction.

**DEFECT-2 (high) — The MVP depends on a deferred capability.** §26.1 defers storage; R-F3-CUT-2 says "no MVP item may rely on a deferred item"; but D2 (MVP) needs storage-level scope enforcement, and §17A.5 needs replay datasets materialized in storage. The cut line violates its own rule. Either storage is partially in MVP (a scope-capable store) or the rule is false. This must be resolved, not finessed.

**DEFECT-3 (high) — AC-5 (differential simulation) is the demo and the riskiest line item, but acceptance treats it as one bullet.** "If I add rule X, N more denied" requires: a stable §13 replay set, a re-evaluation engine (B1) over historical events, a diff classifier (§17.4 four-quadrant), and scoped dataset materialization — across C2, B1, E1, D2. If any slips, the headline demo fails. AC-5 should be decomposed with its own sub-milestones and an early stubbed-data spike.

**DEFECT-4 (medium) — Capacity math is steady-state and ignores concurrency bursts.** SPEC §2 sizes audit at ~6/sec avg and worker-pool of 2, but 50 concurrent analysts (the §22 GUI max) each launching a 30-day namespace replay is a concurrency burst the plan doesn't size (echoes F2 DEFECT-6). "Functional validation not throughput" (R-F3-PERF-4) is a fair dodge, but the demo dies if a single replay takes 20 minutes under contention.

**DEFECT-5 (medium) — Base-first vs AI-first is asserted, not argued against the market.** The positioning memo explicitly flags Wedge-7 (AI agent governance) as "the optional long bet, category-defining," and Wedge-5+1 (OPA successor + simulation) as the fastest revenue. F3 picks base-first on engineering grounds (reframe doc: no refactor needed) — correct technically — but does NOT reconcile with the GO-TO-MARKET sequencing, where an AI-first or simulation-first wedge might be the actual first ship. F3 should state: *engineering builds base-first; product may MARKET a wedge-first slice of it.* These are not the same axis and the doc conflates them.

**DEFECT-6 (medium) — "3 contracts freeze in 1 week" is optimistic.** The §13 schema, the JWT→subject mapping, AND the Gemara/lineage model each have open questions (§26.3) — per-product replay fields, claim transforms, lineage shape. Freezing all three in a week before any engine emits real events invites a late, expensive re-freeze (DEFECT in F1/F2 too). Recommend: freeze a v0 contract, but explicitly version it and budget one re-freeze after the first engine emits real data.

**DEFECT-7 (low) — C3 "2 detections" may be too thin to prove value.** The Gatekeeper-bypass and JWT-drift detections (§14.2) are the platform's proof-of-correctness, but both require *correlation across sources* (admission event vs audit vs decision log). That correlation is itself non-trivial and depends on correlation_id discipline across B1/B2/C2. "2 detections" hides a cross-component integration.

**DEFECT-8 (low) — No explicit rollback/kill-switch for enforce-mode in the MVP plan.** The plan promotes to enforce (AC-1) but the sequence never builds the demote/rollback path as MVP. A bad enforce policy in `payments-prod` during a demo with no fast rollback is an own-goal. Pull F1 `lifecycle:demote` into MVP.

## 3. Inconsistencies vs other components

- **vs F1/F2:** all three independently flag the storage-authz contradiction (§26.1 vs §17A.1) — this is the platform's central unresolved risk and belongs in the cross-cut adversarial.
- **vs E1:** F3 calls E1 "MVP" and "critical-path long pole" simultaneously without reconciling effort.
- **vs the positioning memo:** base-first sequencing vs wedge-first GTM (DEFECT-5).

## 4. "Won't survive contact because…"

…the schedule presents 14 components as "9 plus thin slices," with the two hardest (scope-enforced storage authz and differential simulation) mislabeled as thin, on top of a storage layer the spec deferred — so the date slips precisely on the two things that make the product novel, and the headline demo (AC-5) is the first casualty.

## 5. Top fixes to merge into SPEC/PLAN

1. Re-label D2 and E1 as **MVP-core, high-effort**; size honestly (DEFECT-1).
2. Resolve the deferred-storage contradiction: a scope-capable store IS in MVP (DEFECT-2).
3. Decompose AC-5 into sub-milestones with an early stubbed-data spike (DEFECT-3).
4. Separate the **engineering** sequence (base-first) from the **GTM** wedge sequence; state both (DEFECT-5).
5. Freeze contracts as **v0 with a budgeted re-freeze**, not a one-shot lock (DEFECT-6); pull `lifecycle:demote` into MVP (DEFECT-8).
