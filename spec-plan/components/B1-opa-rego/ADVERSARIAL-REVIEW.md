# B1 — OPA / Rego Integration & Signed Bundles — ADVERSARIAL REVIEW

**Reviewer persona:** hostile principal engineer who has shipped OPA at scale and seen it
break. Mandate: attack assumptions, find what will not survive production. Pull no punches.

---

## A. Existential / strategic

- **AR-1 (HIGH) — OCP bet is a single point of strategic failure.** The SPEC leans on OPA
  Control Plane (D-B1-01) which was open-sourced *weeks/months ago* after Apple acqui-hired
  Styra. Freshly-dumped commercial code with the originating team gone has a real chance of
  bit-rot, breaking changes, or de-facto abandonment. "Wrap it behind BundleService" is the
  right instinct, but the SPEC then quietly assumes OCP for **regression testing against
  historical decision logs** (R32/M6) — a non-trivial capability that the fallback `opa build`
  path does NOT provide. If OCP slips, the differential-simulation story (E1/DT-49) and the
  promotion gate (A2) lose their engine. **Demand:** a costed answer to "what if OCP is not
  production-ready?" The fallback must cover regression testing, or the abstraction is a fig leaf.

- **AR-2 (MED) — "Shared semantics across all engines" is asserted, not guaranteed.** The whole
  platform thesis (§17C) is that one Rego rule means the same thing in Gatekeeper, Conftest,
  app PDP, and replay. But Gatekeeper wraps OPA with its own `violation` convention and input
  shape (`input.review.object`), Conftest uses `deny`/`warn`/`violation` over a *different*
  input (raw parsed file, no `review` envelope), and app PDPs invent their own input. The SPEC's
  fix (a canonical `decision` rule, R8) is sound **only if the `input` is normalized identically**
  — which it is **not** across these engines. The same package literally cannot read
  `input.review.object` in Gatekeeper and `input` in Conftest. **This is under-specified and is
  the biggest correctness hole.** Either (a) packages must take normalized input via an adapter
  layer per engine, or (b) "shared semantics" is marketing. The conformance suite (R30) will
  pass only if it feeds each engine *engine-shaped* input, which means the Rego is NOT identical
  — it has engine-specific input plumbing. Make this explicit or it explodes at integration.

## B. Security

- **AR-3 (HIGH) — Cosign verification identity is the actual trust root and it's hand-waved.**
  R19 says "verify before activate," R20 says "configure the allowed signer identity per
  environment." But *who* governs that config, and how is it distributed to thousands of agents
  without becoming the soft underbelly? If an attacker can edit an agent's `--certificate-identity`
  flag or point it at a malicious Fulcio/Rekor, signing is theater. The SPEC says it "SHOULD be
  governance-controlled" — SHOULD is not good enough for the platform's root of trust. Make it MUST,
  and specify the bootstrap (how does agent #1 trust the verification policy?).

- **AR-4 (MED) — Decision-log redaction at source is necessary but the SPEC under-specifies the
  failure mode.** R27 redacts secrets/PII "at source," but if a *new* policy reads a new sensitive
  input field that the redaction config doesn't know about, the raw value leaks into the log before
  anyone notices. Redaction is a denylist by default → leaks by omission. Needs an allowlist posture
  for `input` logging on sensitive packages, or field-level tainting driven by `required_claims`.

- **AR-5 (MED) — Keyless cosign depends on public good infrastructure (Fulcio/Rekor) being up and
  trustworthy at *activation* time.** If the agent must reach Rekor to verify on every bundle load,
  that's a new availability dependency on the enforcement hot path. The SPEC offers KMS as "optional"
  but doesn't say activation verification can be done offline against cached roots. Specify offline
  verification or you've coupled enforcement availability to sigstore uptime.

## C. Operational / correctness

- **AR-6 (HIGH) — Fail-closed cold start (F3) will cause an outage the first time it fires.** "No
  bundle → deny for runtime class" is the *security*-correct default and the *availability*-wrong
  one. The first time the bundle server is down during a cluster cold-start / mass pod reschedule,
  every admission gets denied and the cluster can't recover (can't even schedule the bundle server's
  own pods if it's in-cluster). This is a classic policy-engine self-DoS. The SPEC needs a documented
  break-glass and an explicit carve-out for system/kube-system and the policy stack's own namespace,
  or it bricks clusters. Gatekeeper's `failurePolicy` interplay (B2) must be reconciled here, not deferred.

- **AR-7 (MED) — nd_builtin_cache capture (R26) is fragile and not free.** Capturing every
  non-deterministic builtin for replay is correct in theory, but (a) it bloats decision logs, (b)
  it only works if authors actually route nondeterminism through capturable builtins — a policy
  that derives "now" indirectly (e.g. compares two timestamps from input) won't be captured, and
  replay silently diverges. The SPEC's R10 (no http.send) helps but doesn't close the timestamp
  case. "Deterministic replay" is a headline feature (§17.4) being propped up by an honor system.

- **AR-8 (MED) — Per-domain bundles with explicit roots (OQ7/R12) creates a composition problem
  the SPEC doesn't address.** If bundle A owns `governance.kubernetes.*` and a shared helper lives
  in `governance.lib.*`, who owns `lib`? If every bundle vendors its own copy of `lib`, you get
  version skew (bundle A on lib v1, bundle B on lib v2) and the "shared semantics" claim dies *within
  OPA itself*. If one bundle owns `lib`, you've reintroduced a mega-dependency and cross-bundle
  ordering. Needs an explicit shared-library distribution + versioning story.

- **AR-9 (LOW/MED) — `allowed := action in {allow,warn,annotate,notify}` (R8/§5) is a footgun.**
  Collapsing 13 actions into a boolean loses information at exactly the boundary where it matters.
  `require_approval` is neither allowed nor denied — it's *suspended*. Treating it as `allowed:false`
  (deny) vs `allowed:true` will produce wrong behavior in some caller that only reads the boolean.
  The derived boolean should not exist, or `require_approval`/`require_async_check` must be a third state.

## D. Process / scope

- **AR-10 (MED) — 80% decision-branch coverage floor (R31) is a vanity metric.** Branch coverage
  on Rego does not catch the dangerous case: a policy that's *silently undefined* for an input it
  should deny (undefined → not deny → allow). The real test is adversarial input fuzzing + the
  golden corpus, not a coverage percentage. Coverage gives false confidence.

- **AR-11 (LOW) — Dual metadata (Go vars R1 + METADATA blocks R4.2) is redundant and will drift.**
  Requiring authors to write `__control_id__` AND `custom.control_id` and keeping them in sync via a
  build check is busywork that will generate friction and PR churn. Pick the METADATA block (it's the
  OPA-native, tool-discoverable one) and *generate* the Go var if some legacy consumer truly needs it.

- **AR-12 (LOW) — Conformance corpus maintenance burden is unbudgeted.** R30's four-engine corpus is
  the right idea but every new package and every engine upgrade must extend it. Without an owner and a
  generation strategy (e.g. auto-derive cases from unit tests), it rots and the central guarantee erodes.

---

## Prioritized defect list

| ID | Sev | Defect | Required resolution before GA |
|---|---|---|---|
| AR-2 | **HIGH** | "Identical Rego across engines" ignores divergent `input` shapes | Specify per-engine input adapters; redefine "shared semantics" as shared *decision logic over normalized input* |
| AR-6 | **HIGH** | Fail-closed cold start can brick a cluster | Break-glass + system-namespace carve-out; reconcile with B2 failurePolicy |
| AR-1 | **HIGH** | OCP single strategic dependency incl. regression testing | Costed fallback covering regression testing; do not assume OCP for the differential-sim engine |
| AR-3 | **HIGH** | Cosign verification identity (trust root) only SHOULD-governed | MUST-govern verification policy; specify bootstrap/distribution |
| AR-4 | MED | Redaction denylist leaks by omission | Allowlist `input` logging on sensitive packages |
| AR-5 | MED | Keyless verify couples enforcement to sigstore uptime | Offline verification against cached roots |
| AR-7 | MED | Replay determinism relies on author discipline | Static check for indirect nondeterminism; mark non-replayable decisions |
| AR-8 | MED | Shared-library versioning across per-domain bundles | Explicit lib distribution + version-pin story |
| AR-9 | MED | `allowed` boolean mis-collapses require_approval | Three-state or drop the boolean |
| AR-10 | MED | Coverage floor is a vanity metric | Replace/augment with adversarial fuzz + golden corpus as the gate |
| AR-11 | LOW | Dual metadata drifts | Single source of truth (METADATA), generate the rest |
| AR-12 | LOW | Conformance corpus rots | Assign owner; auto-derive cases from unit tests |

**Verdict:** The signing/versioning/distribution spine is solid and well-aligned to the
post-Styra market reality. The two things that will *not* survive contact with production are
**AR-2** (the cross-engine semantics claim is over-stated given divergent input shapes) and
**AR-6** (fail-closed cold start as a cluster-bricking footgun). Fix those two before anyone
believes the "one Rego everywhere" pitch.
