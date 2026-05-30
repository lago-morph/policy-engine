# B4 — Engine Selection, Action Taxonomy & CRDs — ADVERSARIAL REVIEW

**Reviewer persona:** hostile principal engineer who has seen "approval workflow" CRDs become the
permanent bypass mechanism for every policy in the org. Mandate: break the decision/CRD layer.

---

## A. The approval CRD (the load-bearing mechanism) and its abuse surface

- **AR-1 (HIGH) — PolicyApprovalRequest is the single most dangerous object in the platform and the
  SPEC under-secures it.** This CRD converts a deny into an allow. Whoever can (a) create one with a
  forged `subject`/`controlId`, (b) write the `approved` subresource, or (c) tamper with the
  external-data cache that admission reads — bypasses governance entirely. R-B4-12 RBAC-gates the
  approve subresource and forbids self-approval, which is necessary but not sufficient:
  - Nothing stops a user from creating a request with a *different* `requiredApproval.value` than the
    control actually mandates (downgrading "needs prod-release-approver" to "needs any-teammate") if
    the request's required-approval field is author-controlled. **The required approver MUST come from
    the control/policy, not from the requester.** The SPEC's schema puts `requiredApproval` in `spec`
    (author-writable) — that's the hole. Make it controller-derived from the controlId, immutable.
  - "Automated" and "external" approver types (§17B.1) mean a *system* can approve. If that system's
    credential leaks, every approval is forgeable. The SPEC says nothing about securing automated approvers.

- **AR-2 (HIGH) — Single-use idempotency (R-B4-13) matched by "correlationId + resourceRef + subject"
  is forgeable and race-prone.** The match key is all attacker-influenceable (a user controls their
  own resourceRef and can observe correlationIds). Two near-simultaneous admissions with the same
  resourceRef could both consume one approval before `phase: consumed` is written (TOCTOU), unless the
  consumption is a strongly-serialized compare-and-swap on the CRD. The SPEC says "optimistic
  concurrency" but admission webhooks are distributed and may race against the controller's status
  write. Specify the exact serialization point or approvals get double-spent.

- **AR-3 (MED) — Approvals become the org's permanent bypass.** Every approval-gated control, in
  practice, trains users to "just get it approved." Without rate-limiting, anomaly detection, and
  *expiry that actually re-prompts*, the approval flow becomes a rubber stamp. The SPEC bounds
  individual approvals (good) but has no notion of "this control is being approved-around 50x/week,
  which means it's mis-calibrated." That signal must feed C3 analytics or approvals silently gut the
  policy. (Same critique applies doubly to PolicyException — see AR-5.)

## B. Exceptions

- **AR-4 (MED) — PolicyException scope matching (`name: legacy-*`) is a wildcard footgun.** Scope
  globs (`legacy-*`) are convenient and catastrophic: a too-broad glob silently exempts resources the
  granter never intended (anything matching `legacy-*` forever, including future resources). Bounded
  *time* (R-B4-15) doesn't bound *scope creep within the glob*. Need: scope-match preview at grant
  time (show exactly what this exception currently exempts), and re-evaluation/alert when new
  resources start matching an existing exception's glob.

- **AR-5 (MED) — Exception expiry "re-enforces" but the SPEC doesn't say what happens to the
  resources that exist *because of* the exception.** When SC-IMG-001's exception for unsigned legacy
  images expires, those unsigned Deployments are already running. Admission only fires on *new/changed*
  requests, so expiry doesn't remove the existing violations — it just means a *future* change gets
  denied. The platform will show the control as "enforcing" while violating workloads keep running.
  The §19/audit-mode (B2-R8) detective scan is the only thing that catches this, and the SPEC doesn't
  wire exception-expiry → forced audit re-scan. Expiry without re-scan is a false sense of compliance.

## C. The action taxonomy and precedence

- **AR-6 (MED) — Closed 13-action enum (R-B4-5) will be wrong within a year.** "New effects require a
  spec change" is a governance virtue and an agility killer. The first time a customer needs
  "throttle" or "redirect-to-quarantine-registry" or an AI-governance action (block-model-deploy),
  the closed enum forces either a spec revision (slow) or a square-peg mapping into an existing action
  (loses fidelity). The F4/AI-governance extension will *certainly* want new actions. A closed enum
  for the *audit/analytics contract* is right; a closed enum for *what policies can do* is too rigid.
  Consider a closed *core* enum + a registered-extension mechanism with explicit schema.

- **AR-7 (MED) — The precedence order (R-B4-7) is asserted without justification and has wrong cases.**
  `deny > require_approval` means: if control X denies and control Y says "needs approval," the user
  gets a flat deny and never learns approval was even an option for Y — confusing and arguably wrong
  (the deny from X is unrelated to Y's approvable concern). Also `mutate > require_scan > warn`: should
  a mutation really silently happen before a required scan? And where do `notify`/`annotate` sit — they
  should be *non-exclusive* (always applied alongside the primary action), but a linear precedence
  treats them as mutually exclusive. The model conflates "the admission disposition" (one of
  allow/deny/suspend) with "side-effects" (mutate/annotate/notify/scan, which can co-occur). A single
  linear precedence over 13 heterogeneous actions is the wrong data model. Split into
  *disposition* (exclusive) + *obligations* (a set).

## D. The rubric

- **AR-8 (MED) — The scored rubric (§3.2) is decoration; nobody will run it per control.** A 12×5
  scoring matrix applied to every control is process theater — in practice engineers pick the engine
  by habit/familiarity and back-fill the justification (R-B4-1). Worse, the scores are hand-assigned
  and arguable (why is Kyverno "1" on complex logic and not "2"?). The rubric's *value* is the small
  number of clear rules (decision→OPA, effect→Kyverno, approval→CRD, build-time→Conftest); the scoring
  veneer adds false rigor. Keep the decision-tree (§3.1 questions), drop the pseudo-quantitative scores
  or be honest they're qualitative.

- **AR-9 (LOW) — "Record the rubric score for every control" (R-B4-1) is unbudgeted governance
  overhead** that will rot like any mandatory-metadata field. If it's not enforced and used, it's noise.

## E. CRDs and controllers generally

- **AR-10 (MED) — Six CRDs in v1alpha1 with six controllers is a lot of surface for an MVP.** The SPEC
  treats all six as peers, but only PolicyApprovalRequest + PolicyException are on the critical path.
  PolicyActionLibrary/PolicyEvidenceSchema are "data CRDs" (config dressed as CRDs — why are they CRDs
  and not ConfigMaps/bundle data? CRD-ness adds RBAC/versioning/etcd cost for static data).
  PolicyRemediationAction is a destructive-action engine that probably shouldn't ship in MVP at all.
  The SPEC should phase these, not present six co-equal controllers.

- **AR-11 (MED) — "Controllers reconcile and call external webhooks" inherits all of webhook-delivery's
  unreliability** (retries, dedup, ordering, poison messages) and the SPEC's failure handling (F1) is
  one line. External workflow integration is explicitly out-of-scope (§17B.3) yet the entire approval
  flow *depends* on it. So the platform's headline approval feature has a hard dependency on a system
  the spec refuses to specify. That's a real gap: at minimum the platform needs a built-in default
  approver UI/path so approval works with *zero* external workflow system, or the feature is
  non-functional out of the box.

- **AR-12 (LOW) — Remediation default "dry-run/propose unless armed" (R-B4-21) is good, but "armed"
  isn't defined.** Who arms it, with what authority, and is arming itself approval-gated? An un-gated
  arming switch turns the safe default into a foot-gun the first time someone arms quarantine cluster-wide.

---

## Prioritized defect list

| ID | Sev | Defect | Required resolution |
|---|---|---|---|
| AR-1 | **HIGH** | `requiredApproval` is author-writable → approver-downgrade bypass | Controller-derive required approver from controlId; immutable; secure automated approvers |
| AR-2 | **HIGH** | Single-use approval match key forgeable + TOCTOU double-spend | Strongly-serialized CAS consumption; non-attacker-controlled match identity |
| AR-5 | MED | Exception expiry doesn't remove existing violating workloads | Wire expiry → forced §19 audit re-scan; don't show "enforcing" while violations run |
| AR-7 | MED | Linear precedence conflates exclusive disposition with co-occurring obligations | Split into disposition (exclusive) + obligations (set); re-derive precedence |
| AR-11 | MED | Headline approval flow hard-depends on out-of-scope external workflow | Ship a built-in default approver path so it works with zero external system |
| AR-3 | MED | Approvals become permanent rubber-stamp bypass | Rate/anomaly analytics (C3) on approval-around frequency; mis-calibration signal |
| AR-4 | MED | Exception scope globs silently over-exempt incl. future resources | Grant-time scope preview + alert when new resources match an existing glob |
| AR-6 | MED | Closed 13-action enum too rigid for AI-gov / future effects | Closed core + registered-extension mechanism for actions |
| AR-8 | MED | Scored rubric is false rigor / process theater | Keep the decision-tree rules; drop or honestly label the scoring |
| AR-10 | MED | Six co-equal v1alpha1 CRDs/controllers is too much MVP surface | Phase: Approval+Exception first; data-CRDs may be ConfigMaps; Remediation post-MVP |
| AR-12 | LOW | "Armed" remediation undefined | Define arming authority; arming itself approval-gated for destructive actions |
| AR-9 | LOW | Per-control rubric score recording rots | Only require it where engine choice is non-obvious |

**Verdict:** The decision/effector split (R-B4-2) and the deny-with-approval-required pattern are the
right answers to the §17B.4 hard constraint — this is the strongest idea in the domain. But the
approval/exception CRDs are **the platform's biggest abuse surface** and the SPEC treats them as
plumbing. **AR-1** (author-controlled required-approver) and **AR-2** (forgeable single-use) are
outright security defects; **AR-5** (expiry ≠ remediation) and **AR-11** (headline feature depends on
out-of-scope external system) make the approval story partly non-functional. Fix AR-1/AR-2 before
anything ships, and re-model actions per AR-7 (disposition vs obligations) before B1/B2/C2 bake in the
linear-precedence assumption.
