# E2 — Governance Console / Headlamp GUI — PLAN

**Component:** E2 · **Spec:** `SPEC.md` (this dir) · **Date:** 2026-05-30

---

## 1. Dependency DAG

```
  ┌──────────────────────────────────────────────────────────────┐
  │ D1/D2 OIDC + storage-layer scope (§17A.4/§17A.5) — HARD BLOCKER│
  └───────────────┬──────────────────────────────────────────────┘
                  ▼
        [WS-P Plugin shell + OIDC + RBAC discovery]
                  │
   ┌──────────────┼─────────────────────────────────────────────┐
   ▼              ▼                 ▼              ▼              ▼
[WS-G Graph]  [WS-R RegoExpl]  [WS-E RuntimeView] [WS-A AuditCorr] [WS-N NsAuthoring]
   │   (A1,B1,    (B1,§8.3,        (B2-5,C2,C3,4,   (C2,C3,4,E1)     (E1 M6,C2,D3,D2)
   │   C2,E3,D3)   E1)              D3)
   ▼
[WS-GB Graph backend / lineage assembly+query]
```

**Critical path:** D1/D2 scope → WS-P plugin shell (auth + RBAC discovery) → WS-GB graph backend → WS-G Graph View (the differentiator).

---

## 2. Parallel workstreams

| WS | Scope | Blocks on | Parallel-with |
|---|---|---|---|
| **WS-P** Plugin shell | Headlamp plugin manifest, signed OCI packaging, OIDC auth-code+PKCE, RBAC discovery, backend API client | D1/D2 | — (gates the rest) |
| **WS-GB** Graph backend | Lineage assembly from A1/B1/B-engines/C2/E3; typed node/edge model; query API; coverage/drift queries | A1,B1,C2 metadata contracts | WS-R/E/A/N |
| **WS-G** Graph View | Render typed graph, search, expand, side-panels, deep-links | WS-GB, WS-P | other views |
| **WS-R** Rego Explorer | packages, deps, coverage, claims/audit-fields, WASM preview, launch sim | B1 §8.3, E1 | WS-G |
| **WS-E** Runtime Enforcement | constraints/bundles/policies, stats, recent denies, drift | B2-5, C2, C3/4 | WS-G |
| **WS-A** Audit Correlation | decisions, gaps, timelines, FP/FN, newly-allowed/blocked, tag UI, create-fixture | C2, C3/4, E1 | WS-G |
| **WS-N** Namespace Authoring | scoped lists, hard-locked authoring form, scoped simulate, approval submit | D2 §17A.5, E1 M6, D3 | WS-G |

**The 5 views (WS-G/R/E/A/N) are embarrassingly parallel** once WS-P (auth shell) + the backend API contracts exist — each binds to different backend sources and shares only the plugin shell + scope middleware.

---

## 3. Milestones

- **MVP-0:** WS-P shell — install signed OCI plugin, OIDC login, RBAC discovery, scope middleware (DT-44). Nothing renders without this.
- **MVP-1:** WS-GB + WS-G Graph View (the differentiator) — control→Rego→enforcement→audit lineage, forward/backward traversal, side-panels (DT-39, HL-01).
- **MVP-2:** WS-R Rego Explorer (DT-40) + WS-E Runtime Enforcement (DT-41) — parallel.
- **MVP-3:** WS-A Audit Correlation incl. tagging + create-fixture surface for E1 (DT-42, DT-49/HL-17).
- **MVP-4:** WS-N Namespace Authoring with storage-enforced scope (DT-43, HL-08).
- **GA:** Backstage/OpenLens API parity; streaming decision stats; graph caching/materialization; coverage/drift overlays polished.

**Sequencing rationale:** Graph View first — it is the market gap (research §8) and the highest-value defensible feature; the other four views are higher-coverage but lower-differentiation and fully parallel.

---

## 4. What can be built concurrently / what blocks what

- **Blocks everything:** WS-P (auth + scope). Until scope middleware exists, no view can be trusted (§17A.1).
- **Concurrent after WS-P:** all 5 views, given backend API contracts are frozen early (define them in WS-P).
- **WS-GB blocks WS-G** but not the other four views.
- **External blockers:** Graph needs A1 control metadata, B1 §8.3 metadata, C2 audit index; Audit Correlation needs E1 diffs; Namespace Authoring needs D2 storage scope + E1 M6.

---

## 5. Test strategy

| Layer | Tests |
|---|---|
| Auth | OIDC code+PKCE flow; token refresh; claim→scope derivation; expired/invalid token rejection |
| **Scope (security, critical)** | crafted out-of-scope API call returns empty/403 **even when UI hides** the affordance (DT-43); namespace form refuses clusterScope; storage-layer filter verified independently of GUI |
| Graph | lineage traversal forward/backward; coverage-gap query (control with no enforcement edge); drift detection (declared vs actual mode); side-panel §8.3 metadata correctness |
| Per-view golden | one E2E per view wired to its scenario (DT-39..44) |
| Cross-view | deep-links (graph node → owning view); tag in Audit Correlation writes E1 SimulationTag; create-fixture hands E1 a valid fixture |
| Resilience | source-down partial render + banner; large-graph depth cap; degraded cluster |
| Packaging | signed OCI verify on install; plugin manifest schema |

**Coverage anchors:** DT-39 (lineage), DT-43 (storage-scope security), DT-44 (OIDC+packaging) are the must-pass E2E gates.

---

## 6. Risks / mitigations

| Risk | Mitigation |
|---|---|
| GUI-only scope leak (the §17A.1 trap) | Storage-layer filter is the source of truth; pen-test asserts UI bypass fails |
| Graph assembly cost over large corpora | Assemble-on-query + cache hot subgraphs; depth caps; coverage/drift as queries |
| Source-contract drift (A1/B1/C2 metadata) | Freeze backend API contracts in WS-P; adapter per source |
| Headlamp plugin API churn | Pin Headlamp version; isolate Headlamp-specific code so Backstage/standalone reuse the backend |
| WASM preview mistaken as authoritative | Label advisory; authoritative eval via E1 |
| Lineage graph is the differentiator but depends on many sources | Build graph backend early; degrade gracefully when a source is missing |
