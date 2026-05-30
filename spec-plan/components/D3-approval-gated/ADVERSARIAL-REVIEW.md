# D3 — Approval-Gated Policy Decisions — ADVERSARIAL REVIEW

**Red-team persona:** impatient developer racing a deadline + an attacker who wants to ship an
unapproved workload. **Target:** `SPEC.md`, `PLAN.md`.

---

## 1. Attacks on the Kubernetes deny-with-approval-required pattern

**A1 — TOCTOU between approval and apply (approve-then-swap).** The spec binds approval to a
resource spec digest (D3-R3, OQ-3) — good. But the **external-data provider** reads CR state and
the PDP compares it to the *current* request. If the digest binding is on `(controlId,
resourceRef)` but **not** the actual manifest content at admission time, an attacker gets
`Deployment/api` approved with a benign image, then re-applies `Deployment/api` with a malicious
image — same `resourceRef`, different content. **The CR matches by name/ref, the image differs.**
DT-59 step 6 matches on `(controlId, resourceRef, requestedBy)` — **not on spec digest.** This
is a direct contradiction with D3-R3. **Critical defect:** the admission re-eval MUST compare the
current manifest digest to the approved digest, not just the resource ref.

**A2 — External-data staleness as a bypass *and* a DoS.** §7 says stale provider ⇒ user retry
denies until refresh. But the inverse is the danger: if the provider caches an `approved` CR and
the approval is later **revoked** (OQ-5) or **expired**, a stale cache could admit a
now-unapproved workload. The spec bounds poll interval for *grant* freshness but not for
*revocation* freshness. **Revocation/expiry must propagate at least as fast as grant** — ideally
faster (fail-closed direction). Gap.

**A3 — `failurePolicy` is the whole ballgame and it's only in §7.** If the approval-gate webhook
is down or times out, `failurePolicy: Ignore` ⇒ **everything is admitted unapproved** (gate
silently disabled); `Fail` ⇒ all deploys blocked (self-DoS). The spec says use `Fail`, but this
is buried in a failure-mode row, not a top-level MUST. For an approval gate, `failurePolicy:
Fail` is **the** security-critical config. **Defect:** promote to a normative MUST with a test
that asserts it on every approval-gate constraint, and document the operational blast radius
(approval gate down = deploys frozen).

**A4 — Idempotency key omits the requester's intent change.** D3-R2 makes CR creation idempotent
on `(controlId, resourceRef, requestedBy)`. But if Sam requests, gets **denied**, then re-applies
hoping for a different approver — does the existing `denied` CR get reused (correctly blocking
retries) or does a new `pending` CR get created (letting him shop for a lenient approver)? Spec
says reuse `pending`; it's silent on reuse of `denied`. **A denied CR must block re-request for a
cooldown / require explicit re-open**, else approver-shopping defeats the gate.

## 2. Attacks on the webhook / callback path

**A5 — Forged or replayed callback grants approval.** D3-R9/R10 require signed, authorized,
idempotent callbacks — good — but the **threat is the highest-value in the system**: a valid
callback transitions `pending→approved`. If the HMAC secret leaks, or the callback endpoint
trusts a workflow system that itself has weak auth, anyone can mint approvals. The spec puts the
*authorization* check on `approved_by` (must satisfy D3-R6) — but a forged callback can simply
**claim** `approved_by: a-real-approver`. Unless the callback signature is bound to the
approver's identity (not just the workflow system's shared secret), `approved_by` is attacker-
controlled. **Defect:** the callback must carry **proof the named approver actually approved**
(e.g. approver-signed token / OIDC-authenticated approval action), not just a workflow-system
HMAC that any holder can use to assert any `approved_by`.

**A6 — Webhook endpoint as exfil/SSRF (mirror of D2 A8).** Workflow Integrator sets the outbound
endpoint; the payload carries `subject.sub`, `groups`, resource identity. Malicious/compromised
config ⇒ every approval request leaks subject+resource to an attacker collector, **and** the
platform makes outbound requests to an attacker-chosen URL (SSRF into internal network). Spec
defers endpoint allow-listing to a SHOULD (D3-R12). For an egress channel carrying identity data,
this must be a MUST with allow-listing + no-internal-address validation.

**A7 — `require_async_check` resolver trust.** `require_async_check` calls an external check and
resolves allow/deny on its response. If that response channel isn't authenticated like the
approval callback, an attacker forges a "check passed" → allow. The spec details auth for
*approval* callbacks (D3-R9) but the `require_async_check` resolution path inherits "shares the
CRD + webhook machinery" without restating the auth requirement. Gap — must be equally hardened.

## 3. State-machine & expiry attacks

**A8 — Expiry reconciler is a single timer; if it lags, expired approvals stay honored.** D3-R4
says expired approvals aren't honored, but enforcement reads CR `status` — if the reconciler that
flips `approved→expired` is down, the CR still says `approved` past `expires_at`. **Enforcement
must independently check `now < expires_at`, not trust the `status` field alone.** Otherwise a
reconciler outage = approvals never expire. The spec's §7 controller-down row says "honored only
if within TTL and reconciler can verify" — but the admission external-data path may not re-check
the timestamp. Must mandate: **PDP compares `now` to `expires_at` at every evaluation**, status
is advisory.

**A9 — Re-authorization scope-widening.** HL-19 lets the approver *tighten* scope on renewal.
Can they **widen** it (extend the exception to more namespaces) under cover of a "renewal"? A
renewal that widens scope is effectively a new, larger grant masquerading as a routine extension
and may skip fuller review. **Defect:** widening on re-auth must be treated as a new request
(full approval), not a renewal.

**A10 — Break-glass abuse.** HL-10 break-glass has shorter TTL + post-hoc review. But "post-hoc
review" is detective, not preventive — the unapproved action already shipped. If break-glass is
self-serviceable (even with later review), it's an **intentional bypass of the entire gate.**
The spec allows an "emergency role" to override SoD with retrospective sign-off. Without a hard
rate-limit + automatic escalation + the action being *reversible*, break-glass becomes the
attacker's preferred path. Under-specified.

## 4. Cross-component inconsistencies

**A11 — SoD depends on D2's broken additive-role model.** D3-R6 (requester≠approver) is the
backbone, but D2's adversarial A5 shows additive roles let one subject hold both author and
approver. So a subject could be `requestedBy` *and* hold `approval:approve` for the scope —
D3-R6's "must not be the requestedBy subject" catches *self*-approval, but **not** the subtler
case where the requester uses a *second identity/service account* they control to approve. SoD is
only as strong as identity separation. Cross-cut defect with D2.

**A12 — Stale token approver (D1 A5).** The approver's `roles` come from their token. A
deprovisioned approver whose token hasn't expired can still satisfy D3-R6. Approval is high-value
enough that it should **re-check the grant store at approval time** (D2 OQ-6's "privileged op"),
not trust the token's `roles`.

**A13 — `deployment_approval` claim (D1/§15.3) vs CRD authority.** D1 OQ-6 says the
`deployment_approval` *claim* is non-authoritative (CRD is truth). Good — but is there any code
path where a PDP reads `input.subject.deployment_approval` and short-circuits to allow? If so, a
subject with a stale/forged approval claim bypasses the CRD. Must assert: **no enforcement path
trusts the approval claim; only the CRD/approval store.**

## 5. Prioritized defect list
| ID | Severity | Defect | Fix |
|---|---|---|---|
| A1 | **Critical** | Approve-then-swap: admission matches resource ref, not spec digest (contradicts D3-R3) | Re-eval MUST compare current manifest digest to approved digest |
| A5 | **Critical** | Forged callback can assert arbitrary `approved_by` with a shared HMAC | Callback must carry approver-bound proof (OIDC/approver-signed), not just system HMAC |
| A8 | **Critical** | Expiry relies on reconciler; stale `status=approved` honored past TTL | PDP MUST compare `now` to `expires_at` every eval; status advisory |
| A3 | **High** | `failurePolicy` security-critical but only a failure-mode note | Normative MUST `Fail`; test on every approval-gate constraint |
| A6 | **High** | Webhook endpoint exfil/SSRF (identity data egress) | MUST allow-list endpoints; block internal addresses |
| A2 | **High** | Revocation/expiry must propagate ≥ as fast as grant | Bound revocation-freshness tighter than grant; fail-closed |
| A4 | High | Approver-shopping via re-request after deny | Denied CR blocks re-request (cooldown / explicit re-open) |
| A7 | Medium | `require_async_check` resolver not auth-hardened like callbacks | Same auth/idempotency requirements as D3-R9/R10 |
| A11/A12 | Medium | SoD defeated by additive roles / second identity / stale approver token | Re-check grant store at approval; identity-separation w/ D2 |
| A9 | Medium | Scope-widening renewal skips full review | Widening = new request, full approval |
| A10 | Medium | Break-glass = intentional self-serviceable bypass | Rate-limit + auto-escalation + reversibility + mandatory review |
| A13 | Low | Possible PDP trust of `deployment_approval` claim | Assert no path trusts the claim; CRD only |

**Verdict:** The deny-with-CRD-and-retry pattern is the correct answer to the §17B.4 admission
constraint and the scenarios are well-modeled. But the **resolution-trust** parts are where it
breaks: **(A1)** approval must bind to the actual manifest content, not just the resource name;
**(A5)** a callback must prove *who* approved, not just that *some* holder of a shared secret
said so; and **(A8)** expiry must be checked against the clock at every evaluation, not trusted
from a status field a reconciler might fail to update. Fix those three and the gate is sound.
