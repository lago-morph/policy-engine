# C1 — Privateer Integration — SPEC

**Component ID:** C1 · **Domain:** C — Evidence, Audit & Analytics
**Spec sources:** §11 (Privateer Integration: §11.1 Responsibilities, §11.2 Evidence Correlation), with inputs from §6 (Gemara governance hierarchy), §13 (C2 schema — consumed), §17E.1/§17E.3 (reports), §23 (evidence integrity), §20.1 (supply chain).
**Status:** SPEC (depends on C2 frozen schema v1.0).
**Scenarios exercised:** DT-22 (Privateer evaluation log), DT-24 (signed evidence export), DT-23 (SBOM attestation → control), HL-06/HL-20 (correlated evidence).

---

## 1. What Privateer is in this platform

Privateer is the **Gemara-native evaluation and evidence-correlation engine**. Where C2 normalizes raw enforcement events into a replay-capable substrate, **Privateer raises that substrate up to the governance layer**: it executes Gemara evaluations, correlates governance controls to the runtime/build-time evidence that demonstrates (or refutes) them, and produces **Gemara Evaluation Logs** — the per-control, per-period evidence rows an auditor samples (DT-22).

The differentiator (market research §5): existing GRC tools *scrape* evidence from systems after the fact and map it to frameworks. Privateer instead **correlates evidence that the platform's own enforcement decisions emitted** (via C2) to Gemara controls — governance-traceable evidence produced *by* enforcement, not reconstructed from logs. C1 is the component that closes the G1 loop (governance objective → control → evidence → verdict) into a queryable, exportable artifact.

### 1.1 Responsibilities (§11.1, normative)
Privateer SHALL:
- **N-C1-1** Execute **Gemara-native evaluations** — evaluate a Gemara control's satisfaction over a scope and period from correlated evidence.
- **N-C1-2** **Generate evaluation evidence** — produce a per-evaluation record tying a control to the underlying C2 events and supply-chain artifacts.
- **N-C1-3** **Correlate governance artifacts to runtime evidence** — join Gemara controls (A1) to C2 events, scanner output, and attestations.
- **N-C1-4** Produce **Gemara Evaluation Logs** — the queryable, per-control evaluation history (DT-22).
- **N-C1-5** **Integrate with supply-chain evidence** — SBOM attestations, signature verification (§20.1).

### 1.2 Evidence correlation scope (§11.2, normative)
Privateer SHALL correlate, per evaluation:
- Governance controls (from A1 Gemara store)
- OPA decisions (C2 `policy_engine=opa` events)
- Gatekeeper audit events (C2 `policy_engine=gatekeeper`)
- Conftest evaluations (C2 `policy_engine=conftest`, build-time)
- Runtime observations (scanner/mesh C2 events)
- SBOM attestations (in-toto)
- Signature verification (Sigstore/cosign — appears as C2 `external_data_refs`)

### 1.3 In/out of scope
- **In:** the Gemara Evaluation Log model; the control↔evidence correlation engine; the evaluation verdict model; the supply-chain evidence adapters (SBOM, signature); Gemara Evaluation Log query + export (delegating the integrity envelope to C2 §7.6).
- **Out:** the raw enforcement-event normalization (C2); the analytic *detections* over events (C3/C4); report *layout* (C5 — C1 supplies the Gemara Evaluation Log as one report source); the Gemara control model itself (A1 owns it; C1 consumes it).

---

## 2. Data model

### 2.1 Gemara Evaluation
A **Gemara Evaluation** is the atomic unit: "control X, evaluated over scope S and period P, with verdict V, supported by evidence E."

```
GemaraEvaluation {
  evaluation_id: string (UUIDv7)
  control_id: string            // FK → A1 Gemara control (e.g. SC-IMG-001)
  control_version: string       // the control definition version evaluated
  scope: { cluster?, namespace?, tenant?, region?, environment? }   // mirrors C2 scope
  period: { from: rfc3339, to: rfc3339 }
  verdict: enum                 // satisfied | not_satisfied | partially_satisfied | indeterminate
  verdict_reason: string
  evidence_refs: EvidenceRef[]  // the correlated supporting evidence
  coverage: {                   // how much of the period/scope had observable evidence
     expected_evaluations: int, observed: int, missing: int, coverage_pct: float
  }
  replay_completeness_rollup: { complete: int, best_effort: int, insufficient: int }
  produced_at: rfc3339
  produced_by: { component: "privateer", version: string }
  integrity: { content_hash, prev_hash, chain_seq, signature? }   // via C2 integrity primitive §7
}

EvidenceRef {
  kind: enum                    // c2_event | sbom_attestation | signature_verification | scanner_finding | conftest_record
  ref: string                   // C2 event_id, or attestation digest, or external ref
  correlation_id: string        // the C2 join key
  source_system: string
  decision?: string             // for c2_event kinds
  digest?: string               // for attestation/signature kinds
}
```

### 2.2 Verdict semantics (the honesty model, inherited from C2)
- **`satisfied`** — every required evidence type present, all decisions consistent with the control objective, and the evidence is `replay_completeness=complete` (or `best_effort` is disclosed). E.g. SC-IMG-001 satisfied = every production admission in scope has a paired signed-image decision and no bypass.
- **`not_satisfied`** — at least one piece of contrary evidence (a deny that shouldn't exist, a bypass, a violating workload found by scan).
- **`partially_satisfied`** — satisfied for part of the scope/period, gaps elsewhere (e.g. one namespace uncovered — DT-33).
- **`indeterminate`** — evidence is insufficient to render a verdict (e.g. the supporting C2 events are `insufficient`; a coverage gap with no reconstructable input). **N-C1-6: Privateer MUST NOT render `satisfied` over `insufficient` evidence; the honest verdict is `indeterminate`.** (Direct inheritance of C2's no-silent-promotion tenet.)

### 2.3 Gemara Evaluation Log
An append-only log of `GemaraEvaluation` records, chained via the C2 integrity primitive (§7.3). Queryable by `control_id`, scope, period, verdict (DT-22). This is the auditor's sampling frame (DT-22 step 2: "one row per evaluation with the §11.2 sources joined").

---

## 3. Correlation engine

### 3.1 Inputs
- A1 Gemara control definitions (control → required evidence types, objective).
- C2 events via the consumer API (§10): filtered by `control_id`, scope, period.
- C2 `correlation_id` to join multiple events of one flow.
- Supply-chain evidence: SBOM in-toto attestations and Sigstore/cosign verification results — the latter usually already present as a C2 `external_data_refs` entry (the Rego consumed it), the former pulled from the attestation store and joined by image digest / `resource_id`.

### 3.2 Correlation algorithm (per control, per period)
```
for each control C in scope:
  required = A1.required_evidence_types(C)            // e.g. {gatekeeper_decision, signature_verification}
  events   = C2.query(control_id=C, scope, period)
  groups   = group(events) by correlation_id
  for each group g:
     present_kinds = kinds_of(g) ∪ supplychain_join(g)   // attach SBOM/signature by digest/resource_id
     evidence_refs = build_refs(g, present_kinds)
     classify(g):
        - missing required kind            → contributes to not_satisfied / coverage.missing
        - contrary decision (deny present where allow expected, or bypass)  → not_satisfied
        - all required present & consistent & complete  → satisfied-contribution
        - supporting evidence insufficient → indeterminate-contribution
  rollup → GemaraEvaluation.verdict + coverage + replay_completeness_rollup
```

### 3.3 Bypass & coverage handling (delegation, not duplication)
- Bypass detection (a workload with no paired decision — §14.2/§19) is **C3/C4's** job. C1 **consumes** C3/C4 findings as `not_satisfied` evidence rather than re-deriving them. **N-C1-7: C1 does not re-implement §14.2 detections; it ingests them.** (Prevents the cross-component duplication the adversarial review flags.)
- Coverage gaps (DT-33) likewise feed `partially_satisfied`.

### 3.4 Supply-chain evidence integration (§20.1, DT-23)
- **SBOM attestation → control:** an in-toto SBOM attestation for an image is joined to the C2 admission event for the workload running that image (by image digest / `resource_id`), and tied to the supply-chain control (e.g. SC-SBOM-001). DT-23 is the canonical flow.
- **Signature verification:** the cosign verification result that the Rego consumed is recorded as an `EvidenceRef{kind: signature_verification}` and cross-checked against the `external_data_refs` digest in the C2 event so the evaluation reflects exactly what the engine saw.

---

## 4. Gemara Evaluation Log export (DT-24)

C1 assembles the **content** of an evidence package; the **integrity envelope is C2's primitive** (SPEC §7.6, and adversarial defect D10: one signing format platform-wide).

- **N-C1-8** Export assembles: `manifest.json` (controls, period, scope, per-source row counts, Privateer query hash, in-scope bundle versions), `evaluations/<control>.ndjson` (GemaraEvaluation rows), `correlations/` (linking artifacts by `correlation_id`), `attestations/` (SBOM in-toto), and the C2 audit-derived violation report (§17E.3) as supporting content.
- **N-C1-9** C1 calls the C2 export primitive to compute the Merkle root, write `merkle.json`, and sign `manifest.json` (ed25519, published key). The auditor verifies with the public key alone (DT-24 step 3, HL-18).
- **N-C1-10** Exports are scope-filtered by the caller's authorization (Auditor read-only scope, §17A.2/§17A.5) — Daniel cannot export outside his authorized controls/period (DT-24).

---

## 5. Failure modes
- **Evidence source unavailable** (C2 query times out): evaluation is `indeterminate` with a `source_unavailable` reason, never silently `satisfied`.
- **Supply-chain attestation missing** for an image: the supply-chain control contributes `not_satisfied`/`partially_satisfied`, with the missing attestation enumerated.
- **A1 control definition changed mid-period:** evaluations are pinned to `control_version`; a mid-period change splits the period at the version boundary (two evaluations) rather than blending.
- **Correlation explosion** (one `resource_id` across many correlation groups): bounded by scope+period query windows; large evaluations are materialized as C2 datasets (§8.5) for reuse (DT-22→DT-24 handoff).

## 6. Decisions
| ID | Decision | Rationale |
|---|---|---|
| D-C1-01 | C1 **consumes** C3/C4 detections; does not re-derive bypass/coverage | Avoids triple-implementation (adversarial D10/A11); single source of truth for detections. |
| D-C1-02 | Verdict over `insufficient` evidence = **`indeterminate`**, never `satisfied` | Inherits C2 honesty tenet (N-C1-6). |
| D-C1-03 | C1 calls **C2's integrity primitive** for export signing | One verifiable package format platform-wide (adversarial D10). |
| D-C1-04 | Evaluations pinned to `control_version`; period split on control change | Auditor must know which control definition produced a verdict. |
| D-C1-05 | Signature/SBOM evidence cross-checked against C2 `external_data_refs` digest | Evaluation reflects exactly what the engine consumed, not a later re-fetch. |

## 7. Open questions (with defaults)
- **OQ-1:** Evaluation cadence — continuous vs. on-demand? *Default:* continuous incremental rollup per control per scope, with on-demand re-evaluation for a named period (DT-22/DT-24 are on-demand reads of the continuous log).
- **OQ-2:** Does Privateer store its own evaluation log or project from C2? *Default:* its own append-only Gemara Evaluation Log (chained via C2 primitive), because evaluations are higher-level artifacts with their own retention and audit value.

## 8. Dependencies
- **Consumes:** C2 (frozen schema, query API §10, correlation view §10.3, integrity primitive §7.6, datasets §8.5); A1 (Gemara controls + required-evidence map); C3/C4 (detections as evidence); D2 (scope authz); supply-chain attestation store + Sigstore.
- **Consumed by:** C5 (Gemara Evaluation Log is a report source); auditors (DT-22/DT-24); HL-20 federated rollup.
- **Blocks on:** C2 schema freeze (M-FREEZE) and A1 control model.
