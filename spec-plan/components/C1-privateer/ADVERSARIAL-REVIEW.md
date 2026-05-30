# C1 — Privateer Integration — ADVERSARIAL REVIEW

**Reviewer persona:** skeptical external auditor + GRC-tool veteran who has seen "automated control satisfaction" lie.

## 1. Attacks on premise
- **A1 — "satisfied" is a high-stakes word the engine renders mechanically.** A control marked `satisfied` becomes audit evidence of operating effectiveness. If the verdict logic over-trusts thin or stale evidence, the platform *manufactures* false assurance — worse than no tool, because it carries institutional weight. The spec's `indeterminate` default (N-C1-6) is the right instinct, but the boundary between `satisfied` and `partially_satisfied` is under-specified: *how much* coverage is "satisfied vs partial"? 100%? 99%? A single uncovered namespace? Without a stated threshold, the verdict is a judgment call dressed as a computation. **Severity: HIGH.** *Demand:* define the coverage threshold and require the threshold + observed coverage_pct to appear on every `satisfied` verdict so an auditor sees the basis.
- **A2 — Garbage-in from A1.** Verdicts depend on A1's "required evidence types" per control. If A1 says SC-IMG-001 requires only `gatekeeper_decision` (and not `signature_verification`), a workload could be `satisfied` while signature checking was never actually consulted. C1 inherits A1's modeling errors and amplifies them into audit conclusions. **Severity: HIGH.** *Demand:* C1 must surface *which* required-evidence definition (and version) produced the verdict, so an A1 modeling gap is visible, not buried.

## 2. Correlation gaps
- **A3 — Supply-chain join by image digest is fragile.** SBOM/signature evidence is joined to C2 events by image digest / `resource_id`. Mutating admission (a registry rewrite, a digest-pinning mutation) can make the *admitted* digest differ from the *attested* digest, silently breaking the join → evidence looks missing when it isn't (or vice versa). **Severity: MEDIUM.** *Demand:* join must follow the post-mutation digest (use `after_state`/`mutation_diff` from C2), and a join miss must be a disclosed `partially_satisfied` reason, not silent.
- **A4 — Re-fetch vs recorded-value drift.** D-C1-05 cross-checks signature evidence against the C2 `external_data_refs` digest the engine consumed — good. But if C1 ever *re-fetches* current signature status (instead of the recorded value) it evaluates against today's truth, not the decision-time truth, producing a verdict that doesn't match what was enforced. **Severity: MEDIUM.** *Demand:* explicitly forbid re-fetch for historical evaluations; only the recorded `external_data_refs` value is authoritative (this is the whole point of C2 replay fidelity).

## 3. Cross-component contradictions
- **A5 — Detection ownership.** C1 says it *consumes* C3/C4 detections (N-C1-7); but DT-22/DT-24 describe Privateer as correlating evidence including bypass conditions. If C3/C4 and C1 disagree on whether a bypass occurred (different windows, different correlation rules), which verdict stands? **Severity: MEDIUM.** *Demand:* C3/C4 findings are authoritative for *detection*; C1's verdict cites the finding id; a C1 verdict may never contradict an unresolved C3/C4 finding.
- **A6 — Export-format ownership (shared with C2 D10).** Three components sign packages (C1 DT-24, C5 DT-46/78, C2 §7.6). C1's D-C1-03 defers to C2's primitive — correct — but this MUST be enforced domain-wide or auditors get three formats. **Severity: MEDIUM (domain-level).**

## 4. Abuse / failure
- **A7 — `indeterminate` as a dumping ground.** If too much evidence is `insufficient`, everything becomes `indeterminate` and the platform asserts nothing useful — an auditor reasonably reads pervasive `indeterminate` as "the control is not being monitored." The honest label can still be operationally useless. **Severity: MEDIUM.** *Demand:* `indeterminate` rate is itself a reported metric (a control mostly-indeterminate is a *finding about the evidence pipeline*, surfaced to C5/C3, not just a shrug).
- **A8 — Period-split explosion on frequent control changes.** D-C1-04 splits the period on each control-version change. A control edited daily yields fragmented evaluations that are hard to sample. **Severity: LOW.** *Demand:* cap/aggregate; warn when a control churns within an audit period.

## 5. Prioritized defect list
| # | Defect | Sev | Fix |
|---|---|---|---|
| D1 | `satisfied`/`partial` coverage threshold unspecified (A1) | HIGH | C1 §2.2: state threshold; show coverage_pct on every satisfied verdict |
| D2 | Inherits A1 required-evidence modeling errors invisibly (A2) | HIGH | C1: surface required-evidence-def version on verdict |
| D3 | Supply-chain digest join breaks on mutation (A3) | MEDIUM | C1 §3.4: join post-mutation digest; disclose misses |
| D4 | Re-fetch could evaluate against today not decision-time (A4) | MEDIUM | C1 §3.4: forbid re-fetch for historical evals |
| D5 | C1↔C3/C4 detection disagreement unresolved (A5) | MEDIUM | C3/C4 authoritative; C1 cites finding id; no contradiction |
| D6 | Export-format ownership (A6) | MEDIUM | domain index: C2 primitive only |
| D7 | `indeterminate` pervasiveness hides a broken pipeline (A7) | MEDIUM | report `indeterminate` rate as a finding |
| D8 | Period-split fragmentation (A8) | LOW | aggregate + churn warning |

**Top must-fix:** D1 (the `satisfied` threshold is the difference between honest and misleading evidence) and D2 (A1 modeling errors must be visible, not laundered into audit conclusions).
