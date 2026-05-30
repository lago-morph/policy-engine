# F3 — POC Scale, MVP Scope & Sequencing — SPEC

**Component:** F3 · **Domain:** F · **Spec source:** §22 (POC scale), §26 (resolved guidance & open questions), §27 (recommended MVP)
**Status:** Authored (domain-lead fallback) · **Date:** 2026-05-30
**Persona lens:** Founding product/eng lead drawing the MVP cut line and sequencing the build.

> NOTE: The cross-component, whole-platform build sequence (which of the 23 components A–F build concurrently, the critical path, MVP→GA phases) lives in this component's **PLAN.md** §"Whole-platform build sequence." This SPEC defines the *cut line, sizing math, and acceptance criteria*; PLAN defines the *sequence*.

---

## 1. Scope

F3 specifies **(a)** the POC scale envelope and what "sized for functional validation, not production telemetry" means concretely (§22); **(b)** the resolved design decisions that constrain MVP (§26.1); **(c)** the explicit MVP-in vs deferred cut line with rationale (§27); and **(d)** acceptance criteria + phasing that make the MVP demonstrable.

This is the **sequencing component**: it owns the cross-cut between "what we build first" and "what the §22 numbers require."

---

## 2. POC scale envelope (§22) — interpreted

| Metric | POC target (§22.1) | F3 interpretation / sizing implication |
|---|---|---|
| Clusters | 1–5 | Hub-and-spoke; 1 hub, ≤5 spokes (F2). Single-cluster is the demo default. |
| Namespaces | 10–100 | Scope predicate must perform at 100 namespaces, not 10k. |
| Policy evals/sec | 100–1,000 | **Edge-served** (admission/OPA sidecar). Control plane sees logs, not evals. |
| Audit events/day | 10k–500k | ≈6/sec avg, burst ~50/sec. 1–2 ingest workers. 30d ≈ 15M events ≈ 30–75 GB. |
| Replay window | 7–30 days | Bounded audit queries; default window when unspecified (F1 R-F1-FILTER-1). |
| Concurrent GUI users | 5–50 | 2–3 API + 2 console replicas. |
| Controls modeled | 25–100 | Trivial metadata; the value is traceability, not count. |
| Product integrations | 3–6 initially | Pick the 3–6 that make the demo story (see §5). |

### 2.1 Performance acceptance (§22.2)
- **R-F3-PERF-1 (MUST):** A single-policy simulation over 10k audit events completes interactively or as a short background job.
- **R-F3-PERF-2 (MUST):** Namespace-scoped replay is the default simulation mode; full-cluster replay supported only for small POC datasets.
- **R-F3-PERF-3 (SHOULD):** Reports generate from bounded datasets; no terabyte/petabyte telemetry (§26.1).
- **R-F3-PERF-4 (MUST):** "Functional validation, not production telemetry processing" — the POC's success metric is *correct, traceable, replayable decisions*, not throughput.

---

## 3. Resolved design decisions that constrain MVP (§26.1)

These are **locked** and every component must honor them:

| Topic | Locked decision | MVP constraint |
|---|---|---|
| Gemara→Rego generation | Auto-generate only when complete/deterministic/safe; else templates+stubs+tests+comments | A1/A2/B1: ship the safe-subset generator + template fallback; do NOT promise full synthesis (§27.2 defers automated control synthesis). |
| OCSF | Optional reference model; platform defines its own minimum replay schema (authoritative) | C2 owns the authoritative §13 schema; OCSF is an optional export mapping (F2 adapter). |
| Wasm | Not a user-facing requirement; later optimization | F2 plugins out-of-process; no Wasm dependency in MVP. |
| Policy lineage | Preserve graph-shaped lineage; do NOT require a graph DB at spec phase; storage out of scope for POC | A1 emits lineage records; storage can be ordinary; graph DB deferred (positioning memo Wedge-2 is post-MVP). |
| Telemetry scale | No TB/PB; small POC sizing | C3 analytics is NOT a large streaming cluster (§24.2 caveat). |
| Signed bundles | Bundles signed; signing impl unspecified at POC | B1/F2: bundle carries signature metadata + verification hook; specific signer (cosign/Sigstore) is an impl choice, not blocking. |

---

## 4. MVP cut line (§27)

### 4.1 IN — the 9 MVP items (§27.1) mapped to components

| # | §27.1 MVP item | Owning component(s) | Why it's in |
|---|---|---|---|
| 1 | Gemara governance definitions | **A1** | The governance spine; nothing traces without it. |
| 2 | OPA/Rego runtime policies | **B1** | The primary decision engine + replay engine (§17C). |
| 3 | Gatekeeper admission enforcement | **B2** | Real-time K8s enforcement (the demo's "deny"). |
| 4 | Conftest CI validation | **B3** | Build-time gate; cheap, high-signal. |
| 5 | Basic audit normalization | **C2** | The replay-capable schema — the platform's novel core. |
| 6 | Keycloak JWT mappings | **D1** | Identity-aware decisions + scope. |
| 7 | Lightweight Headlamp GUI | **E2** | The minimum useful console: graph, replay, simulation, scoped authoring (§26.3). |
| 8 | Historical replay simulation | **E1** | The differential simulation killer feature (§17.4). |
| 9 | Policy lineage graph | **A1** (lineage records) | Governance-to-runtime traceability (the G1 requirement). |

### 4.2 MVP enablers NOT in the §27.1 list but REQUIRED to ship it

These are implied; F3 makes them explicit so they aren't dropped:

- **D2 (scoped roles + storage authz)** — §17A.1 makes GUI-only authz insufficient; MVP cannot demo multi-namespace without it. **MVP-required (thin slice).**
- **F1 (API)** — the §21.1 8 endpoints; the console + simulate need them. **MVP-required (the 8).**
- **F2 (deployment/operator + core CRDs)** — you must install it; `PolicyApprovalRequest`+`PolicySimulationRun` minimum. **MVP-required (thin slice).**
- **C3 (compliance analytics — bypass/drift detection)** — §14.2 examples (Gatekeeper bypass, JWT drift) are the proof the platform "works"; at least the two example detections. **MVP-required (2 detections).**
- **B4 (engine selection / action taxonomy)** — needed to route OPA vs Gatekeeper vs Conftest; thin matrix. **MVP-required (config only).**

### 4.3 DEFERRED (§27.2 + derived)

| Deferred item | §27.2 / derived | Rationale |
|---|---|---|
| Full SIEM integration | §27.2 | Export adapters are post-MVP (positioning memo Wedge-4). |
| Large-scale streaming analytics | §27.2 | POC is not TB/PB (§26.1). |
| Cross-cloud federation | §27.2 | §22 is 1–5 clusters; hub-spoke suffices. |
| AI-assisted policy authoring | §27.2 | Generation is template-first (§26.1). |
| Automated control synthesis | §27.2 | Safe-subset only. |
| **F4 AI/agent governance extension** | derived | Built as DELTAS on the base; sequenced AFTER the base MVP proves the architecture. The reframe doc explicitly says the base needs no change for agents — so base-first. |
| **D3 approval-gate mesh (full)** | derived | The `suspend_pending_approval` outcome + webhook is MVP-thin; full ServiceNow/Jira/Slack mesh is post-MVP (Wedge-6). |
| **C4 retrospective detection (advanced)** | derived | Beyond the 2 §14.2 examples; advanced bypass/drift detection is phase-2. |
| **C5 reporting (rich)** | derived | One human-readable report category at MVP; rich multi-framework crosswalk later. |
| **E3 per-product PDP libraries (full catalog)** | derived | MVP ships the K8s + 1–2 libraries; full §17D catalog (and F4's agent catalog) is phased. |
| Lineage **graph database** | §26.1 | Lineage *records* at MVP; queryable graph DB post-MVP (Wedge-2). |

### 4.4 Cut-line rationale (the principle)
- **R-F3-CUT-1 (MUST):** MVP = the minimum that demonstrates **governance-to-runtime traceability + replay/differential simulation** end-to-end on ≥1 real product (Kubernetes). Everything that doesn't serve that single demo loop is deferred.
- **R-F3-CUT-2 (MUST):** No MVP item may rely on a deferred item. (E.g. E1 simulation must not require the graph DB; it uses lineage records.)
- **R-F3-CUT-3 (SHOULD):** Each deferred item must have a clean seam (plugin/adapter/CRD) so it bolts on without refactor (F2 extensibility) — especially F4.

---

## 5. The 3–6 MVP integrations (§22.1) — chosen

To tell the demo story (deny an unsigned image, replay it, simulate a relaxation):
1. **Kubernetes** (Gatekeeper admission) — the headline enforcement.
2. **Conftest in CI** — build-time gate on the same control.
3. **Keycloak** — identity for scoped authoring/approval.
4. **Trivy** (scan evidence feeding the image-signature/CVE control) — optional 4th.
5. **GitLab or Jenkins CI** — optional 5th (where Conftest runs).
6. **(reserved)** — leave headroom; do not add a 6th unless it serves the demo.

- **R-F3-INT-1 (SHOULD):** Pick integrations that share one control (e.g. `SC-IMG-001` signed-image) so traceability is visible across build-time, admission, and audit-replay.

---

## 6. MVP acceptance criteria (the demo passes if…)

- **AC-1:** Author a Gemara control → generate/stub a Rego policy → bundle (signed) → promote draft→dry-run→enforce (§9.2). (A1/A2/B1/B4)
- **AC-2:** Gatekeeper denies an unsigned-image Deployment in `payments-prod`; Conftest fails the same in CI. (B2/B3)
- **AC-3:** The deny emits a §13 replay-capable audit event with control_id, subject, jwt_claims, request_object. (C2)
- **AC-4:** A namespace-scoped user can replay the last 7–30 days for their namespace ONLY (D2 scope), and cannot see other namespaces. (D2/E1/F1)
- **AC-5:** Differential simulation: "if I add rule X, N more events would be denied / M newly allowed" over 10k events interactively. (E1, §17.4)
- **AC-6:** Console shows the lineage graph (control→policy→enforcement event→audit), the replay, and the simulation. (E2)
- **AC-7:** C3 flags a synthetic Gatekeeper bypass and a JWT-drift case (§14.2). (C3)
- **AC-8:** All of the above enforced server-side (storage + API), not GUI-only (§17A.1). (D2/F1)

---

## 7. Dependencies

F3 is meta: it consumes every component's scope and produces the cut line + sequence. Tight coupling to F1 (which endpoints are MVP), F2 (install thin slice), and every domain's "thin slice" definition. The whole-platform sequence (PLAN.md) is the primary's reconciliation input.

---

## 8. Open questions — decided defaults

| # | Question | Decided default | Rationale |
|---|---|---|---|
| OQ-F3-1 | Is D2/F1/F2/C3 "in MVP" despite not being in §27.1's 9? | **Yes, as thin slices** | §27.1's 9 are undeployable without them (authz, API, install, proof-of-correctness). |
| OQ-F3-2 | Base-first or AI-first (F4)? | **Base-first** | Reframe doc: base needs no change for agents; prove the architecture, then bolt F4 on as deltas. |
| OQ-F3-3 | How many integrations at MVP? | **3 core (K8s, Conftest, Keycloak) + up to 2 optional** | One shared control across build/admission/replay tells the story. |
| OQ-F3-4 | Graph DB in MVP? | **No — lineage records only** | §26.1 locks this. |
| OQ-F3-5 | Which §14.2 detections at MVP? | **The 2 examples (Gatekeeper bypass, JWT drift)** | They are the platform's proof-of-value. |
| OQ-F3-6 | Signed-bundle signer choice | **cosign/Sigstore as default impl, but unspecified per §26.1** | Don't block MVP on signer debate. |

---

## 9. Normative requirements summary

R-F3-PERF-1..4, R-F3-CUT-1..3, R-F3-INT-1; acceptance AC-1..8. The cut line (§4) + acceptance (§6) are the conformance definition of "MVP done."
