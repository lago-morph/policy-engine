# Domain C — Evidence, Audit & Analytics — DOMAIN SUMMARY

## 1. The domain in one paragraph

Domain C is the platform's evidence spine and its primary differentiator. Enforcement decisions (Domain B) are normalized by **C2** into a replay-capable, tamper-evident, append-only audit event whose schema is the **frozen cross-domain contract**. **C3** runs continuous detectors over that stream (bypass, drift, coverage, correlation, integrity). **C4** runs on-demand retrospective sweeps over whole windows ("did anything *ever* get bypassed?"), reconstructing inputs and driving replay. **C1 (Privateer)** raises events to the governance layer — Gemara evaluations correlating controls to evidence, producing the auditor's sampling frame. **C5** renders the four §17E report types and produces signed, independently-verifiable evidence packages. The through-line discipline across all five: **honesty over coverage** — the schema makes "we don't know" (`insufficient`, `best_effort`, `indeterminate`, `confidence=low`) a first-class, queryable, *disclosed* state, never silently promoted to a clean verdict.

## 2. Shared data model

- **The C2 event** (frozen, §3.13) is the atom. Everything else references it.
- **`correlation_id`** (canonical anchor = cluster-scoped Kubernetes Admission Review UID) joins events across Gatekeeper ↔ OPA ↔ K8s-API ↔ Conftest ↔ Privateer.
- **`replay_completeness`** (`complete | best_effort | insufficient`) is the honesty label that propagates upward: C1 `indeterminate` verdicts, C3 reduced-confidence findings, C4 confidence bands, and C5 disclosed denominators all derive from it.
- **Findings** (C3) and **GemaraEvaluations** (C1) and **violation rows** (C4) are higher-level artifacts, each chained via the **C2 integrity primitive** so the whole evidence chain is tamper-evident with one signing format.
- **Materialized datasets** (digest-addressable, immutable) are the unit of reuse across engineering replay and auditor walkthrough.

## 3. THE FROZEN C2 SCHEMA — field list other domains depend on

**Schema:** `c2.audit-event/1.0` — 36 top-level fields. Required (R) on every event; Conditional (C) per rule; Optional (O).

**Required (14):** `schema_version`, `event_id` (UUIDv7), `correlation_id`, `timestamp`, `ingest_timestamp`, `event_type` (`policy.decision|resource.change|scan.result|approval.request|approval.decision|auth.event|replay.synthetic`), `producer`, `subject`, `scope` (`{cluster,namespace,tenant,region,environment}`), `replay_completeness` (`complete|best_effort|insufficient`), `source`, `content_hash`, `prev_hash`, `chain_seq`.

**Conditional (18):** `decision` (`allow|deny|warn|mutate|suspend_pending_approval|require_approval|unknown`), `policy_engine` (`opa|gatekeeper|kyverno|conftest|application|scanner`), `policy_version`, `policy_ref` (`{rego_package,rule,constraint_template,constraint_name}`), `control_id`, `outcome_reason`, `mutation_diff` (RFC 6902), `jwt_claims`, `jwt_claims_completeness` (`full|partial|reconstructed`), `operation`, `resource_type`, `resource_id`, `before_state`, `after_state`, `request_object`, `external_data_refs` (`[{name,provider,version,digest,value_ref}]`), `replay_completeness_reasons`, `confidence_level` (`high|medium|low`).

**Optional (4):** `parent_correlation_id`, `action_performed`, `engine_context` (typed per-engine bag), `signature`.

**Frozen invariants other domains MUST honor:**
- **N-C2-FWD** — consumers ignore unknown fields; v1.x changes are additive only.
- **`replay_completeness` middle state is `best_effort`**; `partial` is a **deprecated alias normalized away at ingest** (no stored event carries `partial`) — *Decision D-C2-01*.
- **N-C2-SYNTH / N-C2-NOPROMOTE** — `replay.synthetic` (reconstructed) events are capped at `best_effort`, never `complete`; nothing is silently promoted to a verdict.
- **Correlation anchor** = cluster-scoped Admission Review UID; OPA MUST echo it into `correlation_id` (D-C2-03); federated dedup uses the *scoped* id.
- **Integrity** = per-source hash chain (`prev_hash`/`chain_seq`) + signed Merkle checkpoints + RFC-8785 canonical `content_hash`; one export-signing primitive platform-wide (D-C2-04 / D-C5-01 / D-C1-03).
- **Consumer API** (§10): `/events`, `/events/{id}`, `/correlations/{id}` (+ members view), `/coverage`, `/datasets`, `/verify`.

*Other domains may begin coding against this list now (M-FREEZE is satisfied by C2 SPEC §3.13).* The only post-freeze refinements proposed (C2 ALT) — CloudEvents wire envelope, log+read-model storage, OCSF export profile — **do not change this field list or the state machine**, so they are safe to adopt later without breaking consumers.

## 4. The 5 hardest decisions (domain-level)

| # | Decision | Where | Why it's hard / chosen rationale |
|---|---|---|---|
| **1** | **Freeze the schema now and make it additive-only.** | C2 §3.13 | C3/C4/C5/E1/F4 all block on it; churn after they code against it is the #1 risk. Freezing early (contract, not implementation) unblocks the whole domain in parallel. The cost is committing before all consumers are built — mitigated by additive-only + N-C2-FWD. |
| **2** | **Honesty over coverage — `insufficient`/`indeterminate`/`best_effort`/`low` are first-class and never silently promoted.** | C2 §5, C1 §2.2, C4 §2.6, C5 §4 | The temptation is to maximize "clean" coverage. But a falsely-`complete` record or a falsely-`satisfied` control is worse than an honest gap — it manufactures assurance an auditor will rely on. This is the platform's integrity (and its legal defensibility). |
| **3** | **Custom replay schema is authoritative; OCSF is one-way export only.** | C2 §9, ALT §A | OCSF is the market standard and tempting to adopt natively — but it cannot represent replay inputs (`request_object`, versioned `external_data_refs`, `replay_completeness`), which is the entire differentiator (market §5). OCSF survives as a published export *profile*. |
| **4** | **Tamper-EVIDENT append-only log (hash chain + signed Merkle checkpoints), not a mutable relational store.** | C2 §7, ALT §B | A relational store is easier to query but its evidence is "trust our DBA"; the hash chain gives an auditor who trusts only a published public key an independent verification path (HL-18/DT-24). Synthesis: log = system of record, relational read-model = query tier (ALT R-ALT-2). |
| **5** | **Strict ownership split: C3 detects · C4 reconstructs+sweeps · E1 replays · C1 evaluates · C5 reports · C2 signs.** | all SPECs + DOMAIN-ADVERSARIAL | Multiple components touch "bypass," "replay," and "signed export." Without a hard split they triple-implement and diverge (auditor sees disagreeing numbers). The split is enforced by shared libraries (C3/C4 detector lib) and a single C2 integrity primitive. |

## 5. Consolidated open questions (with decided defaults)

| OQ | Question | Decided default | Revisit at |
|---|---|---|---|
| C2-OQ1 | Physical store backend? | Backend-agnostic log; ALT recommends log-of-record + relational read-model projection (R-ALT-2) | scale (§22) |
| C2-OQ3 | Retain raw external-data *values* vs versions? | Values for volatile providers (signature status), versions for stable; catalog flags which | per-provider |
| C2-OQ4 | PII in `jwt_claims`? | Redaction-aware keyed-HMAC (per adversarial D13), deployment-time allowlist | D4 security review |
| C3-OQ1 | Streaming vs batch detectors? | Batch reconciliation (15m) authoritative; streaming fast-path for bypass/integrity | MVP+ |
| C4-OQ2 | Sweep look-back horizon? | Bounded by C2 raw retention (≥30d POC); older = disclosed best-effort | retention policy |
| C5-OQ1 | PDF authority? | PDF is a rendering; signed package is the evidence (made normative per adversarial D5) | — |

## 6. Critical path & sequencing (intra-domain)

```
C2 M-FREEZE (DONE — SPEC §3.13)
   │  (unblocks design/contract-testing for ALL downstream NOW)
   ▼
C2 M1 walking skeleton ─▶ C2 M2 completeness+correlation ─▶ C2 M4 query API ─▶ C2 M5 signed exports
   │                                                            │
   ├─▶ C3 framework ─▶ C3 D-BYPASS ──────────┐                  │
   │                                          ▼                  ▼
   ├─▶ C1 Gemara eval (needs A1) ────────────┤            C5 R1 (C2-only) ─▶ C5 signed export
   │                                          ▼                  ▲
   └─▶ C4 (needs C3 lib + E1 replay) ─▶ violation population ────┘ (R2/R3/R4)
```

**The domain critical path runs through C2 M-FREEZE → C2 query API → C4 (the most-blocked component: C2 + C3 + E1).** C4 is the tail; everything else parallelizes off C2's contract. **C4's hard external dependency is E1 (replay)** — the only Domain-C dependency on another domain's *engine* rather than a contract. Mitigation across all PLANs: build against C2's frozen contract + stubs immediately; only final replay wiring waits on E1.

## 7. What other domains should take away
1. **Code against the §3 frozen field list today.** It will not break under you (additive-only).
2. **Never emit or expect `replay_completeness=partial`** — it's normalized to `best_effort` at ingest.
3. **Use the cluster-scoped Admission Review UID as `correlation_id`**, and make OPA echo it.
4. **Sign exports only via C2's integrity primitive** — do not roll your own.
5. **E1 owes Domain C a deterministic replay** that ties out to ~0% divergence on `complete` events (C4 adversarial D4); that determinism is the platform's auditor-trust anchor.
