# F1 — API Requirements — ADVERSARIAL REVIEW

**Reviewer persona:** Red-team API security engineer + integration partner skeptic. Mandate: break the contract before production does.

---

## 1. Headline finding

The §21.1 seed list (8 endpoints) is so thin it hides the real risk: **the API is asserting it can enforce the §17A "storage and GUI" dual-authorization requirement, but the spec puts authorization in D2 (storage) and out-of-scope-for-POC storage (§26.1).** If storage authz is "out of scope for the POC," then **R-F1-AUTHZ-3 (push predicate into storage query) has no backend to push into during the POC** — and the fallback is exactly the GUI/API-only authorization §17A.1 calls "insufficient." This is the central contradiction the SPEC papers over. **DEFECT-1 (critical).**

## 2. Defect list (prioritized)

**DEFECT-1 (critical) — Authz has no enforcement substrate in POC.** §26.1 defers storage; §17A.1 forbids API-only authz. F1 needs D2 to define an in-process scope-predicate library usable even over "ordinary relational/document storage" (§22.2). Resolution: require D2 to ship the predicate as a library F1 links, with row/document-level scope columns mandatory even in the POC store. Escalate to Domain D and cross-cut adversarial.

**DEFECT-2 (high) — `/audit/events` is a data-exfil and DoS vector.** Even with bounded windows, a namespace-scoped user querying `decision=deny` over 30 days × 500k/day can pull other-scope reasoning via `outcome_reason`/`request_object`. The scope predicate MUST apply to nested fields too, and `request_object` (which F4 says carries full agent context incl. prompts) must be field-level redactable by scope. SPEC §6/§9 mention bounded windows but not field-level scope on the event body. **Fix:** field-level projection enforced by authz, not just row-level.

**DEFECT-3 (high) — Idempotency on approval decisions is underspecified.** R-F1-IDEM-1 covers job submit, but `POST /approvals/{id}:decide` is the dangerous one: a replayed approve after a reject (or vice versa) is a security event. Needs optimistic concurrency (`If-Match` ETag on approval state) in addition to idempotency-key, and an explicit "already-decided" 409.

**DEFECT-4 (high) — Filter ∩ scope claims "filter can never widen scope" but cursor signing is hand-waved.** If cursors encode scope and are merely "signed," a stolen cursor from a higher-scope user widens scope. Cursors MUST be bound to the subject (`sub`/session), not just signed, and expire.

**DEFECT-5 (medium) — 404-vs-403 leakage via timing/latency.** Returning 404 for cross-tenant is good, but a control that exists is slower to "not find" than one that never existed. Timing oracle. Acknowledge as accepted-risk for POC or constant-time the negative path.

**DEFECT-6 (medium) — `partial` simulation results are a correctness trap.** R-F1-JOB-2 returns `partial` + `replay_completeness`, but consumers (the console, GRC export per the positioning memo) may treat a partial differential as authoritative ("would have denied 42 more") when the dataset was truncated. The API MUST refuse to let `partial` results be exported as compliance evidence (or watermark them). Ties to F4's explicit "replay-completeness for exact model outputs is best-effort" caveat — same trap, worse for agents.

**DEFECT-7 (medium) — No write path for controls, but promotion needs an approval_ref that comes from where?** `promote(target_mode=enforce)` takes optional `approval_ref`, but there's no specified gate forcing it. If enforce-promotion can proceed without an approval in scopes that require one, the §17B approval-gated model is bypassable at the API. Make `approval_ref` conditionally required per policy-domain config.

**DEFECT-8 (low) — Versioning story is single-version optimism.** `/v1` only; additive-only is fine until C2's audit schema gains F4 fields (`evaluator_results`, trace context). That IS additive, but consumers that pinned strict schema validation will break. Need a documented "audit event is open/extensible" contract, flagged now.

**DEFECT-9 (low) — Rate limiting protects the wrong thing.** R-F1-RATE-1 rate-limits per subject, but the real scarce resource is the simulation/report job queue (a single 30-day full-cluster replay can starve it). Need per-queue admission control + cost-based quotas, not just per-subject request rate.

**DEFECT-10 (low) — `/whoami` over-discloses.** Returning the full resolved subject (all namespaces/domains/tenants) to the browser is a recon aid if the token is later stolen; consider returning only what the UI needs.

## 3. Inconsistencies vs other components

- **vs D2:** SPEC assumes a storage predicate library D2 may not have planned for POC (DEFECT-1).
- **vs C2:** API returns §13 schema verbatim, but C2 + F4 mutate that schema; strict-validation clients break (DEFECT-8).
- **vs E1:** async `partial` semantics must match E1's simulation completeness model exactly, or the console shows contradictory states.
- **vs F4:** agent `request_object` carries prompts/context — far more sensitive than K8s objects; field-level scope (DEFECT-2) becomes mandatory, not optional.

## 4. "Won't survive production because…"

…the spec treats authorization as a property of the API while deferring the storage layer that is supposed to be the real enforcement point. In production a single missed scope predicate on one nested field of `/audit/events` leaks cross-tenant policy reasoning, and the POC's "ordinary storage is acceptable" stance means nobody built row+field-level scope until it was a breach.

## 5. Top fixes to merge into SPEC

1. Require D2 to ship a linkable scope-predicate library covering **row and field** level, mandatory in POC store (DEFECT-1, 2).
2. Subject-bound, expiring cursors; ETag/`If-Match` on approval decisions (DEFECT-3, 4).
3. Watermark/forbid `partial` simulation results as exportable evidence (DEFECT-6).
4. Conditionally-required `approval_ref` on enforce-promotion (DEFECT-7).
5. Declare the audit event schema explicitly extensible; cost-based job-queue admission (DEFECT-8, 9).
