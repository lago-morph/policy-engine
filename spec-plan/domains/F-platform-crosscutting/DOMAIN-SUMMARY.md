# Domain F — Platform & Cross-cutting — SUMMARY

**Date:** 2026-05-30 · **Components:** F1 (API), F2 (Deployment & Extensibility), F3 (MVP scope & Sequencing), F4 (AI/Agent extension)

Domain F is the **connective tissue** of the platform: it owns the external API surface, the deployment/extensibility substrate, the MVP cut line + whole-platform sequence, and the AI-agent extension. Unlike domains A–E (which own vertical capabilities), F components are mostly **horizontal** — they consume and bind every other domain. F4 is the exception: it is a vertical capability built entirely as *deltas* on the rest.

---

## 1. Shared data model (what F components consume/expose)

F adds few new entities; it mostly exposes and packages others'. The entities it touches:

- **Authorization subject** (§17A.4, owned D1/D2) — F1 enforces it per request; F2 puts it on every CRD; F4 extends it into the *agent subject chain* (originating_user → agent → model → tools → capability_token → delegation).
- **§13 replay-capable audit event** (owned C2) — F1 reads it (`/audit/events`); F2 makes every plugin/CRD emit it; F4 adds agent fields (evaluator_results, trace context, agent subject) — all additive.
- **Policy package / signed bundle** (owned B1) — F1 promotes it; F2 ships it as `GovernanceBundle` + verifies signatures.
- **CRDs** (§17C.6) — F2 owns the in-cluster API: `PolicyApprovalRequest`, `PolicySimulationRun`, `PolicyActionLibrary`, `PolicyEvidenceSchema`, `PolicyException`, `PolicyRemediationAction`, + new `GovernanceBundle`, `PolicyEnginePlugin`, `ExportAdapter`.
- **Plugin capability descriptor** (NEW, F2) — kind, version, supported §17C.3 actions, real-time hook, replay schema, scope. The contract every extension (incl. F4's agent PDPs) implements.

**The platform has three load-bearing contracts** every domain keys off (F3 PLAN §3.1): **(1) C2 §13 audit schema, (2) D1+D2 §15.4 mapping + §17A scope predicate, (3) A1 Gemara controls + lineage.** F1, F2, and F4 all bind to these; locking them first is the highest-leverage scheduling move.

---

## 2. Internal dependencies (within F)

- **F3 → everyone:** F3's cut line and sequence reference every F (and non-F) component's thin slice; its PLAN is the first-draft whole-platform sequence the primary reconciles.
- **F1 ↔ F2:** F1 exposes `/plugins` + `/engine-bindings`; F2's plugin SPI surfaces new sub-resources under F1's envelope/authz.
- **F4 → F1+F2:** agent PDPs and evaluators are F2 plugins; agent resources are F1 sub-collections. F4 adds no primitive — it rides F1's API contract and F2's plugin SPI.
- **F1 + F2 share** the D2 scope-predicate dependency (the #1 cross-domain risk, below).

---

## 3. The 3–5 hardest decisions

1. **Storage scope-predicate vs deferred storage (§26.1) — THE platform contradiction.** §17A.1 demands storage-enforced authz; §26.1 defers storage; §22.2 says "ordinary storage is acceptable." F1 (DEFECT-1) and F2 (DEFECT-1) both hit this. **Decision:** D2 must ship a *linkable scope-predicate library* (row AND field level) usable even over ordinary storage; F2 defines a *minimum storage contract* (scope columns, append-only/versioned audit, content hashing). Escalated to cross-cut.
2. **failurePolicy default (F2).** Fail-open when the admission webhook is down is a real-time bypass; fail-closed risks availability. **Decision:** default-closed for any namespace mapped to an `enforcement: required` control + a webhook-health SLO that pages; configurable elsewhere.
3. **Base-first vs AI-first (F3/F4).** **Decision:** engineering builds base-first (reframe doc: no refactor needed); product MAY market an agent-first slice (ALT-1). Two different axes — stated explicitly so they aren't conflated.
4. **Behavioral evaluators inline vs async (F4).** Inline `require_async_check` breaks deterministic replay and adds fatal latency. **Decision (per ALT-2):** two-tier — deterministic pre-filters inline (preserve the guarantee), non-deterministic judge-model evaluators async/best-effort feeding trust-grade + human-review, never conflated with deterministic evidence.
5. **CRD ownership (F2 vs B4).** Both define the §17C.6 CRD surface. **Decision:** nominate a single CRD-schema owner (recommend B4 owns the schema, F2 owns the controllers/operator); reconcile at cross-cut.

---

## 4. Consolidated open-questions list (decided defaults)

| # | Question | Decided default |
|---|---|---|
| F1 | REST vs GraphQL; sync vs async simulate; API sole authz point; write controls via API | REST (MVP) + lineage query later; async job + interactive fast-path; storage co-enforces (not sole); controls via GitOps not API |
| F2 | Hub-spoke vs standalone; one operator vs many; in/out-of-process plugins; failurePolicy; storage shipped vs BYO; federation; plugin signing | Hub-spoke pull-based; one operator/many controllers; out-of-process default (Wasm optional); default-closed for required scopes; BYO ordinary storage w/ minimum contract; federation deferred; signing mandatory |
| F3 | D2/F1/F2/C3 in MVP?; base vs AI first; #integrations; graph DB in MVP; which §14.2 detections; signer | Yes (thin slices); base-first; 3 core + ≤2 optional; no graph DB (records only); the 2 examples; cosign/Sigstore default but unspecified |
| F4 | Separate product or extension; inline vs async evaluators; exact-output replay authoritative; new engine for evaluators; ship before base; trust authority | Extension (deltas) + market a wedge (ALT-1); two-tier async (ALT-2); no (best-effort outputs); no new engine; after base (Phase-3); derived from evaluators+sim, auto-demote with optional approval |

---

## 5. The whole-platform sequence (pointer)

F3 PLAN.md contains the first-draft 23-component build sequence: **three foundation contracts frozen first (C2 schema, D1+D2 identity+scope, A1 Gemara+lineage)** → engines+API+lifecycle wave → simulation+analytics wave → console+integration wave → Phase-2 depth → Phase-3 F4 agent deltas. **Critical path: C2 schema → B1 engine → E1 differential simulation → E2 console.** Longest poles: E1 (differential simulation, the novel core) and D2 (scope predicate over deferred storage). This is the primary's reconciliation input.
