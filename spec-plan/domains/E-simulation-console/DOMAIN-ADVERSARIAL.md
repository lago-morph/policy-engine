# Domain E — Simulation & Console — DOMAIN ADVERSARIAL (reconciliation)

**Date:** 2026-05-30 · Reconciles the per-component adversarial reviews (E1, E2, E3) and surfaces contradictions *between* the three components and *across domain boundaries*. Companion to the per-component `ADVERSARIAL-REVIEW.md` files.

---

## 1. The domain-wide theme: the differentiators are the riskiest surfaces

Domain E's three highest-value features — **E1 differential simulation**, **E2 lineage graph**, **E3 PDP catalog** — are each (a) the platform's strongest market differentiator and (b) the most over-claimable. The unifying adversarial finding: **all three quietly depend on data fidelity the platform may not actually have**, and all three can *present confidence they haven't earned*:

- E1 reports "47 newly blocked, 0 newly allowed" — authoritative only over the `complete` subset, which real audit logs rarely achieve.
- E2 renders a "defensible" governance→runtime lineage — only as complete as every upstream source's metadata and scope tagging.
- E3 presents a uniform enforcement catalog — most non-K8s rows are detect-only or proxy-conditional.

**The shared fix is honesty-by-construction:** coverage/completeness/enforceability must be *prominent, headline, sortable* facts — never footnotes — so the differentiators don't mislead the exact users (Priya, Daniel) who rely on them for compliance defensibility.

---

## 2. Contradictions *between* E components

**C1. `replay_completeness` is computed in three places with three meanings.**
- C2 stores a per-event flag (§13).
- E1 *recomputes* it per-bundle (a new bundle reading a new field changes completeness — E1 ADVERSARIAL A2/D1).
- E3 *predicts* it per product via `replay_completeness_notes`.
- E2 *displays* required-audit-fields in Rego Explorer.

If these four disagree, the console misleads the author about whether a policy is replayable, and E1 may over-report authority. **Resolution:** E1's per-bundle introspection is authoritative for a *given run*; C2's flag is a lower bound; E3's notes are a static prior; E2 must render **E1's** computed completeness, not B1/C2's static value. One number, one owner per context, reconciled in `DATA-MODEL.md`.

**C2. Action taxonomy drift.** E3's libraries use verbs (`detect`, `alert`, `fail build`, `block promotion`, `clear hold`, `attach evidence`) absent from §17C.3 (E3 ADVERSARIAL D2). E1's differential normalizes outcomes to allow/deny-class, and E2's graph renders decisions by action. If E3 invents verbs E1/E2 don't recognize, classification and rendering break. **Resolution:** one canonical action taxonomy (expand §17C.3); E3 maps product verbs onto it (detect→notify-class, fail build→deny-class).

**C3. Tagging has two write paths.** E1's API and E2's Audit Correlation View both write `SimulationTag` (E2 ADVERSARIAL D7, E1 ADVERSARIAL D6). Plus neither enforces separation of duties (author ≠ tagger ≠ approver) — the exact "blind promotion" abuse HL-17 claims to prevent. **Resolution:** single tag-write service; enforce SoD; E2 is a client of E1's tag service, not a second source of truth.

**C4. CRD ownership split three ways.** `PolicySimulationRun` (E1 controller), `PolicyActionLibrary` (E3 instances), both *defined* by B4 (§17C.6). E1/E2/E3 each touch CRDs. **Resolution (consistent rule): B4 defines all governance CRDs; E-components own their controllers/instances.** Stated in each component; must be ratified in the B4 SPEC.

---

## 3. Contradictions *across* domain boundaries

**X1. C2 audit schema is asserted, not agreed (E1 §7, E3 §10, E2 graph).** All three E components hard-reference §13 field names, but C2's component dir was empty at authoring time (parallel build). A C2 rename/omission (e.g. dropping `request_object`/`before_state` for PII) breaks replay (E1), per-product schemas (E3), and evidence nodes (E2). **Resolution:** the §13 field list must be promoted to the cross-cutting `DATA-MODEL.md` as a *versioned shared contract*; all E components code against a `ReplayEventV1` adapter. **This is the #1 cross-domain reconciliation item for Domain E.**

**X2. PII/legal redaction (D4) vs replay fidelity (E1/E3).** §17.3 wants full input preserved; D4 security wants sensitive data minimized. In regulated namespaces (payments-prod) — exactly where differential simulation is most valuable — redaction forces `partial`/`insufficient` replays. The platform's differentiator degrades precisely where it matters most. **Unresolved cross-domain tension; must be a DECISIONS.md entry.** Candidate mitigation: tokenized/structural-only replay inputs that preserve policy-relevant shape without raw PII.

**X3. K8s RBAC vs governance-claim authz (E2 vs D2/§16.2).** Kubernetes-native RBAC discovery (§16.2) and governance `namespaces` claims (§17A.4) are two authorization systems with no defined precedence (E2 ADVERSARIAL D3). **Resolution:** governance claim authoritative for *governance data*; K8s RBAC for *K8s objects*; reconcile cluster-list-vs-data-scope UX. Owned jointly E2 + D2.

**X4. External-data snapshot-value store is unowned (E1 D2, E3 D3).** Both E1 and E3 need the *value* of external data at decision time (image-signature status, CVE feed) to replay signing/CVE policies — the flagship examples. §13 stores only `external_data_refs` (name+version), not the value. No component owns a content-addressed external-data store. **Without it, the headline examples replay as `partial`.** Must be assigned (C2 or a new component) in cross-cutting.

**X5. Coverage/drift overlays (E2) depend on C3/C4 which may slip.** E2's most valuable graph overlays (unenforced controls, declared-vs-actual mode drift) delegate to analytics built in Domain C. If C3/C4 slip, the differentiator degrades to a topology with no insight (E2 ADVERSARIAL D8). **Resolution:** graceful "analytics pending" degradation; sequence C3/C4 ahead of E2 GA.

---

## 4. Cross-component severity roll-up

| Sev | Finding | Components | Cross-ref |
|---|---|---|---|
| **Critical** | Completeness/coverage must be headline, not footnote; replay_completeness computed in 4 places must reconcile | E1, E2, E3 | C1, E1-D1, E2-D1 |
| **Critical** | §13 schema is an unratified cross-domain contract | E1, E2, E3 ↔ C2 | X1 |
| **Critical** | External-data value store unowned ⇒ flagship examples replay partial | E1, E3 ↔ C2 | X4, E1-D2, E3-D3 |
| **Critical** | Enforceability honesty: detect-only/proxy-conditional sold beside true enforcement | E3 | E3-D1 |
| **High** | Action taxonomy drift between E3 and §17C.3 | E1, E2, E3 | C2, E3-D2 |
| **High** | Multi-source graph scope/topology leak | E2 | E2-D1 |
| **High** | Tag write-path duplication + no separation of duties | E1, E2 | C3, E1-D6, E2-D7 |
| **High** | PII redaction vs replay fidelity in regulated namespaces | E1, E3 ↔ D4 | X2 |
| **High** | K8s RBAC vs governance-claim precedence | E2 ↔ D2 | X3 |
| **Medium** | Assemble-on-query graph won't scale; materialization mandatory ⇒ staleness | E2 | E2-D2 |
| **Medium** | Cross-product replay only authoritative for K8s | E1, E3 | E3-D3, X4 |

---

## 5. Top reconciliation actions for cross-cutting wave

1. **Promote §13 to a versioned shared `DATA-MODEL.md` contract**; all E components via `ReplayEventV1` adapter. (X1)
2. **Assign the external-data snapshot-value store** to a component (C2 or new). (X4)
3. **One canonical action taxonomy** (expand §17C.3); E3 maps onto it. (C2)
4. **One completeness number per context**, E1-introspection authoritative; E2 renders it; reconcile the 4 sources. (C1)
5. **Single tag-write service + separation of duties**; E2 is a client. (C3)
6. **Ratify CRD ownership rule:** B4 defines, E-components own controllers/instances. (C4)
7. **Resolve PII-vs-replay** as a DECISIONS.md entry (tokenized/structural replay inputs). (X2)
8. **Resolve K8s-RBAC vs governance-claim precedence** jointly E2+D2. (X3)
9. **Make coverage/completeness/enforceability headline facts** across E1/E2/E3 UX so differentiators don't mislead compliance users. (domain theme)
