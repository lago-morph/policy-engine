# B2 — Gatekeeper Integration — ADVERSARIAL REVIEW

**Reviewer persona:** hostile principal engineer, ex-platform-SRE, has been paged at 2am by a
Gatekeeper webhook taking down a cluster. Mandate: find what breaks.

---

## A. The admission/approval reconciliation (the marquee feature) is the riskiest part

- **AR-1 (HIGH) — "Read approval state via external-data at admission" reintroduces the exact
  availability and latency problem the deny-with-CRD pattern was supposed to avoid.** R-B2-19
  says retry "consults approval state via external-data or cached PolicyApprovalRequest status."
  External-data on the admission hot path means: (a) every retry of an approval-gated resource
  makes a synchronous call within the admission deadline, and (b) if that provider is slow/down,
  the constraint falls back to failurePolicy — which for prod is **Fail**, so a healthy, already-
  approved deployment gets **denied** because the approval-lookup service blipped. The SPEC waves
  at "cached" but never specifies cache freshness vs. a just-granted approval. Stale cache → user
  approved, retry still denied, user files a ticket at 2am. Specify the cache coherence model and
  what happens when approval state and cache disagree, or this feature generates support load forever.

- **AR-2 (HIGH) — Who creates the PolicyApprovalRequest, and is it guaranteed?** R-B2-18 says the
  webhook "emits an event the controller acts on" rather than creating the CRD synchronously
  (to stay within deadline). But admission webhooks are *stateless request/response* — Gatekeeper
  does not natively "emit an event for a controller." If the deny happens but the CRD is never
  created (because the event-emission path is lossy, or no controller is watching, or the user
  never retries), the approval simply never exists and the user is permanently stuck with an
  opaque deny. There is no durable hand-off specified. **This is a correctness gap:** the deny and
  the CRD creation must be atomic-ish or reconciled, and right now they're decoupled with no
  guarantee. The realistic implementation needs a controller that watches *denied admission events*
  or the user's client must create the CRD — neither is specified.

- **AR-3 (MED) — Retry semantics are undefined.** "Allow a later retry once approval exists" — but
  who retries, and how does the *same* request get re-submitted? A `kubectl apply` retry creates a
  new AdmissionReview with a new UID; the correlation between the original denied request and the
  approval is via control_id + resourceRef + subject, which is fuzzy (what if two people request
  the same Deployment name?). The SPEC needs an explicit identity for "the thing being approved"
  and a TTL/idempotency story, or approvals get applied to the wrong retry.

## B. Self-DoS and failurePolicy

- **AR-4 (HIGH) — The system-namespace carve-out is necessary but insufficient, and it's a bypass
  vector.** R-B2-4/14 excludes `kube-system`, `gatekeeper-system`, `governance-system` to avoid
  bricking. Good. But: (1) an attacker who can deploy into an excluded namespace bypasses ALL
  governance — the carve-out is a hole, and §19 detection (C4) is the only backstop, which is
  *detective, after the fact*. (2) The carve-out list is static; real clusters have more critical
  system namespaces (CNI, CSI, monitoring, ingress) that, if denied during cold start, also brick
  the cluster. The SPEC gives three namespaces; production needs a curated, environment-specific
  list, and getting it wrong either bricks or holes. This deserves its own hardening doc, not a
  one-line `excludedNamespaces`.

- **AR-5 (MED) — failurePolicy=Fail + a slow external-data provider = correlated cluster-wide
  outage.** If image-signature verification (external-data) degrades, every signed-image
  constraint with failurePolicy=Fail starts denying every deployment cluster-wide simultaneously.
  This is not a single-resource failure; it's a fleet-wide admission outage triggered by one
  dependency. The SPEC's "bounded timeout → fall back to failurePolicy" makes this *worse* (fast
  failure → fast mass-deny). Need a circuit breaker that, on systemic external-data failure,
  degrades to dryrun/warn (with loud alerting + a §19 catch-up scan) rather than mass-deny.

## C. Conformance claim

- **AR-6 (HIGH) — The template adapter quietly breaks the "identical Rego" story (echoes B1 AR-2).**
  The ConstraintTemplate in §4.1 wraps `data.<pkg>.decision` in a `violation[...]` rule that reads
  `data.governance...decision` — but in Gatekeeper, the decision package reads `input.review.object`,
  while the *same* package run in Conftest reads a bare `input`. So either the B1 package is written
  for Gatekeeper's input envelope (and Conftest must wrap its input to match), or vice versa. The
  SPEC asserts conformance (R-B2-1) but the input-shape divergence means the "thin adapter" is doing
  real semantic plumbing. Until B1/B2/B3 jointly specify ONE normalized input contract, the
  conformance suite is testing engine-specific glue, not identical policy. **Cross-cutting blocker.**

- **AR-7 (MED) — Gatekeeper's `violation` aggregation can disagree with B1's `decision`.** B1's
  `decision.action` is singular (one action per decision); Gatekeeper produces a *set* of violations.
  If a resource triggers two sub-rules — one `deny`, one `require_approval` — what's the resolved
  action? The SPEC doesn't define action-precedence when multiple controls fire on one admission.
  deny-vs-approval ordering matters: should a hard deny override an approval path, or block the
  approval? Undefined → inconsistent behavior across resources.

## D. Detective mode & §19

- **AR-8 (MED) — "No Gatekeeper event ⇒ bypass" (R-B2-8, F5) has a false-positive problem.** Absence
  of an event is also produced by: a resource out of any Constraint's match scope, a Constraint in
  dryrun, the audit controller being behind, or the event simply being dropped by C2's pipeline.
  Treating absence as bypass (DT-30) will cry wolf. C4 needs positive confirmation (Kubernetes audit
  log shows the resource AND it's in-scope of an enforce-mode constraint AND no decision exists), and
  B2 must give C4 enough scope metadata to compute "should there have been an event?" — which the SPEC
  doesn't enumerate. Without it, §19 is noisy and gets muted, defeating its purpose.

- **AR-9 (MED) — Admission↔audit reconciliation (R-B2-9) assumes both evaluate the same logic, but
  they evaluate different *inputs*.** Admission sees the request object; audit sees the stored object
  after mutations/defaulting/other controllers. A Deployment that passed admission can legitimately
  differ at audit time (a mutating webhook or Kyverno changed it). "Divergence = discrepancy" will
  flag legitimate mutation as a violation. Reconciliation must account for post-admission mutation
  or it's a false-positive engine.

## E. Operational / scope

- **AR-10 (MED) — Generated-only templates (R-B2-3, no hand edits) collides with operational
  reality.** During a 2am incident (HL-03), the on-call engineer's fastest fix is often editing a
  Constraint/Template in-cluster. R-B2-3 says any hand-edit fails the lifecycle gate and is flagged
  as drift. So either incident response is blocked by the pipeline, or drift detection screams during
  every incident. Need an explicit break-glass that allows a tracked, time-boxed manual edit with
  mandatory reconciliation-back-to-bundle afterward.

- **AR-11 (LOW) — Coexistence with cloud-managed Gatekeeper (D-B2-04) is mentioned but the conflict
  modes aren't enumerated.** Two Gatekeeper installs in one cluster fight over the same CRDs
  (ConstraintTemplate is cluster-scoped, single CRD). You cannot run two Gatekeepers with conflicting
  template definitions. "Namespace-scope the constraints" doesn't solve a cluster-scoped CRD collision.
  This may be flatly impossible in some managed environments — needs a real compatibility matrix.

- **AR-12 (LOW) — `generateName` resources have no name at admission (field #5).** The 17-field
  contract requires Resource Name, but at admission a resource using `generateName` has none yet.
  The SPEC doesn't say what to log. Minor, but it's a "required field" that's literally unavailable.

---

## Prioritized defect list

| ID | Sev | Defect | Required resolution |
|---|---|---|---|
| AR-2 | **HIGH** | Deny↔CRD-creation hand-off has no durability guarantee | Specify a durable, reconciled approval-request creation (watch denied events or client-creates); make it not-lossy |
| AR-1 | **HIGH** | External-data approval lookup re-imports availability/latency risk + stale cache | Define cache coherence; degraded-lookup must not deny already-approved resources |
| AR-4 | **HIGH** | System-ns carve-out is both a bypass hole and an incomplete brick-guard | Curated per-env critical-ns list; pair with strong §19 detection on excluded ns |
| AR-6 | **HIGH** | Input-shape divergence breaks the "identical Rego" conformance claim | Cross-cut: ONE normalized input contract across B1/B2/B3 |
| AR-5 | MED | failurePolicy=Fail + slow external-data = fleet-wide mass-deny | Circuit breaker → degrade to warn on systemic external-data failure + §19 catch-up |
| AR-7 | MED | Action precedence undefined when multiple controls fire | Define deny>approval>warn>... precedence on one admission |
| AR-8 | MED | "Absence ⇒ bypass" false positives | B2 emits scope metadata so C4 computes "should there have been an event?" |
| AR-9 | MED | Admission↔audit reconciliation flags legitimate post-admission mutation | Account for mutation between admission and audit |
| AR-10 | MED | Generated-only templates block incident hot-fixes | Tracked, time-boxed break-glass with mandatory reconcile-back |
| AR-3 | MED | Retry identity/idempotency for approvals undefined | Explicit approved-thing identity + TTL/idempotency |
| AR-11 | LOW | Coexistence with managed Gatekeeper has cluster-scoped CRD collision | Real compatibility matrix; may be impossible in some managed envs |
| AR-12 | LOW | generateName → no Resource Name at admission | Define logging (generateName + post-create reconciliation) |

**Verdict:** §9.3's 17-field audit contract and the four-mode FSM are strong and implementable.
The approval-at-admission machinery (the platform's headline §17B.4 capability) is **under-durable
and under-specified** (AR-1/2/3) and is where this will hurt in production. And the conformance
claim (AR-6) cannot be honestly made until the input-normalization contract is settled cross-domain.
Fix AR-2 and AR-6 first.
