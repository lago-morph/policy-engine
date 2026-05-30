# F2 — Deployment & Extensibility — SPEC

**Component:** F2 · **Domain:** F · **Spec source:** §24 (Deployment Model), §25 (Extensibility Model) (+ §22 scale, §17C.6 CRDs)
**Status:** Authored (domain-lead fallback) · **Date:** 2026-05-30
**Persona lens:** Kubernetes platform/SRE engineer + plugin-ecosystem maintainer.

---

## 1. Scope

F2 specifies **(a)** the Kubernetes-native deployment topology — operators, CRDs, HA, scaling to POC targets, multi-cluster reach, install/upgrade — and **(b)** the plugin/extensibility model — how new PDPs, policy engines, IdP/JWT mappers, evidence collectors, visualization extensions, SIEM integrations, and export adapters plug in without forking the platform.

§24 names deployment targets (K8s, OpenShift, managed K8s) and a per-component deployment table; §25 lists six plugin categories. This spec turns those into a concrete operator/CRD topology and a versioned plugin contract.

**In scope:** packaging, operators, CRDs, HA/scaling, install/upgrade, the plugin SPI. **Out of scope:** the internal logic of each component; the storage backend choice (§26.1 out-of-scope for POC, but F2 defines *where* it plugs in).

---

## 2. Deployment topology (K8s-native)

### 2.1 Control plane vs data plane

- **Governance control plane** (one logical install, HA-capable): API tier (F1), governance/lineage services (A1/A2), simulation engine (E1), analytics (C3), audit store interface (C2), authz/identity broker (D1/D2), console/Headlamp backend (E2), operator(s) + CRD controllers. Runs in a dedicated `governance-system` namespace (the "hub").
- **Enforcement data plane** (per managed cluster / spoke): Gatekeeper (admission), Kyverno (optional, Kubernetes-native actions §17C), OPA (sidecar or centralized §24.2), decision-log + audit shippers, approval CRD controllers. Conftest runs in CI/CD (not in-cluster). Privateer runs as evaluation services.

This hub-and-spoke matches §22 (1–5 clusters): one hub, up to 5 enforcement spokes; the hub MAY also be a spoke (single-cluster POC).

### 2.2 Per-component deployment (from §24.2, made concrete)

| Component | K8s shape | HA at POC | Notes |
|---|---|---|---|
| OPA | Sidecar (app PDP) **or** centralized Deployment | 2+ replicas if centralized | Bundle pulled from signed-bundle server |
| Gatekeeper | Cluster admission controller (per spoke) | webhook 2+ replicas, leader-elected audit | Constraint templates from B2 |
| Kyverno (optional) | Admission controller (per spoke) | 2+ replicas | Only where Kubernetes-native actions needed (§17C) |
| Conftest | CI/CD job image | n/a | Invoked by pipelines / `POST /conftest/run` |
| Privateer | Evaluation Deployment | 2+ replicas | Evidence collectors |
| Governance API (F1) | Deployment + Service + Ingress | 2–3 replicas | Stateless; HPA on CPU/QPS |
| Simulation engine (E1) | Deployment + job workers | 2 + worker pool | Bursty; scale workers for replay jobs |
| Analytics (C3) | Streaming workers | 1–2 (POC, not large-scale §26.1) | NOT a large streaming cluster at POC |
| Audit store (C2) | StatefulSet or external managed store | 1 (POC) | "Ordinary relational/document/object storage" §22.2 |
| Console/GUI (E2) | Headlamp plugin **or** standalone Deployment | 2 | Plugin-everywhere per positioning memo |
| Operator + CRD controllers | Deployment, leader-elected | 2 (active/standby) | Reconciles governance CRDs |
| Keycloak/IdP (D1) | External or in-cluster Deployment | per IdP | "One of many" IdPs; JWT-mapping is the contract |

### 2.3 Custom Resource Definitions (the API of the platform-in-cluster)

Built on §17C.6. Group `governance.example.io`, version progression `v1alpha1 → v1beta1 → v1`.

| CRD | Purpose | Reconciled by |
|---|---|---|
| `PolicyApprovalRequest` | Approval-gated decisions (§17B); admission "can't wait" pattern | Approval controller → webhook |
| `PolicySimulationRun` | Async simulation job (E1) as a cluster object | Simulation controller |
| `PolicyActionLibrary` | Per-product action/PDP library (§17D) registration | Library controller |
| `PolicyEvidenceSchema` | Per-product replay schema (§26.3) | Schema controller |
| `PolicyException` | Approved exception with scope+expiry | Exception controller |
| `PolicyRemediationAction` | Remediation hooks (quarantine/cleanup §17C.3) | Remediation controller |
| `GovernanceBundle` (new) | Signed policy bundle + Gemara/OSCAL control metadata + required JWT claims | Bundle controller |
| `PolicyEnginePlugin` (new) | Registers an engine/PDP plugin (§25) | Plugin controller |
| `ExportAdapter` (new) | Registers a SIEM/GRC export sink (§25, positioning memo) | Adapter controller |

- **R-F2-CRD-1 (MUST):** All CRDs carry the §17A.5 authorization metadata (scope) so controllers and storage enforce scope uniformly.
- **R-F2-CRD-2 (MUST):** Controllers are idempotent, leader-elected, and emit §13 audit events on every reconcile that changes enforcement-relevant state.
- **R-F2-CRD-3 (SHOULD):** Status subresources + printer columns for `kubectl get`/console.

### 2.4 Operator(s)

- **R-F2-OP-1 (MUST):** A `governance-operator` installs/upgrades the control plane via a top-level `GovernancePlatform` CR (declarative install: which engines, which spokes, storage backend ref, IdP ref). One operator, multiple CRD controllers (can be split later).
- **R-F2-OP-2 (SHOULD):** OLM (OperatorHub/OpenShift) + Helm chart + raw manifests, all generated from one source. Targets §24.1 (K8s, OpenShift, managed K8s).

---

## 3. HA, scaling, and the POC sizing envelope

POC targets (§22.1): 1–5 clusters, 10–100 namespaces, **100–1,000 evals/sec**, **10k–500k audit events/day**, 7–30 day replay window, **5–50 GUI users**, 25–100 controls, 3–6 integrations.

- **R-F2-SCALE-1 (MUST):** Stateless services (API, console, simulation API) horizontally scale via HPA; enforcement (Gatekeeper/OPA) scales with cluster admission load, not platform load.
- **R-F2-SCALE-2 (SHOULD):** Eval throughput (1k/sec) is served at the **enforcement edge** (admission webhooks / OPA sidecars), so the control plane is not on the eval hot path. The control plane sees decision *logs*, sized at 500k/day ≈ **~6/sec average, bursty** — trivially handled by 1–2 ingest workers + ordinary storage.
- **R-F2-SCALE-3 (MUST):** Audit storage sized for 30 days × 500k/day = **15M events** max retained; at ~2–5 KB/event ≈ **30–75 GB**. Ordinary storage suffices (§22.2). Replay datasets materialized as scoped subsets (§17A.5) before simulation.
- **R-F2-SCALE-4 (SHOULD):** Simulation worker pool sized so a single-policy replay over 10k events completes "interactively or as a short background job" (§22.2). Namespace-scoped replay is the default; full-cluster only for small datasets.
- **R-F2-HA-1 (SHOULD):** Each control-plane service ≥2 replicas with PDBs; admission webhooks ≥2 with `failurePolicy` chosen deliberately (see failure modes). Single-node POC profile allowed (1 replica) explicitly as a non-HA tier.

### 3.1 Capacity math (documented for F3 reuse)

| Quantity | POC max | Implication |
|---|---|---|
| Eval rate | 1,000/sec | Edge-served; control plane sees logs only |
| Audit ingest | 500k/day ≈ 6/sec avg (burst ~50/sec) | 1–2 ingest workers |
| Audit volume @30d | ~15M events ≈ 30–75 GB | Ordinary relational/object store |
| Simulation job | 10k events single policy | Seconds–minutes; worker pool of 2 |
| GUI concurrency | 50 users | 2–3 API replicas, 2 console replicas |
| Controls | ≤100 | Trivial metadata footprint |

---

## 4. Install, upgrade, multi-cluster

- **R-F2-INSTALL-1 (MUST):** Single-command install (Helm/operator) for the single-cluster POC; spokes added by registering cluster credentials in `GovernancePlatform` and deploying the spoke agent bundle.
- **R-F2-UPGRADE-1 (SHOULD):** CRD conversion webhooks for `v1alpha1→v1beta1`; rolling control-plane upgrades; signed-bundle promotion is independent of platform upgrade (policy and platform versioned separately).
- **R-F2-MC-1 (SHOULD):** Hub reaches spokes via pull (spoke agent pulls bundles + pushes decision logs) to avoid hub→spoke firewall holes. Matches the §22 small-fleet reality; cross-cloud federation is **deferred** (§27.2).

---

## 5. Extensibility / plugin model (§25)

Six plugin categories (§25.1): new evidence collectors, additional policy engines, custom JWT mappers, visualization extensions, SIEM integrations, alternative governance schemas. The positioning memo adds **export adapters** (Vanta/Drata/ServiceNow/Splunk/OCSF) as first-class.

### 5.1 Plugin taxonomy and SPI

| Plugin kind | SPI contract (the stable interface) | Registered via | Example |
|---|---|---|---|
| **Policy engine / PDP** | `Evaluate(input)→Decision{action,reason,control_id}` + capability descriptor (which §17C.3 actions, real-time vs replay, replay schema) | `PolicyEnginePlugin` CRD | Add Cedar, Kyverno, a model-judge evaluator (F4) |
| **Evidence collector** | `Collect(scope)→[]AuditEvent(§13)` | `PolicyActionLibrary` / collector CRD | New scanner, cloud audit source |
| **JWT/IdP mapper** | `Map(rawClaims)→AuthZSubject(§17A.4)` per §15.4 transforms | config + mapper plugin | Okta/Entra/SPIFFE mapper |
| **Visualization extension** | Console plugin (Headlamp/Backstage/OpenShift/Rancher) consuming F1 read API | console plugin registry | Trust-gradient view (F4) |
| **SIEM / export adapter** | `Export(evidencePackage)→sinkAck` (signed, framework-mapped) | `ExportAdapter` CRD | Splunk, AWS Security Lake (OCSF), Vanta evidence |
| **Governance schema** | `Ingest(doc)→GraphNodes/Edges` | schema plugin | OSCAL Compass, FINOS AIGF, Gemara variants |

### 5.2 Plugin contract requirements

- **R-F2-PLG-1 (MUST):** Every plugin declares a **capability descriptor**: kind, version, supported actions/outcomes, required input fields, produced audit fields, scope it operates over. The platform refuses to load a plugin whose descriptor is incompatible with the core schema version.
- **R-F2-PLG-2 (MUST):** Plugins run **out-of-process** (sidecar/Deployment + gRPC/HTTP SPI) by default — a faulty/malicious plugin MUST NOT crash or read outside the control plane. In-process (Wasm/Go-plugin) is an optional optimization, NOT required (§26.1 Wasm "not a user-facing requirement").
- **R-F2-PLG-3 (MUST):** Plugin decisions and collected evidence flow through the **same §13 audit schema and §17A scope** as native components. No plugin can emit unscoped or unaudited decisions.
- **R-F2-PLG-4 (MUST — supply chain):** Plugin artifacts (engine images, evidence collectors, schema bundles) are signed/attested (Sigstore-style); the bundle/plugin controller verifies signatures at admission (§23 integrity). Unsigned → load refused or quarantined per policy.
- **R-F2-PLG-5 (SHOULD):** Versioned SPI with capability negotiation; additive SPI changes are backward-compatible; a plugin pins a min/max core version.
- **R-F2-PLG-6 (MUST):** A PDP plugin MUST provide both a real-time hook descriptor AND a replay schema (§17C.5) — engines that only enforce but can't be replayed degrade the simulation guarantee and MUST be marked `replay: none` so the platform can warn.

### 5.3 How a new PDP plugs in (worked path)

1. Author a `PolicyEnginePlugin` CR with the capability descriptor + image ref + signature.
2. Plugin controller verifies signature, schedules the sidecar/Deployment, health-checks the SPI.
3. Plugin registers its PDP entry into the §17D-style catalog (`PolicyActionLibrary`), exposing decision points + replay schema (F1 surfaces it under `/engine-bindings`).
4. Policies targeting that engine are bundled (`GovernanceBundle`) with control metadata; promotion lifecycle (§9.2) applies unchanged.
5. Decisions emit §13 events; simulation/differential replay works if a replay schema was provided.

This is exactly how **F4's agent PDPs** (model gateway, MCP gateway, RAG, memory, runtime, output sink, eval gate, resource accounting) plug in — they are PDP plugins + evaluator plugins, no new platform primitive (F4 SPEC).

---

## 6. Failure modes

| Failure | Behavior |
|---|---|
| Admission webhook down (Gatekeeper/Kyverno) | `failurePolicy` decision: **Fail-closed for high-assurance namespaces, fail-open for others**, configurable per scope; both emit an audit event so C3 can detect bypass (§14.2) |
| Spoke→hub link down | Spoke buffers decision logs locally; enforcement continues from last bundle (cached); hub flags stale-evidence |
| Bundle signature invalid on pull | Spoke refuses new bundle, keeps last-good, raises alert (§23) |
| Plugin SPI crash | Plugin marked unhealthy; native enforcement unaffected (out-of-process isolation, R-F2-PLG-2); decisions that required it fail per its declared `failurePolicy` |
| Storage backend down | Reads 503; enforcement edge continues; decision logs buffered |
| Operator reconcile conflict | Leader election + optimistic concurrency; CRDs are source of truth |

---

## 7. Dependencies

- **F1:** exposes `/plugins`, `/engine-bindings`; plugin sub-resources surface under the same envelope/authz.
- **B1–B5:** the engines deployed in the data plane; B4 engine-selection matrix maps to `PolicyEnginePlugin`/`ExportAdapter` config.
- **C2:** every plugin/CRD emits §13 events.
- **D1/D2:** IdP mapper plugins implement §15.4; scope metadata on every CRD.
- **A1/A2:** `GovernanceBundle` carries Gemara/OSCAL control metadata; promotion lifecycle.
- **E2:** visualization-extension SPI = console plugins.
- **F4:** agent PDPs/evaluators are plugins (no new primitive).

---

## 8. Open questions — decided defaults

| # | Question | Decided default | Rationale |
|---|---|---|---|
| OQ-F2-1 | Hub-and-spoke vs per-cluster standalone | **Hub-and-spoke, pull-based** | Matches 1–5 cluster POC; avoids firewall holes; one console. |
| OQ-F2-2 | One operator vs many | **One operator, multiple controllers**, splittable later | Simpler install; POC scale. |
| OQ-F2-3 | In-process vs out-of-process plugins | **Out-of-process default; Wasm optional optimization** | §26.1 Wasm non-required; isolation/safety. |
| OQ-F2-4 | failurePolicy default | **Fail-closed for high-assurance scopes, configurable** | Safety vs availability tradeoff is scope-dependent. |
| OQ-F2-5 | Storage shipped or BYO | **BYO ordinary storage (ref in `GovernancePlatform`)** | §26.1 storage out-of-scope for POC; §22.2 ordinary storage acceptable. |
| OQ-F2-6 | Cross-cloud federation | **Deferred** | §27.2 explicitly defers. |
| OQ-F2-7 | Plugin signing mandatory? | **Yes, verified at admission** | §23 integrity; supply-chain story (positioning memo). |

---

## 9. Normative requirements summary

R-F2-CRD-1..3, R-F2-OP-1..2, R-F2-SCALE-1..4, R-F2-HA-1, R-F2-INSTALL-1, R-F2-UPGRADE-1, R-F2-MC-1, R-F2-PLG-1..6. MUST set is the deployment/extensibility conformance floor.
