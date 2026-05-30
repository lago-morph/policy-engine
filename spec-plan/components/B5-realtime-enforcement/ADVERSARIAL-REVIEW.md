# B5 — Real-Time Enforcement Flow — ADVERSARIAL REVIEW

**Reviewer persona:** hostile principal engineer / SRE who owns the admission webhook's pager.
Mandate: attack the *flow*, especially the seams between components and the latency/availability story.

---

## A. The latency budget is optimistic and the components don't fit in it

- **AR-1 (HIGH) — A ≤2s p99 budget (B5-R6) that must contain D1 JWT mapping + OPA eval + an
  external-data signature verification call is not credible at scale.** Each of these is itself a
  network/compute hop: D1 mapping may hit Keycloak/a claim cache; external-data signature verification
  may hit cosign/Rekor or an OCI registry (B1-AR-5); OPA eval is fast but not free. Summing three
  bounded timeouts to stay under 2s means each gets <700ms, which is tight for a cold cache or a
  cross-region call. And admission webhooks block the *entire* API operation — every Deployment,
  every CI-driven apply, every controller reconcile that touches an in-scope resource pays this. Under
  a deploy storm (e.g. a GitOps sync of 500 manifests), the budget compounds into API-server
  saturation. The SPEC treats the budget as a config knob; it's actually the platform's central
  scaling risk and needs a real performance model, caching strategy, and a load-shedding plan, not a number.

- **AR-2 (HIGH) — "External-data times out to failurePolicy" (B5-R6/F3) means a degraded verifier
  causes fleet-wide mass-deny (this is B2-AR-5 at the flow level, and it's worse here).** B5 makes it
  a *flow invariant* that external-data must not blow the budget, so it times out fast → falls to
  failurePolicy=Fail (prod) → every signed-image admission across the cluster denies *simultaneously*
  the instant the verifier degrades. The SPEC has no circuit breaker at the flow level. A single
  dependency (the image-signature verifier) becomes a cluster-wide admission kill switch. This needs
  a flow-level degraded mode (drop to warn + loud alert + §19 catch-up), not just "time out to Fail."

## B. correlation_id as a single point of traceability

- **AR-3 (MED) — "One correlation_id minted server-side, propagated through every artifact" (B5-R1)
  is the right goal and the most fragile contract in the platform.** It must survive: gateway →
  API-server → Gatekeeper → embedded OPA → decision log → audit event → Privateer → analytics → CRD.
  That's ~8 hops across 5 components owned by 5 teams, several of which are off-the-shelf (Gatekeeper,
  OPA) and don't natively thread a custom id through their internals. How exactly does the
  correlation_id get *into* the embedded OPA decision log when Gatekeeper calls OPA? Via input
  injection? Then it's part of `input` and pollutes the decision (and replay). The SPEC asserts the
  propagation but never specifies the *mechanism* at each hop, and at least two hops (into Gatekeeper's
  audit, into embedded OPA's decision log) are non-obvious or may require forking/patching upstream.
  Without a per-hop mechanism, B5-R1 is aspirational.

- **AR-4 (MED) — Retry breaks correlation_id continuity for the approval flow.** A denied-with-approval
  admission has correlation_id X. The user re-runs `kubectl apply` later — that's a *new* AdmissionReview
  with a *new* server-minted correlation_id Y (B5-R1 says mint at earliest point, server-side). So the
  approved request (keyed to X via B4) and the retry (Y) don't share a correlation_id, and the
  end-to-end trace fragments exactly across the most important governance event (an approved
  production deploy). The SPEC's correlation contract and B4's approval-consumption identity are in
  tension; one of them has to give.

## C. Replayability claims

- **AR-5 (HIGH) — "The decision a user hit in prod can be replayed exactly" (B5-R2/R7, M7) is the
  platform's flagship claim and the flow undermines it.** Exact replay requires capturing *all* inputs
  to the decision, including external-data (the signature-verification result) and the D1-mapped
  identity *as they were at t3*. But external-data is, by definition, fetched live and not part of the
  bundle; the nd_builtin_cache (B1-R26) captures nondeterministic *builtins*, not external-data
  document state. If the verifier said "signed" at t3 and "unsigned" at replay time (key rotated, image
  GC'd), replay diverges — and the SPEC's replay story doesn't capture the external-data snapshot in
  the decision log. So "replay exactly" holds only for pure-bundle-data decisions, not for the
  signature-verification decisions that are the headline §18.1 example. The SPEC must capture the
  external-data *values used* into the decision evidence, or scope the replay-exactness claim down.

## D. The §19 expectation contract

- **AR-6 (MED) — "Export the in-scope decision set so C4 can confidently detect bypass" (B5-R11) is a
  moving target that's stale the moment it's computed.** The set of "decisions that should have
  happened" depends on the active Constraints/policies *at the time each resource was created* — but
  policies change (promotions, exceptions, rollbacks). A resource created during a dryrun→deny
  transition, or while an exception was active-then-expired, legitimately has no enforce-mode decision.
  Computing "should there have been a decision" requires reconstructing the policy state *as of each
  resource's creation time*, not the current state. The SPEC exports a current expected-set; C4 needs a
  time-travel expected-set. Without it, §19 either false-positives on every recent policy change or
  misses real bypasses. This is harder than the one-line R11 implies.

## E. Fail-closed flow vs. availability

- **AR-7 (HIGH) — The composed fail-closed defaults create a correlated, self-amplifying outage mode.**
  F1 (JWT unresolved → deny), F2 (decision undefined → deny), F3 (external-data slow → Fail), F5
  (Gatekeeper down → Fail for prod) are individually defensible. Composed, they mean: a single shared
  dependency (Keycloak, the bundle server, the verifier) degrading causes the flow to deny broadly,
  which can prevent the *recovery* of that very dependency if it (or its dependencies) run in-cluster
  in a non-carved-out namespace. The SPEC defers this to the B2 system-namespace carve-out, but B5 is
  where the *composition* happens and B5 doesn't model the blast radius of correlated fail-closed.
  Need an explicit flow-level "infrastructure-degraded" mode distinct from "policy says no."

- **AR-8 (MED) — Identity-aware control fail-closed on JWT-unresolved (F1/OQ4) will deny legitimate
  system/service-account traffic.** Many in-cluster requests come from service accounts, not Keycloak
  JWTs (controllers, kubelet, operators). If an identity-aware control denies when it can't resolve a
  *human* JWT, it'll deny every service-account-driven operation that hits an in-scope resource. The
  SPEC's "subject=null+reason, identity-aware controls fail-closed" will brick controller reconciles.
  Identity-aware controls must distinguish "no identity (system)" from "human identity unresolvable
  (suspicious)" — the flow doesn't.

## F. Generalization

- **AR-9 (MED) — App-PDP replayability "iff decision input is logged" (B5-R9) makes replayability
  opt-in and therefore usually-absent.** Application teams embedding OPA/Wasm (E3) will not reliably
  log full decision input (perf, PII, effort). So the unified-replay claim degrades to "replayable for
  the components we control (admission), best-effort elsewhere." The SPEC should be honest that
  cross-product *replay* is strong for admission and weak for app PDPs, rather than implying uniformity.

- **AR-10 (LOW) — Identity-PDP via Keycloak (B5-R10) is SHOULD, so it won't happen, leaving identity
  decisions outside the unified evidence stream** — which is exactly the gap §17A/§15 care about. A
  SHOULD on the one PDP type that ties identity into governance means the identity governance story is
  optional and thus absent in practice.

---

## Prioritized defect list

| ID | Sev | Defect | Required resolution |
|---|---|---|---|
| AR-1 | **HIGH** | ≤2s budget containing D1+OPA+external-data isn't credible at scale; admission storms saturate API server | Real perf model + caching + load-shedding; per-hop sub-budgets; measure under deploy-storm |
| AR-2 | **HIGH** | External-data degradation → fleet-wide mass-deny (no flow circuit breaker) | Flow-level degraded mode: drop to warn + alert + §19 catch-up, not time-out-to-Fail |
| AR-5 | **HIGH** | "Replay exactly" fails for external-data decisions (the headline example) | Capture external-data values used into decision evidence; or scope the exactness claim |
| AR-7 | **HIGH** | Composed fail-closed defaults = correlated self-amplifying outage | Explicit flow-level "infra-degraded" mode distinct from "policy deny"; model blast radius |
| AR-3 | MED | correlation_id per-hop propagation mechanism unspecified (esp. into GK/embedded OPA) | Specify the mechanism at each of the 8 hops; verify upstream supports it |
| AR-4 | MED | Approval retry mints a new correlation_id → trace fragments across approved deploys | Reconcile B5-R1 (mint per request) with B4 approval identity; thread a stable governance-tx id |
| AR-6 | MED | §19 expected-set must be time-travel, not current | Export policy-state-as-of-creation-time; C4 reconstructs historical expectation |
| AR-8 | MED | Identity-aware fail-closed denies service-account/system traffic | Distinguish "no identity (system)" from "human identity unresolvable" |
| AR-9 | MED | App-PDP replay is opt-in → usually absent | Be honest: replay strong for admission, weak for app PDPs; or make input-logging mandatory+enforced |
| AR-10 | LOW | Identity-PDP integration is SHOULD → identity governance optional | Make identity-PDP integration MUST if §17A's identity-governance claims are to hold |

**Verdict:** B5 correctly identifies the load-bearing invariant (decision sync / evidence async /
approvals never block) and the correlation_id contract — these are the right bones. But it
**under-models the two things that actually decide production viability: latency/scale (AR-1) and
correlated fail-closed blast radius (AR-2/AR-7).** And the flagship "replay exactly" claim (AR-5) is
false for the very signature-verification example §18.1 leads with, because external-data state isn't
captured. Fix AR-1/AR-2/AR-7 (the outage modes) and AR-5 (the replay honesty) before this flow meets a
real cluster under load.
