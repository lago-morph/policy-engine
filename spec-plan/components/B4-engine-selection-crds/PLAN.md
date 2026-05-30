# B4 — Engine Selection, Action Taxonomy & CRDs — PLAN

**Component:** B4 · **Pairs with:** SPEC.md, ALT-ocp-substrate.md, ALT-kyverno-first.md · **Date:** 2026-05-30

---

## 1. Dependency DAG

```
[B1 decision contract] ──► W1 Action taxonomy (closed 13) + precedence ──► (consumed by B1/B2/B3/C2)
[A1 control catalog] ────► W2 Engine-selection rubric (scored) ──► [A2 lifecycle metadata]
                                          │
                                          ▼
[D1/D2 identity+RBAC] ──► W3 CRD schemas (6) ──► W4 Controllers (reconcile loops)
                                          │                 │
                          ┌───────────────┼─────────────────┼───────────────┐
                          ▼               ▼                 ▼               ▼
                W4a ApprovalRequest   W4b Exception   W4c SimulationRun   W4d Remediation/
                (load-bearing)        (waiver)        (→E1)               ActionLib/EvidenceSchema
                          │
                          ▼
                [B2 deny-with-approval handshake]  (the critical integration)
```

Critical path: **W1 → W3 → W4a (PolicyApprovalRequest controller) → B2 handshake.** The approval
flow is the riskiest, highest-value integration in the whole domain.

---

## 2. Parallelizable workstreams

| WS | Title | Deps | Parallel with |
|---|---|---|---|
| W1 | Closed 13-action enum + precedence (R-B4-5/7) | B1 decision contract | W2 |
| W2 | Scored engine-selection rubric + per-control recording (R-B4-1/2) | A1 stub | W1 |
| W3 | OpenAPI schemas for all 6 CRDs (R-B4-22) | W1 | — |
| W4a | PolicyApprovalRequest controller: reconcile + webhook + durable creation (R-B4-11/14) | W3, D2, B2 | W4b/c/d |
| W4b | PolicyException controller: bounded/scoped, usage counting (R-B4-15/17) | W3, D2 | W4a |
| W4c | PolicySimulationRun controller → drives E1 (R-B4-18/19) | W3, E1 | W4a |
| W4d | Remediation + ActionLibrary + EvidenceSchema (R-B4-20/21) | W3 | W4a |
| W5 | PDP profile schema (§17C.5) + classification (R-B4-9/10) | W2 | W4 |

The four controllers (W4a–d) are independent and parallelizable once schemas (W3) are frozen.

---

## 3. Critical path & milestones

- **M1 — Taxonomy frozen (W1):** the closed 13-action enum + precedence published; every other
  component (B1/B2/B3/C2/E1) consumes it. Unblocks the whole domain's `action` field.
- **M2 — Rubric usable (W2):** a control can be scored → engine(s) selected → justification recorded.
  Demo on SC-IMG-001 (OPA decides, Gatekeeper/Kyverno effects). (HL-14.)
- **M3 — CRD schemas applied (W3):** all six CRDs install; validation rejects unbounded exception,
  ad-hoc action, missing controlId.
- **M4 — Approval flow end-to-end (W4a + B2):** denied admission → durable CRD creation →
  webhook → approve (RBAC-gated, no self-approval) → single-use consumption on retry → audited.
  **Keystone milestone.** Demo HL-10 break-glass, DT-58/59.
- **M5 — Exception flow (W4b):** scoped+bounded exception converts would-deny→exception; usage
  counted; expiry re-enforces. (HL-19, DT-03.)
- **M6 — Simulation + remediation (W4c/d):** PolicySimulationRun drives E1 differential (DT-49);
  remediation defaults dry-run.
- **M7 — PDP profiles (W5):** each integrated product declares its §17C.5 profile; replay-incapable flagged.

---

## 4. Test strategy

1. **Taxonomy:** closed-enum enforcement (ad-hoc action rejected, F8); precedence resolution for
   every conflicting pair (deny>approval>quarantine>mutate>...>allow, R-B4-7); `exception` only
   converts same-scope deny.
2. **Rubric:** golden set of controls → expected engine selection; disqualification on a required-
   capability zero; selection recorded + auditable.
3. **Approval (critical, M4):**
   - durable creation: simulate lost fire-and-forget → controller still creates from durable queue (F2);
   - webhook delivery failure → retry, no auto-approve, manual subresource path works (F1);
   - RBAC: only required role approves; self-approval rejected (F5/R-B4-12);
   - single-use: two requests can't share one approval; double-consume race rejected (F3/R-B4-13);
   - expiry: pending past expiresAt → expired; re-request needed (R-B4-11);
   - integration with B2: denied admission → CRD → approve → retry admits, correlation_id end-to-end.
4. **Exception:** unbounded/unscoped rejected (R-B4-15); usage counted; expired exception does NOT
   convert deny even if cached (F4); grant requires approval (R-B4-16).
5. **Simulation:** PolicySimulationRun never mutates cluster/blocks traffic (R-B4-18); differential
   diff produced (DT-49).
6. **Remediation:** destructive action dry-run by default; armed + audited to actually execute (F6/R-B4-21);
   idempotent re-reconcile safe.
7. **CRD lifecycle:** v1alpha1 conversion path exists; all transitions audited with correlation_id (R-B4-23);
   level-triggered reconcile idempotent.
8. **RBAC:** namespace author scope enforced on CRD writes (R-B4-24).

---

## 5. What blocks what

- **Blocks B1/B2/B3/C2/E1:** the action enum (W1) — freeze it first.
- **Blocks B2:** the deny-with-approval handshake + exception/approval consultation (W4a/b).
- **Blocked by B2:** the admission-side of the approval handshake (co-develop W4a with B2-W4).
- **Blocked by E1:** simulation engine that PolicySimulationRun drives (W4c).
- **Blocked by D2:** RBAC scoping + approver identity resolution.

---

## 6. Risks & mitigations

| Risk | Mitigation |
|---|---|
| Approval flow durability/race (the keystone) | Durable queue for creation (R-B4-14); optimistic-concurrency single-use (R-B4-13); spike W4a first |
| Exception becomes a permanent governance hole | Bounded+scoped mandatory; usage analytics flags abuse (C3); grant approval-gated |
| Action precedence wrong → inconsistent behavior | Exhaustive pairwise precedence tests (#1) |
| OCP vs custom engine bet | See ALT-ocp-substrate; abstract substrate; spike OCP early |
| Kyverno-first pressure (AI tooling momentum) | See ALT-kyverno-first; keep decision/effector separable so either can be primary effector |
| v1alpha1 churn breaks stored CRDs | Conversion webhook before any GA contract (R-B4-22) |
| Remediation destructive accident | Dry-run default; armed+approved for destructive (R-B4-21) |

---

## 7. Estimated sequencing (relative)

Week 1: W1 taxonomy + W2 rubric + W3 schemas. Week 2: W4a approval spike (co-dev with B2). Week 3:
W4a complete (M4) + W4b exception. Week 4: W4c sim + W4d remediation/libs + W5 PDP profiles. Week 5: hardening, RBAC, conformance.
