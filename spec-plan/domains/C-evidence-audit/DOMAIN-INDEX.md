# Domain C — Evidence, Audit & Analytics — DOMAIN INDEX

**Domain lead deliverable.** This domain turns enforcement decisions into governance-traceable, replay-capable, tamper-evident evidence, then detects, evaluates, and reports on it. It is the platform's **single most-differentiated domain** (market research §5: *no vendor emits governance-traceable evidence from enforcement itself; they scrape logs after the fact*).

**The keystone is C2.** Its frozen audit-event schema is the contract C1, C3, C4, C5, E1 (simulation) and F4 (AI extension) all depend on. C3/C4/C5 **block on C2 schema freeze (M-FREEZE)**.

---

## Component table

| ID | Component | Spec § | Dir | Status | Docs | Exercised by (scenarios) |
|---|---|---|---|---|---|---|
| **C1** | Privateer integration (Gemara evaluation + evidence correlation) | §11 | `components/C1-privateer/` | SPEC+PLAN+ADV | SPEC, PLAN, ADVERSARIAL-REVIEW | DT-22, DT-24, DT-23, HL-06, HL-20 |
| **C2** | Audit schema framework & standardized replay-capable event schema **(KEYSTONE)** | §12–13 | `components/C2-audit-schema/` | **SCHEMA FROZEN v1.0** + PLAN + ADV + ALT | SPEC, PLAN, ADVERSARIAL-REVIEW, ALT-ocsf-eventlog-cloudevents | DT-16, DT-25, DT-26, DT-27, DT-28, DT-42, HL-18 |
| **C3** | Compliance analytics engine (detections) | §14 | `components/C3-compliance-analytics/` | SPEC+PLAN+ADV | SPEC, PLAN, ADVERSARIAL-REVIEW | DT-30, DT-31, DT-32, DT-33, DT-80, DT-28, HL-09, HL-20 |
| **C4** | Retrospective audit detection (window sweep) | §19 | `components/C4-retrospective-detection/` | SPEC+PLAN+ADV | SPEC, PLAN, ADVERSARIAL-REVIEW | HL-06, HL-12, DT-30, DT-42, DT-46, DT-78, HL-18 |
| **C5** | Reporting (4 report types + signed export) | §17E | `components/C5-reporting/` | SPEC+PLAN+ADV | SPEC, PLAN, ADVERSARIAL-REVIEW | DT-34, DT-77, DT-78, DT-79, DT-80, HL-20, DT-24 |

---

## One-line purpose

- **C1 Privateer** — raises C2 events to the governance layer: executes Gemara evaluations, correlates controls↔evidence (OPA/Gatekeeper/Conftest/runtime/SBOM/signature), produces the Gemara Evaluation Log (the auditor's sampling frame).
- **C2 Audit schema** — the replay-capable, tamper-evident, append-only standardized audit event; the frozen cross-domain contract; correlation, completeness state machine, integrity primitive, consumer query API.
- **C3 Analytics** — continuous interval detectors over C2: bypass, inconsistent enforcement, policy/JWT drift, coverage gaps, correlation gaps, chain integrity. Emits findings.
- **C4 Retrospective** — on-demand sweep over a whole window: "did anything ever get bypassed?"; reconstructs inputs, drives replay (E1), produces the audit-derived violation population; handles the silent-regression (HL-12) differential.
- **C5 Reporting** — renders the four §17E report types (Real-Time Enforcement, Audit-Derived Violation, Simulation, Coverage-Gap & Drift) with fields/filters/scheduling and **signed export packages** (via C2's integrity primitive).

---

## Internal dependency graph (intra-domain)

```
                 ┌──────────────────────────── C2 (KEYSTONE, frozen schema) ───────────────────────────┐
                 │  events · correlation · completeness · integrity primitive · query API · datasets   │
                 └───┬─────────────────┬──────────────────┬─────────────────────┬─────────────────────┘
                     │                 │                  │                     │
      consumes ▼     │ consumes        │ consumes         │ consumes            │ consumes
            ┌────────┴───┐      ┌──────┴──────┐    ┌───────┴────────┐    ┌──────┴───────┐
            │    C3      │─────▶│     C4      │    │      C1        │    │     C5       │
            │ detections │ finds│ retrospect. │    │  Gemara eval   │    │  reporting   │
            └─────┬──────┘ trig │ (shares C3  │    │  (ingests C3/  │    │ (renders C1/ │
                  │ findings    │  detector   │    │   C4 findings) │    │  C3/C4; C2   │
                  │             │  library;   │    └───────┬────────┘    │  signs)      │
                  │             │  requests E1│            │             └──────┬───────┘
                  │             │  replay)    │            │                    │
                  └──────────────┴────────────┴────────────┴────────────────────┘
                                         all flow into C5 reports + auditor exports
```

**Cross-domain edges:** C2 ← B1 (Rego metadata → policy-dependency catalog), D1 (JWT normalization), D2 (storage authz). C3/C4 ← A1 (in-scope controls), A2/B1 (source-of-truth versions). C4 → **E1 (replay)**. C1 ← A1 (Gemara controls). C5 ← E1 (simulation results). F4 (AI) consumes C2 schema.

---

## Spec-section cross-reference

| Spec § | Owned/used by | Notes |
|---|---|---|
| §11.1–11.2 Privateer | C1 | Gemara eval + §11.2 correlation sources |
| §12 Audit framework | C2 | normalized schemas; "does not replace logging" |
| §13.1–13.5 Standardized event | C2 | design principle, fields, example, OCSF, **replay_completeness** |
| §9.3 Gatekeeper audit fields | C2 §3.11 | superset mapping (no field lost) |
| §10.3 Conftest evidence | C2 §3.10 | build-time normalization |
| §14.1–14.2 Analytics | C3 | the detections |
| §15.2 JWT claims | C2 §3.12, C3 D-JWT-DRIFT | claim capture + drift |
| §17.2/17.3 replay/audit-driven sim | C2 (completeness), C4 (reconstruction), E1 (replay) | shared boundary |
| §17E.1–17E.4 reporting | C5 | four report types |
| §19 Retrospective detection | C4 | window sweep + bypass |
| §23 Evidence integrity | C2 §7 (primitive), used by C1/C5 | one signing format |

---

## Status summary
All 5 components: **SPEC + PLAN + ADVERSARIAL-REVIEW present**; C2 additionally has **ALT**. C2 schema is **FROZEN v1.0** (SPEC §3.13). See `DOMAIN-SUMMARY.md` for the consolidated frozen field list and hardest decisions; `DOMAIN-ADVERSARIAL.md` for cross-component contradiction reconciliation.
