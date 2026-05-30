# C2 — ALT Architecture: OCSF-native schema · event-sourced log vs. relational · CloudEvents envelope

**Component:** C2 (keystone) · **Role:** genuinely different architecture for the same §12–13 requirement, with trade-off analysis vs. the primary SPEC.
**Author persona:** event-streaming / security-data-lake architect (skeptical of bespoke schemas).

The primary SPEC makes three foundational choices. This ALT challenges each independently, because they are separable decisions:

1. **Schema:** custom replay schema (authoritative) vs. **OCSF-native**.
2. **Storage:** append-only hash-chained log vs. **relational (RDBMS) store**.
3. **Envelope:** bespoke top-level fields vs. **CloudEvents envelope wrapping the payload**.

You can adopt any subset. The most coherent ALT bundle is **OCSF-profiled payload + event-sourced Kafka-style log + CloudEvents envelope**. Below, each axis is argued, then the bundle, then the verdict.

---

## A. Schema axis — OCSF-native instead of a custom schema

### A.1 The OCSF-native proposal
Model every audit event as an **OCSF event** (Open Cybersecurity Schema Framework — now the de-facto normalization standard backed by AWS Security Lake, Splunk, Cisco; market research §5). Use OCSF's *Detection Finding* (2004) and *Compliance Finding* (2003) classes for `policy.decision`/`scan.result`, *Authentication* (3002) for `auth.event`, and the *Resource* / *API Activity* families for `resource.change`. Carry the replay-critical fields (`request_object`, `external_data_refs`, `jwt_claims`, `replay_completeness`) inside OCSF's `unmapped` and `enrichments` extension points, and publish a **C2 OCSF profile** that standardizes those extensions.

### A.2 Why it is attractive
- **Zero new schema to teach.** Auditors, SIEMs (Splunk, Security Lake, Sentinel), and SOAR tools already parse OCSF. The platform's evidence flows into existing security data lakes with no adapter.
- **Ecosystem leverage.** OCSF mapping libraries, validators, and a governance process already exist; you inherit them.
- **Future-proofing against the market.** If OCSF wins as the universal event standard, a custom schema becomes an integration tax. Betting on OCSF is betting with the market.
- **One less thing to defend.** "We emit OCSF" is an easier external story than "we invented a schema."

### A.3 Why the primary SPEC rejected it (and the rebuttal is weak)
- **OCSF normalizes for SIEM detection, not for policy replay.** Its classes model "what happened" (a finding), not "the complete decision input needed to deterministically re-run the policy." `request_object` at full fidelity, versioned `external_data_refs`, and `replay_completeness` have **no first-class OCSF home** — they live in `unmapped`/`enrichments`, which are by design un-standardized and tool-ignored. The replay contract would therefore depend entirely on a *custom OCSF profile* — at which point you have a custom schema wearing an OCSF costume, with all the bespoke-ness and none of the "tools understand it for free" benefit, because no SIEM interprets your replay extensions.
- **`replay_completeness` honesty has no OCSF analog.** OCSF has confidence/severity, but not "this record cannot be authoritatively replayed." Shoehorning it loses the semantics that make the platform's differentiator (market §5) defensible.
- **The mapping is lossy and one-way anyway** — which the primary SPEC already concedes by providing C2→OCSF as an *export adapter* (SPEC §9). That captures ~90% of the ecosystem benefit (evidence flows to the SIEM) without making the lossy schema authoritative.

### A.4 Verdict on the schema axis
**Keep the primary SPEC's choice; adopt the ALT's export adapter (already in SPEC §9).** OCSF-native-as-authoritative trades away the one thing that is genuinely novel (replay fidelity) for ecosystem familiarity that the export adapter already delivers. **But** the ALT sharpens a real requirement: the C2→OCSF export should be a **published, validated profile**, not an afterthought, so the SIEM integration story is first-class. *Recommendation folded back into primary:* elevate the OCSF export profile to a named deliverable in C5/F1.

---

## B. Storage axis — relational store instead of append-only hash-chained log

### B.1 The relational proposal
Store events in a relational database (Postgres-class): an `audit_event` table, a `correlation` table, `external_data_ref` and `coverage_cell` tables, with foreign keys and indexes. Tamper-evidence via DB audit triggers, WAL archiving, and row-level signatures rather than an application-level hash chain.

### B.2 Why it is attractive
- **Query ergonomics.** C3 detections (joins across decisions, coverage matrices, drift comparisons) are natural SQL. The primary SPEC's append-only-log + secondary-index model reinvents querying that an RDBMS gives for free.
- **Operational familiarity.** Backups, replicas, point-in-time recovery, access control — all mature.
- **Coverage matrix (DT-33/DT-80) and cross-cluster drift (DT-32/HL-09) are set operations** that SQL expresses cleanly.

### B.3 Why it is dangerous for *this* use case
- **Mutable-by-default is the wrong default for audit evidence.** The whole point (HL-18, DT-24, §23) is tamper-*evidence*. An RDBMS where a privileged user can `UPDATE`/`DELETE` a row and a trigger can be disabled is exactly the threat model (insider deleting the embarrassing HL-06 bypass event). DB triggers and WAL are tamper-*resistant*, not tamper-*evident to an external auditor* who does not trust your DBA. The append-only hash chain + signed Merkle checkpoints give an **auditor who trusts only the published public key** a verification path; an RDBMS does not.
- **Independent verification is harder.** With the hash chain, an auditor recomputes hashes from event JSON (T-DET-1). With an RDBMS, "trust our audit triggers" is not independently verifiable.

### B.4 The synthesis the primary SPEC already implies — and the ALT improves
The primary SPEC's logical contract is **backend-agnostic** (SPEC §8.1, OQ-1). The right architecture is **both**: an append-only hash-chained log as the **system of record / integrity tier**, and a **relational (or columnar) projection as a query tier** materialized from the log (CQRS / read-model). This is the standard event-sourcing pattern:
- Writes go to the append-only log (integrity).
- A projector tails the log and maintains relational/columnar read models (the `coverage_cell` table, `correlation_members`, drift views) for C3/C4/C5 query ergonomics.
- The read models are **rebuildable from the log** and never authoritative — if they disagree with the log, the log wins and the projection is rebuilt.

**Verdict on the storage axis: adopt the ALT's read-model tier as a refinement, keep the log as system of record.** *Recommendation folded back into primary:* PLAN W4/W8 should explicitly build a projected relational/columnar read model for query-heavy consumers (C3 coverage matrices, C5 reports), tailed from the append-only log. This resolves OQ-1 better than the primary's "secondary indexes on the log" phrasing.

---

## C. Envelope axis — CloudEvents wrapper instead of bespoke top-level fields

### C.1 The CloudEvents proposal
Wrap every C2 event in a **CloudEvents 1.0 envelope** (`id`, `source`, `type`, `time`, `datacontenttype`, `dataschema`, `subject`, plus extension attributes), carrying the C2 payload in `data`. Map: CloudEvents `id`→`event_id`, `time`→`timestamp`, `type`→`event_type`, `source`→`producer`/`source.system`, `dataschema`→`schema_version`, `subject`→`resource_id`, and `correlation_id` as a CloudEvents extension attribute.

### C.2 Why it is attractive
- **Transport-neutral, broker-friendly.** CloudEvents is the CNCF standard; Knative, Argo Events, Kafka, NATS, and most eventing infra speak it natively. Routing, filtering, and dead-lettering audit events through standard eventing infra becomes free.
- **Clean envelope/payload separation.** The envelope carries routing/identity; the payload carries the domain schema. This is cleaner than the primary SPEC's flat top-level mixing of envelope fields (`event_id`, `producer`, `source`) with domain fields (`decision`, `external_data_refs`).
- **`dataschema` gives versioning a standard home** — exactly the forward-compat concern (N-C2-FWD).

### C.3 Caveats
- **Hashing/canonicalization must be defined over the `data` payload (or the whole envelope) carefully**, because CloudEvents has multiple wire formats (structured vs. binary) and intermediaries may rewrite envelope attributes. The hash chain must bind a canonical form that brokers cannot mutate — so the chain hashes the canonicalized **payload + a pinned subset of envelope attributes**, not the wire bytes.
- **Slight redundancy** (envelope `id` vs payload `event_id`); resolve by making the payload authoritative and the envelope a projection of it.

### C.4 Verdict on the envelope axis
**Adopt CloudEvents as the transport/wire envelope; keep the C2 schema as the `data` payload and as the canonicalization/hashing unit.** This is low-cost and high-value: it makes C2 events first-class citizens of CNCF eventing infra with no loss to the integrity model, *provided* the hash chain binds the canonical payload (not the mutable envelope). *Recommendation folded back into primary:* SPEC §4/§7 should state the wire envelope is CloudEvents 1.0 with `data` = the C2 schema, and `content_hash` is computed over the canonical `data` plus pinned envelope attributes (`id`, `time`, `type`, `source`).

---

## D. The coherent ALT bundle and its trade-off table

**ALT bundle = OCSF-profiled-export + event-sourced-log-with-relational-read-model + CloudEvents-envelope.**

| Axis | Primary SPEC | Pure ALT | Recommended synthesis |
|---|---|---|---|
| Schema authority | Custom replay schema | OCSF-native | **Custom authoritative + OCSF *published export profile*** (adopt ALT's rigor on the export) |
| Storage | Append-only hash-chained log (backend-agnostic) | Relational store | **Log = system of record; relational/columnar read-model projection for queries** (adopt ALT's read tier) |
| Envelope | Bespoke flat top-level | CloudEvents wrapper | **CloudEvents 1.0 envelope; `data` = C2 schema; hash binds canonical payload** (adopt ALT) |
| Tamper-evidence | Hash chain + Merkle + signed checkpoints | DB triggers + WAL | **Keep primary** (only it gives auditor-independent verification) |
| Replay fidelity | First-class (`replay_completeness`) | Lost in OCSF `unmapped` | **Keep primary** (the differentiator) |

## E. Overall verdict

The pure OCSF-native bundle **loses the platform's core differentiator** (replay fidelity, market §5) and should be rejected as the authoritative schema. But two of the three ALT axes — **CloudEvents envelope** and the **event-sourced-log-plus-relational-read-model** storage pattern — are strict improvements that the primary SPEC's backend-agnostic stance already invites. The third (OCSF) survives as a first-class *export profile*, not as the source of truth.

**Net recommendation:** fold three refinements back into the primary SPEC/PLAN:
1. **R-ALT-1:** wire envelope = CloudEvents 1.0; `data` = C2 schema; `content_hash` over canonical `data` + pinned envelope attrs.
2. **R-ALT-2:** storage = append-only hash-chained log (system of record) + projected relational/columnar read model (query tier), rebuildable from the log — resolves SPEC OQ-1.
3. **R-ALT-3:** C2→OCSF export is a **published, validated profile** (named C5/F1 deliverable), not an afterthought.

None of these change the **frozen field list** (SPEC §3.13) or the **completeness state machine** (SPEC §5) — they are envelope/storage/export concerns. So the cross-domain contract other components depend on is **unaffected** by adopting the ALT refinements, which is exactly why they are safe to adopt post-freeze.
