# E1 — ALT architecture: Replay-as-batch-job vs. always-on shadow service

**Component:** E1 · **Type:** Alternative architecture (high-value tree) · **Date:** 2026-05-30
**Question framed:** Should the primary simulation substrate be an **on-demand batch replay job** (the SPEC's default) or an **always-on live shadow service** that continuously evaluates production traffic against candidate bundles?

The primary SPEC chooses **batch `PolicySimulationRun` as default**, with shadow (M3) as an opt-in add-on. This doc argues the *opposite* arrangement and weighs it honestly.

---

## 1. The alternative: shadow-first

**Architecture.** A long-lived **Shadow Evaluator** sidecar/deployment sits beside each enforcing PDP. Every admission/decision request is tee'd to the shadow, which evaluates it against *N candidate bundles* in parallel with the live enforcing bundle, emitting a decision-pair stream to a time-series store. Differential analysis becomes a *continuous* query over that stream rather than a discrete job.

```
request ──► enforcing PDP ──► (real decision, enforced)
        └─► Shadow Evaluator ──► [bundle:v12 dec, bundle:v13 dec, ...] ──► stream store ──► continuous diff
```

- M3 (Live Shadow) becomes the *primary* substrate; M5 (Differential) is a *view* over the shadow stream.
- Historical replay (M2) is reframed as "replay into the shadow store" for back-fill.

---

## 2. Trade-off analysis vs. primary (batch) spec

| Dimension | Batch replay (primary) | Shadow-first (this ALT) |
|---|---|---|
| **Reproducibility** | **D-FULL** — same evidence+bundles ⇒ identical result, forever | D-LIVE — each window sees different traffic; not re-runnable; weaker for signed promotion evidence |
| **Coverage of rare events** | Replays *all* historical denies incl. rare ones (M7) | Only sees what flows during the window; rare/seasonal events may never appear |
| **Forward-looking signal** | None until you replay | **Real production traffic, live** — catches drift the moment it happens |
| **Cost** | Pay per run; cheap, bounded | Continuous compute per PDP; N-bundle fan-out multiplies cost |
| **Operational risk** | Read-only job; nothing in the hot path | Sidecar in/near the admission hot path; a shadow bug can add latency or fail open/closed incorrectly |
| **External-data determinism** | Pin to historical snapshot per event | Uses *live* external data — conflates policy change with data change |
| **Promotion gate fit** | Signed, reproducible §17E.4 report (HL-17, DT-49 SC) | Hard to sign "the policy is safe" against a non-reproducible window |
| **Audit-schema dependency** | Hard dependency on C2 replay fields | Lower — generates its own decision-pair stream; but loses cross-product/historical reach C2 gives |
| **Catches "previously blocked"** | Yes — M7 replays prior denials directly | Only future analogues of prior denials |

---

## 3. Where shadow-first genuinely wins

- **Continuous drift detection** (§16 Runtime Enforcement View, §14 analytics): you learn a candidate bundle would newly-block legit traffic *as it happens*, not at next replay.
- **No audit-schema completeness problem for the live window**: the shadow captures the exact input the enforcing PDP saw, so `replay_completeness` is `complete` by construction for shadow-observed events.
- **Validates the input-reconstruction itself**: shadow decisions on live input can be diffed against C2's reconstructed-input replay to *verify* the audit schema is actually replay-faithful — a powerful C2 correctness oracle.

## 4. Where shadow-first loses (why primary stays batch)

- **The flagship scenario (HL-17) demands reproducibility and a signed report over a fixed 30-day set.** Shadow cannot produce "47 newly blocked over the same evidence set" deterministically.
- **Rare-event coverage**: the whole point of §17.1 ("this event happened and should not happen again") is often a *rare* event; you cannot wait for it to recur in a shadow window.
- **Cost and hot-path risk** at every PDP across 9 products is operationally heavy for a "lightweight operational model" goal (§3 G5).

---

## 5. Recommended synthesis (and why the SPEC already reflects it)

**Batch is the authoritative substrate; shadow is an opt-in continuous signal that *feeds* the batch substrate.** Concretely:

1. Default and MVP = batch `PolicySimulationRun` (reproducible, signable, C2-replay-based).
2. M3 shadow runs as an **opt-in** evaluator that **archives its decision-pairs as immutable EvidenceSets**, so a later *batch* differential over a shadow-derived set is reproducible (SPEC §3 M3 "authoritative-when" + Failure-mode "archive observed decisions").
3. Use shadow output as a **correctness oracle for C2's reconstructed input** (diff shadow-on-live-input vs replay-on-reconstructed-input; divergence ⇒ audit schema bug).

This keeps reproducibility + signed-report fitness as the backbone while harvesting shadow's forward-looking value — capturing most of the ALT's upside without paying its reproducibility/cost penalty as the default.

**Verdict:** primary (batch-first) chosen. Shadow-first rejected as default due to non-reproducibility against the promotion-gate requirement, rare-event blindness, and hot-path/cost risk — but its live-window strengths are absorbed via archive-to-EvidenceSet and the C2 oracle pattern.
