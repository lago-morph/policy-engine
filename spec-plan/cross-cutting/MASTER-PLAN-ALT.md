# MASTER-PLAN-ALT — Wedge-First Sequencing

**Status:** ALT (alternative to MASTER-PLAN.md) · **Date:** 2026-05-30 · **Author:** Alternative-sequencing strategist
**Companion to:** `00-ORCHESTRATION-PLAN.md`, `components/F3-mvp-sequencing/PLAN.md` (platform-first), the positioning memo (*reframed market position*), and the market research memo.

---

## 0. Thesis: invert the build order, not the architecture

The platform-first plan (MASTER-PLAN.md / F3 PLAN.md) is **correct as engineering** and **wrong as go-to-market**. It freezes three foundation contracts (C2 §13 audit schema, D1+D2 identity+scope, A1 Gemara+lineage), builds the enforcement spine (B1–B5) bottom-up, then layers simulation (E1), console (E2), analytics (C3), and reporting (C5). Time-to-first-**demo** is end of Wave 4; time-to-first-**dollar** is later still, because the thing you can sell ("a unified governance platform") is exactly the thing the market research says **no buyer wants to buy** — every horizontal layer already has an incumbent, and the only uncontested ground is the *connective tissue* (replay, lineage, evidence, approval gates, cross-engine lifecycle).

This ALT keeps the **same 23 components, the same DAG edges, and the same three frozen contracts** — but re-orders the build so each phase ships a **standalone sellable wedge** from the positioning memo, and each wedge is a strict subgraph of the full platform DAG. Nothing is thrown away; later wedges add components to earlier ones. The bet: **converge on the full platform by accretion of sold wedges, not by a big-bang MVP.**

The non-negotiable engineering insight that makes wedge-first safe: **the three foundation contracts must still freeze early even in a wedge-first build.** You cannot defer the C2 §13 schema, the D2 scope predicate, or the A1 `control_id` allocator, because every wedge emits/consumes at least one of them and re-cutting a contract after a wedge ships in production is the one unrecoverable mistake. Wedge-first changes *which components you build*, not *which contracts you freeze*. (See §6, "Contracts you must still freeze early.")

---

## 1. Component → wedge mapping (reference)

The 23 components (per `00-ORCHESTRATION-PLAN.md` §2):

A1 Governance model+Gemara+lineage · A2 Policy lifecycle · B1 OPA/Rego+bundles · B2 Gatekeeper · B3 Conftest · B4 Engine selection/action taxonomy/CRDs · B5 Real-time flow · C1 Privateer · C2 Audit schema · C3 Compliance analytics · C4 Retrospective detection · C5 Reporting · D1 Keycloak/JWT+mapping · D2 Scoped roles+storage authz · D3 Approval-gated decisions · D4 Security baseline · E1 Simulation/dry-run · E2 Console/Headlamp · E3 Per-product PDP libraries · F1 API · F2 Deployment/extensibility · F3 MVP sequencing(meta) · F4 AI/agent governance.

Which components each wedge **needs at minimum** (detailed in §2):

| Wedge | Memo § | Minimum component set |
|---|---|---|
| **W1 Compliance Digital Twin** (replay+sim) | Wedge 1 | C2, E1, B1, + thin A1, thin C5, thin F1, thin D4 |
| **W5 OPA Control Plane successor** | Wedge 5 | B1, B4, A2, + C2 (decision-log subset), E1 (regression-gate), thin A1, thin D4, thin F1 |
| **W2 Governance Lineage Graph** | Wedge 2 | A1, C2, F1, E2 (graph view only), + D1, D2 (scope), thin C3 |
| **W4 Vanta-compatible Runtime Evidence** | Wedge 4 | C2, C5, C1, + B2/B1 (one effector emitting §13), A1 (control crosswalk), D4 (signing), thin F1 |
| **W6 Approval Gate Mesh** | Wedge 6 | B4 (CRDs), D3, B2/B5 (deny-with-approval), F1 (webhook), + thin C2, thin D1 |

"Thin" = a deliberately reduced slice (the same thin slices F3's cut-line table already authorizes), not the full component.

---

## 2. The wedges — minimum components and build order

Each wedge below is a buildable, demoable, sellable product **on its own**. The ordering within each wedge is a topological order over the platform DAG edges (from F3 PLAN §3 / Domain summaries), so building a wedge never violates a real dependency.

### W1 — Compliance Digital Twin (replay + differential simulation as a standalone SaaS)

> *Memo Wedge 1: "Just be the thing that takes normalized enforcement events from anywhere and answers 'what would happen if I changed this policy?'"* This is the **highest-novelty, lowest-incumbent** wedge (market research §8, §13: no open-source differential-simulation tool exists) and maps onto the platform's single longest pole (E1).

**Minimum components, in order:**

1. **C2 (audit schema)** — *freeze the §13 schema first* (the 36-field `c2.audit-event/1.0`, the `replay_completeness ∈ {complete,best_effort,insufficient}` honesty label, the `external_data_refs` capture). For W1 you build the **ingest + query** halves (C2 M1, M2, M4) but you do **not** need the platform's own producers — you ingest *foreign* events (K8s audit, Gatekeeper audit, Kyverno PolicyReports, OPA decision logs, Conftest output) and normalize them into the schema. This is the memo's "make the simulation engine ingestion-only" pivot.
2. **B1 (OPA/Rego engine), replay subset only** — you need the engine that *re-executes* a bundle against reconstructed input, not the full signing/admission stack. B1's `decision` entrypoint + bundle load + replay-over-§13 interface (the part the Domain-B summary calls "the decision substrate; everything else evaluates *this*").
3. **E1 (differential simulation)** — the product. Re-execute baseline-vs-candidate bundle over the same reconstructed §13 event set; classify `newly_blocked / newly_allowed / unchanged`; recompute `replay_completeness` **per-bundle** (Domain-E hard decision #3 — a candidate that reads a new field can be `insufficient` even when C2 marked the event `complete`; this is the correctness landmine and must be in W1 from day one).
4. **A1 (thin)** — only the `control_id` allocator + a flat control list, so simulation results tag to a control. **No** full Gemara hierarchy, **no** lineage graph yet.
5. **C5 (thin) + F1 (thin) + D4 (thin)** — one signed differential report (the "your policy would have denied 42 more deploys" artifact), behind a minimal REST surface, with signing (cosign/in-toto) and OIDC so the report is independently verifiable. C5's normative rule "the signed package is the evidence; PDF is a rendering" applies from the start.

**Smallest viable version (memo):** accept a tarball of K8s audit + Gatekeeper audit + a policy version diff → return a classified report. That is exactly `{C2-ingest, B1-replay, E1-diff, C5-report}`.

**What W1 deliberately omits:** B2/B5 (no live admission — you replay, you don't enforce), D1/D2 full (single-tenant SaaS, scope is per-customer-account not per-namespace at first), A2 (no promotion pipeline — the customer promotes in their own GitOps), E2 (a report + a thin web view, not the Headlamp console), C3/C4 (no continuous detectors), F2 (SaaS, not an in-cluster operator).

**Why this order:** C2 → B1 → E1 is *literally the platform critical path* (F3 PLAN §4). W1 builds the critical path **and nothing else**, which is why it's the fastest path to the single most defensible differentiator.

---

### W5 — OPA Control Plane successor ("the Chainguard for OCP")

> *Memo Wedge 5: make OCP the substrate, add cross-engine lifecycle + governance-metadata-in-bundles + the simulation primitive as a promotion gate.* Market research §2: Styra is sunsetting post-Apple acqui-hire; the Styra-DAS-orphan buyer (Capital One, Goldman, Netflix, Zalando class) is **sized, qualified, and shopping now** — the fastest revenue path.

**Minimum components, in order:**

1. **B1 (OPA/Rego + signed bundles), full** — the core: Git-sourced authoring, signed/versioned OCI bundles (cosign + in-toto), bundle distribution. This is mostly OCP-as-substrate behind the `BundleService` abstraction (Domain-B hard decision #3: "commit to OCP for MVP, own the trust root regardless").
2. **B4 (engine selection + action taxonomy + CRDs)** — the closed 13-action vocabulary and the rubric that lets the *same lifecycle* drive OPA bundles, Kyverno ClusterPolicies, Gatekeeper constraint templates, and Conftest suites. This is the memo's "cross-engine support — genuinely missing from the ecosystem." Freeze the taxonomy early (Domain-B build order: it's shared vocabulary).
3. **A2 (policy lifecycle)** — Git authoring → simulate → promote (`draft → dry-run → warn → enforce`), bundle promotion, the reconciler. A2 is the "modern Styra DAS" workflow surface.
4. **C2 (decision-log subset)** — you need the decision-log/regression-replay event shape so promotions can be **gated on replay results, not just unit tests** (memo: "the simulation primitive wired into OCP's regression-test pipeline"). This is a *subset* of the full §13 schema — but **emit the full schema anyway** (additive-only, N-C2-FWD) so W5 converges with W1/W4 without a re-cut.
5. **E1 (regression-gate mode)** — the simulation primitive from W1, but wired as an A2 promotion gate rather than a standalone SaaS. If you build W1 first, this is **free reuse**.
6. **A1 (thin) + D4 (thin) + F1 (thin)** — control-id tagging in bundle metadata (the memo's "governance metadata as a first-class field in bundles"), signing baseline, minimal API.

**What W5 deliberately omits:** B2/B5 full admission flow (you manage *bundles and lifecycle*; the customer's existing Gatekeeper/Kyverno enforces), C1/C3/C4 (analytics are not the DAS value prop), E2 full console (a lifecycle/promotion UI, not the lineage graph), D2 multi-tenant storage scope (enterprise single-tenant at first), F4.

**Why this order:** B1 → B4 → A2 is the Domain-B internal build order (freeze B4 taxonomy + B1 decision contract first). W5 reuses W1's C2+E1 if W1 ships first — which is the core argument for the hybrid in §4.

---

### W2 — Governance Lineage Graph ("be the spine, not the muscles")

> *Memo Wedge 2: build only the §3.1 G1 traceability requirement as a queryable graph; don't enforce, don't simulate, just be the substrate other tools query.* Market research §8: "no open-source product offers governance→runtime lineage as a graph" — the genuine §3.1-G1 contribution.

**Minimum components, in order:**

1. **A1 (full governance model + lineage)** — the heart of W2: Domain → Objective → Control → EnforcementReq/EvidenceReq/ExceptionReq, FrameworkRef crosswalks, and **typed lineage edges** as first-class objects (the memo's "treat governance lineage as a graph, not a Rego-bundle metadata field"). This is the only wedge where A1 is the *lead* component, not a thin slice. Note the Domain-A decision: relational + temporal edges for the POC, **graph DB deferred** — W2 can ship on assemble-on-query (Domain-E E2 decision) and materialize later.
2. **C2 (ingest + query)** — to populate the graph with *enforcement-event* nodes from Gatekeeper audit, Kyverno PolicyReports, OPA decision logs, scanner findings. Reuse of W1's C2 ingest.
3. **D1 + D2 (scope)** — the graph spans multiple data sources, and **each source must enforce the same scope** (Domain-E hard decision #4: filter edges and aggregate counts, not just node detail, to prevent topology-inference leaks). W2 is the first wedge that *needs* the real scope predicate, because it's multi-tenant and the graph leaks structure if scope is GUI-only.
4. **F1 (graph/lineage query API)** — the GraphQL/Cypher-like API + canned queries ("every enforcement event tied to control SC-IMG-001 in 90 days"; "which controls have no implementing policies in cluster-A?").
5. **E2 (graph view only)** — the Headlamp/Backstage lineage-graph rendering. The memo's distribution trick: open-source the schema + reference impl, charge for the managed graph + federation + NL query.
6. **C3 (thin)** — a coverage detector ("controls with no implementing policy") to make the graph *answer questions*, not just display nodes.

**What W2 omits:** all of Domain B enforcement (W2 doesn't enforce), A2 (no lifecycle), E1 (no simulation), D3 (no approvals).

**Why this order:** A1 → C2 → D1/D2 → F1 → E2 follows the platform DAG. W2 is the **most defensible long-term** (the connective-tissue spine, Stack A) but **slowest to monetize** (memo §"what I'd weigh hardest": value is "across" tools, not "in" them).

---

### W4 — Vanta-Compatible Runtime Evidence Engine

> *Memo Wedge 4: become the canonical "runtime evidence" backend for the GRC tools; you look like a high-value evidence source, internally you're the audit-schema-plus-signing engine.* Market research §5: the GRC layer's structural weakness is that it scrapes evidence from config snapshots, not decision events — "a real gap in the market." **The buyer who'd never buy a 'policy platform' will buy 'more credible evidence for our existing Vanta deployment'** (memo §"what I'd weigh hardest").

**Minimum components, in order:**

1. **C2 (full schema + integrity)** — the signed, tamper-evident, replay-capable event is *the product's substance*. W4 needs the hash-chain + signed Merkle checkpoints + RFC-8785 `content_hash` (C2 integrity primitive) so the evidence is independently verifiable by an auditor who trusts only a published public key (Domain-C hard decision #4).
2. **One effector emitting §13** — minimally **B2 (Gatekeeper) + B1**, so there's a real stream of admission decisions to package ("here are the 47,392 admission decisions that enforced SOC 2 CC6.6"). For a no-footprint variant you can ingest foreign events (reuse W1's C2 ingest) — but the strongest version owns at least one enforcement point.
3. **A1 (control crosswalk)** — the framework cross-walks (NIST 800-53 → SOC 2 → ISO 27001 → EU AI Act → FINOS AIGF) so one enforcement event proves multiple controls. This is A1's FrameworkRef, reused from W2 if W2 shipped.
4. **C5 (evidence packages)** — framework-mapped, signed evidence packages that **drop directly into Vanta/Drata/Secureframe/Hyperproof/Optro** via export adapters. The "signed package is the evidence" rule is the whole value prop here.
5. **C1 (Privateer)** — Gemara evaluations correlating controls→evidence, producing the auditor's sampling frame. This is what turns raw events into "evidence for control X."
6. **D4 (signing baseline) + F1 (thin)** — signing is MUST-from-day-one here (it's the differentiator), plus a minimal API + the export-adapter surface (the memo's "external-tool adapter layer at the same architectural rank as enforcement engines").

**What W4 omits:** A2 (lifecycle not needed to package evidence), E1 (no simulation in the minimal evidence play — though it's a natural upsell), E2 full console, D2 multi-tenant (enterprise single-tenant), D3.

**Why this order:** C2 → effector → A1 crosswalk → C5/C1 export. W4 is the **wedge into the compliance buyer** and pairs with co-marketing ("Certified by Vanta as a Verified Evidence Source").

---

### W6 — Approval Gate Mesh

> *Memo Wedge 6: the §17B problem — admission webhooks have request deadlines, so you can't hold one open for a human approval. No productized cross-system approval gate exists.* This is the **smallest, sharpest** wedge: "once installed it becomes infrastructure."

**Minimum components, in order:**

1. **B4 (CRDs)** — the `PolicyApprovalRequest` CRD (+ `PolicyException`) and the closed action taxonomy that contains `suspend_pending_approval` / `require_approval`. B4 owns the CRD schema (Domain-F hard decision #5: B4 owns schema, F2 owns controllers).
2. **B2 + B5 (deny-with-approval-required pattern)** — the keystone integration (Domain-B hard decision #2): admission returns deny-with-approval-required, creates a `PolicyApprovalRequest` CRD, re-evaluates on retry via Gatekeeper external data; **never holds the webhook open** (§17B.4 invariant). This is the one vertical slice that *is* the product.
3. **D3 (approval-gated decisions)** — the approval state machine (pending → approved/denied/expired), CR-name-doubles-as-`correlation_id`, **approval binds to the resource's spec digest** so "approve-then-swap" fails (Domain-D hard decision #4), and callbacks carry approver-bound proof, not just a shared HMAC.
4. **F1 (webhook surface)** — integrations *out* to ServiceNow Approvals, Jira Service Management, GitLab MR approvals, GitHub Environment reviewers, PagerDuty, Slack/Teams, Opsgenie. Approval state is the source of truth; downstream engines query it.
5. **C2 (thin) + D1 (thin)** — emit `approval.request` / `approval.decision` events (already in the §13 `event_type` enum) for the audit trail; resolve the requesting subject. Reuse foundation contracts.

**What W6 omits:** A1 full / A2 / E1 / E2 full / C3 / C4 / C1. W6 is almost pure Domain-B/D plumbing — it's the wedge with the **smallest component footprint** and the cleanest scope.

**Why this order:** B4 → B2/B5 → D3 → F1 is exactly the Domain-B/D "approval vertical slice" the summaries flag as the keystone and the highest-risk build. W6 isolates that slice and sells it standalone. It's the memo's "tactical add-on that strengthens whichever stack you pick" — so it's also the natural *second* wedge regardless of which you lead with.

---

## 3. Comparison: platform-first vs wedge-first

| Dimension | Platform-first (MASTER-PLAN.md / F3 PLAN) | Wedge-first (this ALT) |
|---|---|---|
| **Time-to-first-value** | End of Wave 4 (E2 console + AC-1..8). Demo before dollars; the sellable unit ("unified platform") is the *last* thing built. | End of first wedge (W1: C2+B1+E1+thin C5/F1). A signed differential report is sellable on its own — the critical path *is* the product. |
| **Time-to-first-revenue** | Late — the buyer the platform targets ("policy platform") is the one market research says doesn't exist. Revenue waits on a full demo. | Early — each wedge targets a buyer who exists *now* (W5: Styra orphans shopping today; W4: existing Vanta customers; W1: GRC-adjacent). |
| **Throwaway risk** | Low *per component* (DAG is honored) but **high opportunity risk**: 6–18 months before market feedback; if the platform thesis is wrong you learn late. | Low if contracts freeze early (§6). Risk concentrates in **thin slices that must later thicken** (thin A1→full A1, thin D2→real scope predicate). Mitigated because thickening is additive, not a rewrite. |
| **Architectural compromise** | None — it builds the intended architecture in dependency order. | Small, bounded: single-tenant-before-multi-tenant (defer real D2), ingest-foreign-events-before-own-producers (W1), report-before-console (defer E2). Each is a *reduction*, not a *deviation* — same contracts, fewer of them lit up. |
| **Parallelism** | Maximal across all 23 from Wave 1 (needs full team day one). | Sequential across wedges, parallel *within* a wedge. Smaller team can ship W1, then grow. Better fit for a startup; worse use of a large pre-built team. |
| **Contracts you must FREEZE early** | C2 §13 schema, D1+D2 mapping+scope, A1 control_id/Gemara (the three foundation contracts, F3 §3.1). | **Same three — non-negotiable.** Plus the **B4 13-action taxonomy** (W5/W6 bake it in) and the **C2 integrity primitive** (W4 bakes it in). Wedge-first freezes *more* contracts earlier per-dollar-shipped, because a shipped wedge is in production. |
| **Contracts you can DEFER** | C1, C4, C5-rich, E3-full, F4, graph DB, export adapters, multi-IdP (Phase 2/3). | All of those **plus**, per wedge: full A1 hierarchy (defer in W1/W5/W6), real storage-scope D2 (defer until W2 — the first multi-tenant wedge), A2 lifecycle (defer in W1/W4/W6), E2 console (defer until W2), B2/B5 enforcement (defer in W1/W2). |
| **Feedback latency** | One big bet, validated late. | Each wedge is a market experiment; you learn which buyer converts before committing the next wedge's build. |
| **Convergence to full platform** | Already converged (it *is* the platform). | By accretion: each wedge adds components to a shared core; §4 shows the union of W1+W5+W6+W4+W2 ≈ the full 23 with no re-cut. |

**The single most important row:** *contracts to freeze early are identical*. Wedge-first does **not** let you skip the hard contract work — it lets you skip building *components*, not *contracts*. The C2 §13 schema, the D2 `ScopePredicate` library interface, the A1 `control_id` namespace, the B4 13-action taxonomy, and the C2 integrity/signing primitive must be designed and frozen before the *first wedge ships to a paying customer*, because re-cutting any of them after production deployment is the unrecoverable rework the platform-first plan rightly fears (F3 risk #2). The difference is only that wedge-first lets you **implement** them incrementally (additive-only, N-C2-FWD) while **selling**.

---

## 4. Recommended hybrid: lead with W1, converge by accretion

### Lead wedge: **W1 (Compliance Digital Twin)** — then W5, then W6, then W4, then W2.

**Rationale for leading with W1, not W5:**

The positioning memo's own synthesis (§"the most interesting combined play") leads with **Wedge 5 + Wedge 1** for revenue-this-year. I diverge on **build order within that pair**: build **W1 first**, then W5, for three reasons:

1. **W1 *is* the platform critical path** (C2 → B1 → E1 — F3 PLAN §4). Building W1 first means your very first shippable wedge already advances the longest, highest-novelty, highest-risk pole (E1 differential simulation, which every domain summary flags as the longest single build and the central correctness landmine). You de-risk the platform's hardest component while earning. Leading with W5 first would build B1+B4+A2 (lower-novelty, partly OCP-as-substrate) and *defer* E1 — leaving the riskiest work latest, which is the platform-first plan's mistake in miniature.
2. **W5 reuses W1 wholesale.** W5's regression-gate (step 5) *is* W1's E1, and W5's C2 decision-log subset *is* W1's C2. If you build W1 first, W5 becomes "add B4 + A2 + full B1 signing around an E1 you already have" — a fast follow, not a fresh build. Build them in the other order and you build E1 against OCP-decision-logs-only, then have to re-generalize it for foreign-event ingest (the W1 differentiator) — a partial redo.
3. **W1 is the cleanest market experiment.** It's ingestion-only (no in-cluster footprint, no enforcement risk for the customer), so the sales cycle is short and the blast radius is zero. It validates the "replay/simulation is genuinely novel and valued" thesis (market research §8/§13) *before* you commit to the heavier W5 enterprise-support build.

**The accretion path (how the wedges converge onto the full DAG without rework):**

```
W1  C2(ingest+query) · B1(replay) · E1(diff) · A1(thin) · C5(thin) · F1(thin) · D4(thin)
      │  +B1(signing+bundles) +B4(taxonomy+CRDs) +A2(lifecycle) +E1(regression-gate mode)
      ▼
W5  ...above... · full B1 · B4 · A2 · C2(decision-log subset→still emits full §13)
      │  +B4(CRDs already present) +B2/B5(deny-with-approval) +D3(approval state) +F1(webhooks) +D1(thin)
      ▼
W6  ...above... · B2 · B5 · D3 · approval mesh
      │  +A1(full hierarchy+crosswalk) +C2(full integrity) +C5(evidence packages) +C1(Privateer) +D4(signing MUST)
      ▼
W4  ...above... · full A1 crosswalk · C2 integrity · C5 evidence · C1
      │  +D1+D2(real scope predicate) +F1(graph/lineage query) +E2(graph view) +C3(coverage detector)
      ▼
W2  ...above... · D1 · D2(scope) · F1(lineage API) · E2(console) · C3
      │
      ▼
UNION(W1..W2) ≈ {A1,A2,B1,B2,B4,B5,C1,C2,C3,C5,D1,D2,D3,D4,E1,E2,F1} — 17 of 23 components
Remaining to reach full platform GA: B3(Conftest), C4(retro detection), E3(full PDP catalog), F2(operator/extensibility), F4(AI/agent) — all Phase-2/3 in F3 too.
```

Each `+` above is **additive against a frozen contract**, never a re-cut:
- C2 emits the full §13 schema from W1 even though W1 only *reads* a subset — so W4's integrity fields and W6's `approval.*` events are already in the schema (N-C2-FWD, additive-only). No consumer breaks.
- A1's `control_id` allocator is frozen in W1-thin; W2/W4 add the *hierarchy and crosswalks above it*, not a new key. The universal join key never moves.
- D2's `ScopePredicate` library *interface* is frozen early (even if W1/W5 run single-tenant and don't light it up); W2 supplies the real multi-tenant implementation behind the same interface (Domain-D decision: app-interceptor + RLS → OPA-partial-eval evolution, "an evolution, not a rewrite").
- B4's 13-action taxonomy is frozen in W5; W6 consumes `suspend_pending_approval`/`require_approval` from it unchanged.
- E1 built foreign-event-first in W1 generalizes *downward* to OCP-decision-logs in W5 (a subset), never the reverse.

**Convergence claim:** the union of the five wedges, built in this order, reaches 17/23 components with **zero contract re-cuts**, and the remaining 6 (B3, C4, E3-full, F2, F4) are exactly F3's Phase-2/3 set. The wedge-first path therefore lands on the *same* full-platform DAG as MASTER-PLAN.md — it just monetizes four times along the way.

**Where W6 sits:** I place W6 third (after W1, W5) because the memo calls it "a tactical add-on that strengthens whichever stack you pick," and because the deny-with-approval vertical slice (B2+B5+D3) is the keystone integration the Domain-B/D summaries flag as highest-risk — building it as its own focused wedge isolates that risk. It also sells into the *same buyer* as W4 (compliance/risk teams), warming the channel for W4.

---

## 5. Stack alignment (which memo stack each path builds toward)

The positioning memo defines three stacking patterns; the wedge order above is deliberately **Stack-B-then-Stack-A**:

- **W1 + W5 + W6** ≈ **Stack B** ("the OPA-and-friends successor"): simulation + OCP stewardship + approval gates. Fastest revenue, highest competitive risk (Apple/Nirmata). This is the first 6–9 months.
- **+ W4 + W2** pivots toward **Stack A** ("the connective-tissue company"): lineage graph + GRC-compatible evidence. Most defensible, slowest to monetize — but now funded by Stack B revenue and built on the same core.
- **Stack C** (FINOS/regulated vertical, + W7 AI agent governance) is reachable from here by adding **F4** as deltas (per the F4 ALT: "build base-first, *market* an agent-first slice"). F4 stays Phase-3 — the memo itself hedges Wedge 7 as "watch for six months."

This sequencing operationalizes the memo's own recommended combined play (lead Wedge 5+1, layer Wedge 2+8, use Wedge 4 as the compliance wedge), with the one correction that **W1 builds before W5** for critical-path de-risking and free reuse.

---

## 6. Trade-offs and the decision I'd make

### The honest trade-offs

**For wedge-first:**
- Revenue and market feedback in months, not quarters. Each wedge validates a buyer hypothesis before the next build.
- The hardest, highest-novelty component (E1) ships first and gets battle-tested early.
- A small team can execute; the build grows with the revenue.
- Targets buyers who *exist today* (Styra orphans, Vanta customers) rather than the "policy platform" buyer the market research says is a fiction.

**Against wedge-first (and the platform-first plan's genuine strengths):**
- Thin slices carry **thickening debt**: thin A1 must grow a full hierarchy; thin D2 must grow a real scope predicate; a report must grow into the E2 console. If a contract was cut wrong, thickening becomes re-cutting — the one fatal case. *This is why §3's "freeze the same contracts early" row is non-negotiable.*
- Less parallelism: a large pre-existing team is under-utilized building wedges sequentially. Platform-first uses a big team better.
- Two production-shaped surfaces (e.g. W1 SaaS + W5 in-enterprise) before convergence means **two operational footprints** to maintain during the transition.
- Stack-B-first accepts the memo's highest *competitive* risk (Apple/Nirmata could swallow the OCP-successor opportunity) in exchange for the fastest revenue.

### The decision

**Build wedge-first, lead with W1 (Compliance Digital Twin), in the order W1 → W5 → W6 → W4 → W2, and freeze all five load-bearing contracts before the first wedge reaches a paying customer.**

Rationale: the market research is unambiguous that the platform-as-a-whole has no buyer and every horizontal layer has an incumbent — so building the whole stack before selling anything (platform-first) maximizes the time spent on the one bet the research says is weakest. Wedge-first inverts that: it sells the *connective tissue* (the only uncontested ground) from month one, builds the platform's riskiest component (E1) first, and reaches the identical 23-component DAG by accretion. The decisive mitigations that make it safe are already in the domain specs and cost nothing extra to honor: C2 is additive-only (N-C2-FWD), A1's `control_id` is an immutable global namespace from the start, D2 ships as a `ScopePredicate` *library interface* before it ships a multi-tenant implementation, and B4's action taxonomy is closed-and-frozen. Honor those four invariants and wedge-first has **no more throwaway risk than platform-first** while delivering revenue and learning 6–12 months sooner.

The one condition under which I'd flip to platform-first: if a **large team already exists** and **a specific anchor customer has pre-committed to the full platform** — then the parallelism of MASTER-PLAN.md wins and the wedge sales motion is unnecessary overhead. Absent that, lead with W1.

---

## 7. Pointers

- Platform-first counterpart: `cross-cutting/MASTER-PLAN.md` (and its source draft, `components/F3-mvp-sequencing/PLAN.md` §3–9).
- Foundation contracts (must freeze early, both plans): C2 §13 schema — `domains/C-evidence-audit/DOMAIN-SUMMARY.md` §3; D1+D2 mapping+scope — `domains/D-identity-authz/DOMAIN-SUMMARY.md` §1; A1 Gemara+control_id — `domains/A-governance-core/DOMAIN-SUMMARY.md` §1.
- B4 action taxonomy + approval keystone: `domains/B-policy-engines/DOMAIN-SUMMARY.md` §3.
- E1 differential simulation (the W1 core): `domains/E-simulation-console/DOMAIN-SUMMARY.md` §3.
- AI-agent wedge framing (W7/F4, Phase-3): `components/F4-ai-agent-extension/ALT-ai-as-separate-product-and-async-tier.md`.
- Wedge definitions + stacks: `policy engine reframed market position.md`. Incumbent landscape / gaps: `policy engine market research.md`.
