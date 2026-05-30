# Domain G — Operational / Non-Functional Requirements (NFR) — INDEX

**Domain lead:** Domain G parent agent · **Date:** 2026-05-30 · **Status:** Wave (NFR) complete (SPEC+PLAN+ADVERSARIAL for all 8 components; ALT for G1, G3, G4, G5).

Domain G is the **operational architecture the functional corpus (A–F) never wrote**. Every functional component is sized for *correctness*; Domain G supplies *throughput, cost, durability, key custody, tenancy, observability, lifecycle, and human-factors*. It was authored to close the seven "no owner" gaps and the ten new risks the independent architecture-review-board pass surfaced (`cross-cutting/META-ADVERSARIAL-SECOND-OPINION.md` §3–4). **Domain G owns almost no code of its own** — nearly every G requirement is a *contract change on a functional component*, most often **C2** (the evidence keystone) and **D2** (the scope/storage layer). The recurring shape of this domain is: *"this NFR is really a cross-cutting contract."*

---

## Components

| ID | Component | Spec sources | Purpose (one line) | Closes META gap / risk | Files | Status |
|---|---|---|---|---|---|---|
| **G1** | Scale, Performance & Capacity | §22, §18/§17B/§17C.6, §12–14, §17/§17E | Latency/throughput budgets per hot path + a capacity model (POC/10×/100×) + the per-component bottleneck map; the **hash-chain is the #1 scale ceiling** | Risk #2 (no scale/cost model) | [SPEC](../../components/G1-scale-performance/SPEC.md) · [PLAN](../../components/G1-scale-performance/PLAN.md) · [ADVERSARIAL](../../components/G1-scale-performance/ADVERSARIAL-REVIEW.md) · [ALT](../../components/G1-scale-performance/ALT-streaming-sharded-sampled.md) | DRAFT v1 |
| **G2** | Cost & Retention Economics | §22, §24–25, C2 schema | Parameterized TCO model (compute/storage/egress/KMS); build-vs-buy; the **cost-cliff = differentiator** finding | Risk #2 + #4 (retention economics) | [SPEC](../../components/G2-cost-retention-economics/SPEC.md) · [PLAN](../../components/G2-cost-retention-economics/PLAN.md) · [ADVERSARIAL](../../components/G2-cost-retention-economics/ADVERSARIAL-REVIEW.md) | DRAFT v1 |
| **G3** | Availability, DR & Resilience | META §3 / Risk #4, B-domain X-4, C2 §7–8, B5 §5, F2 §2–6 | Chain-continuous DR (RPO=0 + `restore_boundary`) and the per-criticality **circuit breaker / `infrastructure_degraded` mode** (resolves the composed mass-deny) | Risk #4 (DR/RPO/RTO) | [SPEC](../../components/G3-availability-dr-resilience/SPEC.md) · [PLAN](../../components/G3-availability-dr-resilience/PLAN.md) · [ADVERSARIAL](../../components/G3-availability-dr-resilience/ADVERSARIAL-REVIEW.md) · [ALT](../../components/G3-availability-dr-resilience/ALT-DR-TOPOLOGY.md) | DRAFT v1 |
| **G4** | Key Management & Cryptographic Lifecycle | §23, §26.1, C2 §7, B1 §6.2, D1 §4, D4 §2–3 | Every key's type/custody/rotation/revocation/trust-root; **long-lived signature verification across rotation** via `key_id` + append-only Key Transparency Log (KTL) | Risk #3 (key-mgmt unowned); resolves D4-OQ-1 | [SPEC](../../components/G4-key-management/SPEC.md) · [PLAN](../../components/G4-key-management/PLAN.md) · [ADVERSARIAL](../../components/G4-key-management/ADVERSARIAL-REVIEW.md) · [ALT](../../components/G4-key-management/ALT-key-custody-models.md) | DRAFT v1 |
| **G5** | Multi-Tenancy Isolation | META §3 / Risk #9, XD-5, D2/D1/C2/F2 | The **per-tenant isolation dial** (soft→hard) across six planes; per-tenant chain, crypto-shred deletion, residency; *a tenancy contract, not a component* | Risk #9 (tenancy = WHERE clause) | [SPEC](../../components/G5-multitenancy-isolation/SPEC.md) · [PLAN](../../components/G5-multitenancy-isolation/PLAN.md) · [ADVERSARIAL](../../components/G5-multitenancy-isolation/ADVERSARIAL-REVIEW.md) · [ALT](../../components/G5-multitenancy-isolation/ALT-tenancy-models.md) | DRAFT v1 |
| **G6** | Observability, SLOs & Day-2 Ops | META §3 / Risk #10 + #8, F2, C2-rc, B4, G1/G3/G4 | Self-observability (platform telemetry ≠ customer evidence), SLOs/error-budgets, day-2 runbooks; owns the **C2 1.0→1.0-rc chain-epoch migration** | Risk #10 (day-2 burden) + #8 (migration) | [SPEC](../../components/G6-observability-day2-ops/SPEC.md) · [PLAN](../../components/G6-observability-day2-ops/PLAN.md) · [ADVERSARIAL](../../components/G6-observability-day2-ops/ADVERSARIAL-REVIEW.md) | DRAFT v1 |
| **G7** | Data Lifecycle, Retention & Privacy | C2-rc, D2 §5.3–5.4, F4, D4 §2.2/OQ-4, META §3 | Retention/tiering; the **erasure-vs-immutability resolution** (per-subject crypto-shred); PII classification; DSR pipeline; residency hand-off | Risk #4 (retention) + #5 (`complete` minority) | [SPEC](../../components/G7-data-lifecycle-privacy/SPEC.md) · [PLAN](../../components/G7-data-lifecycle-privacy/PLAN.md) · [ADVERSARIAL](../../components/G7-data-lifecycle-privacy/ADVERSARIAL-REVIEW.md) | v1.0 |
| **G8** | Rego-Authoring & Human Factors | META §3 / Risk #7, THESIS-DEVILS-ADVOCATE | The **authoring experience** as an NFR: Regal ruleset, templates, the §26.1 generator hand-off, mass-deny guardrails, and the *measurement of authoring error rate* nobody owned | Risk #7 (no Rego-author owner) + #6 (portability) | [SPEC](../../components/G8-rego-authoring-human-factors/SPEC.md) · [PLAN](../../components/G8-rego-authoring-human-factors/PLAN.md) · [ADVERSARIAL](../../components/G8-rego-authoring-human-factors/ADVERSARIAL-REVIEW.md) | DRAFT v1 |

**High-value components (carry ALT trees):** G1, G3, G4, G5. (G2, G6, G7, G8 ship SPEC+PLAN+ADVERSARIAL.)

---

## NFR cross-cut map — which functional components (A–F) + spec §§ each G component imposes a contract on

Domain G is *defined* by its cross-cuts. Each NFR component lands as a set of MUSTs on the functional corpus; the table below is the reconciliation surface for Wave-3.

| G | Primary contract changes (the load-bearing ones in **bold**) | Functional components touched | Spec §§ |
|---|---|---|---|
| **G1** | **C2: per-source chain must shard by `(source.system, cluster)` + cross-shard roll-up super-checkpoint**; CQRS split of write-index vs analytics store; eval:recorded-event ratio knob; build-time cache-seed of signature status; policy-cost linter | **C2**, B1, B2, B3, D1, E1, F2 | §12–13 (C2 chain/store), §8.1 (indexes), §18 (admission), §17 (replay) |
| **G2** | C2: index/read-model growth line; per-event-signing cost condition; per-year ramp + sensitivity bands; cost-per-tenant attribution (E1 = job-attribution) | C2, F2, E1, (build-vs-buy: whole corpus) | §22, §24–25, §12–13 |
| **G3** | **C2: `infrastructure_degraded` disposition + `degraded_session_id` + `chain.restore_boundary` / `fork_reconciliation` event types**; D1: exception/approval restore reconciled against the D0 log; B1/B2/B5: per-criticality breaker dispositions; G4: public keys embedded in WORM backup | **C2**, B1, B2, B5, D1/D3, F2, G4 | §12–13 (C2), §17C.6/§18 (enforcement), §23 (integrity) |
| **G4** | **C2: `key_id` resolution + append-only Key Transparency Log (KTL) the verifier consults**; D4: signer custody (KMS/HSM, IAM split) promoted to MUST; B1: bundle-signing trust domain separation; D1: JWKS trust-anchor lifecycle | **C2**, D4, B1, C1, C5, D1 | §23, §26.1, §7 (C2 integrity), §6.2 (B1 bundles) |
| **G5** | **C2: per-tenant hash chain (global→per-tenant re-architecture) + global chain-head meta-log**; D2: RLS mandatory + no-direct-store-handle; D1: issuer→tenant binding; C3/C5: scoped analytics; F2: region spokes; G1/G4/G7: per-tenant quota/key/erasure | **D2**, **C2**, D1, C3, C5, F2, G1, G3, G4, G7 | §20.2 (multi-tenant), §12–13 (C2), D2 §5, §14/§17E (analytics) |
| **G6** | **C2: scoped `disposition_context` for degraded ops; chain-epoch migration runbook**; F2: plugin `/metrics` golden-signal SPI; B5/C2: async evidence enrichment off the admission path; OBS/RUN boundary (what is a governed C2 event vs telemetry) | **C2**, F2, B5, G1, G3, G4, G5 | §12–13 (C2 migration), §24–25 (F2 day-2), §18 (SLO) |
| **G7** | **C2: `insufficient:erased_input` + `complete:memoized_post_erasure` completeness sub-states; inline-PII moved out-of-line to CAS so GC is chain-safe**; B1/B5: input-fact capture for post-erasure replay; D2: redaction; G4: per-(subject,tenant) DEK grain; F4: agent-content erasure | **C2**, B1, B5, D2, F4, G4, G5 | §12–13 (C2 completeness/retention), §13.3, D2 §5.3, F4 (agent) |
| **G8** | A2/D3: policy-code review/promotion; B1: Regal ruleset + `__control_id__` generation reconciled with B1-R1; **B1-R30 cross-engine conformance suite (the portability proof)**; E1: mandatory differential sim gate; G3: mass-deny circuit breaker; E3: template/PDP-lib catalog | B1, B4, A2/D3, E1, E3, G3, D2, E2 | §26.1 (generation), §7 (authoring metadata), §17 (sim gate) |

**Reading of the map:** **C2 appears in 7 of 8 rows** and **D2 in the two most structural (G5, G7)** — these two are the domain's gravitational centers. The five **bold** items in G1/G3/G5/G7/G4 are *correctness-class* contract changes that must be folded into the already-open **C2 `v1.0-rc`** ratification pass (META M-1), not left as G-backlog (the M-1 propagation-failure trap — see `DOMAIN-ADVERSARIAL.md`).

---

## META-ADVERSARIAL §3–4 coverage (the gaps Domain G fills)

| META gap (§3) / Risk (§4) | Owner in Domain G |
|---|---|
| Production scale / cost / performance (Risk #2) | **G1** (scale) + **G2** (cost) |
| Audit-log retention economics (Risk #4) | **G2** (economics) + **G7** (retention/tiering mechanics) |
| DR / RPO / RTO for the evidence store (Risk #4) | **G3** |
| Key management for all the signing (Risk #3) | **G4** |
| Multi-tenancy isolation beyond query scoping (Risk #9) | **G5** |
| Day-2 operational burden of a 14-service stack (Risk #10) | **G6** |
| Schema migration in production (Risk #8) | **G6** (chain-epoch migration runbook) |
| "Authoritative replay may be a minority of real events" (Risk #5) | **G7** (`erased_input`/memoized sub-states) + G1 (replayable `|S|`) |
| Cross-engine portability / Kyverno breaks it (Risk #6) | **G8** (honest scope: portable only across Rego-executing engines) |
| Human factors — who writes the Rego (Risk #7) | **G8** |

---

## Reading order

1. `G1/SPEC.md` — the capacity model + bottleneck map everything else sizes against.
2. `G2/SPEC.md` — the cost half (consumes G1's paper model).
3. `G3/SPEC.md` + `G4/SPEC.md` — durability + key custody (the two that make the evidence store trustworthy across time).
4. `G5/SPEC.md` + `G7/SPEC.md` — tenancy + erasure (the two heaviest C2/D2 re-architectures).
5. `G6/SPEC.md` — day-2 ops + the chain-epoch migration runbook.
6. `G8/SPEC.md` — the authoring layer the whole thesis rests on.
7. The eight `ADVERSARIAL-REVIEW.md`, then `DOMAIN-ADVERSARIAL.md` (the ranked XG register + the C2-rc ratification asks).
8. `DOMAIN-SUMMARY.md` (through-lines, 5 hardest decisions, consolidated open questions).
