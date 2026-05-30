# Domain B — Policy Engines & Enforcement — DOMAIN ADVERSARIAL RECONCILIATION

**Domain lead deliverable.** Date: 2026-05-30. Reconciles the five per-component adversarial reviews,
surfaces **contradictions *between* components in Domain B**, and ranks the cross-component defects.

---

## 1. The cross-component contradictions (the important part)

### X-1 (CRITICAL) — "One Rego everywhere" vs. divergent engine input shapes
**Components:** B1 (AR-2), B2 (AR-6), B3 (AR-2). **Three independent reviewers found the same hole.**
- B1 claims cross-engine conformance (R30). B2's Gatekeeper template reads `input.review.object`;
  B3's Conftest reads a bare parsed doc; an app PDP reads its own input. The *same package* cannot
  natively read all three.
- B3 "resolves" this by wrapping Conftest input in a fake `{review:{object}, subject}` envelope
  (R-B3-10). But B3-AR-2 then shows this **fabricates an identity** (the CI runner's, not the
  deploying user's), so identity-aware controls evaluated at build-time are *silently wrong* while
  "conforming."
- **Net contradiction:** the conformance suite (B1-R30) passes on synthetic identical input, but
  B2/B3 feed engine-shaped (and for B3, fake-identity) input in production. "Shared semantics" is
  true for *non-identity, structural* decisions and **false for identity-aware ones at build-time.**
- **Reconciliation required:** (a) make the input-normalization contract a real, jointly-owned spec
  (not a B3 footnote); (b) **explicitly exclude identity-dependent controls from the build-time
  (Conftest) class** — do not fabricate identity; (c) restate the platform claim precisely: *shared
  decision logic over a normalized resource model*, with identity-aware decisions only at runtime PDPs.

### X-2 (CRITICAL) — Action model: B4's linear precedence vs. how B1/B2 actually use it
**Components:** B4 (AR-7), B2 (AR-7), B1 (AR-9).
- B4 defines a single **linear precedence** over 13 actions (R-B4-7). B2-AR-7 independently asks
  "what happens when two controls fire (one deny, one require_approval)?" B1-AR-9 flags that the
  derived `allowed` boolean mis-collapses `require_approval`.
- All three point at the same modeling error: the platform conflates the **exclusive admission
  disposition** (exactly one of allow / deny / suspend-pending-approval) with **co-occurring
  obligations** (mutate, annotate, notify, require_scan — which can all happen together). A linear
  order over a heterogeneous 13 is the wrong data model.
- **Reconciliation required:** split the result into **disposition** (exclusive) + **obligations**
  (a set), *before* C2 (audit schema) and E1 (simulation) bake in the linear-precedence assumption.
  This is a v1alpha contract change that gets much harder later. B1's `allowed` boolean should be
  dropped (or become a 3-state disposition).

### X-3 (HIGH) — Approval flow: durability gap vs. the CRD that's supposed to close it
**Components:** B2 (AR-2, AR-3), B4 (AR-1, AR-2), B5 (AR-4).
- B2-AR-2: the deny→CRD hand-off is best-effort and lossy (admission webhooks are stateless;
  Gatekeeper doesn't "emit an event for a controller"). B4-R14 *asserts* a durable queue fixes it but
  doesn't specify the mechanism. B5-AR-4: the retry mints a *new* correlation_id, fragmenting the
  trace across the approved deploy.
- B4-AR-1/AR-2 add security defects on top: `requiredApproval` is author-writable (approver
  downgrade), and single-use consumption is forgeable/TOCTOU.
- **Net:** the platform's headline §17B.4 capability is, as specified, **neither durable, nor
  traceable across retry, nor secure against approver-downgrade.** Three components each own a piece
  and each assumes another fixes it.
- **Reconciliation required:** build the approval flow as **one vertical slice (B2+B4+B5) with a
  single owner**, specifying: (a) the durable deny→CRD mechanism concretely; (b) a stable
  *governance-transaction id* that survives retry (distinct from per-request correlation_id, B5-AR-4);
  (c) controller-derived immutable `requiredApproval` (B4-AR-1); (d) serialized CAS consumption (B4-AR-2).

### X-4 (HIGH) — Fail-closed defaults compose into a self-amplifying outage
**Components:** B1 (AR-6), B2 (AR-5), B5 (AR-2, AR-7), B3 (AR-5).
- B1-AR-6: fail-closed cold start bricks clusters. B2-AR-5: failurePolicy=Fail + slow external-data =
  fleet-wide mass-deny. B5-AR-2/AR-7: B5 makes "time out to failurePolicy" a *flow invariant*, which
  *guarantees* the mass-deny, and the composed fail-closed defaults can prevent recovery of the very
  dependency that failed. B3-AR-5: the same instinct ("fail loudly") gets the CI gate disabled.
- **These are not four bugs; they're one systemic property:** the domain's security-correct defaults,
  composed, create correlated availability failures with no circuit breaker.
- **Reconciliation required:** a **domain-wide "infrastructure-degraded" mode** distinct from "policy
  says no" — when a *shared dependency* (bundle server, verifier, Keycloak/D1) degrades, the system
  drops to warn/advisory with loud alerting + a §19 catch-up scan, rather than mass-denying. The
  system-namespace carve-out (B2-R4) is necessary but insufficient (B2-AR-4). This needs to be a
  *flow-level* (B5) decision, not five independent component choices.

### X-5 (HIGH) — Replay-exactness claimed by B1/B5, broken by external-data
**Components:** B5 (AR-5), B1 (AR-7), B3 (relatedly AR-1).
- B1-R26 captures nondeterministic *builtins* (nd_builtin_cache). B5-R2 claims "the decision a user
  hit can be replayed exactly." But B5-AR-5: external-data (the signature-verification result — the
  *headline §18.1 example*) is fetched live and is **not** captured by nd_builtin_cache. If the
  verifier's answer changes between t3 and replay, replay diverges.
- B3-AR-1 is the build-time mirror: a green CI check and a runtime deny aren't contradictory because
  they evaluated different facts.
- **Reconciliation required:** decision evidence MUST capture the **external-data values used** (not
  just nd builtins), or the replay-exactness claim must be explicitly scoped to pure-bundle-data
  decisions. B1's `decision.evidence` (§5) should include the external-data snapshot + version (it
  already gestures at this via DT-27 drift, but doesn't make it a replay-completeness requirement).

### X-6 (MED) — §19 "absence-of-evidence" detection needs more than B-domain currently exports
**Components:** B2 (AR-8), B5 (AR-6).
- B2-AR-8: "no Gatekeeper event ⇒ bypass" false-positives on out-of-scope / dryrun / dropped events.
  B5-R11 tries to fix this with an "expected-decision-set" export — but B5-AR-6 shows that set must be
  **time-travel** (policy state *as of each resource's creation*), not current state, or it
  false-positives on every recent promotion/exception change.
- **Reconciliation required:** B5 exports policy-state-as-of-creation-time; C4 reconstructs historical
  expectation. This is a B↔C cross-domain contract, flagged here so the cross-cutting wave catches it.

---

## 2. Non-contradictory but reinforcing findings (multiple reviewers, same direction)

- **Exceptions/approvals as permanent bypass** (B4-AR-3/AR-5, B2): need C3 analytics on
  approval-around / exception-overuse frequency, and exception-expiry must trigger a §19 re-scan of
  *existing* violating workloads (expiry ≠ remediation, B4-AR-5).
- **OCP strategic dependency** (B1-AR-1, B4 ALT-ocp): single freshly-open-sourced project under the
  differential-sim differentiator; abstraction is right but the fallback must cover regression-testing.
- **Cosign verification identity is the real trust root and is under-governed** (B1-AR-3): MUST-govern,
  specify bootstrap/distribution.
- **Conformance corpus tests clean input, not messy build-time shapes** (B1-AR-12, B3-AR-4): extend
  the corpus with multi-doc/computed-value/no-image/list cases or the central guarantee is weak.

---

## 3. Prioritized cross-component defect list (domain-level)

| Rank | ID | Sev | Cross-component defect | Owner (resolve as one slice) |
|---|---|---|---|---|
| 1 | X-3 | **CRITICAL** | Approval flow: not durable, not retry-traceable, not secure (approver-downgrade/forgeable) | B2+B4+B5 vertical slice |
| 2 | X-1 | **CRITICAL** | "One Rego everywhere" false for identity-aware build-time; input-norm contract unowned | B1+B2+B3 joint contract |
| 3 | X-2 | **CRITICAL** | Action model: linear precedence conflates disposition + obligations | B4 (before C2/E1 bake in) |
| 4 | X-4 | **HIGH** | Composed fail-closed defaults = correlated self-amplifying outage; no circuit breaker | B5 flow-level + B1/B2/B3 |
| 5 | X-5 | **HIGH** | Replay-exactness broken by uncaptured external-data (the §18.1 headline case) | B1 evidence + B5 |
| 6 | X-6 | MED | §19 expected-decision-set must be time-travel, not current | B5 → C4 (cross-domain) |
| 7 | — | MED | Exception expiry ≠ remediation of existing violations; approval/exception rubber-stamping | B4 + C3 |
| 8 | — | MED | OCP single dependency incl. regression-testing; trust-root under-governed | B1 + B4 ALT |
| 9 | — | MED | Latency budget unmodeled at scale (deploy-storm) | B5 perf model |
| 10 | — | MED | Conformance corpus tests clean input only | B1 + B3 |

---

## 4. Domain verdict

Domain B's **architecture is sound**: OPA-decides/engines-effect, deny-with-approval-required, signed
bundles, one canonical decision + one correlation_id. The five components are internally coherent and
the contract surface is well-identified.

The danger is concentrated in **four cross-component seams** that no single component owns and each
assumes another handles:
1. **The approval flow (X-3)** — three components, three partial implementations, a security hole.
2. **Input normalization (X-1)** — the conformance claim depends on it and it's a footnote.
3. **The action model (X-2)** — a v1alpha data-model error that hardens fast.
4. **Composed fail-closed (X-4)** — five local "fail safe" choices = one global outage mode.

**Recommendation:** the cross-cutting wave (and any implementation) must treat X-1 through X-5 as
**joint, cross-component work items with single owners**, resolved before C2 (audit schema) and E1
(simulation) consume the B-domain contracts — because X-2 (action model) and X-5 (replay) in
particular get dramatically more expensive to change once those downstream components bake them in.
