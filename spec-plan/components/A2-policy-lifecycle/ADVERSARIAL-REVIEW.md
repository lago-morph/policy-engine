# A2 — Policy Lifecycle — ADVERSARIAL REVIEW

**Persona:** hostile principal engineer + auditor. Guilty until proven innocent.
**Target:** A2 `SPEC.md` / `PLAN.md`. **Posture:** the promotion ladder is where bad policy reaches production
traffic; every gate is a place to hide a false sense of safety. Prove the gates actually gate.

---

## 1. Thesis under attack

A2 sells a "safe, gated, reversible" path from intent to enforcement. The implicit claim is: *if a policy
reached `enforce`, it was simulated, soaked, and approved, so it's safe.* That claim is only as strong as the
**weakest gate and the data it ran on**. A green ladder with a hollow gate is *worse* than no ladder — it
launders an unsafe policy into production with an audit trail that says "we checked." This review hunts hollow
gates.

---

## 2. Findings

### A2-DEF-01 — [CRITICAL] G-SIM gates trust E1's simulation, which trusts the audit history's *completeness*
G-SIM-BASELINE and G-DIFF (the gates that "prove" a promotion is safe) run over historical audit logs. If that
history is **incomplete** (DT-25 is literally "replay completeness insufficient"; events missing
`correlation_id`/`policy_version` per DT-16/DT-28), the differential simulation sees fewer would-blocks than
reality and reports "zero Requires-review rows" — a **false all-clear**. A2 promotes to enforce, and the
*actual* traffic includes the un-simulated cases that now get blocked in production. A2's SPEC never requires
the gate to **check the replay-completeness of its own input dataset**. **Remedy:** G-SIM-BASELINE MUST consume
and assert a minimum `replay_completeness` over the gate window and MUST surface "simulated against N% complete
history" on the PromotionEvent; promotion to `enforce` over low-completeness data MUST require explicit
acknowledgement. Otherwise the soak/diff gates are decorative.

### A2-DEF-02 — [CRITICAL] G-SOAK "warn volume within tolerance of dry-run forecast" assumes traffic stationarity
G-SOAK (DT-05 step 4) passes when warn volume ≈ dry-run forecast. But dry-run ran over the *last 30 days* and
warn observes *now*. A weekly batch job, a quarter-end surge, a Black-Friday spike, or a deploy freeze means
the forecast and the soak window sample **different traffic distributions**. A policy that looked safe on
30-day-old traffic blocks a Monday-morning batch that never appeared in the replay window. The gate's core
assumption — that past traffic predicts future traffic — is unstated and frequently false. **Remedy:** state
the stationarity assumption explicitly; SHOULD require the soak window to span at least one full business
cycle relevant to the control's applicability, or flag low-confidence when traffic variance is high.

### A2-DEF-03 — [HIGH] Demotion is ungated and instant (D4) — correct for safety, but it's also an attack vector
D4 makes demotion (enforce→warn/dry-run) ungated and broadly authorized (on-call/admin) "so rollback never
causes an outage." Inverted: any on-call engineer can **silently disable an enforcing security control**
("demote to warn") with no approval, and the only trace is a PromotionEvent. That is a legitimate break-glass,
but it is also exactly how an insider neutralizes image-signing at 3 a.m. The SPEC treats demotion purely as a
safety feature and never as an abuse surface. **Remedy:** demotion MUST be ungated (keep D4) BUT MUST trigger a
*post-hoc* mandatory approval / review SLA (a "you demoted a security control, justify within N hours or it
auto-re-promotes / pages the control owner"). The §17B approval becomes *after* the action, not *before*.
Otherwise "reversible" = "trivially defeatable."

### A2-DEF-04 — [HIGH] The reconciler (D5, WS-G) is load-bearing for the whole domain but runs *periodically*
A2 generously claims ownership of the A1↔A2 seam reconciler (good — A1-DEF-03 said nobody owned it). But §8
makes it "periodic + event-triggered," and the dangerous states (`zombie_enforcement`: a *retired* control still
*enforcing*; `premature_enforcement`: a *draft* control already blocking) are **time-windowed**. Between
reconciler runs, a retired control can keep denying production traffic with a `control_id` that no longer
resolves in A1 — denials nobody can explain (the inverse-orphan). For a *critical* finding, periodic is too
slow. **Remedy:** `zombie_enforcement` and `premature_enforcement` MUST be **prevented at the transition**, not
*detected after*: A2 MUST refuse to realize/keep any mode for a control whose A1 status forbids it, checked at
realization time and on every A1 status event, not on a timer.

### A2-DEF-05 — [HIGH] "Most-restrictive wins" on mode_intent conflict (D5/A2-MUST-051) can cause a self-inflicted outage
A2-MUST-051: if A1's `mode_intent` and A2's realized `mode` conflict, "most-restrictive wins for safety." But
consider: Priya edits the EnforcementRequirement's `mode_intent` from `deny` to `warn` (intending to *relax* a
noisy control), while A2 has it at `enforce`. "Most-restrictive wins" keeps it at `enforce`/deny — the **opposite
of the intended relaxation** — and raises a finding nobody looks at until the next incident. So an explicit
human decision to relax is silently overridden by a "safety" rule. **Remedy:** distinguish *tightening* conflicts
(auto most-restrictive is fine) from *loosening* conflicts (an explicit authoritative relaxation from A1 should
win, or at minimum *block and force human resolution immediately* rather than silently keeping the stricter mode).
A blanket "strictest wins" mis-handles deliberate relaxations.

### A2-DEF-06 — [HIGH] `current_policy_version` single-authority (OQ-4) collides with multi-selector modes (OQ-3)
OQ-4 says A2 is the single authority for `current_policy_version` (good, fixes the A1 mirror ambiguity). But
OQ-3 allows the *same control* to run at different versions/modes on different clusters (hipaa-dev v25-warn,
hipaa-prod v24-enforce). So "the" `current_policy_version` is **not a scalar** — yet A1 mirrors it as one field
and *seals a single hash at retire* (A1-MUST-012). Which version gets sealed when there are three live? The
single-scalar mirror is a lie under multi-selector rollout. **Remedy:** `current_policy_version` must be a
*set keyed by selector*, and A1's seal must record all live (selector→version) pairs, not one.

### A2-DEF-07 — [MEDIUM] G-APPROVE separation-of-duties is checkable but the "second admin" pool may be size 1
A2-MUST-040 requires a *different* admin to approve enforce. In a small POC team (the spec repeatedly says
POC-scale, §26.1), there may be exactly one Platform Governance Admin. Then SoD is **unsatisfiable** and either
(a) promotion is permanently blocked, or (b) someone grants themselves a second account — defeating the control.
The SPEC mandates SoD without addressing the small-team reality it elsewhere assumes. **Remedy:** define the
behavior when the approver pool < 2 (e.g. explicit break-glass single-admin promotion *with* heightened audit
+ post-hoc review), rather than leaving an unsatisfiable requirement.

### A2-DEF-08 — [MEDIUM] Gates bind to artifacts "immutably" but the artifacts (E1 reports) may themselves be mutable
A2-MUST-011 binds gate artifacts (simulation report ids) into the PromotionEvent "immutably." But A2 only stores
a *reference*; if E1's report store lets a report be regenerated/overwritten under the same id, the "immutable"
binding points at mutable content — the rollback reconstruction (DT-06) replays a *different* report than the one
that justified the promotion. **Remedy:** bind by *content digest*, not by id; or require E1 reports to be
write-once. A reference is not immutability.

### A2-DEF-09 — [MEDIUM] Build-Time "enforce = CI hard-fail" gives no dry-run, so first deployment IS production
§4.2/DT-07: Conftest controls go `draft→enforce` with G-AUTHOR only — no dry-run, no soak. So the *first time*
a Build-Time control runs against real PRs, it's already blocking merges. A noisy/false-positive Conftest rule
(e.g. the resource-limits rule misfiring on a valid chart shape) hard-fails every developer's PR on day one,
with no warn period. The SPEC removed the safety ladder for Build-Time without giving Build-Time its own ramp.
**Remedy:** Build-Time SHOULD support a non-blocking "annotate/warn" CI mode as a real ramp stage (the SPEC
mentions it parenthetically but doesn't make it a gate stage), so a new CI rule can soak as warnings before
hard-failing builds.

### A2-DEF-10 — [MEDIUM] No gate on "does the realized policy actually match the control's intent?"
Every gate checks *operational* safety (will it block too much? did someone approve?). **No gate checks
semantic fidelity:** that the Rego Marcus wrote (especially a hand-filled DT-09 template) actually implements
Priya's EnforcementRequirement. DT-09 step 8 has Priya "review and confirm the logic matches" — but that's a
manual courtesy, not a gate. A2 will happily promote a syntactically-valid, well-simulated policy that enforces
the *wrong thing*. **Remedy:** add an optional but recommended G-INTENT gate: the control owner (Priya) signs off
that the implementation matches intent before enforce, recorded like G-APPROVE.

### A2-DEF-11 — [LOW] Idempotency key (impl, version, mode) doesn't cover the gate *inputs*
A2-MUST-030 makes `:promote` idempotent on (impl, version, mode). But two promotions with the same key could
ride on *different* gate runs (different simulation datasets). Replaying the "same" promotion may bind a newer
report. **Remedy:** include the gate-result digest in the idempotency identity.

### A2-DEF-12 — [LOW] Deprecation wind-down (DT-04) overlaps A1's grace window — double-owner ambiguity
DT-04 has A1 set a grace window AND A2 move the constraint to dry-run for 7 days. Two timers, two components,
one wind-down. If they disagree (A1 grace = 14d, A2 dry-run soak = 7d), which governs retire? **Remedy:** define
that A1's grace window is the *outer* bound and A2's dry-run is *within* it; retire requires *both* elapsed.

---

## 3. Cross-component contradictions

- **vs A1:** A1-DEF-03 (lifecycle seam) is claimed-fixed by A2's reconciler, but A2-DEF-04 shows the reconciler
  being periodic re-opens the critical cases. The fix must be *transition-time prevention*, agreed by both.
- **vs E1:** A2's entire gate value rests on E1 simulation fidelity (A2-DEF-01/08). If E1's completeness/
  immutability guarantees are weaker than A2 assumes, the ladder is hollow. This contract must be explicit.
- **vs B4:** abstract `enforce`→concrete mapping lives in B4 (D2). If B4 maps `enforce` differently than A2's
  rollback timing assumes, the <2-min demotion SLA (A2-MUST-060) may be unmeetable on some engines.
- **vs C2:** soak forecasting and reconciliation both consume audit; if C2's audit is lossy, A2's forecasts and
  zombie-detection silently degrade.

---

## 4. "Will not survive production because…"

1. …the gates run on historical audit data whose completeness A2 never checks (DEF-01) — the single biggest
   hollow-gate risk. A green promotion can be green because the simulation was blind.
2. …`zombie_enforcement` is detected on a timer, not prevented at the transition (DEF-04) — production will, at
   some point, emit denials for a `control_id` A1 has retired, and on-call won't be able to explain them.
3. …"most-restrictive wins" silently overrides deliberate relaxations (DEF-05) — operators will lose trust when
   their intended warn-down keeps denying.
4. …demotion is a one-click, no-approval security-control kill switch (DEF-03) with only a log entry — fine as
   break-glass, dangerous as the *only* control.

---

## 5. Prioritized defect list

| Rank | ID | Sev | Demand |
|---|---|---|---|
| 1 | A2-DEF-01 | CRITICAL | Gates must assert replay-completeness of their input dataset; low completeness ⇒ explicit ack. |
| 2 | A2-DEF-04 | CRITICAL | Prevent zombie/premature enforcement at the *transition*, not via a periodic reconciler. |
| 3 | A2-DEF-02 | HIGH | State the stationarity assumption; soak ≥ one business cycle or flag low confidence. |
| 4 | A2-DEF-05 | HIGH | Distinguish tightening vs loosening conflicts; don't silently override deliberate relaxations. |
| 5 | A2-DEF-03 | HIGH | Demotion stays instant but requires post-hoc justification/review SLA (abuse surface). |
| 6 | A2-DEF-06 | HIGH | `current_policy_version` is a selector→version *set*; A1 seal records all live pairs. |
| 7 | A2-DEF-08 | MEDIUM | Bind gate artifacts by content digest, not mutable id. |
| 8 | A2-DEF-09 | MEDIUM | Give Build-Time a real warn/annotate ramp before hard-fail. |
| 9 | A2-DEF-10 | MEDIUM | Add a G-INTENT sign-off gate (impl matches control intent). |
| 10 | A2-DEF-07 | MEDIUM | Define SoD behavior when approver pool < 2 (small POC team). |
| 11 | A2-DEF-12 | LOW | Make A1 grace the outer bound, A2 dry-run inner; retire needs both. |
| 12 | A2-DEF-11 | LOW | Fold gate-result digest into the idempotency identity. |

**Bottom line:** the state machine and the gate *structure* are well-designed and buildable. The danger is not
the ladder — it's that **the gates run on data they don't validate** (DEF-01) and that the **most dangerous
illegal states are detected late instead of prevented** (DEF-04). Those two turn a safety system into a
confidence-laundering system. Fix them before A2 is allowed to promote anything to `enforce`.
