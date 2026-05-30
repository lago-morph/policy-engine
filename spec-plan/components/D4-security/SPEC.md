# D4 — Security Requirements — SPEC

**Domain:** D · Identity, Authz & Security
**Spec sources:** §23 (Security Requirements), with cross-cuts to §8 (signed bundles), §13
(audit), §15 (OIDC/JWT), §17A (server-side authz), §24–25 (deployment/extensibility),
§26.1 (signed-bundles guidance)
**Scenarios exercised:** HL-18 (evidence integrity / independent re-execution), DT-56/DT-57
(auditor onboarding, redacted export), plus every D1/D2/D3 scenario relies on D4 controls.
**Status:** AUTHORED (parent-authored)
**Author persona:** Marcus (Platform Security Engineer) + security architect

---

## 1. Scope

### 1.1 In scope
D4 is the **platform-wide security baseline** — the numbered, normative security requirements
that the other components inherit and that the platform's own supply chain must meet. It
consolidates §23.1's seven areas into testable requirements and extends them where §23 is
under-specified (it is deliberately terse for the POC). Areas:

1. **Policy integrity** — signing + versioning of policy bundles (§23, §26.1).
2. **Evidence integrity** — audit fixtures and replay datasets tamper-evident or versioned.
3. **Access control / Authorization** — GUI + storage scope-aware, server-side (delegates to
   D2; D4 states the requirement and the verification).
4. **Identity** — OIDC/JWT, Keycloak initial IdP (delegates to D1).
5. **Auditability** — edits, simulations, approvals, promotions logged (ties to D3 + Domain C).
6. **Transport** — TLS for service-to-service.
7. **Supply chain of the platform itself** — the governance platform must practice what it
   preaches (signed images, SBOMs, provenance) — §23 implies but doesn't state this; D4 makes
   it explicit because a governance product that isn't itself governed is indefensible.

### 1.2 Out of scope
- The *implementations* of D1/D2/D3 (D4 states cross-cutting requirements + acceptance).
- Choosing the specific signing tech (§26.1: "signing implementation intentionally unspecified
  at this stage") — D4 specifies the **properties**, not the product.
- Infra hardening (cluster CIS benchmarks, network policy) beyond the platform's own footprint
  (Domain F).

### 1.3 §23 is a floor, not a ceiling
> §23 uses "should" for signing/TLS and is POC-scoped. D4 keeps the POC floor but tags which
> requirements **must** harden before GA, so the security posture is a deliberate trajectory,
> not an accident.

---

## 2. Numbered security requirements

### 2.1 Policy integrity (§23 Policy integrity; §8; §26.1)
- **SEC-1 (MUST)** Every policy bundle is **versioned** with an immutable, content-addressable
  identifier (digest); the version is recorded on every decision (§13.3 `policy_bundle_version`)
  so a decision is always traceable to the exact bundle (HL-18 replays by digest).
- **SEC-2 (SHOULD→MUST@GA)** Policy bundles are **signed**; the signing implementation is
  unspecified for the POC (§26.1) but the **verification property** is required: a consumer
  (Gatekeeper/OPA/Conftest) can verify provenance + integrity before loading a bundle.
- **SEC-3 (MUST)** Bundle scope metadata (§17A.5 D2-R7) is part of the signed/versioned content
  — scope cannot be altered without invalidating the version/signature.
- **SEC-4 (MUST)** Unsigned/unverifiable bundles in an environment requiring verification are
  **rejected** (fail closed), not loaded with a warning.

### 2.2 Evidence integrity (§23 Evidence integrity; §13)
- **SEC-5 (MUST)** Audit events and replay-capable fixtures are **tamper-evident or versioned**:
  each event/fixture has a content hash; the audit store is append-only/immutable for the
  retention window.
- **SEC-6 (MUST)** Replay datasets are **materialized + hashed** before use (ties D2-R8); a
  replay's inputs are addressable by digest so an auditor's run is reproducible (HL-18).
- **SEC-7 (MUST)** Evidence exports compute and record a **pre-redaction `original_hash`** in the
  export manifest before any redaction is applied (DT-57 step 3) — the tamper-evidence anchor
  survives redaction.
- **SEC-8 (SHOULD)** The audit log supports a **chained/notarized integrity proof** (hash chain
  or periodic signed checkpoint) so tampering anywhere in the window is detectable, not just
  per-event. (Hardens SEC-5 toward GA.)

### 2.3 Access control & authorization (§23 Access control + Authorization; §17A)
- **SEC-9 (MUST)** Authorization is enforced **server-side** at both the GUI/API layer **and**
  the storage layer (§17A.1) — GUI-only authz is non-conformant. (Verification: D2 §4.2
  scope-escape suite.)
- **SEC-10 (MUST)** Namespace, tenant, and policy-domain access are enforced server-side
  (§23 Authorization); identical denial regardless of caller (Console/curl/CI/MCP).
- **SEC-11 (MUST)** Separation of duties: requester ≠ approver; author ≠ enforce-promoter
  (D3-R6, D2 matrix); enforced at grant + action time, including a **mutual-exclusion** on
  holding both author and approver roles for the same scope (D2 adversarial A5).
- **SEC-12 (MUST)** Privileged/global actions (`role:grant`, `mapping:edit`, global
  `policy:promote-enforce`, `config:manage`) are **audited**, and SHOULD require dual-control /
  approval (D2 adversarial A6, D1 adversarial A6).

### 2.4 Identity (§23 Identity; §15)
- **SEC-13 (MUST)** Authentication uses **OIDC/JWT**, Keycloak as initial IdP; tokens are
  signature-verified, with `iss` (exact-match allow-list), `aud` (+`azp`), `exp`/`iat`/`nbf`
  validated; `alg` pinned per issuer; **unpinned alg ⇒ reject**; `alg:none`/RS-HS confusion
  rejected (D1-R1..R3, D1 adversarial A1/A2/A4).
- **SEC-14 (MUST)** Privileged-role tokens have a **bounded lifetime**; high-value operations
  re-validate the role against the server-side grant store (mitigates stale-token; D1 A5, D2
  OQ-6, D3 A12).
- **SEC-15 (MUST)** The JWT-to-policy mapping config is an **authz-relevant artifact**: editing
  it is a privileged, audited, approval-worthy action (D1 §8, D1 adversarial A6).

### 2.5 Auditability (§23 Auditability)
- **SEC-16 (MUST)** Policy **edits, simulations, approvals, promotions, mapping changes, role
  grants, and exports** are logged with actor (JWT subject), timestamp, before/after value where
  applicable, and `correlation_id`.
- **SEC-17 (MUST)** Authorization denials (`authz_denied`) and admin boundary-crossings
  (`boundary_crossing`) are audited (D2-R9, DT-54/DT-55).
- **SEC-18 (MUST)** The audit trail is itself **scope-aware** (auditors see only in-scope events)
  yet **complete and immutable** within scope (no silent gaps).

### 2.6 Transport (§23 Transport)
- **SEC-19 (SHOULD→MUST@GA)** Service-to-service communication uses **TLS** (mTLS preferred for
  the normalized-subject and approval-callback channels, which carry trust-bearing data).
- **SEC-20 (MUST)** The normalized authorization subject (D1) and approval callbacks (D3) travel
  only over authenticated, integrity-protected channels; they are never reconstructed from an
  untrusted source.

### 2.7 Supply chain of the platform itself (D4 extension of §23)
- **SEC-21 (SHOULD→MUST@GA)** Platform container images are **signed** (e.g. Sigstore) and carry
  an **SBOM**; deployment verifies signature + SBOM presence — the platform meets the supply-
  chain bar it enforces on tenants (§20.1 dogfooding).
- **SEC-22 (MUST)** Platform CRDs' controllers and admission webhooks run with **least-privilege
  RBAC**; the validating webhooks that protect CRD immutability (D3-R1) are themselves protected
  from tampering.
- **SEC-23 (MUST)** Secrets (webhook HMAC keys, JWKS-pin material, signing keys) are managed
  outside source, rotated, and never logged.

---

## 3. Secrets / key inventory (what must be protected)
| Secret | Used by | Risk if leaked | Control |
|---|---|---|---|
| Bundle signing key | SEC-2 | Forge trusted policy | HSM/KMS, rotation, SEC-23 |
| JWKS / issuer pin material | SEC-13 | Accept forged tokens | Verified channel, pinning |
| Approval-callback signing key | D3-R9 | Forge approvals (D3 A5) | Per-approver binding, rotation |
| Webhook HMAC / mTLS certs | D3-R8/R12 | Exfil / SSRF (D2 A8/D3 A6) | Allow-list endpoints, rotate |
| Audit-chain checkpoint key | SEC-8 | Undetectable tampering | Separate trust domain |
| Platform image signing key | SEC-21 | Supply-chain compromise | KMS, isolated CI |

---

## 4. Threat model (platform-level, consolidating D1–D3)
| Threat | Primary control(s) |
|---|---|
| Forged/confused token | SEC-13 |
| Stale privileged token after deprovision | SEC-14 |
| Scope escape / GUI bypass | SEC-9, SEC-10 (D2 §4.2 suite) |
| Self-approval / SoD defeat | SEC-11 |
| Admin self-escalation (mapping/role) | SEC-12, SEC-15 |
| Approve-then-swap (manifest) | D3-R3 + digest re-eval (D3 A1) |
| Forged approval callback | SEC-13-style binding (D3 A5) |
| Approval not expiring | clock-checked expiry (D3 A8) |
| Evidence tampering | SEC-5..SEC-8 |
| Unsigned/forged bundle | SEC-1..SEC-4 |
| PII leak in audit/export | SEC-7 + redaction (DT-57), at-rest decision (D1 A12) |
| Webhook SSRF/exfil | SEC-23 + endpoint allow-list |
| Supply-chain compromise of platform | SEC-21, SEC-22 |
| Self-DoS (fail-closed authn/admission) | documented break-glass + RTO (D1 A14, D3 A3) |

---

## 5. POC-floor vs GA-hardening trajectory
| Req | POC | GA |
|---|---|---|
| SEC-2 bundle signing | versioned + verification interface, signing impl stubbed | full signing + verification enforced |
| SEC-8 audit chain | per-event hash | hash chain / signed checkpoints |
| SEC-19 TLS | TLS in deployed envs | mTLS for trust-bearing channels |
| SEC-21 platform supply chain | SBOM produced | signed images + verified deploy |
| SEC-12 dual-control | audit-only | approval-gated admin verbs |
Everything else (SEC-9, 10, 11, 13, 16, 17, 18, 20, 22, 23) is **MUST at POC** — these are the
non-negotiable floor because they are correctness/trust controls, not hardening.

---

## 6. Dependencies
| Depends on | For |
|---|---|
| D1 | Identity controls (SEC-13..15, SEC-20) |
| D2 | Authz controls (SEC-9..12) |
| D3 | Approval/SoD/callback controls (SEC-11, SEC-20) |
| Domain C (C1/C2/C4) | Evidence/audit integrity (SEC-5..8, 16..18) |
| Domain B (B1) | Signed bundles (SEC-1..4) |
| Domain F | Deployment, supply chain (SEC-19, 21, 22) |

| Depended on by | For |
|---|---|
| All components | Inherit the security floor; map their controls to SEC-n |

---

## 7. Open questions — decided defaults
| # | Question | Decided default | Rationale |
|---|---|---|---|
| OQ-1 | Signing tech for bundles? | **Unspecified (§26.1)**; require the *verification property* + versioning now. | Don't bikeshed crypto in POC; keep the interface. |
| OQ-2 | Audit immutability mechanism? | **Append-only store + per-event hash** at POC; hash-chain/checkpoint at GA (SEC-8). | Tamper-evidence floor without heavy infra (§22). |
| OQ-3 | mTLS or TLS at POC? | **TLS floor; mTLS for subject + approval-callback channels.** | Trust-bearing channels get the stronger control first. |
| OQ-4 | PII at rest in audit `jwt_claims`? | **Hash-at-rest for `sub`/`email` with grant-gated reveal** (resolves D1 A12). | Minimizes retention-window PII exposure. |
| OQ-5 | Dual-control for admin verbs at POC? | **Audit-only at POC, approval-gated by GA** (SEC-12). | Phased; audit is the floor, prevention the target. |
| OQ-6 | Does the platform dogfood its own supply-chain controls? | **Yes — SEC-21/22** (signed images, SBOM, least-priv). | A governance product that isn't governed is indefensible. |
