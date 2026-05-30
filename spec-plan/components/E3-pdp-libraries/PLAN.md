# E3 — Per-Product PDP Libraries — PLAN

**Component:** E3 · **Spec:** `SPEC.md` (this dir) · **Date:** 2026-05-30

---

## 1. Dependency DAG

```
   ┌──────────────────────────────────────────────────────────┐
   │ §13/C2 replay schema  +  §17C.3 action taxonomy  +         │
   │ §17C.6/B4 PolicyActionLibrary CRD  (contract inputs)       │
   └───────────────┬──────────────────────────────────────────┘
                   ▼
        [WS-T Template + catalog tooling]   (the reuse contract)
                   │
   ┌────────┬──────┼──────┬──────┬──────┬──────┬──────┬──────┐
   ▼        ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼
  K8s    Keycloak Jenkins GitLab Trivy OWASP Sonar Grafana Elastic   (9 product libs, parallel)
   │        │      │      │      │      │      │      │      │
   └────────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
                   ▼
        [WS-V Validation: examples replayable by E1, scope-mappable by D1]
```

**Critical path:** template + catalog tooling (WS-T) → then **all 9 product libraries are embarrassingly parallel** → validation.

---

## 2. Parallel workstreams

| WS | Scope | Blocks on | Parallel-with |
|---|---|---|---|
| **WS-T** Template + tooling | §4 template, CRD packaging, build-time YAML expansion, validation of `recommended_action ∈ supported_actions`, completeness-notes lint | §13, §17C.3, §17C.6 contracts | — (gates the 9) |
| **WS-K8s** | Kubernetes library (§5.1) | WS-T | all other product WS |
| **WS-KC** | Keycloak (§5.2) | WS-T | " |
| **WS-JEN** | Jenkins (§5.3) | WS-T | " |
| **WS-GL** | GitLab (§5.4) | WS-T | " |
| **WS-TRV** | Trivy (§5.5) | WS-T | " |
| **WS-OWA** | OWASP (§5.6) | WS-T | " |
| **WS-SON** | SonarQube (§5.7) | WS-T | " |
| **WS-GRA** | Grafana/Prometheus (§5.8) | WS-T | " |
| **WS-ELK** | Elasticsearch/Kibana (§5.9) | WS-T | " |
| **WS-V** Validation | each library's examples replay in E1; subject_mapping lands in §13 + §17A.4; example controls link to A1 | WS-T + ≥1 lib | overlaps later libs |

**The 9 product libraries are the textbook embarrassingly-parallel workload** — once the template is frozen, nine authors (ideally domain SMEs per product) work fully independently. No inter-library dependency.

---

## 3. Milestones

- **MVP-0:** WS-T template frozen + CRD packaging + lint (recommended_action validity, completeness notes present).
- **MVP-1 (anchor):** Kubernetes library complete + validated end-to-end via E1 (DT-68, DT-64) — proves the template against the richest, most-enforceable product.
- **MVP-2:** CI/CD cluster — Jenkins, GitLab, Trivy, SonarQube, OWASP (DT-70..74) — these share CI/CD PDP class and gate semantics; batch them.
- **MVP-3:** Identity (Keycloak, DT-69) + Data/API (Elastic, DT-76) — heavier on detect-only/proxy nuance.
- **MVP-4:** Observability (Grafana/Prometheus, DT-75) — indirect-enforcement (promotion-gate) pattern.
- **GA:** all 9 validated for cross-product differential replay (E1) + graph nodes (E2); extensibility doc for adding product #10.

**Sequencing rationale:** K8s first (richest enforcement, validates template), then batch by PDP class so shared semantics (CI/CD gates, detect-only, proxy-required) are solved once per class.

---

## 4. What can be built concurrently / what blocks what

- **WS-T blocks all 9.** Freeze the template first; a late template change forces rework across nine libraries.
- **After WS-T: all 9 fully parallel** — assign one author per product.
- **WS-V can start as soon as the first library + E1 stub exist**, then runs continuously as libraries land.
- **External inputs that must be stable before WS-T:** §13 field names (C2), §17C.3 action list, §17C.6 CRD shape. Drift here ripples into every library's `replay_input_schema` and `supported_actions`.

---

## 5. Test strategy

| Layer | Tests |
|---|---|
| Template lint | every `recommended_action ∈ supported_actions`; `realtime_hook.available=false` ⇒ actions ⊆ {detect,alert,require review,notify}; `replay_completeness_notes` present |
| Schema mapping | every `replay_input_schema.required` is a valid §13 field; `subject_mapping.normalized_to` matches §13 subject + §17A.4 claims |
| Example replay (per product) | each example control's policy replays in E1 over a synthetic fixture and yields `recommended_action` (DT-68..76) |
| Cross-product | a Keycloak deny and a K8s deny share §13 shape; cross-product differential run over a mixed evidence set classifies correctly |
| CRD | `PolicyActionLibrary` instance validates; version pinning; signature verify |
| Limitations honesty | detect-only decision points reject enforcement actions at lint time |

**Coverage anchors:** one passing E1-replay test per product (DT-68 K8s, DT-69 Keycloak, DT-70 Jenkins, DT-71 GitLab, DT-72 Trivy, DT-73 OWASP, DT-74 Sonar, DT-75 Grafana, DT-76 Elastic).

---

## 6. Risks / mitigations

| Risk | Mitigation |
|---|---|
| Template churn after libraries authored | Freeze template (WS-T) before fan-out; version the template |
| §13/§17C contract drift | Pin to spec; adapter for renames; libraries reference fields symbolically |
| Over-promising enforceability (detect-only sold as block) | Lint enforces `realtime_hook.available=false` ⇒ detect-only actions; ADVERSARIAL flags this as top risk |
| Proxy-required products (ES, Grafana) have no enforcement without deployment work | Document requirement; mark indirect/detect-only; deployment (F2) owns proxy |
| 9 SMEs produce inconsistent style | Build-time YAML expansion from a single template enforces uniformity |
| Examples rot vs real product APIs | Pin to library_version; E1 regression fixtures re-run on bump |
