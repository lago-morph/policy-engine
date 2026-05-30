# D2 — Scoped Roles, Permissions & Storage Authorization — ADVERSARIAL REVIEW

**Red-team persona:** malicious insider (legit but over-curious NS-author) + external pen-tester.
**Target:** `SPEC.md`, `PLAN.md`, `ALT-opa-rls-spicedb.md`.

---

## 1. Attacks on the storage-layer invariant (§17A.1 / D2-R3)

**A1 — The single-choke-point assumption is a single point of failure.** The whole design rests
on "there is no code path from a handler to storage that bypasses the interceptor" (D2-R3),
enforced by "a lint/test forbids direct storage access." Lints are bypassable (`// nolint`,
reflection, a new service that imports the storage client directly, a background job, a
migration script, an analytics ETL). **The moment one component reads storage outside the
interceptor, the invariant is silently void.** The ALT's RLS recommendation exists precisely
because of this — but the Primary spec treats RLS as optional. **Defect:** for a P0-grade
invariant, the *architecture* (not a lint) must prevent bypass — RLS-under-interceptor should be
**mandatory**, not "defense-in-depth nice-to-have."

**A2 — Aggregations and analytics leak.** D2-R5 covers counts/cursors on *list* endpoints, but
the platform has a **§14 analytics engine** and **§17E reporting** that run aggregate queries
(drift counters, violation timelines, per-subject cross-tenant counters in HL-13 step 4). Do
those run through the interceptor? A "cross-tenant attempt counter" inherently aggregates across
tenants. If analytics has its own storage path (it almost certainly does, for performance), it's
an **A1 bypass by design**. **Defect:** spec is silent on analytics/reporting read paths —
likely the largest real escape surface.

**A3 — Timing / side-channel existence oracle.** D2-R6 returns 404 for out-of-scope-by-ID. But
if an in-scope-but-unauthorized object returns 403 while a truly-nonexistent ID returns 404, or
if response *latency* differs (cache hit on existing row vs. miss), an attacker can still
distinguish existence. Spec gives 404 for out-of-scope but doesn't mandate **constant-shape /
constant-time** responses across {exists-out-of-scope, not-exists}. Partial gap.

**A4 — Write-scope check vs. scope *widening* on update.** D2-R2 checks scope at *create*. What
about **update**? If `authz` metadata is "immutable after write" (D2-R1), can an attacker create
an object as `namespace-scoped` then patch `visibility:global`? Spec says authz block is
immutable but doesn't explicitly forbid a *separate* visibility-escalation path. Also: who can
*move* an object between namespaces? Gap.

## 2. Attacks on the role/permission matrix

**A5 — Additive roles + union can defeat SoD.** §2 says roles are additive and "effective
permissions = union ∩ scope." But SoD (author ≠ approver) relies on a person *not* holding both
roles. Nothing stops a subject being granted **both** `namespace-policy-author` and
`namespace-policy-approver` for the same namespace — then they author and self-approve
`policy:promote-enforce`. The matrix enforces per-role grants but **union breaks SoD**. **Defect:**
SoD must be a **mutual-exclusion constraint** ("no subject may simultaneously hold author and
approver for the same scope"), enforced at grant time (role 1 `role:grant`), not assumed.

**A6 — `Y*` global admin is unbounded.** Plat-Admin is `Y*` on *every* permission including
`mapping:edit`, `role:grant`, `config:manage`. One compromised admin token = total platform
compromise, and the only mitigation is *after-the-fact* audit (DT-54). There is **no
preventive** control on admin (no dual-control for `role:grant` or `mapping:edit`, no break-glass
distinction). For a governance/compliance product this is ironic. **Defect:** high-impact admin
verbs (`role:grant`, `mapping:edit`, `policy:promote-enforce` globally, `config:manage`) should
require **dual-control / approval** (route through D3), not merely be logged.

**A7 — Matrix ambiguity: `(own)` vs `(own NS)` vs role scope.** The matrix mixes role-layer and
scope-layer semantics in one cell (e.g. `Y(own NS)`). "Own NS" for a Developer who is *also* a
Namespace Policy Author — does `(own)` mean authored-by-subject or in-subject's-namespace? DT-67
shows a Developer with `exception:create` scoped to a namespace, not to objects they authored.
The spec should split **resource ownership** from **namespace scope** explicitly; conflating them
will produce inconsistent enforcement.

**A8 — Workflow Integrator narrowness is asserted, not enforced against escalation.** Role 9 has
`workflow:configure` only. But configuring an approval webhook means choosing the *endpoint* the
`approval.requested` payload (containing `subject.sub`, `groups`, resource info) is sent to. A
malicious Workflow Integrator points the webhook at their own collector and **exfiltrates every
approval's subject/resource data** — without ever holding `report:view`. **Defect:** webhook
*endpoints* must be allow-listed/approved; `workflow:configure` is more dangerous than the
"no read" framing implies (the data flows *out* via config).

## 3. Cross-component inconsistencies

**A9 — Boundary-crossing audit "blocking" vs availability (OQ-4).** D2 makes cross-tenant audit
a *precondition* (blocking) for global subjects. During an **incident**, the admin doing the
cross-tenant investigation (DT-54's exact scenario) is blocked if the audit sink is down — the
worst possible time. This conflicts with incident-response reality. Reconcile with a documented
break-glass that logs locally + alarms, rather than hard-blocking the responder.

**A10 — Stale-token scope vs server-side grant store (OQ-6).** OQ-6 says "server-side grant store
is authoritative; token scope validated against it for privileged ops" — but **cheap reads still
trust the token.** A deprovisioned NS-author keeps read access to their old namespace until token
expiry. For a *governance* product, "you could still read violations for 30 min after offboarding"
may be unacceptable. The boundary between "privileged op (re-checked)" and "cheap read (token-
trusted)" is undefined. **Defect:** define exactly which verbs re-check the grant store.

**A11 — D1↔D2 role contract (mirrors D1 A11).** A registry role with no mapping path is a dead
grant; a mapping role with no registry entry is a no-permission role. Neither is detected. Need a
reconciliation lint — owned jointly with D1.

**A12 — Materialized replay dataset (D2-R8) staleness/scope-at-materialization.** HL-18: the
Auditor replays a *materialized* scoped dataset. But materialization snapshots scope **at
materialization time**. If a namespace is re-scoped to a different tenant afterwards, the
auditor's snapshot may now contain rows that should no longer be in their scope — or miss rows
that should. Who re-validates the snapshot's scope at *read* time? Gap.

## 4. ALT-review cross-check

**A13 — The ALT correctly identifies that the Primary's choke point is its weakness (A1), but
the Primary spec doesn't adopt the ALT's mitigation.** This is an internal inconsistency: the
ALT *recommends* mandatory RLS-under-interceptor, the SPEC's OQ-1 makes RLS optional/"later."
Resolve by promoting RLS-under-interceptor (relational subset) into the SPEC's normative
requirements.

**A14 — OPA partial-eval (ALT-B) over-permissive-residual risk is real and under-weighted** for
analytics (A2). If the post-POC migration to OPA filtering happens, the analytics aggregate
paths (A2) are exactly where a bad residual silently leaks — the migration must include
analytics, or A2 persists across architectures.

## 5. Prioritized defect list
| ID | Severity | Defect | Fix |
|---|---|---|---|
| A2 | **Critical** | Analytics/reporting aggregate read paths likely bypass the interceptor | Route §14/§17E reads through the same scope predicate; or RLS makes it moot for relational |
| A5 | **Critical** | Additive roles break author≠approver SoD via self-grant | Mutual-exclusion constraint at grant time |
| A1 | **Critical** | Single choke point enforced only by a lint; one bypass voids the invariant | Make RLS-under-interceptor mandatory for relational store (promote from ALT) |
| A6 | **High** | Global admin verbs preventively uncontrolled (audit-only) | Dual-control / D3 approval for `role:grant`, `mapping:edit`, `config:manage` |
| A8 | **High** | Workflow Integrator exfiltrates approval payloads via webhook endpoint config | Allow-list/approve webhook endpoints; treat as data egress |
| A10 | **High** | Stale-token reads after deprovisioning undefined | Define which verbs re-check the grant store; cap read-token lifetime |
| A4 | Medium | Visibility-escalation / namespace-move after immutable create | Forbid visibility escalation + namespace move without re-authorization |
| A9 | Medium | Blocking cross-tenant audit fails the incident responder | Break-glass: local-log+alarm instead of hard block |
| A3 | Medium | Existence oracle via 403/404 shape + timing | Constant-shape/constant-time responses |
| A7 | Medium | `(own)` vs `(own NS)` matrix ambiguity | Split ownership from namespace scope in the matrix |
| A12 | Low | Materialized replay snapshot scope staleness | Re-validate snapshot scope at read time |
| A11/A13 | Low | D1↔D2 role contract; SPEC/ALT RLS inconsistency | Reconciliation lint; adopt RLS in SPEC |

**Verdict:** The matrix and the storage-filter design are strong and the DT-55 pen-test discipline
is exactly right. But three things will cause real escapes/incidents: **(A2)** the analytics and
reporting read paths almost certainly skirt the choke point; **(A5)** additive roles silently
defeat the SoD the whole approval story depends on; and **(A1/A13)** a P0 invariant is guarded by
a lint when the project's own ALT already shows the fix (mandatory RLS). Adopt RLS, make SoD a
hard mutual-exclusion, and bring analytics/reporting under the predicate.
