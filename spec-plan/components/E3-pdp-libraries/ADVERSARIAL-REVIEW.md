# E3 — Per-Product PDP Libraries — ADVERSARIAL REVIEW

**Component:** E3 · **Reviewer persona:** Daniel (Auditor) + Priya (Compliance) + red-team hat · **Date:** 2026-05-30
**Target:** `SPEC.md`, `PLAN.md` in this dir.

---

## 1. Attacks on core assumptions

**A1. "Detect-only" is sold next to "enforce" and buyers will conflate them.** The catalog's honesty (marking `realtime_hook.available=false`) is good, but the *presentation* lists detect-only and enforceable decision points in the same table. A compliance lead reading "Exec into pod → require review" (K8s) or "Document delete → deny if proxied" (ES) may design a control assuming enforcement that **does not exist without extra infrastructure** (break-glass tooling, a reverse proxy). The catalog's value is also its risk: it makes ad-hoc hooks look like a uniform enforcement fabric when most rows are observe-only or proxy-conditional. → enforceability must be a **prominent, sortable column**, and controls authored against detect-only points must be flagged "detective control — not preventive."

**A2. "deny if proxied" is doing enormous load-bearing work.** Elasticsearch (§5.9), Grafana (§5.8), and parts of Keycloak (§5.2) only enforce *if* a proxy/app-PDP/extension is deployed. The catalog declares the requirement but OQ-5 punts ownership to deployment (F2). In practice **most installs will not have the proxy**, so the entire ES/Grafana "enforcement" story collapses to detect-only. The catalog risks over-promising a capability that depends on un-owned infrastructure.

**A3. Replay completeness is asserted "Yes" for every product, but the audit sources vary wildly.** Every library claims `retrospective_replay: Yes`. But Jenkins build logs, Grafana alert history, and Keycloak event listeners are *nothing like* the structured §13 admission event. Reconstructing a faithful policy input from a Jenkins build log is far harder than from an AdmissionReview. The uniform "Yes" overstates how authoritative cross-product replay will actually be — most non-K8s products will yield `partial`/`insufficient` replays (E1's gate), undermining cross-product differential simulation.

**A4. Subject normalization across 9 identity systems is hand-waved.** `subject_mapping.normalized_to` assumes Jenkins users, GitLab CI identities, ES API keys, Keycloak service accounts, and K8s service accounts all map cleanly to `{sub, groups, roles, tenant, namespaces}`. They don't. An ES API key has no `groups`; a Jenkins job has no `tenant`. Identity-aware policies (the platform's G4 goal) degrade silently when the mapping is partial, and the catalog provides no per-product mapping-confidence signal.

---

## 2. Gaps / unhandled cases

**G1. Versioning of the *product itself*, not just the library.** Keycloak event listener APIs, GitLab compliance frameworks, Trivy report formats, and ES audit schemas change across product versions. The library pins `library_version` but not the **supported product version range**. An example policy for GitLab 16 may silently break on GitLab 17. No product-version compatibility matrix.

**G2. No coverage metric for "how much of a product is covered."** The catalog lists *some* decision points per product but never claims completeness. Is "8 Kubernetes decision points" the whole attack surface or a sample? Without a coverage statement, a control author can't know whether the absence of a decision point means "not policy-relevant" or "not yet catalogued."

**G3. Conflicting actions across overlapping hooks.** GitLab "MR approved" (approval rules) and "Pipeline completed" (status check) can both gate the same merge with different outcomes. Trivy "image scan" and SonarQube "quality gate" both gate the same build. The catalog lists decision points independently; **composition/conflict resolution when multiple PDPs fire on one logical event is unspecified.**

**G4. External-data examples (signatures, CVE feeds) inherit E1's external-data store gap.** "Require signed image / block critical CVEs" (K8s, Trivy) depend on external data whose *value at decision time* must be captured for replay (E1 ADVERSARIAL D2). The catalog's `external_data_refs` notes the dependency but the snapshot-value store is unowned — so these flagship examples replay as `partial`.

**G5. No threat model for the catalog as an attack map.** A read-only catalog of "what cannot be enforced directly" (`limitations`) is a **gift to an attacker**: it enumerates exactly which actions are detect-only or unproxied. Access to the limitations data should itself be scoped (Security Reviewer / Admin), not broadly readable.

**G6. "Recommended engine" mixes incompatible execution models.** Examples recommend OPA, Gatekeeper, Kyverno, Conftest, "tool gate," and "app PDP" interchangeably. A control author picking "OPA" for a Grafana alert-rule change has no OPA execution context there — it's a webhook/CI gate. The recommended_engine field needs validation that the engine can actually run at that decision point.

---

## 3. Inconsistencies vs other components

**X1. §13 field subset vs C2's actual schema.** Each library's `replay_input_schema.required` references §13 fields. If C2 (built in parallel) renames/omits fields, every library's required-list breaks (same risk as E1 §7). The libraries reference fields by name in prose — needs the shared versioned `DATA-MODEL.md` contract.

**X2. Action taxonomy drift vs §17C.3.** Libraries use actions like `detect`, `alert`, `attach evidence`, `clear hold`, `block promotion`, `fail build` — several of which are **not in the §17C.3 taxonomy** (which has allow/deny/warn/mutate/generate/cleanup/quarantine/suspend/require_approval/require_scan/notify/annotate/exception). "detect", "alert", "fail build", "block promotion", "clear hold", "attach evidence" are product-specific verbs. The catalog has silently *extended* the taxonomy. Either §17C.3 must be expanded or these must map onto canonical actions (e.g., detect→notify, fail build→deny).

**X3. CRD ownership (`PolicyActionLibrary`) split with B4.** E3 §7 ships libraries as `PolicyActionLibrary` CRD instances; B4 (§17C.6) owns the CRD definition. Same E1/B4 split — must reconcile: B4 defines, E3 populates.

**X4. Example controls must link to A1 controls, but A1's control IDs are not yet fixed.** Examples use IDs like `SC-IMG-001`; if A1's Gemara hierarchy assigns different IDs, the lineage graph (E2) edges break. Examples should reference A1 controls by a stable key, not invent IDs.

---

## 4. "Won't survive production" findings

**P1. The cross-product differential promise rests on the weakest product's replay fidelity.** The platform's headline ("simulate any policy change over normalized audit logs across all products") is only as strong as the *least* replayable product. Grafana/Jenkins/ES replays will be `partial` at best (A3, G4), so cross-product differential is authoritative mainly for Kubernetes. The catalog should set expectations: K8s is the gold replay; others are best-effort.

**P2. 9 SMEs, one template — drift is inevitable.** Build-time YAML expansion enforces *structure* but not *semantic* consistency (what counts as a "decision point," how granular, how an action is named). Without a strong style guide + review gate, the catalog becomes nine differently-grained documents wearing one schema.

**P3. Maintenance burden is permanent.** Nine third-party products, each evolving APIs/audit formats independently. The catalog is a living document requiring continuous SME maintenance, not a one-time spec. The plan treats it as a build task; it's an ongoing program. Without an owner per product post-GA, examples rot (G1).

---

## 5. Scope-creep watch

- Adding product #10, #11… is "extensible by construction," which invites unbounded product sprawl. Bound the supported set; treat new products as funded additions, not free.
- The catalog edges toward becoming a universal integration layer for 9 products before the Kubernetes core is proven. PLAN's "K8s first" mitigates; hold that line.

---

## 6. Prioritized defect list

| # | Severity | Defect | Fix |
|---|---|---|---|
| D1 | **Critical** | Detect-only / "deny if proxied" rows presented beside true enforcement ⇒ buyers design controls against enforcement that doesn't exist (A1, A2, P1) | Prominent enforceability + proxy-required columns; flag detective-only controls; set "K8s gold, others best-effort" expectation |
| D2 | **Critical** | Action verbs (detect/alert/fail build/block promotion/clear hold/attach evidence) not in §17C.3 taxonomy (X2) | Reconcile: expand §17C.3 or map product verbs to canonical actions |
| D3 | **High** | Uniform `retrospective_replay: Yes` overstates non-K8s replay fidelity (A3, G4, P1) | Per-product replay-fidelity rating; external-data snapshot store ownership (with E1/C2) |
| D4 | **High** | Subject normalization across 9 identity systems is partial/hand-waved (A4) | Per-product mapping-confidence + missing-claim notes; degrade identity-aware policies explicitly |
| D5 | **High** | Multi-PDP composition/conflict on one logical event unspecified (G3) | Define precedence/composition when multiple decision points gate the same action |
| D6 | **Medium** | recommended_engine can be incompatible with the decision point (G6) | Validate engine executes at that hook; lint |
| D7 | **Medium** | §13 field + A1 control-ID contracts not yet fixed; libraries hard-reference them (X1, X4) | Reference via shared versioned DATA-MODEL.md + stable A1 keys |
| D8 | **Medium** | No product-version compatibility matrix (G1, P3) | Add supported product-version range; assign per-product maintainer |
| D9 | **Medium** | Catalog limitations are an attacker's map (G5) | Scope `limitations` data to Security Reviewer/Admin |
| D10 | **Low** | No coverage statement per product (G2) | State whether decision-point list is exhaustive or sampled |
| D11 | **Low** | CRD ownership split E3/B4 (X3) | B4 defines, E3 populates — state in both |
| D12 | **Low** | 9-SME semantic drift under one schema (P2) | Style guide + review gate beyond structural lint |
