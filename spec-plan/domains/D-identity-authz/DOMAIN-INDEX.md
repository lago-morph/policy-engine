# Domain D — Identity, Authz & Security — INDEX

**Domain lead deliverable.** Entry point for the four components covering identity ingestion,
authorization (incl. the storage-layer enforcement requirement), approval-gated decisions, and
the platform security baseline.

**Spec sections owned:** §15 (Keycloak/JWT), §17A (Scoped roles, permissions, storage authz),
§17B (Approval-gated decisions), §23 (Security requirements). Cross-cuts §16.3, §17C.6, §8, §13.

---

## 1. Components

| ID | Title | Spec § | Docs | ALT | Status |
|---|---|---|---|---|---|
| **D1** | Keycloak/JWT integration & mapping layer | §15, §17A.4 | [SPEC](../../components/D1-keycloak-jwt/SPEC.md) · [PLAN](../../components/D1-keycloak-jwt/PLAN.md) · [ADVERSARIAL](../../components/D1-keycloak-jwt/ADVERSARIAL-REVIEW.md) | — | AUTHORED |
| **D2** | Scoped roles, permissions & storage authorization | §17A | [SPEC](../../components/D2-scoped-rbac-storage/SPEC.md) · [PLAN](../../components/D2-scoped-rbac-storage/PLAN.md) · [ADVERSARIAL](../../components/D2-scoped-rbac-storage/ADVERSARIAL-REVIEW.md) | [ALT](../../components/D2-scoped-rbac-storage/ALT-opa-rls-spicedb.md) | AUTHORED |
| **D3** | Approval-gated decisions | §17B, §17C.6 | [SPEC](../../components/D3-approval-gated/SPEC.md) · [PLAN](../../components/D3-approval-gated/PLAN.md) · [ADVERSARIAL](../../components/D3-approval-gated/ADVERSARIAL-REVIEW.md) | — | AUTHORED |
| **D4** | Security requirements | §23 | [SPEC](../../components/D4-security/SPEC.md) · [PLAN](../../components/D4-security/PLAN.md) · [ADVERSARIAL](../../components/D4-security/ADVERSARIAL-REVIEW.md) | — | AUTHORED |

Robustness rule met: every component dir has SPEC.md + PLAN.md (+ ADVERSARIAL, + ALT for D2).
All authored by the domain lead (no subagent runtime available in this environment).

---

## 2. One-line purpose
- **D1** — Turn OIDC/JWT from one-or-many IdPs into one canonical normalized subject; mapping
  layer is the single indirection so claim evolution is config, not fleet-wide policy edits.
- **D2** — The 9-role × permission matrix and **storage-layer** scope enforcement: every query
  filtered server-side, no GUI bypass, no scope escape.
- **D3** — Non-terminal decisions (`suspend_pending_approval`, `require_async_check`) resolved
  via approval CRDs + webhooks; the K8s-admission "deny-with-approval-required + retry" pattern.
- **D4** — The numbered security baseline (SEC-1..23) the whole platform inherits, incl. the
  platform's own supply chain.

---

## 3. Spec-section cross-reference
| Spec § | Component | Key requirement |
|---|---|---|
| §15.2 / §17A.4 | D1 | Required claims; normalized subject |
| §15.4 | D1 | Mapping layer: transform/normalize/alias/inherit/expand/hierarchy |
| §17A.2 | D2 | 9 roles |
| §17A.3 | D2 | Permission primitives + scope dimensions |
| §17A.5 | D2 | Storage-level access controls (the hard requirement) |
| §17A.1 | D2/D4 | "GUI-only authorization is insufficient" → SEC-9 |
| §17B.2 | D3 | 6 decision outcomes (2 non-terminal) |
| §17B.3 | D3 | Workflow webhook schema |
| §17B.4 | D3 | Enforcement-point behavior; K8s admission constraint |
| §17C.6 | D3 | `PolicyApprovalRequest` / `PolicyException` CRDs |
| §23.1 | D4 | 7 security areas → SEC-1..23 |

---

## 4. Scenario coverage (HL / DT)
| Scenario | Components | What it exercises |
|---|---|---|
| DT-35 | D1 | Add a claim via mapping layer (no Keycloak realm change) |
| DT-36 | D1 | Normalize `tenant`/`org_id` across two IdPs |
| DT-37 | D1 | Decommission an obsolete claim (prove-then-remove) |
| DT-38 | D1 | Group hierarchy → role expansion at issuance |
| DT-53 | D1/D2 | Grant Namespace Policy Author via Keycloak |
| DT-54 | D2 | Audit a global admin's cross-tenant boundary crossing |
| DT-55 | D2 | Storage-layer scope enforcement pen-test (the crown jewel) |
| DT-56 | D2 | Onboard external Auditor (read-only scoped) |
| DT-57 | D2 | Evidence export with redacted JWT subjects |
| DT-58 | D3 | `suspend_pending_approval` for a prod deploy |
| DT-59 | D3 | K8s admission deny-with-approval-required |
| DT-60 | D3 | Jenkins pipeline pauses for approval |
| DT-61 | D3 | GitOps controller suspends sync |
| DT-62 | D3 | Approval expiry → re-authorization |
| DT-65 | D3 | `PolicyApprovalRequest` CRD lifecycle |
| DT-67 | D3 | `PolicyException` CRD lifecycle |
| HL-04 | D2/D3 | Developer onboarding through policy gates |
| HL-10 | D3 | Production deploy approval with break-glass exception |
| HL-13 | D1/D2 | Cross-tenant access attempt detected and audited |
| HL-16 | D1 | Keycloak IdP change drives JWT claim evolution |
| HL-18 | D2/D4 | Auditor independent re-execution (signed bundles, evidence integrity) |
| HL-19 | D3 | Policy exception expiry and re-authorization |

---

## 5. Sibling docs
- [DOMAIN-SUMMARY.md](DOMAIN-SUMMARY.md) — shared model, hardest decisions, open questions.
- [DOMAIN-ADVERSARIAL.md](DOMAIN-ADVERSARIAL.md) — reconciliation of per-component red-team
  findings + contradictions *between* D components.
