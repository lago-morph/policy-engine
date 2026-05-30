# E2 — Governance Console / Headlamp GUI — ADVERSARIAL REVIEW

**Component:** E2 · **Reviewer persona:** Marcus (Platform Security Engineer) + red-team hat · **Date:** 2026-05-30
**Target:** `SPEC.md`, `PLAN.md` in this dir.

---

## 1. Attacks on core assumptions

**A1. "Storage-layer scope enforcement" assumes the storage layer *can* scope.** The SPEC asserts D2/§17A.5 strips out-of-scope rows. But the graph is *assembled on query* from many heterogeneous sources (A1 control store, B1 bundle metadata, C2 audit index, B-engine registries). **Does each source enforce the same scope model?** If C2 enforces namespace but B1 bundle metadata is global, a Namespace Author can see global Rego packages they shouldn't, or infer cross-tenant controls from graph topology. Scope must be enforced *uniformly across every source the graph touches* — the SPEC delegates to D2 but the graph's multi-source assembly is exactly where uniform scope is hardest. **This is the central unverified assumption.**

**A2. Graph topology leaks information even when nodes are hidden.** Hiding out-of-scope *nodes* may still leak via *edges/counts*: "Control X has 14 enforcement points" reveals other tenants' enforcement even if their detail is hidden. Lineage graphs are notorious for inference attacks. Scope filtering must remove edges *and* aggregate counts that cross scope boundaries, not just node detail panels.

**A3. Headlamp is the official SIG UI — but it's CNCF *sandbox*.** Research §8 notes Headlamp is sandbox-stage and was Kinvolk→Microsoft. Plugin API stability is not guaranteed. Betting the default GUI on a sandbox project's plugin API is a real maturity risk; the SPEC's "isolate Headlamp-specific code" is right but the *default* posture should acknowledge churn risk.

**A4. WASM Rego preview is a trust footgun.** §16.2 mandates WASM Rego execution in-browser. The SPEC labels it "advisory," but users *will* trust a green "ALLOW" in the editor. Browser eval lacks external data, lacks the scope-enforced input, and may run a different OPA/WASM version than production — so it can show ALLOW where production DENIES. The divergence between preview and authoritative E1 eval must be made *visually unmistakable*, not just labeled.

---

## 2. Gaps / unhandled cases

**G1. Graph staleness vs decision freshness.** `assembled_at` is carried, but a SOC2 workpaper (DT-39/HL-01) built on a stale graph could omit yesterday's denials. What is the freshness SLA? An auditor relying on a 6-hour-stale lineage for a "defensible" workpaper is a finding waiting to happen.

**G2. No write-conflict / concurrency model for authoring.** Two Namespace Authors editing the same policy, or an author editing while an approver is mid-approval (HL-08). Optimistic locking / version pinning is unspecified.

**G3. Tagging surface lives in two views.** Audit Correlation View (E2 V4) writes E1 SimulationTags, and E1's own API also tags. Two write paths to the same tag state ⇒ consistency and separation-of-duties questions (cf. E1 ADVERSARIAL D6). Who is the source of truth for a tag, and does the console enforce author≠tagger?

**G4. Coverage-gap and drift overlays depend on C3/C4 which may not exist yet.** The graph's most valuable views (unenforced controls, declared-vs-actual mode drift) are delegated to analytics components built in other domains. If C3/C4 slip, the graph degrades to a pretty topology with no insight. The differentiator partially depends on downstream analytics.

**G5. Approval state in the graph can be stale/race-y.** `ApprovalGate.state` is read from D3; a gate approved 1s ago may render as pending. For an operator deciding whether a deploy is unblocked, stale approval state is operationally dangerous.

**G6. No offline / air-gapped story.** OIDC to Keycloak + OCI plugin pull + multi-cluster API. Regulated/air-gapped deployments (the payments-prod audience) need an offline install + token story; unaddressed.

---

## 3. Inconsistencies vs other components

**X1. RBAC discovery (`SelfSubjectAccessReview`) vs governance scope (`namespaces` claim).** Kubernetes RBAC (what the kubeconfig can do) and the governance `namespaces` claim (§17A.4) are *two different authorization systems*. A user may have K8s read on a namespace but not a governance authoring claim there, or vice versa. Which wins per view? The SPEC uses both (§2.3 + §3) but never reconciles the precedence. This is a real conflict between §16.2 (K8s-native RBAC) and §17A (governance RBAC).

**X2. §8.3 `__required_audit_fields__` vs C2/E1 completeness.** Rego Explorer shows "required audit fields," and E1 downgrades replays when those fields are absent. The console must show the *same* required-field set E1 uses for completeness — if B1 metadata and E1 introspection disagree, the console misleads the author about whether their policy is replayable.

**X3. Lineage `Exception` node vs E1 exception linkage.** The graph shows `Control --has_exception--> Exception`, and E1 links tags to exceptions. Two views of the same exception lifecycle (B4/D3 owns it). Ownership and a single source of truth must be pinned.

---

## 4. "Won't survive production" findings

**P1. Multi-source assemble-on-query graph will be slow at scale.** Joining A1 + B1 + C2 (millions of audit events) + B-engine registries per graph render, scope-filtered, is a fan-out join across services. The "interactive console" experience will degrade to seconds-per-render without aggressive caching/materialization — which then reintroduces staleness (G1). The MVP "assemble-on-query" (OQ-2) will not survive a real fleet; materialization is not optional, it's required, and it changes the freshness model.

**P2. The differentiator depends on the most components.** The Governance Graph View needs A1, B1, B2-5, C2, C3, C4, E1, E3, D3 all emitting consistent, scope-aware, correctly-linked metadata. It is the single most integration-dependent surface in the platform. If any upstream omits `__control_id__` linkage or scope tags, the graph silently shows broken/partial lineage — and "partial lineage" presented as "defensible workpaper" is worse than no graph.

**P3. Token scope vs cluster reachability mismatch.** RBAC discovery shows clusters the kubeconfig reaches; the governance backend scopes by claim. A user could see cluster-b in the cluster list (K8s reachable) but get empty governance data (no claim) — confusing, looks broken.

---

## 5. Scope-creep watch

- Backstage + OpenLens parity (§16.2) is a tempting "while we're here" that triples the frontend surface. Keep Headlamp-first; expose backend API for later parity (PLAN does).
- In-console policy *authoring* with WASM preview edges toward becoming a full Rego IDE. Bound it to preview + launch-sim; the authoritative loop is E1.

---

## 6. Prioritized defect list

| # | Severity | Defect | Fix |
|---|---|---|---|
| D1 | **Critical** | Multi-source graph scope not uniformly enforced; topology/count inference leaks (A1, A2) | Uniform scope contract across every graph source; filter edges + aggregate counts, not just node detail |
| D2 | **Critical** | Assemble-on-query won't scale; differentiator depends on most components (P1, P2) | Materialize/cache hot subgraphs with explicit freshness SLA; graceful "partial lineage" labeling; integration contract tests across sources |
| D3 | **High** | K8s RBAC vs governance-claim precedence unreconciled (X1, P3) | Define precedence (governance claim authoritative for governance data; K8s RBAC for K8s objects); reconcile cluster-list vs data-scope UX |
| D4 | **High** | WASM preview can show ALLOW where prod DENIES (A4) | Make preview/authoritative divergence visually unmistakable; show engine/version + missing-external-data warning |
| D5 | **High** | Stale approval/decision state in graph is operationally dangerous (G5, G1) | Freshness SLA + live-refresh for approval gates; show `assembled_at` prominently on workpaper exports |
| D6 | **Medium** | Required-audit-fields shown in Rego Explorer must match E1 completeness logic (X2) | Single source for required-field set; console reads E1's introspection, not just B1 metadata |
| D7 | **Medium** | Tag write paths in two places; separation-of-duties unenforced (G3) | Single tag-write service; enforce author≠tagger in console |
| D8 | **Medium** | Coverage/drift overlays depend on C3/C4 maybe-absent (G4) | Degrade gracefully; mark overlays "analytics pending" rather than empty |
| D9 | **Low** | No concurrency/locking for authoring (G2) | Optimistic version pinning on policy edits |
| D10 | **Low** | No air-gapped install/token story (G6) | Document offline OCI mirror + token flow |
| D11 | **Low** | Headlamp sandbox-stage plugin API churn (A3) | Pin version; isolate plugin layer; keep backend portable |
