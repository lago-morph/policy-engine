# Domain D — Identity, Authz & Security — ADVERSARIAL RECONCILIATION

Domain-level reconciliation of the four per-component red-team passes. Focus: **contradictions
*between* D components**, findings that only appear when the components are seen together, and the
prioritized domain-wide defect list.

Component reviews: [D1](../../components/D1-keycloak-jwt/ADVERSARIAL-REVIEW.md) ·
[D2](../../components/D2-scoped-rbac-storage/ADVERSARIAL-REVIEW.md) ·
[D3](../../components/D3-approval-gated/ADVERSARIAL-REVIEW.md) ·
[D4](../../components/D4-security/ADVERSARIAL-REVIEW.md)

---

## 1. Cross-component contradictions (must be reconciled before build)

**X1 — Additive roles (D2) vs Separation-of-Duties (D3/D4).** D2's SPEC describes roles as
**additive with union semantics**; D3-R6 and D4 SEC-11 depend on **requester ≠ approver** and
author ≠ enforce-promoter. A subject granted both `namespace-policy-author` and
`namespace-policy-approver` for the same scope can author *and* approve. D2 adversarial A5 and D3
A11 and D4 A8 all flag this from different angles.
**Resolution (domain ruling):** SEC-11's **mutual-exclusion is normative**. Amend D2 SPEC §2 to
state that author/approver (and any defined SoD pair) cannot be co-held for the same scope;
enforce at `role:grant` time and re-check at action time. This is a *correctness* constraint, not
hardening.

**X2 — Hot-path purity (D1-R10) vs grant-store re-check (D4 SEC-14 / D2 OQ-6 / D3 A12).** D1
forbids synchronous network in normalization; D4/D2/D3 require high-value ops to re-validate the
subject's roles against the server-side grant store (to defeat stale-token use). D1 adversarial
A15 and D4 A9 both flag the collision.
**Resolution:** Split the path. **Normalization stays pure** (no network). A **separate
authorization-time check** (not normalization) re-validates the grant store for the enumerated
high-value verbs (`policy:promote-enforce`, `approval:approve`, `exception:approve`, `role:grant`,
`mapping:edit`, `config:manage`). Cheap reads trust the token within a bounded lifetime. Document
the exact verb list once, in D4, and reference it from D1/D2/D3.

**X3 — Fail-closed everywhere (D1/D2/D3) vs availability (no D-wide requirement until D4 A3).**
D1 (JWKS-expiry reject), D2 (audit-blocking cross-tenant), D3 (`failurePolicy: Fail`) each
correctly fail closed — but *together* they create a platform where Keycloak down = no authn,
audit sink down = no admin investigation, approval webhook down = no deploys. No component owned
the resulting self-DoS until D4 A3.
**Resolution:** D4 adds a break-glass + RTO requirement (consolidating D1 A14, D2 A9, D3 A3).
Each fail-closed boundary gets a documented, **audited, rate-limited, time-boxed** break-glass.
Notably D2 A9: cross-tenant audit for an **incident responder** must degrade to local-log+alarm,
not hard-block.

**X4 — Mapping config (D1) is an authz artifact but gated by the role it can mint (D2).** D1 A6
and D2 A6: editing the §15.4 mapping can grant `role:org-finance` (group→role expansion); the
edit is gated behind Platform Governance Admin — the same power. Self-escalation with no
four-eyes.
**Resolution:** `mapping:edit` and `role:grant` are **dual-control / approval-gated** (route
through D3). A mapping change *is* an approval-gated decision. D4 SEC-12/SEC-15 carry this;
elevate the highest verbs to POC-MUST dual-control (D4 A10), not GA-deferred.

**X5 — D4 SHOULD-deferrals vs the components' reliance on them.** D4 defers bundle signing
(SEC-2), audit-chaining (SEC-8), and platform supply-chain (SEC-21) toward GA — but HL-18
(auditor independent re-execution, D2/D4) leans on **signed, digest-addressable bundles** and
**tamper-evident evidence** to be *meaningful*. An auditor re-running against an unsigned bundle
proves little. D4 A1/A2/A11.
**Resolution:** For the HL-18 assurance story to hold, **bundle versioning + digest addressing is
POC-MUST** (it is, SEC-1), and **at least minimal signing is strongly recommended at POC** rather
than stubbed — otherwise the marquee auditor scenario is hollow.

---

## 2. Findings visible only at domain scope

**Y1 — The `correlation_id` is overloaded but never centrally defined.** It threads D1
(audit `jwt_claims`), D2 (`authz_denied`/`boundary_crossing`), and D3 (approval chain, == CR
name). Each component assumes a compatible shape but no component **owns** the correlation-id
contract. → Define it once at the domain/cross-cutting layer (likely C2's audit schema), with
rules for who mints it and how it's propagated.

**Y2 — PII flows through all four and is reconciled nowhere consistently.** D1 emits `email`/`sub`
into `jwt_claims` (A12); D2 redacts on export (DT-57) but the at-rest store still holds raw; D4
OQ-4 says hash-at-rest but D4 A7 notes immutability vs right-to-erasure conflict. → A single
**identity-data handling policy** must span D1 (emit), C2 (store), D2 (export), D4 (retention/
erasure). Currently three half-answers.

**Y3 — Stale-token theme recurs in every component** (D1 A5, D2 A10, D3 A12). It's the same root
cause seen four times. X2's resolution (enumerated re-check verbs + bounded privileged-token
lifetime) fixes it *once* if applied consistently — don't let each component invent its own
partial mitigation.

**Y4 — Analytics/reporting read paths are the domain's biggest unguarded surface (D2 A2).** §14
analytics and §17E reporting aggregate across scopes by nature (e.g. per-subject cross-tenant
counters). If they bypass D2's interceptor, the whole §17A.1 invariant is void *for the most
data-rich queries*. This is a **Domain D ↔ Domain C** seam that neither domain fully owns. → Must
be resolved at cross-cutting: analytics/reporting reads go through the scope predicate (or RLS).

**Y5 — Webhook endpoints are a data-egress channel touched by D2 (role 9 config) and D3 (emit).**
D2 A8 + D3 A6: a Workflow Integrator can point approval webhooks (carrying subject/resource PII)
at an attacker URL → exfil + SSRF, without holding any read permission. → Endpoint allow-listing
+ internal-address blocking is a joint D2/D3 control; make it a D4 SEC requirement.

---

## 3. Prioritized domain-wide defect list
| ID | Sev | Theme | Components | Resolution |
|---|---|---|---|---|
| X1 | **Critical** | Additive roles defeat SoD | D2,D3,D4 | Mutual-exclusion on SoD pairs; SEC-11 normative |
| D3-A1 | **Critical** | Approve-then-swap (ref vs digest) | D3 | Bind approval to manifest digest; re-eval compares digest |
| D3-A5 | **Critical** | Forged callback asserts any approver | D3,D4 | Approver-bound callback proof, not shared HMAC |
| D2-A2/Y4 | **Critical** | Analytics/reporting bypass storage authz | D2,(C) | Route aggregate reads through scope predicate / RLS |
| X2/Y3 | **Critical** | Stale privileged token | D1,D2,D3,D4 | Enumerated re-check verbs + bounded privileged-token TTL |
| D3-A8 | **Critical** | Approvals not expiring (status-trust) | D3 | PDP compares now vs expires_at every eval |
| X4 | **High** | Mapping/role-grant self-escalation | D1,D2,D4 | Dual-control / D3 approval on highest verbs (POC-MUST) |
| D2-A1/D4-A1 | **High** | P0 invariant guarded by a lint; signing deferred | D2,D4 | RLS-under-interceptor mandatory; minimal POC signing |
| Y5/D2-A8/D3-A6 | **High** | Webhook exfil/SSRF | D2,D3,D4 | Allow-list endpoints; block internal addrs |
| D4-A4 | **High** | No injection/input-validation requirement | D4 | Validate/parameterize identity-derived values in queries/templates |
| X3 | **High** | Fail-closed self-DoS, no break-glass | D1,D2,D3,D4 | Audited, rate-limited, time-boxed break-glass + RTO |
| Y2/D4-A7 | Medium | PII handling fragmented; erasure vs immutability | D1,D2,D4,(C) | Single identity-data handling policy across emit/store/export/retain |
| Y1 | Medium | `correlation_id` contract unowned | D1,D2,D3,(C) | Define once in C2 audit schema |
| D1-A7/A10 | Medium | Snapshot drift breaks replay; multi-IdP disagreement first-wins | D1 | Bind snapshot digest to mapping version; disagreement ⇒ degraded |
| D2-A3 | Medium | Existence oracle via 403/404 shape+timing | D2 | Constant-shape/constant-time responses |
| D3-A4 | Medium | Approver-shopping after deny | D3 | Denied CR blocks re-request (cooldown) |

---

## 4. Domain verdict
The domain's *architecture* is sound and the scenarios are faithfully supported. The risk is
concentrated in **trust boundaries and the seams between components**:

1. **Trust state, not tokens** (X2/Y3, D3-A8) — the recurring stale-data theme; fix once.
2. **Bind decisions to content, not names** (D3-A1) — approval must follow the actual manifest.
3. **Prove who, not just that someone, approved** (D3-A5) — callbacks need approver-bound proof.
4. **The choke point must be architectural, and it must include analytics** (D2-A1/A2/Y4) — a
   lint is not a control, and the richest queries are the ones most likely to escape.
5. **Make the SoD a hard constraint** (X1) — the entire approval story is built on requester ≠
   approver, which additive roles silently break.

Resolve X1, X2, X4, D3-A1, D3-A5, D3-A8, and D2-A2 and Domain D goes from "looks secure in the
diagram" to "survives the pen-test and the audit."
