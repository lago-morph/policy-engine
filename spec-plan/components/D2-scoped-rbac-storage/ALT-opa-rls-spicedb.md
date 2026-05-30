# D2 — ALT Architecture — Enforcement Mechanism for Storage Authorization

**Question this ALT explores:** §17A.5 mandates that authorization is enforced *at the storage
layer*, server-side, for every query (§17A.1: "GUI-only authorization is insufficient"). The
primary `SPEC.md` enforces this via an **application-layer mandatory query-rewrite interceptor**.
This document evaluates three genuinely different architectures for the *same* requirement and
recommends a path.

The four candidates:
- **P (Primary):** App-layer interceptor / mandatory query rewrite (bespoke).
- **ALT-A:** Database **Row-Level Security (RLS)** — push the predicate into the DB engine.
- **ALT-B:** **OPA-as-authz-layer** — externalize the decision to OPA/Rego (data filtering).
- **ALT-C:** **SpiceDB-style ReBAC** (Zanzibar) — relationship-based authorization service.

---

## 1. ALT-A — Database Row-Level Security (Postgres RLS / equivalent)

### Design
Every table holding platform objects carries the `authz` columns (`tenant`, `namespaces[]`,
`policy_domains[]`, `visibility`). A **session GUC** (`SET app.subject = '<json>'`) is set from
the D1 subject at connection checkout. RLS **policies** on each table enforce:
```sql
CREATE POLICY scope_read ON violations FOR SELECT
  USING (
    tenant = ANY (current_subject_tenants())            -- or tenants @> '{*}'
    AND namespaces && current_subject_namespaces()
    AND policy_domains && current_subject_domains()
  );
```
The DB engine appends the predicate to **every** query automatically — even ad-hoc SQL,
reporting tools, or a leaked connection string.

### Pros
- **Strongest "storage-layer" guarantee**: the predicate is enforced by the engine itself, the
  lowest possible layer. A developer who forgets the interceptor (the Primary's biggest risk)
  *still* cannot escape — RLS is not bypassable from application code (only by the table owner /
  `BYPASSRLS`).
- Counts, aggregates, joins, cursors are filtered natively — no bespoke pagination math
  (eliminates the Primary's D2-R5 leak risk by construction).
- Auditable, declarative policies; mature in Postgres.

### Cons
- **Ties the platform to an RLS-capable relational store.** §22 allows "relational/document/
  object storage"; object stores (R2/S3) and many document stores have **no RLS** — the audit
  fixtures, bundles, and replay datasets often live there. So RLS covers *part* of the surface
  only; you still need an app-layer guard for object storage → two enforcement models.
- Setting the subject GUC correctly per request is itself a choke point that can be forgotten
  (connection pooling reuse bug → wrong subject) — a different but real footgun.
- 404-vs-403-by-ID and "no existence leak in error bodies" still need app-layer handling; RLS
  gives empty result sets but the *API contract* shaping is still bespoke.
- Cross-DB-engine portability is poor; the §17A.5 predicate logic gets duplicated in SQL.

### Verdict
**Best raw guarantee for the relational subset; insufficient alone** because the platform's
evidence/bundle/replay corpus lives partly in object storage. Strong candidate as a
**defense-in-depth layer *under* the Primary interceptor**, not a replacement.

---

## 2. ALT-B — OPA-as-authz-layer (externalized decision + data filtering)

### Design
The platform already runs OPA/Rego for *policy* decisions. Reuse it for *platform* authz:
- **Decision mode:** API handler calls OPA `POST /v1/data/authz/allow` with
  `{subject, verb, object}`; OPA returns allow/deny. (Classic external authz.)
- **Filter mode (the hard part):** for list endpoints, OPA returns a **partial-evaluation
  residual** (Rego *partial eval* / "compile" API) that the data layer translates into a query
  predicate. This is exactly how Styra DAS / OPA's `compile` endpoint does data filtering.

```rego
package authz
import future.keywords
allow if {
  role_grants[input.verb][input.object.type]
  scope_match
}
scope_match if {
  input.object.tenant in input.subject.tenants
  count(input.object.namespaces & input.subject.namespaces) > 0
}
```
For lists, OPA `compile` yields conditions over `object.tenant`/`object.namespaces` that the
data layer compiles to SQL/Mongo filters — unifying decision and filtering in Rego.

### Pros
- **Single policy language** for runtime policy *and* platform authz — strong "unified policy
  semantics" alignment (§28). The 9-role matrix becomes Rego data, diffable and simulatable
  with the *same* §17 simulation framework.
- The §17A matrix and scope logic live as **policy-as-code**, testable with OPA's test tooling
  and replayable — authz changes go through the same lifecycle (§7) as everything else.
- Partial eval gives real **server-side data filtering**, satisfying §17A.5 without bespoke
  predicate code per resource.
- Naturally pluggable (§25.1) and consistent with the platform's OPA-centric architecture.

### Cons
- **Partial-eval data filtering is operationally subtle** — translating residuals to every
  store (SQL, Mongo, object-store list) is non-trivial and a known source of correctness bugs
  (an over-permissive translation = silent scope escape — the worst failure).
- Adds OPA to the **hot read path** of every query (latency, availability coupling); needs a
  sidecar/cache. For 5–50 GUI users (§22) fine, but it's more moving parts than the POC needs.
- The decision is only as good as the data OPA sees — it still needs the same `authz` metadata
  and subject; doesn't remove the metadata-modeling work.
- Debugging "why did this row leak" now spans Rego partial-eval + the translation layer.

### Verdict
**Architecturally the most "on-brand"** (everything is OPA/Rego) and the best long-term answer
for *consistency and lifecycle*, but the **partial-eval-to-query translation is the riskiest
single thing** in any of these options for a scope-escape-sensitive system. Excellent **post-POC
target**; risky as the first implementation where correctness must be obvious.

---

## 3. ALT-C — SpiceDB-style ReBAC (Zanzibar)

### Design
Model authz as a **relationship graph**. Define a schema:
```
definition user {}
definition tenant { relation member: user }
definition namespace { relation tenant: tenant; relation author: user }
definition policy {
  relation namespace: namespace
  permission view  = namespace->author + namespace->tenant->member
  permission edit   = namespace->author
  permission promote_enforce = namespace->approver
}
```
Every object→scope→subject relationship is a tuple. Reads use **`LookupResources`** ("which
policies can subject X view?") to get the allowed-ID set, or **check** per object. SpiceDB's
consistency tokens (Zookies) handle the read-after-write/staleness problem explicitly.

### Pros
- **Purpose-built for exactly this** ("can subject view object, filtered lists"); `LookupResources`
  is *designed* for server-side scoped listing — no bespoke predicate or partial-eval translation.
- Cleanly models **hierarchy and delegation** (namespace→tenant→member, NS-author delegation),
  which maps naturally to the §15.4 group/tenant-inheritance story (D1) and §17A namespace
  delegation. The D1 group→role expansion could even be *replaced* by graph relations.
- Explicit consistency model addresses the **stale-token / read-after-grant** issues that bite
  both Primary and RLS.
- Strong audit story; battle-tested at scale (overkill for POC scale, but future-proof).

### Cons
- **Heaviest new dependency** — a whole authorization database + service to run, sync, and
  keep consistent with the object store. The object's *scope facts* now live in two places
  (object store metadata **and** the relationship graph) → a **dual-write/consistency problem**
  (the classic Zanzibar "tuple drift" issue): if an object's namespace changes and the tuple
  doesn't, you get an escape.
- `LookupResources` returns IDs; you still join back to the object store and must ensure the
  store can't be queried *without* the ReBAC gate (so you *still* need a choke point) — it
  doesn't remove the storage-layer-enforcement question, it relocates it.
- Steep modeling learning curve; over-engineered for §22's 25–100 controls / 10–100 namespaces.
- ReBAC schema becomes a **second policy language** alongside Rego — cuts against §28 unified
  semantics (the opposite of ALT-B's main virtue).

### Verdict
**The "right" answer at large scale / heavy delegation**, but the dual-write consistency risk
plus a second policy language make it **disproportionate for the POC** and partly *in tension*
with the platform's OPA-unified thesis.

---

## 4. Comparison

| Dimension | P: App interceptor | A: RLS | B: OPA partial-eval | C: SpiceDB ReBAC |
|---|---|---|---|---|
| Storage-layer guarantee strength | Good (1 choke point) | **Strongest** (engine) | Good | Good (separate service) |
| Covers object storage too | **Yes** | No (relational only) | Yes | Yes (but join-back) |
| Scope-escape failure mode | forgot interceptor | forgot GUC / BYPASSRLS | bad residual translation | tuple drift / forgot gate |
| Existence-leak handling (counts/404) | bespoke (D2-R5/R6) | native sets + bespoke API shaping | bespoke | native lookup + API shaping |
| Unified w/ platform policy (§28) | No | No | **Yes (Rego)** | No (2nd lang) |
| Lifecycle/simulatable authz | No | No | **Yes (§7/§17)** | partial |
| New operational deps | none | RLS DB feature | OPA sidecar | full ReBAC DB+svc |
| Hierarchy/delegation modeling | manual | manual | manual | **native** |
| Fit for §22 POC scale | **Best** | Good | Good | Overkill |
| Correctness-obviousness (audit) | **High** | High | Medium | Medium |

---

## 5. Recommendation — layered, evolve-in-place

1. **POC / MVP: Primary (app interceptor) + ALT-A (RLS) as defense-in-depth** on the relational
   subset.
   - The interceptor is the single, *obvious-to-audit* choke point — its correctness is easy to
     demonstrate (DT-55 suite), which matters most for a scope-escape-sensitive control.
   - Add Postgres RLS *underneath* for the relational tables so a forgotten interceptor call
     still fails closed — turning the Primary's #1 risk (a bypass path) into a non-issue for the
     relational store. Object storage stays interceptor-guarded (no RLS there anyway).
   - Keep the scope predicate **storage-engine-agnostic** (a `ScopePredicate` value object) so
     it compiles to either the interceptor filter or RLS policy from one source of truth — no
     dual-maintenance.

2. **Post-POC, when authz must be lifecycle-managed and consistent with runtime policy: migrate
   to ALT-B (OPA partial-eval).**
   - This realizes §28's unified-semantics thesis: the 9-role matrix becomes Rego data,
     simulatable/replayable through §17, and changed via §7 lifecycle.
   - Migrate *behind* the same `ScopePredicate` interface so endpoints (W7+) don't change.

3. **Adopt ALT-C (ReBAC) only if** heavy cross-namespace delegation, deep tenant hierarchies,
   or large scale make graph relationships the natural model — and accept the second policy
   language + dual-write consistency cost deliberately.

**Bottom line:** the requirement is *server-side, every-query, caller-independent* enforcement.
The Primary gives the most **auditable** version of that for the POC; RLS hardens it for free on
the relational subset; OPA partial-eval is the principled long-term home that keeps authz inside
the platform's own policy engine. The `ScopePredicate` indirection is what makes this an
evolution rather than a rewrite.
