# HANDOFF — Detailed Spec + Plan Corpus

**Date:** 2026-05-30 · **Branch:** `claude/spec-plan-review-parallel-cDAcL` · **PR:** #5

This is the **pick-up-here** document. If you read one file after `00-MASTER-INDEX.md`,
read this. It states what exists, what's decided, what's still open, and what to do next.

---

## 1. What this corpus is (and is not)

A maximally-parallel, cooperative-**and**-adversarial spec + implementation plan for
**every piece** of the policy-engine design corpus — the unified governance / policy-
enforcement platform (Gemara + OPA/Rego + Conftest + Gatekeeper + Kyverno + Privateer +
Keycloak), plus its operational/NFR layer and an AI-agent extension.

- **It is** a scrutinized design corpus: SPEC + PLAN + adversarial review per component,
  alternative-architecture trees on contested pieces, cross-cutting reconciliation, and
  three independent "should this even exist / is it right-sized" reviews.
- **It is NOT** blessed or final. Multiple alternative trees are kept deliberately for you
  to choose between and discard. Every "frozen" contract that an adversarial pass reopened
  is marked provisional. Nothing here has been built.

Entry point: [`00-MASTER-INDEX.md`](00-MASTER-INDEX.md). Method: [`00-ORCHESTRATION-PLAN.md`](00-ORCHESTRATION-PLAN.md).
Every unattended decision: [`DECISIONS.md`](DECISIONS.md).

---

## 2. What has been done

| Wave | Output | Status |
|---|---|---|
| 0 | Inventory, decomposition (31 components / 7 domains), orchestration plan, traceability seed | ✅ |
| 1 | **23 functional components** (A–F) × SPEC + PLAN + ADVERSARIAL, + 8 ALT trees, + 6 domain index sets | ✅ |
| 2 | Master build plan · wedge-first alt · cross-domain adversarial (22 defects) · data model (51 entities) · 6 frozen inter-domain contracts · traceability | ✅ |
| 3 | Master-index roll-up, decision log, top-level INDEX pointer, link sweep | ✅ |
| 4 | C2 `v1.0-rc` reconciled schema + 9-ticket build-blocking checklist · meta-adversarial second opinion · thesis devil's-advocate | ✅ |
| 5 | **8 NFR components** (Domain G) × SPEC + PLAN + ADVERSARIAL, + 4 ALT trees | ✅ |
| 5-close | Domain-G index · NFR cross-cut adversarial (chain-model composes ✓) · NFR devil's-advocate (POC cut-line) | ✅ |

**~145 files / ~23,000+ lines. 25 agents across 6 waves.** All committed and pushed to PR #5.
**The corpus is complete; the remaining work is decisions + one reconciliation pass (§3, §6).**

### The 31 components
- **A** Governance Core: A1 Gemara model, A2 Policy lifecycle
- **B** Policy Engines: B1 OPA/Rego, B2 Gatekeeper, B3 Conftest, B4 Engine-selection/actions/CRDs, B5 Real-time enforcement
- **C** Evidence/Audit: C1 Privateer, C2 Audit schema *(keystone)*, C3 Analytics, C4 Retrospective detection, C5 Reporting
- **D** Identity/Authz: D1 Keycloak/JWT, D2 Scoped RBAC + storage authz, D3 Approval-gated, D4 Security
- **E** Simulation/Console: E1 Simulation, E2 Governance console/Headlamp, E3 PDP libraries
- **F** Platform/Cross-cut: F1 API, F2 Deployment/extensibility, F3 MVP sequencing, F4 AI-agent extension
- **G** Operational/NFR: G1 Scale, G2 Cost, G3 DR, G4 Keys, G5 Tenancy, G6 Observability/Day-2, G7 Data-lifecycle/Privacy, G8 Rego-authoring/Human-factors

---

## 3. What is still to do (the open items)

### 3a. Must happen before any build — the C2 `v1.0-rc` re-freeze
The keystone audit schema was declared frozen prematurely and **seven independent
components have since asked it to change.** These MUST be reconciled into one schema and
re-frozen first. Consolidated ratification list (see [`C2-v1.0-rc-RECONCILED.md`](cross-cutting/C2-v1.0-rc-RECONCILED.md)
and [`BUILD-BLOCKING-FIXES.md`](cross-cutting/BUILD-BLOCKING-FIXES.md) BB-1..BB-9, plus the NFR additions):

| Source | Ask on C2 |
|---|---|
| BB-1/2 (XD-3/11) | Split `disposition` (exclusive verdict) from `obligations[]` (co-occurring); one canonical action taxonomy |
| BB-3 (XD-1) | External-data **value** capture mandatory for volatile providers (so image-signing replays `complete`) |
| BB-4 (XD-2) | `replay_completeness` is a per-event lower bound; E1 recomputes per-bundle |
| BB-5 (XD-13) | Pin `catalog_version` per `policy_version` |
| BB-6 (XD-12) | Wire `jwt_claims_completeness` (D1 sets → C2 carries → C3 detects) |
| BB-7 (XD-8) | Retry-stable `correlation_id` + `governance_transaction_id` |
| **G1** | **Shard the hash chain by (source, cluster)** — the single serialization bottleneck |
| **G3** | New `infrastructure_degraded` disposition + `chain.restore_boundary` / `fork_reconciliation` event types |
| **G5** | **Per-tenant hash chain** (offboard = crypto-shred, no cross-tenant collateral) |
| **G7** | `insufficient:erased_input` reason code (lawful erasure ≠ capture defect) |
| **G4** | `key_id`-indexed verification + Key Transparency Log binding |

**Answered (was open):** do chain-sharding (G1) + per-tenant-chain (G5) + epoch-boundaries
(G6) + restore-markers (G3) compose into *one* coherent chain model? **Yes, conditionally**
(`NFR-CROSSCUT-ADVERSARIAL.md` NX-1, §"do they compose"). The unified model: chain identity =
**`(tenant, source.system, cluster, epoch)`**, `chain_seq` monotonic within that tuple, plus a
**global signed roll-up super-checkpoint** over all chain-heads (the keystone that keeps
whole-chain deletion globally detectable); G6's schema-epoch seal and G3's failover-epoch
fence unify into one cross-sign mechanism with two triggers; crypto-shred rides inside any
chain without touching `content_hash`. They compose **in principle but conflict on paper** —
the C2-rc still defines `chain_seq` as a bare per-source integer. The fix is the §3a
reconciliation pass folding N-1..N-10 into the rc *before* re-freeze. **This is the single
highest-priority authoring task left.**

### 3b. Other build-blocking correctness fixes (not C2)
- **BB-8 (XD-5):** D2 `ScopePredicate` must be traversed by analytics/reporting aggregate
  reads, not just CRUD. G5 promotes RLS-under-interceptor to mandatory.
- **BB-9 (XD-9):** ratify CRD ownership (B4 schemas / F2 controllers / F1 REST).
- **SoD (XD-4):** enforce author≠approver mutual-exclusion at grant time.

### 3c. Strategic decision you must make (reviewers disagree with the default)
Three independent reviewers (`MASTER-PLAN-ALT`, `THESIS-DEVILS-ADVOCATE`,
`META-ADVERSARIAL-SECOND-OPINION`) + the market memos favor **wedge-first over the
platform-first default**, and the thesis review goes further: *narrow to Wedge 5 (OPA
Control Plane successor) or kill it; buy ~20 components, build ~2.* The go/no-go test:
**name 3 design partners who'll sign a 90-day paid pilot for one named wedge** and the exact
sentence each says about why OPA Control Plane / Wiz / their Vanta+Kyverno stack can't do it.
Both build trees (`MASTER-PLAN.md` and `MASTER-PLAN-ALT.md`) are preserved — your call.

### 3d. Known gaps NOT yet spec'd
- **C2-rc reconciliation pass itself** (3a) — needs a human or a follow-up agent to merge the
  ratification list into one schema and update the docs that still call it "frozen."
- **F4 (AI extension) sequencing tension** — the devil's-advocate argues it's the one
  forming market and is sequenced last (Phase 3); revisit if AI-agent governance is the wedge.
- A few NFR open questions remain inside each G component (e.g. G4 KTL DR owner, G5
  crypto-shred backup residue, G7 third-party PII in agent prompts) — listed in each
  component's SPEC "open questions" and the Domain-G `DOMAIN-ADVERSARIAL.md`.

---

## 4. The 10 findings worth your time (the adversarial layers earned their keep)

1. **C2 was "frozen" while still wrong** → resolved into `v1.0-rc`; 11 components want changes.
2. **The five Wave-2 synthesis docs disagreed with each other** on whether C2 was frozen
   (the meta-adversarial caught it) → one canonical artifact now supersedes them.
3. **The differentiator and the cost cliff are the same decision** (G2): full-population
   replay-complete capture (XD-1) is what turns a cheap event product into a PB-scale one.
4. **The hash-chain append is the one un-parallelizable bottleneck** (G1) — the very
   property that makes it tamper-evident caps central ingest throughput.
5. **You cannot migrate an append-only signed log in place** (G6) — solved with chain-epoch
   boundaries; every schema change AND key rotation becomes an epoch transition.
6. **GDPR erasure vs immutable audit** (G7) — the corpus never noticed the collision; solved
   with per-subject crypto-shredding that preserves chain verification, at an honest replay cost.
7. **Key rotation vs verifying 3-year-old evidence** (G4) — solved with a Key Transparency
   Log; audit-root and supply-chain artifacts need *opposite* custody (KMS vs Sigstore-keyless).
8. **Composed fail-closed defaults → fleet-wide mass-deny** (B + G3) — solved with a
   criticality-tiered circuit breaker + anti-self-brick invariant.
9. **Multi-tenancy is a WHERE clause** (G5) — soft tenancy is unsafe for regulated buyers;
   a per-tenant T0–T4 dial with mandatory RLS, sold at T2+ to regulated customers.
10. **The authoring safety apparatus proves the median engineer fails at raw Rego** (G8) —
    narrow authoring to experts, or re-market from a "competence" to a "containment" claim.

---

## 5. How to navigate

- **5-minute version:** `00-MASTER-INDEX.md` → "Headline reconciliation flags" + "Strategic
  signal" + this handoff's §4.
- **Build planning:** `cross-cutting/MASTER-PLAN.md` (platform-first) or `MASTER-PLAN-ALT.md`
  (wedge-first); then `BUILD-BLOCKING-FIXES.md` for the day-1 fix queue.
- **Per-piece depth:** `components/<id>/SPEC.md` + `PLAN.md`; read its `ADVERSARIAL-REVIEW.md`
  before trusting it; check for `ALT-*.md` where a different architecture is on the table.
- **Risk register:** `cross-cutting/CROSSCUT-ADVERSARIAL.md` (functional XD-1..22),
  `domains/G-operational-nfr/DOMAIN-ADVERSARIAL.md` (NFR XG), `cross-cutting/NFR-CROSSCUT-ADVERSARIAL.md`.
- **Should we build it:** `cross-cutting/THESIS-DEVILS-ADVOCATE.md` + `NFR-DEVILS-ADVOCATE.md`.

## 6. Recommended next moves (in order)

1. **Decide the strategic question (§3c)** — platform-first vs wedge-first. Everything below
   is cheaper if you pick a wedge.
2. **Run the C2 `v1.0-rc` reconciliation pass (§3a)** — merge the 11-item ratification list
   into one schema; confirm the chain-model changes compose (await `NFR-CROSSCUT-ADVERSARIAL`).
3. **Do an operational spike** answering the three load-bearing unknowns: cross-engine Rego
   portability, replay-coverage (what % of real events actually replay `complete`), and the
   per-source chain throughput at target scale.
4. **Then** start the foundation-contract freeze week (C2, A1, D1, B4 + the BB fixes) and the
   first wave of the chosen plan.
