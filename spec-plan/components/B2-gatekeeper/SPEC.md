# B2 — Gatekeeper Integration — SPEC

**Component ID:** B2 · **Domain:** B · **Spec source:** §9 (with §8/B1, §17B/§17C, §18/B5 deps)
**Status:** DRAFT v1 · **Date:** 2026-05-30 · **Author persona:** cooperative engineering author

---

## 1. Scope

B2 makes **OPA Gatekeeper** the Kubernetes admission enforcement point that executes
governance Rego (from B1 bundles) at the API server, emits the platform's required audit
fields (§9.3), and supports the four enforcement modes (§9.2). Gatekeeper is the "runtime"
enforcement class (§7.2) for Kubernetes resources.

**In scope:** Constraint / ConstraintTemplate schema conventions; mapping B1's canonical
`decision` semantics onto Gatekeeper's `violation` model; the four enforcement modes
(deny/warn/dryrun/audit) and their state machine; the **17 required audit fields** Gatekeeper
decisions MUST emit; admission-webhook configuration (failurePolicy, timeouts, scoping);
how `require_approval`/`suspend_pending_approval` is realized at admission (deny-with-approval-
required + CRD — the hard constraint); external-data integration; audit (periodic scan) mode;
reconciliation of admission-time vs audit-time results.

**Out of scope:** Rego authoring/metadata/signing (B1); Kyverno (B4 selects it for mutate/
generate/cleanup/image-verify); CRD controllers themselves (B4); the end-to-end realtime
sequence (B5 owns the full sequence diagram); audit-schema normalization downstream (C2).

---

## 2. Background (market context, §3 market research)

- All three hyperscalers ship Gatekeeper-derived enforcement (Azure Policy for Kubernetes,
  GCP Anthos/Config Policy Controller, and EKS users self-run it). This validates Gatekeeper
  as the admission substrate and means the platform's Constraints must coexist with
  cloud-managed Gatekeeper instances (do not assume exclusive ownership of the Gatekeeper
  deployment — **D-B2-04**: detect and tolerate a pre-existing/managed Gatekeeper).
- Kyverno is **complementary, not competitive** (§17C.1). B2 deliberately does NOT try to do
  mutation/generation/cleanup/image-verification — those route to Kyverno via B4 selection.
- Red Hat ACS (`SecurityPolicy` CR) and cloud-native policy controllers are alternatives; B2
  stays engine-portable by keeping governance semantics in B1 Rego, not in Gatekeeper-specific glue.

---

## 3. Data model / entities

| Entity | Description | Key fields |
|---|---|---|
| **ConstraintTemplate** | CRD defining a reusable constraint kind + its Rego | `crd.spec.names.kind`, `targets[].rego`, `targets[].libs`, parameters openAPIV3Schema |
| **Constraint** | Instance binding a template to scope + params + mode | `kind` (from template), `match{}`, `parameters{}`, `enforcementAction` |
| **GatekeeperDecision** | One admission or audit decision event | the 17 fields of §9.3 + extensions (§5) |
| **ConstraintStatus** | Audit results surfaced on the Constraint | `auditTimestamp`, `violations[]`, `totalViolations` |
| **ExternalDataProvider** | External data feeding decisions (e.g. signature verifier) | `provider`, `url`, cache TTL, version |
| **ApprovalDenyResult** | deny-with-approval-required outcome | `controlId`, `approvalRequestRef`, `reason`, `retryHint` |

**Relationships:** `ConstraintTemplate 1→* Constraint`; each template's Rego MUST originate from a
**B1 signed bundle** (not inlined ad hoc — see R-B2-3); `Constraint → Gemara control` via the
template's `__control_id__`/METADATA; `GatekeeperDecision *→1 Constraint`.

---

## 4. ConstraintTemplate & Constraint schemas (normative)

### 4.1 ConstraintTemplate (REQUIRED shape)

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredimagesignature
  annotations:
    governance.example.io/control-id: SC-IMG-001
    governance.example.io/bundle-revision: "1.4.0+ab12cd3"     # provenance to B1 (R-B2-3)
    governance.example.io/severity: critical
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredImageSignature
      validation:
        openAPIV3Schema:
          type: object
          properties:
            exemptImages: { type: array, items: { type: string } }
            allowedRegistries: { type: array, items: { type: string } }
  targets:
    - target: admission.k8s.gatekeeper.sh
      code:
        - engine: Rego
          source:
            rego: |
              package governance.kubernetes.imagesigning
              import data.lib.governance as g
              violation[{"msg": msg, "details": d}] {
                dec := data.governance.kubernetes.imagesigning.decision
                dec.action == "deny"
                msg := dec.messages[_]
                d := {"control_id": dec.control_id, "severity": dec.severity}
              }
            libs:
              - |
                package lib.governance
                # shared helpers vendored from B1 governance.lib.* at build time
```

### 4.2 Constraint (REQUIRED shape)

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredImageSignature
metadata:
  name: require-signed-images-prod
  annotations:
    governance.example.io/control-id: SC-IMG-001
spec:
  enforcementAction: deny          # deny | warn | dryrun  (+ scoped audit via config)
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
    namespaceSelector:
      matchExpressions:
        - { key: environment, operator: In, values: ["prod"] }
    excludedNamespaces: ["kube-system", "gatekeeper-system", "governance-system"]   # AR fail-closed carve-out
  parameters:
    allowedRegistries: ["ghcr.io/acme"]
    exemptImages: []
```

### 4.3 Normative requirements

- **R-B2-1 (MUST):** Every ConstraintTemplate's Rego MUST delegate to the canonical B1
  `data.<package>.decision` rule and translate it into Gatekeeper `violation[...]` (per 4.1).
  Authors MUST NOT re-implement governance logic inline; the template is a thin adapter so
  cross-engine conformance (B1-R30) holds.
- **R-B2-2 (MUST):** Templates and Constraints MUST carry `governance.example.io/control-id`
  matching the embedded package's `__control_id__`. Admission/audit by an out-of-band Constraint
  with no governance control-id MUST be flagged by retrospective detection (C4) as unmanaged.
- **R-B2-3 (MUST):** The Rego embedded in a ConstraintTemplate MUST be **generated from a signed
  B1 bundle**, with `bundle-revision` recorded in the annotation. Hand-edited template Rego that
  diverges from the bundle MUST fail the lifecycle gate (A2) and be detectable (drift, HL-09).
- **R-B2-4 (MUST):** `match.excludedNamespaces` MUST always include the policy stack's own
  namespaces and `kube-system` to prevent admission self-DoS (cluster-bricking; see B1 AR-6).
- **R-B2-5 (SHOULD):** Constraint `parameters` SHOULD be the only thing operators tune per-env;
  logic stays in the template (which is bundle-sourced). This keeps blast radius small.

---

## 5. Enforcement modes & state machine (normative)

Maps §9.2 onto Gatekeeper's `enforcementAction` plus the audit subsystem.

| Mode (§9.2) | Gatekeeper realization | Blocks admission? | Emits audit? |
|---|---|---|---|
| **Deny** | `enforcementAction: deny` | Yes | Yes (admission event) |
| **Warn** | `enforcementAction: warn` | No (returns warning) | Yes (warning event) |
| **Dry Run** | `enforcementAction: dryrun` | No | Yes (would-deny event) |
| **Audit** | periodic audit controller over existing resources | No (detective) | Yes (audit violations on Constraint status) |

**Promotion state machine (governance lifecycle, A2/§17.4):**

```
[dryrun] --simulate ok--> [warn] --bake period clean--> [deny]
   ^                          |                              |
   |        rollback (DT-06)  v          rollback (DT-14<->) v
   +--------------------------+------------------------------+
```

- **R-B2-6 (MUST):** A new/changed Constraint MUST enter at `dryrun`, be validated against
  simulation (E1) and the audit controller, then be promoted to `warn`, then `deny`. Direct
  authoring into `deny` MUST be blocked by the lifecycle gate (A2). (DT-05, DT-14.)
- **R-B2-7 (MUST):** Each mode transition MUST emit an audit event recording old→new action,
  actor, control_id, and bundle revision (governance traceability; HL-02).
- **R-B2-8 (MUST):** The audit (detective) controller MUST run on a configurable interval and
  reconcile *existing* cluster state against Constraints, surfacing violations that admission
  never saw (e.g. resources created while a Constraint was in dryrun, or before it existed).
  This is the §19 bypass-detection feed (DT-15, DT-30).
- **R-B2-9 (MUST):** Admission-time and audit-time results for the same resource MUST be
  reconcilable (same control_id, same decision logic). A divergence (admission allowed, audit
  flags) MUST be surfaced as a reconciliation discrepancy (DT-17), not silently ignored.

---

## 6. Required audit fields (normative — §9.3 is the floor)

Every Gatekeeper decision event (admission **and** audit) MUST include **all 17** fields:

| # | Field | Source | Notes |
|---|---|---|---|
| 1 | Timestamp | admission/audit time | RFC3339, UTC |
| 2 | Cluster ID | platform config | multi-cluster correlation (DT-32) |
| 3 | Namespace | `request.namespace` | |
| 4 | Resource Kind | `request.kind` | |
| 5 | Resource Name | `object.metadata.name` | may be generateName at admission |
| 6 | User Identity | `request.userInfo.username` | |
| 7 | JWT Subject | from D1 mapping layer | `sub` claim |
| 8 | JWT Groups | from D1 mapping layer | `groups` claim |
| 9 | Control ID | template/decision metadata | Gemara trace |
| 10 | Constraint Template | template kind | |
| 11 | Constraint Name | constraint metadata.name | |
| 12 | Rego Package | decision package path | |
| 13 | Decision Outcome | allow/deny/warn/dryrun | |
| 14 | Violation Reason | `violation.msg` | |
| 15 | Request UID | `request.uid` | |
| 16 | Admission Review UID | AdmissionReview UID | |
| 17 | Correlation ID | injected (B5) | ties to OPA decision log + downstream evidence |

- **R-B2-10 (MUST):** A Gatekeeper decision event missing ANY of the 17 fields MUST be treated
  as a defective event: the audit pipeline (C2) MUST flag it, and B2's webhook layer MUST NOT
  drop the request silently — it logs the gap (DT-16). Fields 7/8 require the D1 mapping layer;
  if JWT claims are unavailable, the field MUST be present with an explicit `null`+reason, never absent.
- **R-B2-11 (MUST):** `Correlation ID` (17) MUST be the same value carried into the OPA decision
  log (B1-R25) and all downstream evidence so a single admission is traceable end-to-end (DT-13, HL-03).
- **R-B2-12 (SHOULD):** Events SHOULD additionally carry: bundle revision, severity, enforcement
  mode at decision time, image digest(s), and external-data versions used.

---

## 7. Admission webhook configuration (normative)

- **R-B2-13 (MUST):** The ValidatingWebhookConfiguration MUST set a **short timeout** (≤ the
  Kubernetes admission deadline, default budget ≤ 2s for the governance decision) and MUST NOT
  attempt long-running work synchronously. (Underlies the §17B.4 hard constraint — §8.)
- **R-B2-14 (MUST):** `failurePolicy` MUST be `Fail` (fail-closed) for production runtime
  constraints **except** for an explicit allowlist of system namespaces (R-B2-4) where it is
  `Ignore`, to avoid the cluster-bricking failure (B1 AR-6). Dryrun/warn constraints SHOULD use
  `failurePolicy: Ignore`. This choice MUST be governance-recorded per environment.
- **R-B2-15 (MUST):** External-data calls (R image-signature verification, etc.) MUST have their
  own bounded timeout and a cache; an external-data timeout MUST NOT silently allow — it falls back
  to the constraint's configured `failurePolicy`. External-data version MUST be captured (DT-27).
- **R-B2-16 (SHOULD):** Webhook scope (`namespaceSelector`, `objectSelector`) SHOULD be narrowed
  so Gatekeeper is only invoked for resources in-scope of at least one Constraint (latency/cost).

---

## 8. Suspend-pending-approval at admission (the HARD constraint — normative)

Per §17B.4 + §17C.6: **Kubernetes admission webhooks have short request deadlines and CANNOT
hold a request open for a human approval.** Therefore:

- **R-B2-17 (MUST):** When a B1 decision yields `action == require_approval` /
  `suspend_pending_approval`, Gatekeeper MUST **deny the admission with an approval-required
  reason**, MUST NOT block waiting, and the platform MUST create/update a `PolicyApprovalRequest`
  CRD (owned by B4) capturing the request. The denied message MUST tell the user how to obtain
  approval and that a retry will succeed once approval state exists.
- **R-B2-18 (MUST):** The decision to deny-with-approval MUST be conveyed by the B1 `decision`
  object's `approval` block (subject, required approver type/value, correlation_id, expiry). The
  Gatekeeper template translates it into (a) a deny `violation` with the reason, and (b) a signal
  to the approval controller (B4) to create the CRD. The webhook itself does NOT create the CRD
  synchronously if that would exceed the deadline; it emits an event the controller acts on.
- **R-B2-19 (MUST):** On retry, the Constraint MUST consult approval state (via external-data or a
  cached `PolicyApprovalRequest` status) and allow iff an unexpired matching approval exists. Once
  approved, the resource admits; the approval consumption is audited. (DT-58, DT-59, HL-10.)
- **R-B2-20 (MUST):** `PolicyException` (B4) MUST be similarly consultable: an unexpired,
  in-scope exception flips a would-deny into allow-with-exception-attached (action `exception`),
  and the exception use is audited (HL-19, DT-03).

This is the load-bearing reconciliation of "real-time admission" with "long-running approvals":
**deny-with-approval-required + CRD + retry**, never a held webhook.

---

## 9. Failure modes

| # | Failure | Required behavior |
|---|---|---|
| F1 | Webhook timeout / Gatekeeper down | `failurePolicy` governs: Fail (deny) for prod runtime, Ignore for system ns (R-B2-14). Emit availability alert; C4 will detect any admissions that slipped during an Ignore window (§19) |
| F2 | External-data provider down/slow | Bounded timeout → fall back to failurePolicy; capture version; never silently allow (R-B2-15) |
| F3 | Constraint Rego drifts from bundle | Lifecycle gate fails (R-B2-3); drift detector flags (HL-09) |
| F4 | Missing audit field (e.g. JWT claim) | Field present as null+reason, event flagged, not dropped (R-B2-10, DT-16) |
| F5 | Gatekeeper temporarily disabled by admin (§19) | No admission/audit events emitted → C4 correlates Kubernetes audit logs vs missing Gatekeeper events → bypass alert (DT-30, HL-06) |
| F6 | Admission deadline exceeded by decision logic | Decision must be bounded (B1-R10); slow policies caught by latency tests (PLAN); never hold for approval (R-B2-17) |
| F7 | Pre-existing/managed Gatekeeper (cloud) | Tolerate; deploy governance Constraints alongside; namespace-scope to avoid conflict (D-B2-04) |

---

## 10. Security / authz notes

- Constraint/Template authoring is privileged; namespace-scoped authors (D2/§17A) may only create
  Constraints matching their own namespaces (admission RBAC + a B2 validating policy on the
  governance CRDs themselves — Gatekeeper guarding its own config).
- `failurePolicy` and `excludedNamespaces` are tamper targets — they MUST be governance-controlled
  and changes audited (an admin widening exclusions is exactly the §19 bypass vector).
- External-data provider URLs/credentials are privileged (D4).

---

## 11. Dependencies

| Depends on | For |
|---|---|
| B1 | signed bundles, canonical `decision`, conformance, decision logs, correlation_id |
| B4 | action taxonomy; `PolicyApprovalRequest`/`PolicyException` CRDs + controllers; engine selection (Gatekeeper vs Kyverno) |
| B5 | the full admission→decision→audit sequence (B2 supplies the Gatekeeper leg) |
| D1 | JWT subject/groups mapping for audit fields 7/8 |
| D2 | scoped authoring RBAC |
| A2 | lifecycle promotion gate (dryrun→warn→deny) |
| C2 | audit-event normalization sink |
| C4 | bypass/drift detection consuming (or noting absence of) Gatekeeper events |

---

## 12. Open questions (decided defaults)

| # | Question | Default | Rationale |
|---|---|---|---|
| OQ1 | Inline template Rego vs bundle-sourced? | **Bundle-sourced, generated, revision-annotated** | Keeps conformance + drift detection (R-B2-3) |
| OQ2 | failurePolicy Fail vs Ignore default? | **Fail for prod runtime; Ignore for system ns + dryrun/warn** | Security default without cluster-bricking (R-B2-14) |
| OQ3 | How is approval state read at admission? | **External-data provider over PolicyApprovalRequest status (cached)** | Bounded, no held webhook (R-B2-19) |
| OQ4 | Own or coexist with managed Gatekeeper? | **Coexist; namespace-scope governance constraints** | Hyperscalers ship Gatekeeper (D-B2-04) |
| OQ5 | Audit interval? | **Configurable; default tighter for prod (e.g. 60s) than dev** | Faster §19 detection where it matters |

---

## 13. Traceability

- **Spec:** §9 (9.1–9.3), §7.2, §17B.4 (suspend-pending-approval at admission), §17C.6 (CRDs), §18, §19.
- **Scenarios:** DT-14 (warn→deny switch), DT-15 (audit-mode scan), DT-16 (missing audit fields),
  DT-17 (reconcile periodic vs admission), DT-30 (bypass detection), DT-58/DT-59 (suspend/approval),
  HL-02 (rollout), HL-03 (2am admission incident), HL-06 (bypass), HL-09 (multi-cluster drift), HL-10 (break-glass).
- **Personas:** Marcus, Jess, Priya, Sam (namespace author).
