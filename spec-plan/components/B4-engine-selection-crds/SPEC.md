# B4 — Engine Selection, Action Taxonomy & CRDs — SPEC

**Component ID:** B4 · **Domain:** B · **Spec source:** §17C (with §17B, §9/B2, §8/B1 deps)
**Status:** DRAFT v1 · **Date:** 2026-05-30 · **Author persona:** cooperative engineering author
**High-value component:** has two ALT architecture trees (ALT-ocp-substrate.md, ALT-kyverno-first.md).

---

## 1. Scope

B4 is the **decision-making and extension layer** that ties the engines together. It owns three
things the spec calls out in §17C and §17B:

1. **Engine-selection rubric** (§17C.1–17C.2): a *real, evaluable* decision matrix that, given a
   policy's required outcome, tells you whether to use OPA-alone, Gatekeeper, Kyverno, a CI/CD
   integration, or a custom CRD controller.
2. **The 13-action taxonomy** (§17C.3) as the platform's authoritative action vocabulary, plus the
   PDP typology (§17C.4) and per-product PDP requirements (§17C.5).
3. **Custom CRDs** (§17C.6) that fill gaps existing engines can't address — with full OpenAPI
   schemas and controller reconcile loops: `PolicyApprovalRequest`, `PolicySimulationRun`,
   `PolicyActionLibrary`, `PolicyEvidenceSchema`, `PolicyException`, `PolicyRemediationAction`.

B4 is where the platform's most important *integration* decisions live, including the hard
constraint that **long-running approvals must not block admission webhooks** (§17B.4) — resolved
here via the `PolicyApprovalRequest` CRD + deny-with-approval-required pattern (B2-R17).

**In scope:** the selection rubric as scored criteria; the canonical action enum + semantics;
PDP typology + per-product definition requirements; CRD OpenAPI schemas, state machines, and
controller reconcile loops; the approval/exception/remediation flows.

**Out of scope:** Rego/bundles (B1); Gatekeeper specifics (B2); Conftest (B3); the realtime
sequence (B5); per-product PDP libraries content (E3/§17D); simulation engine internals (E1/§17).

---

## 2. Background (market shifts, §2/§3 market research)

- **OPA Control Plane (OCP)** is now an open CNCF substrate (post-Styra/Apple) covering bundle
  build/distribution/regression. **Kyverno** has matured (Nirmata AI assistant, Nov 2025) and is
  the YAML-native engine for mutate/generate/cleanup/image-verify. **Gatekeeper** is shipped by all
  three hyperscalers. **Red Hat ACS** offers `SecurityPolicy` CR as an alternative.
- These shifts directly inform B4's two ALT trees:
  - **ALT-ocp-substrate.md:** build B4's lifecycle/distribution on OCP vs. a parallel custom engine.
  - **ALT-kyverno-first.md:** make Kyverno the primary K8s engine (YAML-native, AI-assisted) with
    OPA as the cross-product brain, vs. the spec's OPA/Gatekeeper-first posture.
- **D-B4-01 (decided):** Keep OPA/Gemara as the **cross-product decision brain and common
  vocabulary** (the action taxonomy + control model), and treat Gatekeeper/Kyverno/Conftest/CRD-
  controllers as **interchangeable effectors** selected by the rubric. This is the spec's §17C.8
  posture and it's what makes the platform engine-portable. The ALTs stress-test it.

---

## 3. The engine-selection rubric (normative)

§17C.1/17C.2 give qualitative guidance; B4 turns it into a **scored rubric** so selection is
reproducible and auditable (a control's chosen engine is recorded and justifiable).

### 3.1 Decision inputs (the questions asked of every policy)

| Q# | Question | Drives toward |
|---|---|---|
| Q1 | Is the required outcome a pure *decision* (allow/deny/warn) or an *operational effect* (mutate/generate/cleanup)? | decision→OPA/GK; effect→Kyverno/controller |
| Q2 | Is the enforcement point Kubernetes admission, CI/CD, identity, scanner, app, or data/API? | maps to PDP type (§17C.4) |
| Q3 | Does the decision need cross-product/shared semantics or replay? | yes→OPA/Rego (B1) |
| Q4 | Does it need YAML-native authoring by K8s teams? | yes→Kyverno |
| Q5 | Does it need image-signature verification? | Kyverno or external-data→OPA |
| Q6 | Does it need to create/mutate/cleanup K8s resources? | Kyverno / custom controller |
| Q7 | Does it require long-running approval? | CRD + workflow webhook (never block admission) |
| Q8 | Does it require an action outside all engines (quarantine, remediation)? | custom controller + CRD |
| Q9 | Is identity (JWT claims) part of the decision? | OPA/Rego (consistent claim eval) |

### 3.2 Scored decision matrix (the §17C.1 table, made evaluable)

For a given policy, score each candidate engine 0–3 per capability it needs; pick the highest
total among engines that score ≥1 on every *required* capability (a hard zero on a required
capability disqualifies the engine).

| Capability (need) | OPA alone | Gatekeeper | Kyverno | CI/CD | Custom CRD ctrl |
|---|---|---|---|---|---|
| Complex decision logic | 3 | 3 | 1 | 2 | 1 |
| K8s admission deny/warn | 0* | 3 | 3 | 0 | 1 |
| K8s mutation | 0 | 2 | 3 | 0 | 2 |
| Generate companion resources | 0 | 0 | 3 | 0 | 2 |
| Cleanup/delete | 0 | 0 | 3 | 0 | 2 |
| Image signature verification | 1† | 2 | 3 | 1 | 2 |
| Policy reports | 1 | 2 | 3 | 1 | 1 |
| Long-running approval | 0 | 0 | 0 | 2‡ | 3‡ |
| Cross-product consistency | 3 | 1 | 1 | 1 | 1 |
| Identity-aware decisions | 3 | 2 | 1 | 1 | 1 |
| Retrospective replay | 3 | 1 | 1 | 1 | 1 |
| CI/CD / IaC build-time | 2(via Conftest) | 0 | 0 | 3 | 0 |

`*` OPA alone needs an admission integration (= Gatekeeper) to act at admission.
`†` via external-data verifier feeding OPA. `‡` approval is never "in-engine"; the score reflects
ability to *pause* (CI) or *orchestrate* (controller) — final approval state lives in a CRD.

- **R-B4-1 (MUST):** Every governance control MUST record its selected engine(s) and the rubric
  score/justification, so engine choice is auditable and revisitable (HL-14 onboarding a PDP).
- **R-B4-2 (MUST):** A control whose required outcome is a pure decision MUST default to OPA/Rego
  for the *decision* (cross-product consistency), even if the *effector* is Gatekeeper or Kyverno.
  I.e. **the decision and the effector are separable**: OPA decides, an engine effects. This keeps
  cross-product semantics central (D-B4-01).
- **R-B4-3 (MUST):** Kubernetes-native effects (mutate/generate/cleanup/image-verify) SHOULD use
  Kyverno; do NOT add Kyverno merely because a policy touches Kubernetes (§17C.1).
- **R-B4-4 (MUST):** Long-running approval MUST NOT be assigned to any synchronous admission engine
  (Gatekeeper/Kyverno admission); it MUST use the `PolicyApprovalRequest` CRD + webhook (§17B.4).

---

## 4. The 13-action taxonomy (normative — authoritative for the whole platform)

B4 owns the canonical action vocabulary (§17C.3). Every engine's result maps onto exactly one
primary `action`; B1's `decision.action` (B1-R9) draws from this enum.

| # | Action | Semantics | Allowed-at | Typical engines | Side-effect? |
|---|---|---|---|---|---|
| 1 | `allow` | Permit | all PDPs | OPA, GK, Kyverno, app, CI | no |
| 2 | `deny` | Block | all PDPs | OPA, GK, Kyverno, CI | no |
| 3 | `warn` | Permit + record warning | admission, app | GK, Kyverno, app | no |
| 4 | `mutate` | Modify request/resource | admission | Kyverno, GK, controller | yes (resource changed) |
| 5 | `generate` | Create companion resource | post-admission | Kyverno, controller | yes |
| 6 | `cleanup`/`delete` | Remove resource | controller loop | Kyverno, controller | yes |
| 7 | `quarantine` | Isolate workload | controller | custom controller, K8s automation | yes |
| 8 | `suspend` | Pause workflow/reconciliation | CI/CD, GitOps | CI, GitOps, workflow | yes (pauses pipeline) |
| 9 | `require_approval` | Trigger approval flow | admission(via deny+CRD), CI, GitOps | workflow webhook + CRD | yes (creates CRD) |
| 10 | `require_scan` | Trigger scanner | CI, controller | CI, scanner integration | yes (triggers scan) |
| 11 | `notify` | Emit event | any | webhook/SIEM/chatops | no (informational) |
| 12 | `annotate`/`label` | Add metadata | admission | Kyverno, GK mutation, controller | yes (metadata) |
| 13 | `exception` | Attach approved exception → allow | any | platform exception store + CRD | no (records waiver) |

- **R-B4-5 (MUST):** The `action` enum is **closed**: exactly these 13. New effects require a spec
  change, not ad-hoc actions, so downstream consumers (audit, analytics, console) have a fixed set.
- **R-B4-6 (MUST):** `require_approval` at a Kubernetes admission PDP MUST be realized as
  deny-with-approval-required + `PolicyApprovalRequest` CRD (B2-R17, §17B.4). The action is
  `require_approval`; the *admission disposition* is deny; these are not contradictory (the audit
  records both: action=require_approval, disposition=deny-pending-approval).
- **R-B4-7 (MUST):** Action **precedence** when multiple controls fire on one request (resolving
  B2-AR-7): `deny` > `require_approval` > `quarantine` > `mutate`/`generate`/`annotate` >
  `require_scan` > `warn` > `exception` > `allow`. A hard `deny` from any control wins; `exception`
  only applies to a *would-deny from the same control scope*, never overriding an unrelated deny.
- **R-B4-8 (MUST):** `exception` (13) MUST reference an unexpired, in-scope `PolicyException` CRD;
  using an exception is itself audited (HL-19 expiry, DT-03).

### 4.1 PDP typology (§17C.4) and per-product requirements (§17C.5)

- **R-B4-9 (MUST):** Each integrated product MUST declare a **PDP profile** answering §17C.5's 8
  required definitions: event taxonomy, enforcement location, audit source, replay schema, subject
  mapping, resource mapping, decision outcomes, missing-capability notes. (Feeds E3/§17D libraries.)
- **R-B4-10 (MUST):** Each PDP profile MUST classify the PDP per §17C.4 (Admission/Application/
  CI-CD/Identity/Scanner/Observability/Data-API/Approval) and state whether it supports real-time
  enforcement and retrospective replay. Replay-incapable PDPs MUST be flagged (DT-25).

---

## 5. Custom CRDs (normative — §17C.6)

Six CRDs, group `governance.example.io/v1alpha1`. Each has an OpenAPI schema + a controller
reconcile loop. The unifying pattern: **CRDs hold durable state for actions that engines cannot
perform synchronously, and controllers reconcile that state and call out via webhooks.**

### 5.1 `PolicyApprovalRequest` (the load-bearing one — §17B.4 / §17C.6)

```yaml
apiVersion: governance.example.io/v1alpha1
kind: PolicyApprovalRequest
spec:
  controlId: DEPLOY-APPROVAL-001            # required, → Gemara
  requestedBy: alice                        # subject.sub
  subject: { sub: user-123, groups: [team-payments], tenant: acme }
  resourceRef: { apiVersion: apps/v1, kind: Deployment, name: api, namespace: payments-prod }
  requestedAction: "deploy workload"
  requiredApproval: { type: role, value: production-release-approver }  # person|role|group|org|automated|external
  correlationId: uuid                       # ties to the denied admission (B2-R17) + decision log
  expiresAt: "2026-05-13T00:00:00Z"         # bounded; expiry → must re-request
status:
  phase: pending                            # pending|approved|rejected|expired|consumed
  decisions: [ { approver: bob, decision: approved, at: "...", note: "..." } ]
  webhookDelivery: { lastAttempt: "...", delivered: true }
  consumedBy: { admissionReviewUID: "...", at: "..." }   # set when a retry admits using this approval
```

**OpenAPI (abridged):** `spec.controlId` (string, required), `spec.requiredApproval.type`
(enum: person|role|group|org|automated|external, required), `spec.expiresAt` (date-time, required),
`status.phase` (enum, required, default pending).

**Reconcile loop:**
1. On create (pending): validate controlId exists (A1) + requiredApproval well-formed; emit
   `approval.requested` webhook (§17B.3 schema) to the external workflow system; set
   `status.webhookDelivery`. Do **not** approve automatically.
2. Watch for approval input (via webhook callback API or an `approved`/`rejected` subresource
   write by an authorized approver — RBAC-gated to the required role/group, D2). Record in
   `status.decisions`; set `phase=approved|rejected`.
3. On `expiresAt` passed while pending: set `phase=expired`; emit `approval.expired` (HL-19).
4. When an admission retry consumes an approved request (B2-R19): set `phase=consumed`, record
   `consumedBy`; emit audit. A `consumed` or `expired` request cannot be reused.

- **R-B4-11 (MUST):** Approval MUST be **bounded** (`expiresAt` required); no indefinite pending.
- **R-B4-12 (MUST):** Only an identity matching `requiredApproval` (resolved via D1/D2) may approve;
  the approval subresource MUST be RBAC-gated and the approver identity audited. **Self-approval
  (requestedBy == approver) MUST be rejected** unless the control explicitly allows it.
- **R-B4-13 (MUST):** Approval consumption MUST be **single-use and idempotent**: one approved
  request authorizes one admission (matched by correlationId + resourceRef + subject); a second
  distinct request cannot ride the same approval (resolves B2-AR-3 retry identity).
- **R-B4-14 (MUST):** The controller MUST be the durable hand-off for B2-AR-2: it MUST create the
  CRD from a denied-with-approval-required admission signal **reliably** — via watching a
  durable queue/event of approval-required denials, not a best-effort fire-and-forget. If creation
  fails, the user's deny message MUST instruct them how to create the request explicitly (fallback).

### 5.2 `PolicyException`

```yaml
spec:
  controlId: SC-IMG-001
  scope: { namespaces: [legacy-batch], resources: [{kind: Deployment, name: legacy-*}] }
  reason: "Vendor image not yet signable; tracked in JIRA-123"
  grantedBy: priya
  approvalRef: { kind: PolicyApprovalRequest, name: ... }   # exceptions themselves may need approval
  expiresAt: "2026-08-01T00:00:00Z"                          # MUST be bounded
status: { phase: active, usageCount: 7, lastUsedAt: "..." }  # active|expired|revoked
```

- **R-B4-15 (MUST):** Exceptions MUST be **bounded** (`expiresAt` required) and **scoped**; an
  unbounded or unscoped exception MUST be rejected. Expiry flips controls back to enforcing (HL-19).
- **R-B4-16 (MUST):** Exception use MUST be counted and audited (`usageCount`, `lastUsedAt`), so
  analytics (C3) can flag over-used or stale exceptions. Exception grant SHOULD itself require
  approval (`approvalRef`).
- **R-B4-17 (MUST):** Exceptions are consulted by engines at decision time (B2-R20) via external-data
  / cached status; an exception only converts a would-deny *within its scope+control* to `exception`
  (allow-with-waiver) — never a blanket allow (R-B4-7 precedence).

### 5.3 `PolicySimulationRun`

```yaml
spec:
  mode: differential        # differential|historical-replay|shadow|snapshot  (§17.2)
  candidateBundle: { digest: "sha256:..." }     # B1
  baselineBundle: { digest: "sha256:..." }
  inputSource: { type: decision-log-window, from: "...", to: "..." }   # or live-shadow / manifest-set
  scope: { namespaces: [payments-prod] }
status: { phase: running|complete|failed, diffSummary: {...}, reportRef: {...} }
```

- **R-B4-18 (MUST):** `PolicySimulationRun` MUST NOT mutate cluster state or block real traffic
  (it's evaluation-only); it drives the E1 simulation engine (which backs onto B1 RegressionTest/OCP).
- **R-B4-19 (SHOULD):** Differential mode MUST surface decision-flip diffs (candidate vs baseline)
  to feed the A2 promotion gate (DT-49, HL-17).

### 5.4 `PolicyActionLibrary`, `PolicyEvidenceSchema`, `PolicyRemediationAction`

- **R-B4-20 (MUST):** `PolicyActionLibrary` declares, per product, the available actions (subset of
  the 13) and how each maps to an engine effector (feeds §17D/E3). `PolicyEvidenceSchema` declares
  the replay schema per PDP (§17C.5 replay schema, feeds C2/C4). Both are **data CRDs** (no active
  reconcile beyond validation + publication to the bundle data / external-data).
- **R-B4-21 (MUST):** `PolicyRemediationAction` represents an action outside all engines (e.g.
  quarantine a workload, open a ticket, revoke a token). Its controller executes the remediation via
  webhook/integration and records evidence. It MUST be **idempotent** and **auditable**, and MUST NOT
  perform destructive actions (cleanup/quarantine) without an explicit policy + (optionally) approval
  reference. Default for destructive remediation: **dry-run/propose** unless explicitly armed.

### 5.5 Cross-cutting CRD requirements

- **R-B4-22 (MUST):** All six CRDs are `v1alpha1`; the platform MUST define a conversion/upgrade path
  before any becomes a durable contract (no breaking changes to stored objects without conversion).
- **R-B4-23 (MUST):** All controllers MUST emit platform audit events (C2) with correlation_id on
  every state transition, and MUST be **level-triggered/idempotent** (safe to re-reconcile).
- **R-B4-24 (MUST):** CRD writes MUST be RBAC-scoped (D2/§17A): namespace authors may create
  requests/exceptions only in their scope; approval/grant requires elevated roles; controllers run
  with least privilege.

---

## 6. Failure modes

| # | Failure | Required behavior |
|---|---|---|
| F1 | Approval webhook delivery fails | Retry with backoff; surface in `status.webhookDelivery`; user can still approve via subresource; never auto-approve on delivery failure |
| F2 | CRD creation from denied admission lost (B2-AR-2) | Durable queue/event drives creation (R-B4-14); deny message has manual-create fallback |
| F3 | Approval consumed twice (race) | Single-use idempotency via optimistic concurrency on `phase: consumed` (R-B4-13) |
| F4 | Exception expired but still cached at an engine | Engines consult freshness; expired exception MUST NOT convert a deny (R-B4-15/17); cache TTL bounded |
| F5 | Self-approval attempt | Rejected (R-B4-12) |
| F6 | Remediation controller acts destructively in error | Default dry-run/propose; destructive requires armed policy + audit (R-B4-21) |
| F7 | Multiple controls → action conflict | Deterministic precedence (R-B4-7) |
| F8 | Unknown/ad-hoc action emitted | Rejected; enum is closed (R-B4-5) |

---

## 7. Security / authz notes

- Approval and exception grant are the platform's highest-trust operations — they convert denies to
  allows. RBAC (D2), no self-approval (R-B4-12), bounded validity (R-B4-11/15), and full audit
  (R-B4-23) are mandatory. An attacker who can write a `PolicyException` bypasses governance — so
  exception writes MUST be tightly scoped and ideally themselves approval-gated (R-B4-16).
- Remediation actions can be destructive (quarantine/cleanup) — least privilege + dry-run default.
- The engine-selection record (R-B4-1) is itself governance data; changing a control's engine
  (e.g. from deny to advisory) is a governance change and MUST be audited.

---

## 8. Dependencies

| Depends on | For |
|---|---|
| B1 | action enum source-of-truth alignment; bundle digests for simulation; decision contract |
| B2 | deny-with-approval-required handshake; exception/approval consultation at admission |
| B3 | build-time action subset |
| A1 | controlId resolution |
| A2 | promotion gate consumes simulation diffs; engine-choice is lifecycle metadata |
| D1/D2 | subject/approver identity resolution; RBAC scoping of CRD writes |
| E1 | simulation engine driven by PolicySimulationRun |
| E3/§17D | PolicyActionLibrary/EvidenceSchema feed per-product PDP libraries |
| C2/C4 | audit + replay schema (PolicyEvidenceSchema), exception/approval analytics |

---

## 9. Open questions (decided defaults)

| # | Question | Default | Rationale |
|---|---|---|---|
| OQ1 | Decision and effector separable, or engine owns both? | **Separable: OPA decides, engine effects** | Cross-product consistency (R-B4-2, D-B4-01) |
| OQ2 | Closed or open action set? | **Closed 13** | Stable downstream contract (R-B4-5) |
| OQ3 | Approval consumption single-use? | **Yes, idempotent single-use** | Prevent replay (R-B4-13) |
| OQ4 | Exception default validity? | **Bounded + scoped, required** | No silent permanent waivers (R-B4-15) |
| OQ5 | Remediation default posture? | **Dry-run/propose unless armed** | Avoid destructive automation accidents (R-B4-21) |
| OQ6 | CRD creation from denial: best-effort or durable? | **Durable queue/event + manual fallback** | Resolves B2-AR-2 (R-B4-14) |
| OQ7 | Build on OCP or parallel custom engine? | **OCP substrate behind abstraction (see ALT-ocp-substrate)** | Reuse maturing CNCF primitives (D-B4-01) |
| OQ8 | Kyverno-first or OPA/Gatekeeper-first for K8s? | **OPA decision brain + engine effectors (see ALT-kyverno-first)** | Cross-product semantics central (D-B4-01) |

---

## 10. Traceability

- **Spec:** §17C (all), §17B (approval outcomes + suspend-pending-approval), §9 (Gatekeeper effector),
  §8 (decision source), §17D (PDP libraries), §18 (realtime).
- **Scenarios:** DT-58/59 (suspend/approval), HL-10 (break-glass approval), HL-19 (exception expiry),
  DT-03 (exception on control), HL-14 (PDP onboarding), DT-49 (differential sim), DT-25 (replay schema),
  HL-14 (new product/PDP), DT-09 (template when not generatable → CRD/Kyverno effector).
- **Personas:** Marcus, Sam, Priya, Workflow Integrator, Daniel.
