# D1 — Keycloak / JWT Integration & Mapping Layer — ADVERSARIAL REVIEW

**Red-team persona:** hostile penetration tester + skeptical staff identity engineer.
**Target:** `SPEC.md`, `PLAN.md`. **Mandate:** find what breaks in production.

---

## 1. Attacks on the trust boundary

**A1 — `alg:none` / algorithm confusion.** Spec mitigates (D1-R3) but the *default* must be a
per-issuer **allow-list**, not a deny-list. If a new IdP is onboarded and someone forgets to
pin its algs, the safe default must be *reject unknown alg*, not *accept*. **Defect:** spec
says "pin acceptable algorithms per issuer" but doesn't state the global default when an
issuer has no pin. → **Must specify: no pin ⇒ reject.**

**A2 — Issuer spoofing via `iss` substring.** Allow-list must be **exact-match** on `iss`,
not prefix/substring. An attacker registering `https://kc.example.evil.com/realms/platform`
must not match `https://kc.example/...`. Spec says "allow-list" but not match semantics. Gap.

**A3 — JWKS poisoning / key-confusion.** If JWKS is fetched over a compromised channel or the
`kid` is attacker-chosen, verification can be subverted. Spec relies on cached JWKS but doesn't
require TLS-pinned/verified JWKS endpoint or `kid`→key binding validation. Gap.

**A4 — Audience confusion.** `aud` may be an array; "aud includes platform" must reject tokens
minted for a *different* client that merely also lists `platform`. Need: `azp`/client check,
not just membership. Partial gap.

**A5 — Token lifetime vs approval/role staleness.** §15.3 `deployment_approval` and expanded
`roles` are baked into a token at issuance. A revoked role or expired approval is still valid
until `exp`. OQ-6 correctly says the approval claim is non-authoritative — **but the same
logic must apply to `roles`/`namespaces`.** If D2 trusts the token's `roles` for up to the
token lifetime, a deprovisioned admin keeps admin until token expiry. **Defect:** spec doesn't
mandate a max token lifetime or a revocation/introspection path for high-privilege roles.

## 2. Attacks on the mapping layer

**A6 — Mapping config as privilege escalation.** §8 notes a mapping edit *is* a privilege
change (group→role expansion). But the spec gates mapping edits behind "Platform Governance
Admin" — the **same** role that the expansion can mint. A compromised or rogue admin can write
a hierarchy rule that grants themselves `role:org-finance` everywhere. There is no
**four-eyes / approval gate on mapping changes**. → Recommend: mapping changes route through
D3 approval (a mapping edit is an approval-gated decision), or at minimum dual-control.

**A7 — Determinism vs external-data drift.** D1-R6 claims determinism, but transforms like
`tenant_inherit`/`lookup` use snapshots that change. "Same token + same mapping version ⇒
identical subject" only holds if the **snapshot digest is part of the mapping version**. Spec
mentions this in §2.5 but R6 doesn't reference the snapshot. A replay (§17.3) done a week later
with a refreshed snapshot silently diverges. **Defect:** bind snapshot digest into the mapping
version identity and into `claim_provenance` for replay fidelity.

**A8 — `expand_group_hierarchy` cycles / explosion.** Nested groups can form cycles or fan out
to thousands of roles, bloating tokens past Keycloak limits (the very problem DT-37 fixes).
Spec's `/validate` checks "cycles" but sets no **bound on emitted role-set cardinality**. Gap.

**A9 — Sentinel `unknown` masks real failures.** D1-R7 maps unrecognized values to `unknown`
so policies fail closed. But if a *transform table* is wrong (typo'd `tier2`→`medium`),
everyone silently becomes `unknown` and risk-gating collapses to "deny all" or "treat as
lowest". Dry-run (DT-35) surfaces distribution, but there's no **alert threshold** ("if
>X% unknown, block promotion"). Operational gap.

**A10 — Conflict resolution hides multi-IdP identity collisions.** `prefer_first_non_empty`
across `tenant`/`org_id` can mask the case where two IdPs disagree on tenant for the *same*
`sub` (account linking bug). Spec marks unresolved as `degraded` but `prefer_first_non_empty`
**never** leaves it unresolved — it just picks one. → A *disagreement* (both non-empty, values
differ) must be distinct from *absence* and should be `degraded`, not silently first-wins.

## 3. Cross-component inconsistencies

**A11 — D1↔D2 role-name contract is informal.** §5.3 says emitted roles must be registered in
D2's registry, but nothing **enforces** it. A mapping emitting `role:org-finance` with no
registry entry yields a role with *no* permissions (fail-closed, fine) — but the reverse, a
registry role with no mapping path, is an undetectable dead grant. Need a reconciliation lint
across D1 mappings ↔ D2 registry.

**A12 — `jwt_claims` audit block vs PII redaction.** W7 emits `jwt_claims` to C2; DT-57 redacts
`subject.sub`/`email`. If D1 writes raw `email` into the audit event and redaction is a
*downstream export* concern, the **at-rest** audit store holds PII for the full retention
window. Spec doesn't say whether `jwt_claims` is stored raw or hashed at rest. Privacy gap
(GDPR/Schrems). → Decide: store hashed + reversible-only-with-grant, or accept raw-at-rest and
document it.

**A13 — `normalization_status=incomplete` propagation.** D1 returns `incomplete`, but does the
**audit event** record that the decision was made on an incomplete subject? §17.3 marks
*simulation* incomplete; here it's a *live* decision. If an admission allow happens with an
incomplete subject (because the missing claim wasn't required by *that* policy), is that
defensible later? Need: persist `normalization_status` + `missing_required` into every audit
event, not just the API response.

## 4. Failure-mode gaps

**A14 — JWKS fail-closed is a self-DoS.** D1-R1 + §7 reject all tokens if JWKS expires and the
endpoint is down. A Keycloak outage now takes down *the entire platform's authn*, including the
admins who would fix it. Correct for security, dangerous for availability. → Need a documented
**break-glass** (static signed-key bundle, longer grace with alarms) and explicit RTO.

**A15 — Hot-path purity vs. revocation.** D1-R10 forbids synchronous network calls, but
revocation checking (A5) inherently needs a lookup. These requirements **conflict**. Resolve:
either accept token-lifetime-bounded staleness (and shorten lifetimes for privileged roles) or
carve a revocation cache exception. Spec must pick.

## 5. Prioritized defect list
| ID | Severity | Defect | Fix |
|---|---|---|---|
| A6 | **Critical** | No four-eyes on mapping edits; admin self-escalation via hierarchy | Route mapping changes through D3 approval / dual-control |
| A5/A15 | **Critical** | Token-lifetime staleness for revoked roles/approvals; no revocation path | Cap privileged-role token lifetime; define revocation cache exception to R10 |
| A1/A2 | **High** | Default-accept on unpinned alg; non-exact `iss` match | No pin ⇒ reject; exact-match `iss` |
| A7 | **High** | Snapshot drift breaks replay determinism | Bind snapshot digest into mapping-version identity + provenance |
| A12 | **High** | PII (`email`/`sub`) raw at rest in audit for full retention | Decide hash-at-rest vs documented raw + grant-gated reveal |
| A10 | Medium | Multi-IdP value disagreement silently first-wins | Disagreement ⇒ `degraded`, distinct from absence |
| A13 | Medium | Incomplete-subject live decisions not recorded as such | Persist `normalization_status` into audit events |
| A8/A9 | Medium | No bound on role-set explosion; no `unknown`-rate promotion gate | Cardinality cap; promotion blocks above unknown threshold |
| A3/A4 | Medium | JWKS channel/kid binding; aud+azp confusion | TLS-pin JWKS, validate kid binding, check azp |
| A11 | Low | Informal D1↔D2 role-name contract | Reconciliation lint mappings↔registry |
| A14 | Low | JWKS fail-closed self-DoS | Break-glass signed-key bundle + RTO |

**Verdict:** The mapping-layer indirection is sound and the scenarios are well-supported, but
the spec under-specifies **(1) the security defaults at the verification boundary**,
**(2) staleness/revocation of privileged claims**, and **(3) that editing the mapping is itself
a privileged, approval-worthy act.** A6 and A5 are the ones that will cause a real incident.
