# D4 — Security Requirements — ADVERSARIAL REVIEW

**Red-team persona:** auditor who will *fail* the SOC 2 if controls are aspirational + attacker
probing the "should" gaps. **Target:** `SPEC.md`, `PLAN.md`.

---

## 1. The "SHOULD" trap

**A1 — POC "should"s are GA "won't"s.** §23 and D4 mark signing (SEC-2), audit-chain (SEC-8),
TLS (SEC-19), and platform supply-chain (SEC-21) as SHOULD-at-POC. History says POC SHOULDs
never harden unless something forces them. D4's W10 "trajectory gates" is the right idea but
it's a **plan artifact, not a control** — nothing *blocks* GA if the gate isn't met. **Defect:**
each SHOULD→MUST@GA needs a concrete **gate owner + blocking criterion** in the release process,
or it's decoration. For a security/governance product, shipping with unsigned bundles or
plaintext service traffic is a credibility-ending finding.

**A2 — "Verification property without signing" (SEC-2) is half a control.** SEC-2 requires the
*verification interface* but stubs the signing. A verification interface with nothing to verify
is a no-op that **looks** like a control in a diagram and **gives false assurance**. An auditor
will (correctly) treat "we can verify signatures but don't sign" as "no signing control."
Either sign (even with a simple keypair) or state plainly there is no integrity control yet.

## 2. Gaps in the requirement set

**A3 — No availability / DoS requirement.** Every fail-closed control (D1 JWKS-expiry, D3
`failurePolicy: Fail`, D2 audit-blocking) trades availability for security — correctly — but D4
has **no requirement governing the resulting self-DoS surface** (RTO, break-glass, rate-limits).
The platform can be taken down by knocking over Keycloak (all authn fails closed) or the audit
sink (cross-tenant admin blocked). **Defect:** add SEC requirements for break-glass + documented
RTO for each fail-closed boundary (consolidates D1 A14, D2 A9, D3 A3).

**A4 — No input-validation / injection requirement.** D4 covers authn/authz/integrity but says
nothing about **injection** into the query-rewrite layer (D2), the mapping-transform engine (D1
`lookup`/`template`), or webhook payloads (D3). If a tenant/namespace name flows unsanitized into
the storage predicate, a crafted name could break the scope filter (the worst class of bug for
SEC-9). **Defect:** add a requirement for input validation/parameterization on all
identity-/scope-derived values that reach a query or template.

**A5 — No rate-limiting / brute-force / enumeration-throttle requirement.** D2's 404-by-ID
defeats *confirmation* of existence, but an attacker can still enumerate at high volume; break-
glass (D3) can be spammed; approval callbacks can be brute-forced if the HMAC is weak. No SEC
requirement mandates rate-limiting or anomaly throttling. Gap.

**A6 — No key-rotation / compromise-recovery procedure.** §3's secret inventory lists keys and
says "rotate," but there's no requirement for **what happens when a key is compromised** — e.g.
if the bundle signing key or approval-callback key leaks, how are issued tokens/bundles/approvals
revoked en masse? Rotation ≠ recovery. The approval-callback key compromise (D3 A5) is an
existential threat with no defined response.

**A7 — Data residency / retention / right-to-erasure absent.** SEC-5 mandates immutable
append-only audit for the retention window; OQ-4 hashes PII. But immutability **conflicts** with
GDPR right-to-erasure for `sub`/`email` held in `jwt_claims` — you can't erase from an immutable
log. D4 doesn't reconcile "tamper-evident immutable evidence" with "erase my PII." For a product
processing EU identity data this is a real legal control gap (mirrors D1 A12).

## 3. Cross-component / internal inconsistencies

**A8 — D4 inherits D2's broken SoD.** SEC-11 asserts mutual-exclusion on author+approver — good,
it actually *fixes* D2's A5 — but D2's SPEC still describes additive roles with union semantics.
D4 and D2 are now **inconsistent**: which is normative? If D4 is the floor, D2's role-union
section must be amended to honor SEC-11. Reconcile at the domain level.

**A9 — SEC-14 (re-check grant store) conflicts with D1-R10 (no hot-path network).** D4 mandates
high-value ops re-validate against the server-side grant store; D1 forbids synchronous network
in normalization. These collide exactly as D1's A15 noted. D4 inherits the unresolved conflict
and must **pick the boundary**: which verbs re-check (accepting a lookup) vs which trust the
token. Currently both specs hand-wave it.

**A10 — "Audit-only" admin controls (SEC-12 POC) are detective, and this is a governance
product.** Audit-only means a rogue admin's `role:grant` self-escalation is *recorded* but not
*prevented*. For most products fine; for a product whose entire value proposition is preventive
governance, shipping the platform's **own** admin controls as detective-only is self-undermining.
Recommend SEC-12 dual-control be POC-MUST for the *highest* verbs (`role:grant`, `mapping:edit`),
not deferred to GA.

**A11 — Supply-chain dogfooding (SEC-21) is SHOULD; tenants are held to MUST (§20.1).** §20.1
makes "all production images must be signed" a *governance objective the platform enforces on
tenants*. D4 then lets the *platform's own* images be unsigned at POC. This is the exact
hypocrisy an auditor pounces on: "you block my unsigned image but run unsigned yourself."
Make SEC-21 POC-MUST, or document the dogfooding gap loudly.

## 4. Prioritized defect list
| ID | Severity | Defect | Fix |
|---|---|---|---|
| A2 | **High** | Verification-without-signing is false assurance | Sign (even simply) at POC, or state "no integrity control" plainly |
| A4 | **High** | No injection/input-validation requirement for scope/mapping/webhook inputs | Add SEC req: validate/parameterize all identity-derived values reaching queries/templates |
| A9 | **High** | SEC-14 vs D1-R10 unresolved (re-check vs no-hot-path-network) | Define which verbs re-check the grant store; bound the lookup |
| A8 | **High** | SEC-11 (mutual-exclusion) inconsistent with D2's additive-role union | Amend D2; make SEC-11 the normative floor |
| A6 | Medium | Rotation without compromise-recovery (esp. approval-callback key) | Add key-compromise mass-revocation procedure |
| A3 | Medium | No availability/break-glass/RTO requirement for fail-closed boundaries | Add SEC req consolidating D1 A14 / D2 A9 / D3 A3 |
| A1 | Medium | SHOULD→GA gates are plan text, not blocking controls | Assign gate owner + blocking criterion per SHOULD |
| A7 | Medium | Immutable evidence vs GDPR right-to-erasure | Reconcile: pseudonymize/crypto-shred PII while keeping decision evidence |
| A11 | Medium | Platform supply-chain dogfooding deferred while tenants held to MUST | SEC-21 POC-MUST or document loudly |
| A10 | Medium | Admin controls detective-only on a preventive product | Dual-control POC-MUST for `role:grant`/`mapping:edit` |
| A5 | Low | No rate-limit / enumeration-throttle / brute-force requirement | Add SEC req for rate-limiting on auth, callbacks, break-glass |

**Verdict:** D4 correctly *consolidates* the D1–D3 findings (SEC-11/12/14/15 are exactly the
right responses), which is its strength. Its weaknesses are: **(A1/A2/A11)** too much deferred to
"GA" or "verification-only" on a product whose credibility *is* security; **(A4/A3/A5)** whole
control families (injection, availability, rate-limiting) simply absent; and **(A8/A9)** it
states fixes that **contradict the current D2/D1 SPEC text**, so the domain must reconcile. The
single most important message: a governance product that defers signing its own bundles and
images, and runs admin controls as detective-only, will not survive its own first audit.
