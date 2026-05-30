# C5 — Reporting — SPEC

**Component ID:** C5 · **Domain:** C — Evidence, Audit & Analytics
**Spec sources:** §17E (Reporting Requirements: §17E.1 categories, §17E.2 real-time enforcement, §17E.3 audit-derived violation, §17E.4 simulation), with inputs from §14 (C3 findings), §19 (C4 violations), §11 (C1 evaluation logs), §13 (C2 schema/integrity), §16.3 (console views), §17A.2/§17A.5 (scoped roles/storage authz), §23 (evidence integrity).
**Status:** SPEC (depends on C2 frozen schema, and on C1/C3/C4 outputs).
**Scenarios exercised:** DT-34 (weekly compliance report), DT-77 (real-time enforcement report), DT-78 (audit-derived violation report), DT-79 (simulation stakeholder report), DT-80 (coverage-gap report), HL-20 (federated reporting), DT-24 (signed export).

---

## 1. What C5 is

C5 is the **reporting and export layer**: it queries C2 events, C1 Gemara Evaluation Logs, C3 findings, and C4 violation populations, and renders them into the §17E report categories with defined fields, filters, scheduling, and **signed export packages**. C5 owns **layout, filtering, scheduling, and delivery**; it does **not** own detection (C3), violation reconstruction (C4), control verdicts (C1), or the integrity primitive (C2 §7.6 — C5 *calls* it). This division is the resolution of the cross-component "three signing formats" hazard (C2 adversarial D10): **C5 assembles content, C2 signs.**

### 1.1 The report categories (§17E.1)
The full §17E.1 list maps to four primary report *types* plus cross-cutting categories:

| §17E.1 category | Report type | Source |
|---|---|---|
| Real-time enforcement actions; Mutations; Approval requests; Suspended/pending | **R1 Real-Time Enforcement** (§17E.2) | C2 `policy.decision` events |
| Violations detected from audit logs | **R2 Audit-Derived Violation** (§17E.3) | C4 violation population |
| Simulation results; Newly allowed/blocked; False-positive analysis | **R3 Simulation** (§17E.4) | E1/C4 differential + simulation |
| Missing audit fields; Coverage gaps by control/namespace; Policy drift | **R4 Coverage-Gap & Drift** | C3 findings + C2 coverage feed |

---

## 2. The four report types (fields, filters, formats)

### 2.1 R1 — Real-Time Enforcement Report (§17E.2; DT-77)
**Purpose:** what the platform enforced, decision-by-decision, in a window.
**Source:** C2 `policy.decision` events (all engines: opa, gatekeeper, kyverno, conftest, application §17C).
**Filters:** `tenant`, `environment`, `cluster`, `namespace`, `control_id`, `policy_engine`, `decision`, `time window`. (DT-77: `tenant=payments, env=prod, last 30d`.)
**Required fields per row (§17E.2):** decision timestamp · actor (`subject.sub`) · resource (`resource_id`) · namespace (`scope.namespace`) · policy engine (`policy_engine`) · policy version (`policy_version` digest) · control ID (`control_id`) · decision (`decision`) · action performed (`action_performed`) · mutation diff (`mutation_diff`, populated iff `action_performed=mutate`) · approval webhook correlation (populated iff `decision ∈ {suspend_pending_approval, require_approval}`).
**Aggregation:** counts by `decision`, by `policy_engine` (DT-77: "all four engines contributed — no silent engine gap"), by `control_id`.
**Integrity note:** every row dereferences to its C2 `event_id`; the report is a *view* over signed C2 events (no reconstruction).

### 2.2 R2 — Audit-Derived Violation Report (§17E.3; DT-78, HL-06)
**Purpose:** violations discovered retrospectively (bypasses, missing coverage) — the C4 population.
**Source:** **C4** (this report renders C4's output; C5 does not reconstruct).
**Filters:** `control_id`, `window`, `confidence_level`, `cluster/namespace/tenant`, `source_audit_log`.
**Required fields per row (§17E.3, all 9):** violation_timestamp · discovery_timestamp · source_audit_log (dereferenceable, round-trips — DT-78 SC) · reconstructed_policy_input · policy_version (replay bundle) · confidence_level (high|medium|low) · missing_fields (list) · matched control_id · recommended_remediation.
**Honesty rules (inherited from C4):**
- **N-C5-1:** rows MUST visibly distinguish *recorded deny* (a real C2 event) from *inferred deny* (C4 reconstruction) — a reader must never conflate them (C4 adversarial D2).
- **N-C5-2:** `confidence=low` rows are shown but NOT auto-counted in violation totals without operator confirmation; unconfirmed-low rows appear as an aged backlog, never silently dropped (C4 D8, DT-78 SC).
- **N-C5-3:** a drill-in shows the reconstructed input side-by-side with the replay decision trace (DT-78 step 4).
**Re-execution:** the report exposes a "Re-execute" action that replays the reconstructed input via E1 from the viewer's session (DT-78 step 5; auditor independence — must tie out, divergence flagged per C4 D4).

### 2.3 R3 — Simulation Report (§17E.4; DT-79, HL-12)
**Purpose:** the result of a differential simulation / historical replay — newly allowed/blocked classification.
**Source:** E1 simulation + C4 retrospective differential (HL-12).
**Filters:** `policy_version_before`, `policy_version_after`, `audit_dataset`, `control_id`, `scope`.
**Required fields (§17E.4):** policy version before · policy version after · audit dataset used · events evaluated · newly blocked count · newly allowed count · unchanged allowed count · unchanged denied count · tagged intentional changes · untagged risky changes · false-positive candidates · false-negative candidates.
**Honesty rules:**
- **N-C5-4:** events with `replay_completeness=insufficient` are excluded from authoritative counts and disclosed separately (DT-46 step 4); they never silently inflate "unchanged" buckets.
- **N-C5-5:** "untagged risky changes" (newly-allowed not tagged intentional) are highlighted — this is the HL-12 silent-regression signal (a quota control that silently started allowing). DT-79 is the stakeholder-facing rollup.

### 2.4 R4 — Coverage-Gap & Drift Report (§17E.1: coverage gaps by control/namespace, policy drift, missing audit fields; DT-80, DT-33, DT-25)
**Purpose:** where enforcement is *absent* or *inconsistent* — the negative-space report.
**Source:** C3 findings (`coverage_gap`, `policy_drift`, `jwt_policy_drift`) + C2 coverage feed (§10.4) + C2 `insufficient`/missing-fields events.
**Sub-reports:**
- **Coverage gaps by namespace** and **by control** — the (namespace × control) matrix (DT-80) with cells classified `enforced | installed_no_events | not_installed | n/a` (+ `stale_inventory` subtype per C3). Each cell links to source-of-truth (constraint list, bindings, last decision).
- **Policy drift** — C3 `policy_drift` findings: expected vs observed `policy_version` per cluster (HL-09).
- **Missing audit fields** — C2 `insufficient` events + their `replay_completeness_reasons` (DT-25): the report that drove the DT-25 normalizer-fix loop.
**Required fields:** namespace/control cell classification · expected vs evaluated counts · coverage_pct · workloads_observed · drift expected/observed versions · missing-field reason codes.

---

## 3. Cross-cutting reporting capabilities

### 3.1 Scheduling (DT-34 weekly executive brief)
- **N-C5-6:** reports can be scheduled (cron), scoped, and delivered (email/distribution list). DT-34: "Weekly Executive Compliance Brief," `tenant in (...)`, rolling 7-day, `cron: 0 6 * * 1`, with selected §17E.1 categories, aggregations (counts per control, top-10 denying policies, exception issuance/expiry, open bypass alerts, week-over-week deltas), and signed export.

### 3.2 Scoping & authz (§17A.2/§17A.5)
- **N-C5-7:** every report query is scope-filtered by the caller's authorization subject (Compliance Analyst, Auditor, etc.). DT-77: results bounded to `tenant=payments` namespaces. An Auditor sees only authorized controls/period (DT-78). C5 enforces scope at the **query**, delegating to C2's scope-filtered API (§10) and D2's storage authz — not at the UI alone.

### 3.3 Federated rollup (HL-20)
- **N-C5-8:** multi-cluster/multi-cloud reports roll up across clusters/regions, **deduplicating on the cluster-scoped `correlation_id`** (per C2 adversarial D5 — never the bare UID). A single quarterly coverage-gap report across `aws-east-1/aws-west-2/gcp-us-central/onprem-dc1` by control, cluster, and region (HL-20).

### 3.4 Export formats & signed packages (DT-24, DT-46, DT-78, HL-18, §23)
- **N-C5-9:** reports export to: **PDF** (human/stakeholder — DT-79), **CSV/NDJSON** (data), and a **signed evidence package** (auditor — DT-24/DT-46/DT-78).
- **N-C5-10:** the signed package uses the **C2 integrity primitive** (SPEC §7.6): `manifest.json` (controls, period, scope, row counts, query hash, in-scope bundle versions), the report rows as NDJSON, `merkle.json`, and a detached signature over the manifest (ed25519, published key). **C5 assembles; C2 signs.** Verification requires only the published public key (HL-18). One format platform-wide (resolves C2 D10 / C1 A6).
- **N-C5-11:** the **OCSF export profile** (C2 §9 / ALT R-ALT-3): a report may additionally export its underlying C2 events as a published, validated OCSF profile for SIEM ingestion — the first-class export the ALT recommends.

---

## 4. Report integrity & honesty (the through-line)
Every C5 report inherits the domain's honesty discipline:
- **Never silently promote** low-fidelity evidence into authoritative counts (N-C5-2, N-C5-4).
- **Always distinguish recorded vs inferred** (N-C5-1).
- **Always disclose** `insufficient`/`best_effort`/`low-confidence` populations rather than hiding them (the §17E.1 "missing audit fields" category is itself a *feature*, not an embarrassment).
- **Always traceable** — every rendered datum dereferences to a signed C2 event or a cited C3/C4/C1 finding id (no orphan numbers).

## 5. Failure modes
- **Source component down** (C3/C4 unavailable): the report renders what it can and marks the missing section "unavailable — not zero" (distinguish "no findings" from "couldn't query findings").
- **Scope mismatch:** a query exceeding the caller's scope returns only authorized rows + a "results truncated by scope" notice (never silently complete-looking).
- **Scheduled report delivery failure:** retried + alerted; a missed scheduled report is itself logged (audit committee must know a report didn't run).
- **Large export:** materialized as a C2 dataset (§8.5) and delivered by reference/digest (DT-46 reuse).

## 6. Decisions
| ID | Decision | Rationale |
|---|---|---|
| D-C5-01 | C5 assembles content; **C2 signs** (one integrity primitive) | Resolves three-signing-formats hazard (C2 D10, C1 A6). |
| D-C5-02 | R2 renders C4 output; C5 does **not** reconstruct | Single reconstruction owner (C4); C5 is presentation. |
| D-C5-03 | Recorded-deny vs inferred-deny visibly distinguished (N-C5-1) | Evidence admissibility (C4 D2). |
| D-C5-04 | Low-confidence/insufficient disclosed, never auto-counted (N-C5-2/4) | Honesty tenet; DT-78/DT-46 SCs. |
| D-C5-05 | "No findings" ≠ "couldn't query" rendered distinctly | A blank report must not falsely imply compliance. |
| D-C5-06 | Federated dedup on cluster-scoped correlation_id (N-C5-8) | C2 adversarial D5; HL-20 correctness. |
| D-C5-07 | OCSF export profile as a first-class format (N-C5-11) | C2 ALT R-ALT-3; SIEM integration story. |

## 7. Open questions (defaults)
- **OQ-1:** PDF rendering engine / templating? *Default:* server-side templated PDF with embedded signed-manifest digest on the cover page; the PDF is a *rendering*, the signed package is the *evidence*.
- **OQ-2:** Report result caching vs always-live? *Default:* scheduled reports materialize an immutable dataset (reproducible later); ad-hoc reports run live but can be "pinned" to a dataset on export.

## 8. Dependencies
- **Consumes:** C2 (events, coverage feed, datasets, integrity primitive §7.6, OCSF export §9); C1 (Gemara Evaluation Logs as a source); C3 (findings: coverage/drift); C4 (violation population §17E.3, differential for §17E.4); E1 (simulation results §17E.4, re-execute action); D2 (scope authz §17A.5).
- **Consumed by:** Compliance Analysts (DT-34/77/80), Auditors (DT-78/24, HL-18), executives (DT-79), audit committees (HL-20).
- **Blocks on:** C2 schema freeze (M-FREEZE) + integrity primitive; C1/C3/C4 outputs for the respective report types.
