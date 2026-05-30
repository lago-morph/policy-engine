# A1 — Governance Model & Gemara Hierarchy — ADVERSARIAL REVIEW

**Persona:** hostile principal engineer + skeptical lead auditor. This design is guilty until proven innocent.
**Target:** `SPEC.md` (and `PLAN.md`) for A1. **Posture:** find what breaks in production and what an auditor
would reject. Every finding has a severity and a demanded remedy. Defect list is prioritized at the end.

---

## 1. The thesis I'm attacking

A1 claims to be "the single system of record for governance intent" and that `control_id` is "the canonical
foreign key across the entire platform." Fine — then A1 is a **single point of correctness** for the whole
product. If A1 is wrong, everything downstream is wrong *and* believes it is right (it has a traceable
`control_id`, so it looks audited). That is the most dangerous possible failure shape: **confident,
traceable wrongness**. The rest of this review pressure-tests that.

---

## 2. Findings

### A1-DEF-01 — [CRITICAL] §13.3 "superset" validation is a paper guarantee; emission is never verified
SPEC D4/A1-MUST-010 makes A1 reject an EvidenceRequirement that omits a §13.3 core field. Good. But A1
**declares** required fields; it never **observes** whether the running policy actually *emits* them. A control
can pass A1 validation (requirement lists `correlation_id`) while the Rego/Gatekeeper constraint emits events
without it (DT-16, DT-28 are literally "missing audit fields" / "missing correlation_id" scenarios). A1 gives
the auditor a green checkmark that is unverified. **Auditor verdict: the EvidenceRequirement is an
*assertion of intent*, not *evidence of operation* — exactly the gap SOC 2 Type II exists to catch.**
**Remedy:** A1 MUST own (or C2 MUST own and A1 MUST surface) a *conformance signal*: a closed loop that
samples actual emitted events per control and asserts the required fields are present, flipping the control to
`evidence_drift` (F5 only covers schema-version bumps, not field absence in practice). Without this, A1's
central promise is theater.

### A1-DEF-02 — [CRITICAL] Control-plane/data-plane split (D9) silently breaks exception correctness under partition
D9 says enforcement engines cache the ExceptionRequirement and "continue enforcing if A1 is down." But an
*exception* is a **relaxation**. If A1 is down and Priya has just tightened an ExceptionRequirement (e.g.
shrunk `max_duration_days` 90→30, DT-03), the cached projection still authorizes 90-day exceptions. The
"fail-safe" design fails *open* for exceptions while failing closed for controls. The SPEC asserts
"A1 unavailability degrades authoring, never enforcement" — false: it degrades *exception governance*, which
is enforcement. **Remedy:** projection entries for ExceptionRequirements MUST carry a hard TTL and a
`max_staleness` after which the validator treats unknown/stale exception rules as **most-restrictive**
(deny new exceptions), not last-known. A1-MUST-050 says "detect staleness" but never says what to *do* about it.

### A1-DEF-03 — [HIGH] The two-lifecycle split (D7) has no defined reconciliation; drift is inevitable
D7 cleanly separates governance status (A1) from enforcement mode (A2). But a control can be A1-`active`
with A2-`dry-run` *forever*, or A1-`deprecated` while A2 is still `enforce` (the wind-down is "event-driven"
but events get lost). There is **no invariant** stating which combinations are legal/illegal and **no
reconciler** that detects a control that's been `active` for 6 months but never left `dry-run` (i.e. an
intent that is governed but not actually enforced — the auditor's nightmare: "the control exists" vs "the
control operates"). **Remedy:** define the legal cross-product matrix (e.g. `retired`+`enforce` is ILLEGAL,
`deprecated`+`enforce` is `winding_down` and time-boxed) and a reconciliation job that raises findings.
This is the seam between A1 and A2 and nobody owns it.

### A1-DEF-04 — [HIGH] `control_id` as immutable global key is correct but the org reality is renames/merges
D3 forbids reuse and rename of `control_id`. Realistic: frameworks get re-baselined; two controls merge;
a typo'd ID (`SC-IMG-01` vs `SC-IMG-001`) ships and is now load-bearing across Rego/audit/exceptions.
SPEC offers `superseded_by` but that's for deprecation, not for "this ID was a mistake." There is **no alias
mechanism**. In production someone WILL need `SC-IMG-001` to also answer to `SC-IMG-1`. **Remedy:** add an
`aliases[]` field + an alias-resolution rule in the API and lineage, so historical audit events under the old
ID remain joinable without reusing or mutating the canonical ID.

### A1-DEF-05 — [HIGH] Gemara round-trip (D8) lossiness risk; OpenSSF Gemara schema is a moving target
D8 promises byte-stable `yaml↔entity↔yaml` round-trip and stuffs divergence into `x-platform-*`. Two problems:
(1) byte-stability across a *Gemara schema version bump* (OQ-6 admits N/N-1 support) is hand-waved — re-serializing
an N-1 doc through an N serializer will not be byte-stable, which **breaks the deterministic signed export
(A1-MUST-032)** an auditor relies on. (2) "Where OpenSSF Gemara and this model diverge, store `x-platform-*`"
means A1 is **silently extending an external standard** — a Gemara-conformant consumer will ignore those blocks
and get an *incomplete* governance picture (e.g. missing the ExceptionRequirement if Gemara doesn't model it).
**Remedy:** pin the exact OpenSSF Gemara schema version supported, declare conformance level explicitly, and
make signed exports pin the schema version in the signed payload so verification is version-scoped.

### A1-DEF-06 — [HIGH] Coverage "gap" computation is trusted but `coverage: partial` is unfalsifiable
DT-02/§2.11: a CoverageLink is `full` or `partial` with free-text rationale, authored by Priya. The §17E
coverage badge (`covered:3, partial:1, gaps:2`) is then presented to auditors as evidence of framework
coverage. But `full` is a **self-assessment with no machine check** that the linked controls' enforcement
requirements actually satisfy the framework text. An auditor will (correctly) treat "Priya said full" as
*management assertion*, not evidence. The SPEC even concedes (HL-07 notes) "the platform stores it but does
not certify it" — yet the coverage *badge* visually certifies it. **Remedy:** the badge MUST be labeled as
management assertion; SHOULD link each `full` claim to at least one *operating-effectiveness* signal
(EvaluationRequirement producing non-empty evidence) before it can render green. A `full` link to a control
that is `dry-run`-only (never actually enforcing) is a lie the UI currently tells.

### A1-DEF-07 — [MEDIUM] Deprecation `retire` guard depends on a downstream inventory A1 doesn't control
A1-MUST-010/§4: `deprecated→retired` requires "B-layer reports zero active enforcement points." A1 *asks*
B2/B3/B4 for this inventory (`GET .../enforcement-points`). But that inventory is only as good as the
B-layer's self-report. A constraint applied out-of-band (kubectl apply, not via GitOps) referencing the
control would be invisible, and A1 would happily retire a control that is still actively denying admissions —
producing denials with a `control_id` that no longer resolves (the inverse of F9 orphan-implementation, and
arguably worse). **Remedy:** retire MUST also require a *negative* check against recent audit events (zero
decisions emitted for the control in the last N hours, which DT-04 step 6 actually does via the §17E report)
— make that audit-derived check a hard guard, not just a reporting step.

### A1-DEF-08 — [MEDIUM] No concurrency/authority model for the framework cross-reference graph
CoverageLinks are bidirectional (§2.11) and editable by Compliance Analysts. Two analysts mapping overlapping
frameworks (SOC 2 and ISO 27001) to the same control can produce contradictory coverage states with no merge
discipline beyond ETag (F7), which is per-*control*, not per-*link-graph*. The coverage badge aggregate then
flickers. **Remedy:** define ownership of a CoverageLink (the framework owner, not the control owner) and make
the aggregate computation read-consistent (snapshot isolation) so reports don't show torn states.

### A1-DEF-09 — [MEDIUM] "Append-only revisions" + "deterministic export" + edits-return-to-draft interact badly
§4 allows `in_review→draft` (edit) and §6 mandates append-only revisions. So a control under active review can
accumulate many draft revisions. Meanwhile §6 mandates a *deterministic* export. Which revision exports?
The head? The last `active` one? An auditor pulling evidence "as of date X" needs the revision that was
*active at X*, but a control that went `active → deprecated → (re-activated within grace, D7)` has an ambiguous
"active at X." **Remedy:** export and `GET ...?at=` MUST resolve to "the revision whose `[valid_from,valid_to)`
active interval contains X," and the SPEC must state that a control can have *multiple disjoint active intervals*
(re-activation creates a new interval). This is implied but never specified, and it's exactly where audit
queries get wrong answers.

### A1-DEF-10 — [MEDIUM] Multi-tenant isolation deferred to "row-level scoping" (OQ-4) is under-specified for an auditor
OQ-4 puts all tenants' governance in one store with §17A.5 row-level scoping. For an *external* auditor (Daniel)
scoped to one client/tenant, a row-level-security bug leaks another tenant's controls — and governance intent
(control statements, rationale) is often *itself* sensitive (it reveals the org's risk model). The SPEC treats
this as a scale decision; it's actually a **confidentiality boundary** decision. **Remedy:** the SPEC must
state the threat model for cross-tenant governance leakage and require negative tests (DT-54 cross-tenant admin
audit exists — wire it to A1), or isolate at the schema/database level for external-auditor tenants.

### A1-DEF-11 — [LOW] `required_jwt_claims` on the control is advisory and unenforced at A1
A1 stores `required_jwt_claims` (SHOULD) but nothing checks the claims actually exist in any Keycloak realm
(D1). A control can require `compliance_scope` that no IdP issues (DT-37 decommissions a claim — does A1 notice
controls still requiring it?). **Remedy:** SHOULD reconcile `required_jwt_claims` against the D1 claim catalogue
and flag controls requiring decommissioned/absent claims.

### A1-DEF-12 — [LOW] Lineage edges are append-only and temporal but unbounded — no compaction story
D6 stores every edge forever (decision→control edges in particular could be enormous if every decision creates
a lineage edge). SPEC §26.1 caps telemetry scale for POC, but the SPEC's lineage example includes
`decision:<correlation_id>` nodes — that's per-decision lineage, which does not stay POC-sized. **Remedy:**
clarify that per-*decision* lineage lives in the audit store (C2), and A1's LineageEdge is limited to
*governance-structural* nodes (control/rego/constraint/framework/bundle), not per-decision edges.

---

## 3. Cross-component contradictions surfaced

- **vs C2:** A1 vendors a *copy* of §13.3 (`AUDIT_CORE_FIELDS@v1`) to stay off the critical path (PLAN §2).
  If C2's authoritative list diverges from A1's pinned copy, A1 will validate against a stale floor and pass
  controls C2 would reject. The two must reconcile on a single published artifact, not a vendored copy.
- **vs A2:** the enforcement-mode lifecycle (dry-run→warn→enforce) lives in A2, but A1 owns the EnforcementRequirement's
  `mode_intent` (target mode). If A1 changes `mode_intent` from `deny` to `warn` while A2 has already promoted to
  `enforce`, who wins? Undefined (see A1-DEF-03).
- **vs B-layer:** A1's `current_policy_version` is described as "mirror from A2/B1" — a mirrored field that A1
  doesn't author but exposes on the control and seals at retire (A1-MUST-012). If the mirror lags, the sealed
  hash is wrong. The authority for `current_policy_version` must be exactly one component.
- **vs D3/§17C.6:** A1 publishes ExceptionRequirement; D3 enforces it. But A1-DEF-02 shows the failure mode
  when A1 is partitioned. The contract must specify staleness behavior on *both* sides.

---

## 4. "Will not survive contact with production because…"

1. …the central guarantee (every decision traces to a verified, fully-evidenced control) is **declared but not
   measured** (DEF-01). Auditors will downgrade it to a management assertion on first inspection.
2. …the fail-open-for-exceptions behavior under A1 partition (DEF-02) is a latent security finding waiting for
   the first network blip during an exception tightening.
3. …the A1/A2 lifecycle seam (DEF-03) has no owner and no reconciler; "governed but never enforced" controls
   will accumulate and nobody will notice until an incident or an audit.
4. …`control_id` immutability with no alias path (DEF-04) will force an emergency schema hack the first time a
   typo'd-but-load-bearing ID needs to be corrected.

---

## 5. Prioritized defect list

| Rank | ID | Sev | One-line demand |
|---|---|---|---|
| 1 | A1-DEF-01 | CRITICAL | Close the loop: verify required fields are actually *emitted*, not just *declared*. |
| 2 | A1-DEF-02 | CRITICAL | Exception projection must fail *most-restrictive* on staleness, not last-known. |
| 3 | A1-DEF-03 | HIGH | Define the legal A1-status × A2-mode matrix and a reconciler for "governed-not-enforced." |
| 4 | A1-DEF-05 | HIGH | Pin Gemara schema version; scope signed export + verification to it. |
| 5 | A1-DEF-06 | HIGH | Coverage badge = management assertion unless backed by operating-effectiveness signal. |
| 6 | A1-DEF-04 | HIGH | Add `control_id` aliases for renames/typo-corrections/merges. |
| 7 | A1-DEF-07 | MEDIUM | Retire guard must include audit-derived "zero recent decisions," not just B-layer self-report. |
| 8 | A1-DEF-09 | MEDIUM | Specify multi-interval "active at X" semantics for as-of-date audit queries. |
| 9 | A1-DEF-10 | MEDIUM | State cross-tenant governance-confidentiality threat model; isolate external-auditor tenants. |
| 10 | A1-DEF-08 | MEDIUM | Ownership + read-consistency for the CoverageLink graph. |
| 11 | A1-DEF-11 | LOW | Reconcile `required_jwt_claims` against the live D1 claim catalogue. |
| 12 | A1-DEF-12 | LOW | Keep per-decision lineage in C2; A1 lineage is governance-structural only. |

**Bottom line:** the entity model and lifecycle are solid and buildable. The two things that will get A1 *failed
in an audit* are DEF-01 (declared-not-verified evidence) and DEF-06 (self-asserted coverage rendered as
certified). The two things that will get A1 *paged in production* are DEF-02 (fail-open exceptions) and DEF-03
(unreconciled lifecycle seam). Fix those four before anything else.
