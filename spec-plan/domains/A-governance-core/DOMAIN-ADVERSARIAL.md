# Domain A — Governance Core — ADVERSARIAL RECONCILIATION

Domain-level reconciliation of the per-component adversarial reviews (`A1/ADVERSARIAL-REVIEW.md`,
`A2/ADVERSARIAL-REVIEW.md`). Purpose: surface where the two components' findings **converge**, where they
**contradict**, and what the domain must fix as a unit before any cross-domain integration.

---

## 1. The convergent finding (both reviewers, independently, point here)

**The A1↔A2 lifecycle seam is the domain's structural weak point.**
- A1-DEF-03 (HIGH): "the two-lifecycle split has no defined reconciliation; drift is inevitable; *nobody owns it*."
- A2-DEF-04 (CRITICAL): A2 *claims* ownership of the reconciler (good), but runs it *periodically*, so the
  dangerous states (`zombie_enforcement` = retired control still enforcing; `premature_enforcement` = draft
  control already blocking) exist in the windows between runs.

**Reconciled domain decision:** the reconciler is **A2-owned (settled)** and MUST be **transition-time
preventive** for critical states, **periodic-detective** only for soft drift (`governed_not_enforced`). A
`retired` or `draft` control MUST be *refused* a live enforcement mode at realization time and on every A1
status event — never merely *flagged later*. This is the #1 domain-level fix and it spans both SPECs (A1 emits
the status events; A2 enforces the legality at realization). **Track as A-OQ-4.**

---

## 2. The fail-direction problem (the domain's most dangerous theme)

Three separate findings are all instances of **"the safety default points the wrong way under stress":**

| Finding | Stress condition | Wrong default | Right default |
|---|---|---|---|
| A1-DEF-02 (CRIT) | A1 partitioned, exception just tightened | cached exception fails *open* (allows old 90-day) | fail **most-restrictive** (deny new exceptions) |
| A2-DEF-05 (HIGH) | A1 relaxes `mode_intent`, A2 at enforce | "strictest wins" silently *keeps blocking* (overrides relaxation) | distinguish tighten vs loosen; don't override deliberate relaxation |
| A2-DEF-03 (HIGH) | on-call demotes a security control at 3am | ungated, log-only ⇒ *silent disable* | keep instant, add **post-hoc** mandatory review |

**Reconciled domain principle:** *fail-safe is not a single direction.* For **controls/admission**, fail
*closed* (most-restrictive). For **exceptions/relaxations**, "most-restrictive" means **deny the relaxation**,
which is *also* fail-closed — but the SPECs currently conflate "keep last-known" with "fail-safe." The domain
must adopt one rule: **under uncertainty/staleness/partition, resolve to the most-restrictive *effective*
posture, and require an explicit human action (not a timeout) to relax.** Apply uniformly across A1 exceptions
(A1-DEF-02), A2 mode conflicts (A2-DEF-05), and A2 demotion (A2-DEF-03).

---

## 3. The "declared vs verified" theme (the auditor's recurring objection)

| Finding | The gap |
|---|---|
| A1-DEF-01 (CRIT) | EvidenceRequirement *declares* required §13.3 fields; nothing verifies they're *emitted* (DT-16/DT-28 are literally "missing field" scenarios) |
| A1-DEF-06 (HIGH) | Coverage `full` is a *self-assessment* rendered as a *certified* badge; a `full` link to a dry-run-only control is a lie |
| A2-DEF-01 (CRIT) | Promotion gates run on historical audit data whose *completeness they never check* — a green ladder can be green because the simulation was blind |
| A2-DEF-10 (MED) | No gate verifies the implementation actually *matches the control's intent* |

**Reconciled domain decision:** Domain A must everywhere distinguish **assertion** from **evidence**. Concretely:
1. A1 coverage badges and EvidenceRequirements are labeled **management assertions** until backed by an
   *operating-effectiveness signal* (a non-empty EvaluationRequirement result / verified emission).
2. A2 promotion gates MUST consume and surface `replay_completeness`; promoting to `enforce` over low-completeness
   data requires explicit acknowledgement on the PromotionEvent.
3. This closes the single objection an external auditor (Daniel) will raise first, and it ties Domain A's
   credibility to **C2 (audit completeness)** and **C1/C3 (evaluation/analytics)** — a hard cross-domain
   dependency the master plan must sequence.

---

## 4. Contradictions *between* the two components

| # | A1 says | A2 says | Resolution |
|---|---|---|---|
| C-1 | A1 mirrors `current_policy_version` and **seals one hash** at retire (A1-MUST-012) | same control can run **many versions** across selectors (OQ-3); A2 is the authority (OQ-4) | `current_policy_version` is a **selector→version set**; A1 seals **all live pairs** (A2-DEF-06). *Both SPECs must update.* |
| C-2 | A1 vendors a **copy** of §13.3 `AUDIT_CORE_FIELDS@v1` to stay off critical path (PLAN) | A2 gates assert §13.3 emission via C2 | A1's vendored copy and C2's authoritative list **must reconcile**; vendored copy is a *bootstrap*, replaced by C2's published artifact (else A1 validates a stale floor). |
| C-3 | A1 grace window (DT-04, e.g. 14d) governs deprecation | A2 dry-run soak (DT-04, 7d) governs wind-down | A1 grace = **outer** bound, A2 dry-run = **inner**; retire requires **both elapsed** (A2-DEF-12). |
| C-4 | A1 retire guard trusts A2's enforcement-points **self-report** | A2 supplies the inventory | guard MUST **also** require an **audit-derived** "zero recent decisions" check (A1-DEF-07) — belt and suspenders. |
| C-5 | A1 EnforcementReq carries `mode_intent` (intent) | A2 owns realized `mode` | on conflict, **don't blanket "strictest wins"** (A2-DEF-05); block + force human resolution for loosening conflicts. |

None of these are fatal; all are **interface-contract gaps** that must be nailed before A1/A2 integration, not
after.

---

## 5. Severity-ranked domain defect roll-up

| Rank | Source | Sev | Domain-level fix |
|---|---|---|---|
| 1 | A1-DEF-01 + A2-DEF-01 | CRITICAL | "Declared vs verified": gates/requirements must consume real emission/completeness signals, not assertions. |
| 2 | A1-DEF-03 + A2-DEF-04 | CRITICAL | Reconciler = transition-time **prevention** for zombie/premature enforcement, not periodic detection. |
| 3 | A1-DEF-02 + A2-DEF-05 + A2-DEF-03 | CRITICAL→HIGH | Uniform fail-direction: under uncertainty resolve to most-restrictive *effective* posture; relax only by explicit human action. |
| 4 | A2-DEF-06 / C-1 | HIGH | `current_policy_version` is a selector→version set; fix the A1 seal accordingly. |
| 5 | A1-DEF-06 | HIGH | Coverage badge = assertion unless backed by operating-effectiveness signal. |
| 6 | A1-DEF-04 | HIGH | Add `control_id` aliases (renames/typo/merge) without breaking the single-authority rule. |
| 7 | A1-DEF-05 | HIGH | Pin Gemara schema version; scope signed export + verification to it. |
| 8 | C-2..C-5 | MEDIUM | Close the four A1↔A2 interface-contract gaps before integration. |

---

## 6. Verdict

The entity model (A1) and the gated state machine (A2) are **structurally sound and buildable** — neither
reviewer found a reason to redesign. The domain's risk is concentrated in **four cross-cutting themes**:
(1) declared-vs-verified, (2) the lifecycle seam, (3) fail-direction under stress, (4) scalar-vs-set version
mirroring. All four live **at the A1↔A2 boundary or at Domain A's boundary with C2/C1/C3**. They are
contract/semantics fixes, not architecture rewrites. **Gate before integration:** items 1–3 (the three
CRITICAL/HIGH themes) must be resolved in the SPECs before A2 is permitted to promote anything to `enforce` in
a shared environment.
