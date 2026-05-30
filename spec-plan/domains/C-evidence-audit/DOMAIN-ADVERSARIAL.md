# Domain C — Evidence, Audit & Analytics — DOMAIN ADVERSARIAL (reconciliation)

**Domain-lead deliverable.** This reconciles the five per-component adversarial reviews: it surfaces contradictions *between* C-domain components, deduplicates findings that recur across them, and produces a single prioritized domain-level defect list plus the decisions that resolve the cross-component conflicts.

---

## 1. Contradictions *between* components (the dangerous ones)

### X1 — The `partial` → `best_effort` rename ripples through every consumer and the scenarios themselves.
- **Where:** C2 D-C2-01 renames the middle completeness state; C2-ADV D2; C3/C4/C5 all switch on the value; **the authoritative spec (§13.3) and detailed scenarios (DT-30, DT-42) literally assert `=partial`.**
- **Conflict:** if any producer or test emits `partial` and any consumer checks `best_effort`, joins silently fail.
- **RESOLUTION (binding):** **mandatory ingest normalization** — no stored C2 event ever carries `partial`; it is rewritten to `best_effort` at the normalizer (C2 §3.7). The consumer API accepts `partial` as a *query* synonym only. A conformance test asserts "zero stored events with `partial`." Scenario success-criteria that say `=partial` are satisfied by `best_effort` (documented mapping). **One reconciliation point; enforced at ingest.**

### X2 — Multiple components produce "signed evidence packages."
- **Where:** C1 (DT-24), C5 (DT-46/78), C2 §7.6 — three signing paths.
- **Conflict:** three incompatible signed-package formats → an auditor cannot verify uniformly.
- **RESOLUTION (binding):** **C2 owns the integrity primitive** (Merkle + ed25519 sign + manifest format). **C1 and C5 own content assembly and call C2's primitive.** No component implements its own signing. (C2 D-C2-04, C1 D-C1-03, C5 D-C5-01.)

### X3 — "Bypass," "reconstruction," and "replay" have overlapping claimants.
- **Where:** C3 D-BYPASS, C4 three-view sweep, C1 §11.2 (Privateer "correlates bypass conditions"), E1 (replay).
- **Conflict:** if C3 and C4 sweep with different windows/logic, or C1 re-derives bypass, they produce *disagreeing* bypass sets — and the auditor sees the platform contradict itself.
- **RESOLUTION (binding):** **single ownership, shared library.** C3 = continuous interval detection (live alerting). C4 = on-demand window sweep + reconstruction, **sharing C3's detector library** (pinned, version-reported — C4-ADV D7). C1 **ingests** C3/C4 findings, never re-derives (C1 N-C1-7). **E1 is the only replay evaluator**; C3 and C4 *request* replay, never embed one (C3 N-C3-5, C4 D-C4-05). Verdicts cite finding ids so they cannot silently diverge (C1-ADV A5, C4-ADV A6).

### X4 — The completeness/confidence label is computed in three places and may disagree.
- **Where:** C2 live scorer; C4 reconstruction confidence; the replay-time re-score (E1).
- **Conflict:** the same event scored `complete` live but `insufficient` at replay; auditor re-execution diverging from stored verdict (C2-ADV A2, C4-ADV A4).
- **RESOLUTION (binding):** **conservative-label-wins** (`insufficient > best_effort > complete`); any live-vs-replay divergence is recorded as a finding (not hidden); the catalog version that produced each label is captured (C2-ADV D4). The "≥95% tie-out" in DT-78 is **re-framed**: divergence is a *defect requiring root-cause* (non-deterministic policy / time-dependent external data / digest mismatch), not an accepted tolerance (C4-ADV D4). Target divergence on `complete` events is ~0%.

### X5 — Historical replay can silently use *present-day* external data.
- **Where:** C1 §3.4 (re-fetch forbidden), C4 §2.4/confidence (re-resolve external data), C5 re-execute button.
- **Conflict:** a "faithful" historical replay that re-fetches today's signature status is not faithful — it evaluates against current truth, not decision-time truth.
- **RESOLUTION (binding):** **all historical replay uses the C2-retained value as-of-`timestamp`** (C2 §8.3 raw external-data retention), **never a live re-fetch.** Live-only availability caps confidence at `medium` + `external_data_drift_possible` flag. (C1-ADV A4/D4, C4-ADV A2/D1.) This is why C2 §8.3 mandates retaining raw external-data *responses*, not just versions.

---

## 2. Findings that recur across components (deduplicated)

These were independently raised in multiple reviews; they are domain-wide, not component-local:

| Recurring theme | Raised in | Domain resolution |
|---|---|---|
| **Stale/wrong policy-dependency catalog** corrupts the central `complete`/equivalence/coverage logic | C2-A1 (CRITICAL), C3-A2 (equivalence hashing) | Catalog version **pinned per `policy_version`** and bound into events; runtime read-set mismatch → downgrade `best_effort:catalog_stale`. The catalog is a *domain-shared* dependency; its correctness is a domain-level invariant, not a C2-local concern. |
| **Signing-key compromise** defeats all tamper-evidence | C2-A5, C3-A9 (D-CHAIN inherits it), C5-A6 (key trust) | External timestamp/transparency-log anchor in addition to platform-signed checkpoints (C2-ADV D7); out-of-band key distribution named in exports (C5-ADV D6); D-CHAIN verifies the external anchor too. |
| **"No event / no finding / green cell" conflated with "compliant"** | C3-A1 (audit source disabled), C4-A3 (ephemeral bypass), C5-A2 (green cells) | Three independent observation views (decision/audit/inventory) in C4; typed empty-cell states in C5; "unavailable ≠ zero." Honestly scope §19.1 to *detectable* bypasses; document the ephemeral-bypass residual blind spot. |
| **Aggregation/confirmation hides low-fidelity or implicating evidence** | C1-A7 (`indeterminate` rate), C3-A8 (`suppressed` findings), C4-A9 (unconfirmed low-conf), C5-A1 (buried denominators) | All state changes (suppress/confirm/dismiss) are **chained audit events** with actor+reason; every rollup carries denominator evidence-quality; low-confidence/insufficient populations shown adjacent, never dropped. |
| **Version skew across components** silently drops fields/diverges results | C3-A7→shared lib, C4-A7, C5-A8 | Components pin and *report* the schema/detector-library versions they used; C5 refuses-or-flags on source schema mismatch (in a *report*, a dropped field is a lie — opposite of the API forward-compat rule). |

---

## 3. The single domain-level defect list (prioritized)

| # | Defect | Severity | Components | Resolution owner |
|---|---|---|---|---|
| **DC-1** | Stale policy-dependency catalog → confidently-wrong `complete` + wrong equivalence/coverage | **CRITICAL** | C2, C3 | C2 §4.2: pin catalog per policy_version; runtime mismatch → `best_effort:catalog_stale` |
| **DC-2** | `partial`→`best_effort` rename breaks joins/scenarios | **HIGH** | C2→all | C2 §3.7: mandatory ingest normalization + "no stored partial" conformance test (X1) |
| **DC-3** | Historical replay uses present-day external data → unfaithful | **HIGH** | C1, C4, C5 | as-of-timestamp retained value only; live → cap medium + drift flag (X5) |
| **DC-4** | Live-vs-replay label divergence; "95% tie-out" hides non-determinism | **HIGH** | C2, C4, E1 | conservative-label-wins; divergence = defect needing root-cause (X4) |
| **DC-5** | Headline bypass detection blind when audit source itself disabled / ephemeral | **HIGH** | C3, C4 | three-view reconciliation (decision/audit/inventory); document residual ephemeral blind spot |
| **DC-6** | D-DRIFT modal-observed fallback can hide a fleet-wide regression | **HIGH** | C3 | never guess expected version; `sot_unavailable` finding instead |
| **DC-7** | `complete` blob GC'd → label/reality divergence over time | **HIGH** | C2 | retention-lock blobs to event retention OR re-verify+downgrade at replay |
| **DC-8** | Three signed-package formats | **MEDIUM** | C1, C2, C5 | C2 owns primitive; C1/C5 assemble content (X2) |
| **DC-9** | Detection/reconstruction/replay ownership overlap & divergence | **MEDIUM** | C1, C3, C4, E1 | shared detector library; single E1 evaluator; verdicts cite finding ids (X3) |
| **DC-10** | Signing-key compromise unmodeled; key trust assumed | **HIGH** | C2, C3, C5 | external timestamp/transparency anchor + out-of-band key distribution |
| **DC-11** | Correlation-id collision/spoof in federated store | **HIGH** | C2, C3, C5 | cluster-scope the id; dedup on scoped id (HL-20) |
| **DC-12** | Aggregates bury evidence quality; green cells imply safety | **HIGH** | C5 | denominator evidence-quality on every rollup; typed empty cells |
| **DC-13** | Suppress/confirm/dismiss + report scope are unaudited editorial acts | **MEDIUM** | C3, C4, C5 | chained audit events for state changes; signed manifest embeds exact scope |
| **DC-14** | Version skew silently drops report fields | **MEDIUM** | C3, C4, C5 | pin+report versions; C5 flags source schema mismatch |
| **DC-15** | C1 `satisfied`/`partial` coverage threshold unspecified | **HIGH** | C1 | state threshold; show coverage_pct + evidence-quality on every satisfied verdict |

## 4. Top-5 domain risks (the ones that sink the platform's credibility)

1. **DC-1 (catalog staleness).** The entire honesty edifice (`complete`, equivalence, coverage) rests on the policy-dependency catalog being current. A single stale catalog produces *confidently wrong* evidence — the worst outcome, because `complete` is trusted and unscrutinized. Fix is non-negotiable before any consumer trusts `complete`.
2. **DC-4 + DC-3 (replay faithfulness).** A deterministic, decision-time-faithful replay is the platform's auditor-trust anchor. If it disagrees with itself (DC-4) or uses today's data (DC-3), the differentiator collapses *in front of the auditor* — the worst possible moment.
3. **DC-5 (bypass blind spots).** The signature scenario (HL-06, "did it ever get bypassed?") has structural holes (disabled audit source, ephemeral workloads). Overclaiming detection coverage is a credibility bomb; honest scoping + the three-view defense is mandatory.
4. **DC-12 (reporting honesty).** All upstream honesty can be aggregated away at C5. A compliance % without an evidence-quality % is misleading by construction; a green cell during an outage reads as compliance.
5. **DC-10/DC-11 (integrity foundations).** Tamper-evidence and federated correctness rest on key trust and correlation-id uniqueness; both have unmodeled failure modes (key compromise, UID collision/spoof) that an adversary will target precisely because they are foundational.

## 5. Net assessment
The five SPECs are internally coherent *because the cross-component boundaries were authored in one mind* (single domain lead), which is why the ownership splits (X2/X3) and the honesty discipline (X4) are consistent rather than negotiated-after-the-fact. The frozen C2 schema is safe for other domains to depend on now. **The dominant residual risk is not architectural but operational-honesty:** the platform's value is entirely in *not lying* about evidence fidelity, and the defect list above is overwhelmingly about the dozen ways a faithful-looking number can hide an unfaithful reality. Every fix is in service of one property: **a number this platform reports must be traceable to evidence whose fidelity is disclosed.**
