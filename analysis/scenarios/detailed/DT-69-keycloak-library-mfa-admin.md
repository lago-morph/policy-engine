# DT-69 — Keycloak library: require MFA for admin login from unusual network

**Personas:** Marcus (Platform Security Engineer)
**Spec sections:** §17D.3 Keycloak Library (User login → Admin login from unusual network requires MFA), §17B.2 Decision Outcomes, §17.3 Audit-Driven Simulation Requirements
**Type:** Low-level
**Pre-condition:** Marcus has installed the Keycloak library's authentication-flow integration (§17D.3): a custom authenticator that calls an OPA sidecar PDP at the `User login` decision point. The OPA bundle includes the Rego package `governance.keycloak.adminlogin` bound to control `IAM-ADMIN-MFA-001`. The Keycloak realm `platform` has a configured network allow-list (CIDRs for office and VPN), and admin users are members of group `/platform-admins`. Event listener forwards login events to Privateer.
**Trigger:** A user in `/platform-admins` initiates a login to the `platform` realm from an IP outside the allow-list.

## Steps
1. **Login attempt.** Admin user `marcus@example.com` authenticates with username+password to the Keycloak `platform` realm from source IP `203.0.113.42` (not in allow-list).
2. **Authenticator calls PDP.** The custom authenticator (§17D.3 real-time hook = "Authentication flow / event listener") builds a normalized input containing `user.sub`, `user.groups: ["/platform-admins"]`, `client_id`, `realm`, `source_ip`, `risk_level` claim from the device-risk provider, `user_agent`, and `auth_session_id`. It POSTs to the OPA PDP at `v1/data/governance/keycloak/adminlogin/decision`.
3. **Rego evaluates.** The Rego computes `is_admin := "/platform-admins" in input.user.groups`, `is_unusual := not net.cidr_contains_any(data.allowlist.cidrs, input.source_ip)`, and `elevated_risk := input.risk_level >= 50`. With `is_admin == true` and (`is_unusual == true` or `elevated_risk == true`), the decision is `require_mfa` (mapped from §17B.2 — implemented as `suspend_pending_approval` style step-up: the login is not denied but cannot complete without an additional factor).
4. **Authenticator branches.** Receiving `{decision: "require_mfa", reasons: ["unusual_network"], control_id: "IAM-ADMIN-MFA-001"}`, the authenticator pushes the session into the MFA sub-flow (TOTP or WebAuthn), bypassing the default "remember device" shortcut for this session.
5. **User completes MFA.** Marcus approves the WebAuthn prompt on his registered key. The authenticator marks the factor satisfied and completes login. A token is issued.
6. **Decision logged.** The event listener emits a Keycloak `LOGIN` event plus a §17.3-compliant decision record: `control_id=IAM-ADMIN-MFA-001`, `decision=require_mfa`, `subject.sub`, `subject.groups`, `source_ip`, `risk_level`, `mfa_method=webauthn`, `outcome=success`, `correlation_id=<auth_session_id>`, `policy_version`. Privateer ingests it.
7. **Negative case — login from allow-listed IP.** Same admin from office CIDR: Rego returns `decision=allow`; no MFA step-up; decision log records `decision=allow`, `reasons=[]`, same control ID.

## Success criteria (testable)
- Admin login from non-allow-listed IP triggers the MFA sub-flow before token issuance.
- Decision record for the unusual-network login has `decision=require_mfa` and `reasons` containing `unusual_network` (or `elevated_risk`).
- Decision record carries all §17.3 fields: subject, source, policy_version, correlation_id, control_id, full normalized input.
- A non-admin user from the same unusual IP receives `decision=allow` (policy is admin-scoped).
- Admin login from allow-listed CIDR records `decision=allow` with no MFA step-up.
- Failed MFA at step-up records `outcome=failure` and does not issue a token.

## Flowchart

```mermaid
flowchart TD
  L[Admin login attempt §17D.3] --> N[Authenticator builds input]
  N --> PDP[OPA: governance.keycloak.adminlogin]
  PDP --> D{is_admin AND (unusual_network OR elevated_risk)?}
  D -- no --> A[decision=allow → token]
  D -- yes --> M[decision=require_mfa §17B.2]
  M --> F{MFA factor satisfied?}
  F -- yes --> T[issue token, log success]
  F -- no --> X[deny session, log failure]
  A --> LOG[§17.3 decision log to Privateer]
  T --> LOG
  X --> LOG
```

## Notes
`require_mfa` is rendered via the §17B.2 `suspend_pending_approval` family as a synchronous step-up rather than a long-running approval. The Rego must not see the password — only the post-primary-auth subject and context.
