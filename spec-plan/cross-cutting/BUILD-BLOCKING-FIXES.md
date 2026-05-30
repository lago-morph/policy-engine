# Build-Blocking Fixes — Day-One Checklist

**Source of truth:** `CROSSCUT-ADVERSARIAL.md §4` (the build-blocking subset) + the overrides in
`INTER-DOMAIN-CONTRACTS.md` (OV-1..OV-5) + the reconciliations in `DATA-MODEL.md` (R1..R7).
**What this resolves:** the #1 finding of the cross-cut wave — **C2 `c2.audit-event/1.0` was frozen
too early and baked in modeling errors.** This is the actionable work order a team picks up day one.

**The gate:** **un-freeze C2 → `v1.0-rc`** (see `C2-v1.0-rc-RECONCILED.md`), land BB-1..BB-9 **in
order**, verify acceptance criteria, then **re-freeze to final `c2.audit-event/1.0`**. Everything in
Domains C/E/F/A builds against these contracts; each fix gets dramatically more expensive once
consumers bake the old shape in.

**Foundation contracts that must (re-)freeze together after this list:**
C2 audit schema · B4 action taxonomy · D2 scope predicate · shared `replay_completeness` semantics.

---

## Dependency order (the critical path)

```
BB-1 (XD-3  split disposition/obligations) ──┐
                                             ├─▶ BB-2 (XD-11 one taxonomy) ──▶ regenerate C2 enums
BB-3 (XD-1  external-data VALUE capture) ────┤
BB-4 (XD-2  policy-relative completeness) ───┤
BB-5 (XD-13 pin catalog) ──▶ feeds ──▶ BB-6 (XD-12 jwt_claims_completeness scorer)
BB-7 (XD-8  correlation_id + governance_transaction_id)
BB-8 (XD-5  D2 scope-predicate library)   ── gates D2 contract (parallel to C2)
BB-9 (XD-9  CRD ownership ratification)   ── gates B4/F2/F1 (parallel, low-effort)
```

**Order rationale:** BB-1 first — it is the contract change that hardens fastest and poisons the
most downstream consumers (E1/C3/C4/C5/F4). BB-2 depends on BB-1 (enums regenerate *from* the split
taxonomy). BB-5 (pin catalog) must precede BB-6 (the JWT scorer reads the pinned catalog) and
sharpens BB-3/BB-4 (volatility marking, read-set comparison). BB-8 and BB-9 are independent of the
C2 field changes and can run in parallel, but each gates its own foundation contract
(D2 predicate / B4 CRD authority).

| Order | Ticket | XD | Gates which contract |
|---|---|---|---|
| 1 | BB-1 | XD-3 | C2 schema + B4 taxonomy |
| 2 | BB-2 | XD-11 | C2 schema + B4 taxonomy |
| 3 | BB-3 | XD-1 | C2 schema + `replay_completeness` semantics |
| 4 | BB-4 | XD-2 | `replay_completeness` semantics (`DATA-MODEL`) |
| 5 | BB-5 | XD-13 | C2 schema (§4.2 catalog) |
| 6 | BB-6 | XD-12 | C2 schema (D1↔C2 contract) |
| 7 | BB-7 | XD-8 | C2 §6 correlation contract |
| 8 | BB-8 | XD-5 | D2 scope-predicate contract |
| 9 | BB-9 | XD-9 | B4 CRD schema authority |

---

## BB-1 — Split the action model: `disposition` + `obligations[]`

- **XD:** XD-3 · **Severity:** CRITICAL · **Order:** 1
- **Owning component(s):** **B4** (taxonomy/model) + **C2** (schema fields), jointly. Downstream
  adopters: B1 (drop `allowed` boolean), E1 (normalize to disposition+obligations).
- **Unblocks contract:** Contract 5 (Action-Model, OV-3) + Contract 1 (Audit Event).
- **Problem:** C2 field 9 `decision` is a flat enum that conflates the **exclusive verdict** with
  **co-occurring effects**; `mutate` sits as a sibling of `deny`, so a deny-that-mutates or
  allow-that-mutates-and-notifies has no representation. Already frozen — must un-freeze.
- **Acceptance criteria:**
  1. C2 schema adds `disposition` (NEW field 37, C, exclusive: `allow | deny | warn |
     suspend_pending_approval | require_approval | unknown`) and `obligations[]` (NEW field 38, O,
     set from the closed B4 obligation vocabulary).
  2. `decision` (field 9) retained as a **deprecated alias** of `disposition`; for `policy.decision`
     it MUST equal `disposition`; `decision:mutate` is **never emitted** (becomes
     `disposition:allow` + `obligations:[mutate]`).
  3. `action_performed` (field 15) re-scoped to "realized effect of an obligation"; `mutation_diff`
     condition keyed off the `mutate` obligation.
  4. B4 splits its 13-action enum into one exclusive disposition axis + an obligation set and
     **drops the single linear precedence as a cross-axis rule** (precedence applies only *within*
     the disposition axis when multiple controls fire).
  5. B1 drops the derived `allowed` boolean (or makes it the 3-state disposition).
  6. Invariant **N-C2-DISP** asserted; a test rejects any event with `decision:mutate` or with an
     `obligations[]` value outside the closed set.
  7. An "allow + mutate + notify" event round-trips through normalize→store→query and renders all
     three (the v1.0 representability gap is closed).
- **Done when:** the reconciled field table (`C2-v1.0-rc-RECONCILED.md §6.2`) is in effect and the
  representability test (criterion 7) passes.

## BB-2 — One canonical action taxonomy; regenerate C2 enums

- **XD:** XD-11 · **Severity:** HIGH · **Order:** 2 (depends on BB-1)
- **Owning component(s):** **B4** (taxonomy owner) + **C2** (regenerate enums) + **E3** (map product
  verbs onto it).
- **Unblocks contract:** Contract 5.
- **Problem:** four closed, mutually-incompatible vocabularies — B4's 13, C2's `decision`, C2's
  `action_performed`, and E3's verbs (`detect/alert/require review/notify/attach evidence/require
  MFA/fail build/clear hold`). A value legal in one is illegal in another ⇒ guaranteed runtime
  mis-bucket/reject.
- **Acceptance criteria:**
  1. B4 owns **one** canonical taxonomy; C2's `disposition`/`obligations`/`action_performed` are
     **derived from it** (one vocabulary, generated once).
  2. E3 product verbs **map onto** the canonical set (`detect→notify`, `alert→notify`,
     `fail build→deny`, `attach evidence→annotate`, `require review→require_approval`,
     `require MFA→require_approval`-obligation, `clear hold→disposition transition`); E3 invents no
     new verbs.
  3. The canonical reconciliation table (`C2-v1.0-rc-RECONCILED.md §1.3`) is the single mapping.
  4. R-B4-5 (closed obligation vocabulary) holds: an ad-hoc/unknown action is rejected at ingest.
  5. E1 outcome normalization and E2 by-action graph rendering no longer reject any "foreign" verb.
- **Done when:** a fixture carrying each E3 verb normalizes into the canonical disposition+obligation
  set without loss or rejection.

## BB-3 — External-data **value** capture = MUST (flagship replays `complete`)

- **XD:** XD-1 · **Severity:** CRITICAL · **Order:** 3
- **Owning component(s):** **C2** (store + required-ness + CAS) with **B1/B5** (capture at source).
- **Unblocks contract:** Contract 1 + `replay_completeness` semantics.
- **Problem:** C2 §8.3 *claims* to retain external-data values, but field 27 `value_ref` was
  OPTIONAL and §5.2 listed "re-resolvable ⇒ best_effort" as a normal path — a self-contradiction
  that legitimized never capturing the value, so the flagship image-signing example replayed
  `best_effort`, not `complete`.
- **Acceptance criteria:**
  1. `external_data_refs[].value_ref` (field 27) becomes **conditionally required** whenever the
     pinned catalog marks the provider **volatile/non-re-resolvable** (signature status, CVE feed).
  2. B1/B5 decision evidence captures the external-data **value** returned at decision time (not
     just version/digest); C2 stores it in CAS (§8.4) and points `value_ref` at it.
  3. C2 **owns** the content-addressed external-data snapshot store.
  4. §5.2's "re-resolvable ⇒ best_effort" path is **narrowed to catalog-marked stable providers**;
     a volatile provider with no captured value ⇒ **`insufficient`** (reason
     `external_data_value_uncaptured:<provider>`), never `best_effort`. Invariant **N-C2-EDV**.
  5. New reason codes `external_data_value_uncaptured:<provider>` added to the §5.5 vocabulary.
  6. **Acceptance demo:** the canonical Gatekeeper image-signing deny replays **`complete`**
     end-to-end (see `C2-v1.0-rc-RECONCILED.md §8`).
- **Done when:** the flagship example scores `complete`, and a volatile-provider event with no
  captured value scores `insufficient` (not `best_effort`).

## BB-4 — `replay_completeness` is policy-relative; stored flag is a lower bound

- **XD:** XD-2 · **Severity:** CRITICAL · **Order:** 4
- **Owning component(s):** **C2** (semantics + `DATA-MODEL.md`) + **E1** (recompute) + **F1/F4**
  (adopt one vocabulary).
- **Unblocks contract:** shared `replay_completeness` semantics (Contract 6 §6.3/§6.4).
- **Problem:** completeness is computed in ≥5 places (C2 store, E1 per-bundle, F1 jobs, F4 deltas,
  E3 static notes). The *same* event can be `complete` for the deciding bundle but `insufficient`
  for a newer simulated bundle. C2 §5 asserts "normalizer and E1 must agree" without saying inputs
  are bundle-relative — an agreement false-by-construction the moment the bundle changes.
- **Acceptance criteria:**
  1. C2 §5 amended: the stored `replay_completeness` is a **per-event lower bound computed against
     the deciding bundle**. Codified in `DATA-MODEL.md` (R1 / CI-6).
  2. E1 **recomputes** per simulated bundle; that recomputation is **authoritative for the run**
     (E1 introspects bundle input paths via Rego metadata / `opa inspect`, intersects with fixture
     fields). Invariant **N-C2-LB**.
  3. **Conservative-label-wins** on divergence (`insufficient > best_effort > complete`).
  4. A live-vs-replay divergence on a stored-`complete` event is recorded as a **finding**, not an
     accepted tolerance (couples XD-15; ends the "≥95% tie-out hides non-determinism" problem).
  5. **One vocabulary** `complete | best_effort | insufficient` across C2/E1/F1/F4 (R1 / OV-1);
     `partial` is the deprecated alias only, normalized in E1's `ReplayEventV1` adapter.
- **Done when:** an event that is `complete` for bundle v12 and `insufficient` for v13 is reported
  honestly per-run, and the divergence test fires a finding rather than swallowing it.

## BB-5 — Pin the policy-dependency catalog per `policy_version`

- **XD:** XD-13 · **Severity:** CRITICAL · **Order:** 5 (precedes BB-6)
- **Owning component(s):** **C2** (§4.2 pin + reason code) + **B1** (Rego-metadata catalog
  production).
- **Unblocks contract:** Contract 1 (the honesty edifice — `complete`/equivalence/coverage all rest
  on the catalog being current).
- **Problem:** `complete`, input-equivalence (C3-A2), coverage and `jwt_claims_completeness` all
  trust the policy-dependency catalog blindly. A single stale catalog yields *confidently-wrong*
  `complete` — the worst outcome: a trusted label that is wrong because the catalog didn't list an
  input the policy actually consulted.
- **Acceptance criteria:**
  1. Catalog version pinned **per `(policy_ref, policy_version)`** and carried in `catalog_version`
     (NEW field 41), **bound into `content_hash`**. Invariant **N-C2-CAT**.
  2. At runtime, the bundle's **observed** external-data reads / claim reads are compared against
     the catalog; on mismatch the event is **downgraded to `best_effort` with reason
     `catalog_stale`** (added to the §5.5 vocabulary).
  3. C3 input-equivalence hashing derives its relevance set from the **same pinned** catalog.
- **Done when:** a fixture whose policy reads an input absent from its pinned catalog is downgraded
  to `best_effort:catalog_stale`, never `complete`.

## BB-6 — Wire `jwt_claims_completeness` producer→scorer→consumer

- **XD:** XD-12 · **Severity:** HIGH · **Order:** 6 (depends on BB-5)
- **Owning component(s):** **D1** (emit captured-claim set) + **C2** (scorer + contract) +
  **C3** (consumer, DT-31).
- **Unblocks contract:** Contract 1 + Contract 2 (the D1↔C2 seam).
- **Problem:** field 19 `jwt_claims_completeness` gates `complete` (§5.1) and C3 DT-31 detects on it,
  but **no one set it** — D1 never emitted it and the normalizer can't compute it without both the
  catalog and D1's captured-claim set. Risk: silent false-`complete` events that dropped a
  policy-consulted claim.
- **Acceptance criteria:**
  1. D1 emits **`jwt_claims_captured[]`** (NEW field 39) — the exact claim names captured at
     decision time.
  2. C2 normalizer **computes** `jwt_claims_completeness` by diffing `jwt_claims_captured` against
     the pinned catalog's required-claim list (BB-5): captured ⊇ required ⇒ `full`; missing required
     ⇒ `partial`; Keycloak-reconstructed ⇒ `reconstructed`. Invariant **N-C2-JWTC**.
  3. **Catalog absent ⇒ `partial`, never `full`** (honesty tenet, mirrors D-C2-06).
  4. The Keycloak token-issuance-reconstruction path explicitly stamps `reconstructed`.
  5. `jwt_claims_completeness=partial` (field 19) stays **distinct** from replay `best_effort`
     (R1); a `partial`/`reconstructed` JWT caps `replay_completeness` at `best_effort`
     (reason `jwt_partial`/`jwt_reconstructed`), never the reverse.
  6. C3 DT-31 JWT-drift detects against the scorer-computed field.
- **Done when:** an event that captured only a subset of required claims is scored `partial` (and
  `best_effort`), and one that captured all is `full`; no event reaches `full` with the catalog
  absent.

## BB-7 — `correlation_id` once + retry-stable `governance_transaction_id`

- **XD:** XD-8 · **Severity:** HIGH · **Order:** 7
- **Owning component(s):** **C2** (contract + field, §6) + **B4** (mint) + **B5** (propagate);
  D3 anchors the id to the `PolicyApprovalRequest` identity.
- **Unblocks contract:** Contract 6 (`correlation_id` / replay), OV-4.
- **Problem:** Kubernetes mints a fresh AdmissionReview UID per request, so an approval
  deny→approve→**retry** flow gets two different `correlation_id`s — fragmenting the governance
  transaction; C4 sees the approved retry as an unpaired (bypass-looking) event. C2 §6 had no
  transaction-id-that-survives-retry.
- **Acceptance criteria:**
  1. `correlation_id` (field 3) defined **once in C2 §6** as the **retry-stable logical-flow join
     key**; approval flows anchor it to the `PolicyApprovalRequest` identity (CR name derived from
     `(controlId, resourceRef, requestedBy)`); ordinary flows use the AdmissionReview UID.
  2. **`governance_transaction_id`** (NEW field 40) added — minted by B4 at approval-request time,
     propagated by B5 on the retry, carried by C2 on every event in the flow.
  3. The **per-admission AdmissionReview UID is demoted** to
     `engine_context.gatekeeper.admission_review_uid` (per-event), never the cross-event join key
     for approval flows. Invariant **N-C2-TXN**.
  4. ids are **cluster-scoped** to defeat federated collision (couples XD-19).
  5. The headline deny→approve→deploy flow reconstructs **end-to-end**; C4 no longer flags the
     approved retry as unpaired.
- **Done when:** a deny + its approved retry share one `correlation_id` / `governance_transaction_id`
  and the C4 correlation-members view shows them as one transaction.

## BB-8 — D2 row+field-level scope-predicate library; analytics/reporting route through it

- **XD:** XD-5 · **Severity:** CRITICAL · **Order:** 8 (parallel; gates the D2 contract)
- **Owning component(s):** **D2** (predicate library + minimum storage contract) + **F2** (minimum
  storage contract) + **C3/C5** (route aggregate reads through it) + **C2/E1/F1** (link it).
- **Unblocks contract:** Contract 3 (`ScopePredicate`, D2-R5).
- **Problem:** storage-layer authz is mandated (§17A.1/§23) but storage is deferred (§26.1/§22.2);
  the §14 analytics and §17E reporting **aggregate across scopes by nature** and likely **bypass the
  D2 interceptor** — voiding the core security invariant for the most data-rich queries. Today it is
  enforced by a lint, not a control.
- **Acceptance criteria:**
  1. D2 ships a single **scope-predicate library (row AND field level)** that C2's query API
     (N-C2-401), F1 REST, F2 controllers, and the **C3 analytics + C5 reporting read paths** all
     **link** — never reimplement (resolves the triple-impl drift).
  2. **D2-R5 enforced:** counts, aggregates, histogram buckets, and pagination cursors are computed
     over the **filtered** set; an aggregate/coverage-matrix read is a read — out-of-scope rows
     never leak through a total or cursor.
  3. F2 defines a **minimum storage contract** (scope columns, append-only/versioned audit, content
     hashing); "ordinary storage" ⇒ "ordinary storage that supports the minimum contract"; for
     relational, **RLS-under-interceptor** promoted from ALT to mandatory.
  4. D2-R3/R4/R6 hold: no path to storage bypasses the rewrite; out-of-scope by filter ⇒ 403-no-data,
     by id ⇒ 404.
- **Done when:** a cross-tenant analytics aggregate run by a scoped subject returns counts over only
  the in-scope rows, verified by test, with no bypass path.

## BB-9 — Ratify CRD ownership rule in B4 SPEC

- **XD:** XD-9 · **Severity:** HIGH · **Order:** 9 (parallel; low-effort, gates B4 authority)
- **Owning component(s):** **B4** (schema authority) ratifies; **F2** (controllers + 3 new CRDs),
  **F1** (REST projection), **E1/E3** (their controllers/instances) reference.
- **Unblocks contract:** Contract 4 (CRD schema vs controller vs REST), OV-2.
- **Problem:** the §17C.6 CRD surface is claimed by B4 (schema), F2 (controllers + 3 new CRDs), F1
  (REST), and E1/E3 (controllers) — three-to-four owners, one surface ⇒ drifting schemas.
- **Acceptance criteria:**
  1. B4 SPEC ratifies the single rule: **B4 owns all six §17C.6 CRD schemas; F2 owns
     controllers/operator + the three new CRDs (`GovernanceBundle`, `PolicyEnginePlugin`,
     `ExportAdapter`); F1 owns the REST projection; E-components own their controllers/instances.**
  2. F1's `/rego-packages` is a **projection** of `/policies` where `engine=opa`, not a second
     source of truth.
  3. Cross-cutting CRD rules hold for all nine: each carries §17A.5 authz scope metadata, controllers
     are idempotent + leader-elected + emit a C2 event with `correlation_id` on every
     enforcement-relevant reconcile, and a `v1alpha1→v1beta1` conversion path exists before any
     becomes a durable contract.
- **Done when:** B4 SPEC carries the frozen ownership table and no two components define divergent
  field shapes for the same CRD.

---

## Out of the freeze gate (land before *component build*, not before contract re-freeze)

These are **correctness** but live in component SPECs, not the frozen foundation contracts. They do
**not** gate the C2/B4/D2 re-freeze; they gate the build of their respective components.

| ID | XD | Title | Owner | Note |
|---|---|---|---|---|
| C-A | XD-4 | Additive roles defeat author≠approver SoD — make SEC-11 mutual-exclusion normative; single E1-owned tag-write service | D2 + D3/D4 + E1 | enforce at grant-time AND action-time |
| C-B | XD-6 | Declared-vs-verified: surface `replay_completeness` in A2 gates, C5 rollup denominators, E1/E2/E3 headline | A1/A2 + C5 + E1/E2/E3 | builds on C2's honesty label |
| C-C | XD-10 | Approval flow not durable/retry-traceable/secure — one vertical slice (B2+B4+B5+D3); approval bound to manifest digest; approver-bound callback proof; threads `governance_transaction_id` (BB-7) | B2+B4+B5+D3 | single owner |
| C-D | XD-16 | Behavioral (AI-judge) evaluators = labeled best-effort tier, capped at `best_effort` (analogous to N-C2-SYNTH); two-tier async; scope "deterministic replay" claim | F4 + C2 (tier marker) | additive C2 marker |

## Hardening (urgent but deferrable with a documented risk acceptance)

| ID | XD | Title | Owner |
|---|---|---|---|
| H-A | XD-7 | Composed fail-closed ⇒ correlated mass-deny; add "infrastructure-degraded" mode + circuit breaker + break-glass (availability P0) | B5 + B1/B2/B3 + D4 |
| H-B | XD-17 | Fragmented PII/identity handling + agent context in deferred store — single identity-data policy; tokenized/structural-only replay inputs (legally load-bearing) | D4 + C2 + F4 + D2 |
| H-C | XD-19 | Signing-key compromise / federated correlation-id collision — external transparency-log anchor; out-of-band key distribution; cluster-scope ids | C2 + C3/C5 |
| H-D | XD-22 | Webhook endpoints = unguarded PII-egress/SSRF — endpoint allow-listing + internal-address blocking | D4 + D2/D3 |

---

## Report-back: re-freeze checklist

Re-freeze C2 to final `c2.audit-event/1.0` only when **all** hold:

- [ ] BB-1 `disposition`/`obligations` split landed; `decision:mutate` no longer emitted; allow+mutate+notify representable.
- [ ] BB-2 one canonical taxonomy; C2 enums derived from B4; E3 maps, invents nothing.
- [ ] BB-3 volatile-provider `value_ref` captured; flagship image-signing replays `complete`; volatile-no-value ⇒ `insufficient`.
- [ ] BB-4 stored completeness = lower bound; E1 recomputes per-bundle (authoritative); conservative-label-wins; one vocabulary.
- [ ] BB-5 catalog pinned per `policy_version`, bound into `content_hash`; read-set mismatch ⇒ `best_effort:catalog_stale`.
- [ ] BB-6 D1 emits `jwt_claims_captured`; C2 computes `jwt_claims_completeness`; catalog absent ⇒ `partial`, never `full`.
- [ ] BB-7 one `correlation_id` + `governance_transaction_id` across approval retry; per-admission UID demoted to `engine_context`.
- [ ] BB-8 D2 scope-predicate library linked by C2/C3/C5/F1/F2; aggregate reads filtered; minimum storage contract defined.
- [ ] BB-9 B4 SPEC ratifies CRD ownership; no divergent field shapes.

Then the foundation contracts (C2 schema, B4 taxonomy, D2 predicate, `replay_completeness`
semantics) re-freeze together, and E1/C3/C4/C5/F4/C1 build against the corrected `c2.audit-event/1.0`.
