# F2 — Deployment & Extensibility — PLAN

**Component:** F2 · **Source:** §24–25 · **Status:** Authored (domain-lead fallback)

---

## 1. Dependency DAG

```
[CRD schemas (§17C.6 + new)] ──> [CRD controllers] ──> [governance-operator] ──> [GovernancePlatform install]
                                       │                         │
[C2 audit emit] <──────────────────────┘                         │
[D2 scope metadata] <────────────────────────────────────────────┘
[B1–B5 engines] ──> data-plane manifests ──> spoke agent bundle
[§25 plugin SPI] ──> [PolicyEnginePlugin/ExportAdapter controllers] ──> plugin registry (F1 /plugins)
[Sigstore verify] ──> bundle/plugin admission
```

Critical path: **CRD schemas → controllers → operator → installable platform**. Plugin SPI parallels the operator once CRD scaffolding exists.

## 2. Parallel workstreams

- **WS-A — CRD + controller scaffolding:** define all 9 CRDs, scope metadata (R-F2-CRD-1), audit emit (R-F2-CRD-2), leader election. Foundation for everything.
- **WS-B — Operator + packaging:** `GovernancePlatform` CR, Helm/OLM/manifests from one source, single-cluster install. Depends on WS-A schemas.
- **WS-C — Data-plane / spoke:** Gatekeeper/Kyverno/OPA wiring, decision-log + audit shippers, pull-based spoke agent. Parallel to WS-A.
- **WS-D — Plugin SPI:** capability descriptor, out-of-process gRPC SPI, signature verification, six plugin kinds. Parallel once CRD scaffolding exists; this is where F4 agent PDPs land.
- **WS-E — HA/scaling/upgrade:** HPA/PDB, CRD conversion webhooks, capacity tuning to §22 envelope. Late, after install works.

## 3. Critical path

`CRD schemas → controllers (with scope+audit) → operator install (single cluster) → engine data-plane → first end-to-end enforce+log+replay`. HA/multi-cluster/plugins layer on after the single-cluster vertical slice works.

## 4. Milestones

- **M1:** All CRDs + scope metadata + status subresources; one controller (`PolicyApprovalRequest`) reconciling end-to-end.
- **M2:** Operator installs the control plane single-cluster via `GovernancePlatform`.
- **M3:** Spoke data plane (Gatekeeper/OPA) enrolled; decision logs flow to hub; capacity verified at POC envelope (§3.1 SPEC).
- **M4:** Plugin SPI live; one non-native engine + one export adapter loaded with signature verification.
- **M5:** HA profile (2+ replicas, PDBs, failurePolicy config) + upgrade/conversion webhooks; multi-spoke (up to 5).

## 5. What can be built concurrently / what blocks what

- Concurrent: WS-A and WS-C (data-plane manifests don't need the operator to be written, only the CRDs); WS-D plugin SPI alongside WS-B once CRD types exist.
- Blocks: operator (WS-B) blocks one-command install; plugin signature verification (WS-D) blocks loading any third-party engine; scope metadata (WS-A) blocks D2/F1 scope enforcement consistency.
- Cross-domain: F2 is the **delivery vehicle** for B (engines), C2 (audit emit), D (scope), F4 (agent PDPs as plugins). Coordinate CRD group/version with B4 (which also defines CRDs in §17C) to avoid duplicate CRDs.

## 6. Test strategy

- **Operator install/upgrade tests** on kind/k3s + an OpenShift target; idempotent reconcile; CRD conversion.
- **Capacity tests to the §22 envelope:** 500k events/day ingest soak; 10k-event simulation latency; 50 concurrent GUI users; confirm control plane is NOT on the 1k/sec eval hot path (edge-served).
- **Failure injection:** admission webhook down (verify fail-closed/open per scope + bypass audit event for C3); spoke link cut (buffer + cached bundle); plugin SPI crash (native enforcement unaffected, isolation holds); bad bundle signature (refused, last-good kept).
- **Plugin conformance suite:** capability-descriptor validation; a plugin that emits unscoped/unaudited decisions is rejected (R-F2-PLG-3); unsigned plugin refused (R-F2-PLG-4); PDP without replay schema flagged `replay:none` (R-F2-PLG-6).
- **Security:** out-of-process isolation (plugin can't read control-plane secrets/other scopes); RBAC on CRDs.

## 7. Risks

- CRD ownership collision with B4 (§17C also defines CRDs) → reconcile one CRD group across domains (cross-cut).
- failurePolicy default is a real availability/safety tradeoff → make it explicit per-scope config, not a hidden global.
- Plugin SPI becomes a fork-magnet if under-versioned → capability negotiation + signed plugins from day one.
- "Storage out of scope for POC" (§26.1) collides with F1 DEFECT-1 (needs a scope-predicate substrate) → F2 must still define WHERE scope metadata lives even on ordinary storage.
