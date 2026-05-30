# Domain E — Simulation & Console — DOMAIN SUMMARY

**Date:** 2026-05-30 · Companion to [DOMAIN-INDEX](DOMAIN-INDEX.md) and [DOMAIN-ADVERSARIAL](DOMAIN-ADVERSARIAL.md).

Domain E is the platform's **user-facing product surface and its three biggest differentiators**: simulate-before-promote (E1), see/author/trust (E2), reuse-per-product (E3). Where Domains A–D build the governance, enforcement, evidence, and identity machinery, Domain E is what a human actually touches — and per market research (§8, §12), it is where the platform is *most defensible*, since no open-source product offers differential simulation, a governance→runtime lineage graph, or a uniform per-product PDP catalog.

---

## 1. Shared data model (the spine that ties E1/E2/E3 together)

All three components rotate around the **§13 replay-capable audit event** and the **governance lineage** built on top of it:

```
Objective(§6) → Control(§6) → RegoPackage(§8.3) → EnforcementPoint(§17D/B-engines) → AuditEvent(§13)
                                   │                        │                              │
                                   │                        │                  E1 replays these (M2,M5-M9)
                                   │              E3 defines per-product decision points  │
                          E2 renders this whole chain as the lineage graph ───────────────┘
```

- **§13 audit event** (owned by C2) is the lingua franca: E1 replays it, E2 renders it as evidence nodes, E3 defines per-product *replay input schemas* that map into it.
- **`replay_completeness ∈ {complete, partial, insufficient}`** (§13 / §17.3) is the **authoritativeness gate** running through the whole domain: E1 only treats `complete` results as authoritative; E2 must show coverage honestly; E3's per-product `replay_completeness_notes` predict which products yield complete replays (K8s) vs partial (Grafana/Jenkins/ES).
- **§17C.3 action taxonomy** + **§17C.6 CRDs** (`PolicySimulationRun`, `PolicyActionLibrary`, `PolicyException`) are the shared verbs and objects.
- **§17A.4 OIDC claims + §17A.5 storage-layer scope** constrain every read in all three components — *dual-layer authz, GUI-only is insufficient* (§17A.1).

---

## 2. Internal dependencies (within Domain E)

- **E2 → E1:** the console launches simulation runs, renders differential diffs, hosts the tagging UI, and creates regression fixtures from audit events (§17.5). The Audit Correlation View *is* E1's user surface.
- **E2 → E3:** the lineage graph renders `PolicyActionLibrary` nodes; Rego Explorer surfaces per-product example controls.
- **E1 → E3:** E1 replays the decision points E3 catalogs; cross-product differential simulation depends on E3's uniform replay schemas.
- **All three → Domains A–D:** C2 (§13 schema, **hard blocker for E1**), D1/D2 (OIDC + scope, **hard blocker for E2**, honored by E1/E3), A1 (controls), B1–B5 (engines + metadata), C3/C4 (drift/analytics overlays), D3/B4 (approval/exception CRDs), C5 (report rendering).

---

## 3. The 3–5 hardest decisions

1. **Re-execute bundles vs reuse decision logs (E1 OQ-1).** Decided: **re-execute both** bundles on reconstructed input; decision logs are a *validation oracle* + non-deterministic-builtin source only. Pure reuse pollutes the differential matrix with input-reconstruction artifacts and is OPA-only. (ALT-decisionlog-reuse.)
2. **Batch replay vs always-on shadow (E1 OQ-2).** Decided: **batch `PolicySimulationRun` is the authoritative default**; live shadow (M3) is opt-in and *archives its observations as immutable EvidenceSets* so later batch diffs are reproducible. The flagship scenario (HL-17) demands a reproducible, signable report over a fixed evidence set — shadow cannot deliver that. (ALT-replay-batch-vs-shadow.)
3. **`replay_completeness` is policy-relative, so E1 must recompute it per-bundle.** C2's stored flag is a lower bound; a bundle that reads a new field can be `insufficient` even when C2 marked the event `complete`. This is the domain's central correctness landmine (E1 ADVERSARIAL D1).
4. **Scope must be enforced at the storage layer, uniformly across a multi-source graph (E2).** The lineage graph assembles from A1+B1+C2+B-engines; each source must enforce the *same* scope, and the graph must filter edges/aggregate-counts (not just node detail) to prevent topology-inference leaks (E2 ADVERSARIAL D1).
5. **Enforceability honesty in the PDP catalog (E3).** Most non-K8s decision points are detect-only or "deny if proxied." The catalog's value (uniform model) is also its risk (makes ad-hoc hooks look like a uniform enforcement fabric). Decided: enforceability + proxy-required must be prominent, sortable, and controls against detect-only points flagged "detective, not preventive" (E3 ADVERSARIAL D1).

---

## 4. Consolidated open-questions list (with decided defaults)

| From | Question | Decided default |
|---|---|---|
| E1 | Reuse decision logs vs re-execute? | Re-execute both; logs as oracle |
| E1 | Shadow vs batch default? | Batch authoritative; shadow opt-in, archive-to-EvidenceSet |
| E1 | warn/mutate allow-class? | Yes, + within-class secondary diff (gate it) |
| E1 | Replay vs current external data? | Historical snapshot at decision time |
| E1 | Gate promotion on untagged_risky==0? | Yes (+ coverage threshold, per adversarial) |
| E2 | Headlamp vs standalone vs Backstage? | Headlamp default; shared backend enables others |
| E2 | Graph DB vs assemble-on-query? | Assemble-on-query MVP → **must materialize/cache for scale** (adversarial) |
| E2 | WASM preview authoritative? | No, advisory; E1 server-side authoritative |
| E2 | K8s RBAC vs governance-claim precedence? | **Open — must reconcile** (adversarial D3): governance claim authoritative for governance data |
| E3 | Per-product vs mega-library? | Per-product on shared template |
| E3 | Example policies Rego vs native? | Rego primary + native where action is native |
| E3 | Catalog detect-only points? | Yes — explicit, honest |
| E3 | Proxy ownership for ES/Grafana enforcement? | Out of E3; deployment (F2) owns; E3 documents requirement |

---

## 5. Critical path & build order for the domain

1. **Unblock:** C2 §13 schema (E1) + D1/D2 OIDC+scope (E2). Until then: E1 uses a `ReplayEventV1` adapter + synthetic fixtures; E2 builds the auth/scope shell.
2. **E3 template first**, then 9 product libraries in parallel (embarrassingly parallel).
3. **E1 MVP-2 (differential)** + **E2 MVP-1 (lineage graph)** are the two differentiators — prioritize over shadow/snapshot and the lower-value views.
4. **Integration anchor scenarios:** HL-17 (E1 differential + E2 tagging end-to-end), DT-43 (E2 storage-scope security), DT-68 (E3 K8s library replay).

The domain's two flagship differentiators (E1 differential, E2 lineage graph) are also its two most integration-dependent and most easily over-claimed surfaces — see DOMAIN-ADVERSARIAL.
