# B5 — Real-Time Enforcement Flow — SPEC

**Component ID:** B5 · **Domain:** B · **Spec source:** §18 (with §8/B1, §9/B2, §15/D1, §17B/§17C/B4, §11/C1, §14/C3 deps)
**Status:** DRAFT v1 · **Date:** 2026-05-30 · **Author persona:** cooperative engineering author

---

## 1. Scope

B5 owns the **end-to-end real-time enforcement sequence**: the precise, ordered flow from a user
action (authenticated request) through admission/decision to audit evidence, and the contracts
that make that flow *traceable end-to-end* via a single correlation_id. It is the **integration
spec** that stitches B1 (decision), B2 (Gatekeeper effector), D1 (identity), B4 (action/approval),
C1 (Privateer evidence), and C2/C3 (audit/analytics) into one coherent runtime path.

The canonical example (§18.1) is Kubernetes admission of an unsigned image (SC-IMG-001), but B5
generalizes the sequence to any **real-time PDP** (§17C.4: admission, application, identity, approval).

**In scope:** the admission request → decision → audit **sequence** (the precise step ordering and
who does what); the correlation_id propagation contract; the latency budget and the hard constraint
that long-running approvals must not block the admission webhook (§17B.4); the evidence-emission
contract (what evidence each step produces); the failure/degradation behavior of the *flow* (vs.
individual components); the generalization to application + identity real-time PDPs.

**Out of scope:** Rego/bundle internals (B1); Gatekeeper config internals (B2); JWT mapping internals
(D1); CRD schemas (B4); Privateer internals (C1); analytics internals (C3); retrospective/§19 (C4);
build-time flow (B3).

---

## 2. The canonical real-time sequence (normative — §18.1 made precise)

The §18.1 runtime flow lists 10 steps; B5 makes the contract precise, fixes ordering, and assigns
ownership and the correlation_id.

```
Actor (engineer)                                                                  [t0]
  │ 1. authenticate → Keycloak issues JWT                                          (D1)
  ▼
API Gateway / kubectl                                                              [t1]
  │ 2. request carries JWT; gateway/admission injects/propagates correlation_id    (B5-R1)
  ▼
Kubernetes API server                                                             [t2]
  │ 3. AdmissionReview generated (request.uid, userInfo)
  ▼
Gatekeeper (ValidatingWebhook)                                                    [t3]  budget ≤ ~2s (B5-R6)
  │ 4. D1 mapping layer resolves JWT → subject/groups/tenant for the decision      (D1, B2-R10)
  │ 5. Gatekeeper invokes embedded OPA → data.<pkg>.decision                       (B1, B2-R1)
  │ 6. decision returned: action ∈ 13-taxonomy (B4); disposition computed
  ▼
Disposition branch                                                                [t4]
  ├─ allow/warn/annotate ─► admit (warn surfaces warning; annotate via mutation)
  ├─ deny ───────────────► reject with violation reason
  ├─ require_approval ───► DENY with approval-required reason + signal PolicyApprovalRequest (B4-R14)   ← HARD CONSTRAINT
  └─ exception (in-scope, unexpired) ─► admit-with-waiver, exception use audited (B4-R8)
  ▼
Evidence emission (parallel, non-blocking)                                        [t5]
  │ 7. Gatekeeper emits admission audit event (17 fields, §9.3/B2-R10) ──► C2
  │ 8. OPA emits decision log (B1-R25) w/ same correlation_id + nd_builtin_cache ──► C2
  │ 9. Privateer records evaluation evidence (C1) keyed by correlation_id
  │ 10. Compliance analytics correlates events by correlation_id ──► C3
  ▼
Audit store / analytics (async)                                                   [t6]
```

- **B5-R1 (MUST):** A **single `correlation_id`** MUST be generated at the earliest point of the
  request (gateway or admission) and propagated through *every* downstream artifact: the OPA decision
  log (B1-R25), the Gatekeeper audit event (B2-R11, field 17), the Privateer evidence (C1), the
  analytics correlation (C3), and any PolicyApprovalRequest/exception (B4). End-to-end traceability
  (DT-13) depends on this single id; a flow that loses or re-generates it mid-stream is non-conformant.
- **B5-R2 (MUST):** The decision (step 5) MUST be made by the canonical B1 `decision` rule so the
  runtime decision is identical to what simulation/replay would compute (§17.4 — the decision a user
  hit in prod can be replayed exactly; B1-R26 nd capture makes this deterministic).
- **B5-R3 (MUST):** Steps 7–10 (evidence emission) MUST NOT be on the synchronous admission path's
  critical latency — they are emitted but their *delivery* is async/best-effort-with-buffering
  (B1-R28), so a slow audit store never delays or fails an admission decision. The *decision* (steps
  4–6) is synchronous; the *evidence* (7–10) is not.
- **B5-R4 (MUST):** The audit evidence (§18.1 "Audit Evidence") MUST include at minimum: user
  identity, JWT claims, image digest, policy version (bundle_revision), control_id, enforcement
  outcome (disposition + action), timestamp, and correlation_id. (Superset of §18.1 list + correlation_id.)

---

## 3. The hard constraint: approvals must not block admission (normative)

Restates §17B.4 / B2-R17 / B4-R6 as a flow-level invariant, because B5 is where the whole sequence's
latency budget lives.

- **B5-R5 (MUST):** No step in the synchronous admission path may block waiting for a human, an
  external workflow, or any unbounded operation. When the decision is `require_approval`, the flow
  MUST take the **deny-with-approval-required** branch (t4) and return *within the latency budget*,
  delegating durability to the PolicyApprovalRequest CRD (B4) + retry. Holding the webhook open for
  approval is a conformance failure of the *flow*.
- **B5-R6 (MUST):** The synchronous admission path (t3→t4) MUST complete within a configured
  **latency budget** (default ≤ 2s, ≤ the Kubernetes admission webhook timeout) at p99 under load.
  This budget is the sum of: D1 mapping (t3 step 4) + OPA eval (step 5) + any external-data call
  (bounded, B2-R15). If the budget is at risk, external-data calls MUST time out to the configured
  failurePolicy (B2-R14) rather than blow the deadline.
- **B5-R7 (MUST):** The approval **retry** path (a later admission once approval exists, B2-R19) is
  itself a real-time flow subject to the same budget; consulting approval state MUST be bounded
  (cached external-data, B2-AR-1) and MUST NOT itself block.

---

## 4. Generalization to other real-time PDPs (normative)

§18.1 is Kubernetes admission; the same sequence applies to other real-time PDPs (§17C.4) with the
effector swapped. B5 fixes the *shape* so all real-time PDPs are consistent and replayable.

| PDP type (§17C.4) | t1–t2 trigger | t3 effector | Real-time disposition | Replayable? |
|---|---|---|---|---|
| **Admission** | kubectl/API → AdmissionReview | Gatekeeper (+OPA) / Kyverno | allow/deny/warn/mutate/deny-approval | Yes (decision log) |
| **Application** | app API call | app calls OPA before serving | allow/deny/warn | Yes **iff** decision input logged (§17C.4) |
| **Identity** | Keycloak login/token/admin change | Keycloak hook → OPA | allow/deny (token issuance) | Yes (D1 events) |
| **Approval** | change needing approval | workflow + CRD | suspend→deny-approval / allow-on-approval | Yes |

- **B5-R8 (MUST):** Every real-time PDP MUST emit the same correlation_id-keyed evidence (B5-R1/R4)
  and MUST use the canonical B1 `decision` (B5-R2), so all real-time decisions across products are
  uniformly traceable and replayable.
- **B5-R9 (MUST):** Application PDPs (embedded OPA/Wasm, E3/§17D) MUST log the **decision input**
  they evaluated (subject to redaction, B1-R27) — otherwise the decision is not replayable (§17C.4,
  DT-12, DT-25) and the §18.1 "audit evidence" contract is incomplete for that PDP.
- **B5-R10 (SHOULD):** Identity PDPs (Keycloak token issuance) SHOULD evaluate the same governance
  controls (e.g. claim/role constraints) via OPA so identity decisions join the unified evidence
  stream (HL-16 claim evolution).

---

## 5. Flow-level failure & degradation modes

These are *flow* failures (component failures are owned by B1/B2/D1); B5 specifies how the *sequence*
degrades.

| # | Failure | Required flow behavior |
|---|---|---|
| F1 | D1 mapping unavailable (JWT not resolvable) at t3 | Decision proceeds with subject=null+reason; identity-aware controls fail-closed (deny) for runtime class; field present-with-reason (B2-R10) |
| F2 | OPA eval error / decision undefined at t5 | Disposition = fail-closed deny for runtime enforcement class (B1-F4); emit `decision.error` evidence; never silent-allow |
| F3 | External-data (e.g. signature verifier) slow at t5 | Bounded timeout → failurePolicy (B2-R14); never exceed budget (B5-R6); version captured (DT-27) |
| F4 | Evidence sink (C2) down at t5–t6 | Decision unaffected (B5-R3); evidence buffered (B1-R28); correlation_id preserved so backfill correlates later |
| F5 | Whole Gatekeeper path down | failurePolicy governs (Fail for prod runtime, Ignore for system-ns); §19/C4 detects any admissions that slipped during an Ignore window (gap created here is the §19 input) |
| F6 | Approval CRD creation hand-off lost (B4-AR-2) | Durable creation (B4-R14); user deny message includes manual-create fallback; flow does not hang |
| F7 | Clock skew between t0 (JWT iat) and t3 (decision) | Decision uses captured nd cache for replay (B1-R26); JWT exp validated by D1; skew tolerance configured |
| F8 | correlation_id missing/duplicated | Pipeline (C2) flags missing correlation (DT-28); B5 MUST ensure exactly-one id per request (B5-R1) |

---

## 6. The §19 relationship (why real-time flow defines detective gaps)

The real-time flow is what *produces* enforcement evidence; its **absence** is what §19 detects. B5
therefore MUST make the "expected evidence" explicit so C4 can compute "should there have been a
decision here?" (resolving B2-AR-8):

- **B5-R11 (MUST):** For every in-scope real-time decision, the flow MUST produce a decision-log +
  audit event with correlation_id. The *set of in-scope decisions* (what should have produced
  evidence) MUST be derivable from the active Constraints/policies + their match scope, and exported
  to C4, so that "Kubernetes audit log shows a resource but no governance decision exists" is a
  *confident* bypass signal, not a guess (§19, DT-30, HL-06). Absence-as-bypass requires this
  positive expectation contract.

---

## 7. Security / authz notes

- correlation_id is non-sensitive but integrity-relevant: if an attacker can suppress/forge it, they
  can break traceability and §19 detection. It SHOULD be generated server-side (gateway/admission),
  not trusted from the client.
- The JWT at t0–t2 is the identity basis for the whole flow; its validation (D1) and the redaction of
  its claims in evidence (B1-R27/D4) are mandatory.
- The fail-closed defaults (F1/F2/F5) are security choices; their interaction with availability is the
  cluster-bricking risk (B1-AR-6) — the system-namespace carve-out (B2-R4) is the mitigation and is
  itself a §19-monitored bypass surface.

---

## 8. Dependencies

| Depends on | For |
|---|---|
| B1 | canonical `decision`, decision log, nd capture, bundle_revision (policy version) |
| B2 | Gatekeeper admission leg, 17-field audit, failurePolicy, deny-with-approval |
| B4 | action taxonomy/disposition, PolicyApprovalRequest/Exception |
| D1 | JWT → subject/groups/tenant mapping at t3 |
| C1 | Privateer evaluation evidence keyed by correlation_id |
| C2 | audit-event normalization sink; correlation_id contract |
| C3 | analytics correlation of the emitted events |
| C4 | consumes the "expected evidence" contract for §19 bypass detection |
| E1 | replays the exact runtime decision (B5-R2) for simulation/regression |

---

## 9. Open questions (decided defaults)

| # | Question | Default | Rationale |
|---|---|---|---|
| OQ1 | Where is correlation_id minted? | **Server-side at gateway/admission, earliest point** | Integrity + single id (B5-R1, security note) |
| OQ2 | Sync vs async evidence emission? | **Decision sync; evidence async-buffered** | Never couple admission latency to audit-store uptime (B5-R3) |
| OQ3 | Latency budget? | **≤ 2s p99, ≤ admission timeout, configurable** | Admission deadline is hard (B5-R6) |
| OQ4 | Identity-aware control when JWT unresolved? | **Fail-closed deny for runtime; field-with-reason** | Security default (F1) |
| OQ5 | Application PDP replayability | **Mandatory input logging (redacted)** | Otherwise not replayable (B5-R9, §17C.4) |
| OQ6 | How does C4 know what *should* have a decision? | **Export the in-scope decision set from active policies** | Confident §19 detection, not guesswork (B5-R11) |

---

## 10. Traceability

- **Spec:** §18 (18.1 full sequence), §8 (decision), §9 (Gatekeeper/audit), §15 (identity), §17B.4
  (approval-not-blocking), §17C.4 (PDP types), §11 (Privateer), §14 (analytics), §19 (detective gap).
- **Scenarios:** DT-13 (trace decision→bundle→control), DT-28 (missing correlation_id), DT-41
  (recent denies report), DT-42 (audit correlation gap), DT-58/59 (suspend/approval at admission),
  DT-30 (bypass detection), HL-02 (image-signing end-to-end), HL-03 (2am admission incident),
  HL-06 (bypass), HL-16 (claim evolution → identity PDP).
- **Personas:** Marcus, Jess, Priya, Daniel, Sam.
