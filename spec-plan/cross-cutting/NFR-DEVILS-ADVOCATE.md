# NFR — DEVIL'S ADVOCATE: The Scope-Control Review

**Status:** ALT / opposing-voice tree · **Date:** 2026-05-30
**Author persona:** Pragmatic POC-stage engineering director doing a scope-control review.
**Reads against:** the 8 `components/G*/SPEC.md` (Domain G, the NFR wave), `components/F3-mvp-sequencing/SPEC.md` (the MVP cut line), `cross-cutting/THESIS-DEVILS-ADVOCATE.md` (wedge-first), `cross-cutting/MASTER-PLAN-ALT.md`, and §22 (POC scale) of `openssf_opa_unified_governance_platform_spec v1.md`.

> This document is deliberately one-sided. It is the scope-control opposing brief, not a balanced assessment. The honest call is in §5. My job is to be the voice of **"what do we ACTUALLY need for the POC / first paying design partner,"** and to push back on a Domain-G wave that — read against F3 §22 and the wedge-first thesis — is sizing for a **regulated 50M-events/day, 7-year-retention, multi-region enterprise that has not signed, may not exist, and that two prior reviewers argued the platform should not chase yet.**

---

## TL;DR — verdict, cut-line, and the one to fund first

**Verdict: Domain G is OVER-BUILT for the POC and the first design partner — by a wide margin — but it is NOT wrong, and one corner of it is actively UNDER-built.** The G-wave is excellent *scale architecture*. It is the wrong *sequencing*. ~70% of the normative MUSTs in G1–G5 are gated on a "Scale-C" enterprise (G2's own term) that, per the THESIS and ALT reviewers, the company has not proven a buyer for. The G-wave was authored to close META-ADVERSARIAL findings — but META asked *"can this scale?"*, and the answer for a POC is *"it doesn't have to yet."* §22 says so in one sentence: **"sized for functional validation, not production telemetry processing… the POC's success metric is correct, traceable, replayable decisions, not throughput"** (R-F3-PERF-4).

**The POC NFR cut-line (the 5–8 things that genuinely gate a first design partner):**

1. **Don't lose the audit log (G3, single mechanism).** One quorum-or-equivalent durable write so a committed decision survives a node/disk loss + a daily WORM/object-lock backup. RPO≈0 for single-node, restore-and-it-still-verifies. **Not** active-active multi-region DR. (G3 §3.1 row 1 only.)
2. **Don't leak the signing key (G4, floor only).** KMS-backed (not a K8s Secret), `key_id` in the envelope, all old public keys retained so old signatures verify. **That's it.** Defer the air-gapped trust-root, the Key Transparency Log, external timestamp anchoring, dual-control ceremonies, PQC.
3. **The mass-deny circuit breaker (G3 §4).** The one G-requirement that earns its keep at POC: fail-open/degrade-to-warn for a shared-dependency outage + the anti-self-brick carve-out (`governance-system` exempt from its own admission). A first design partner *will* hit this in week one; it is cheap and it prevents a fleet-wide self-inflicted outage.
4. **The admission latency budget + external-data timeout/cache (G1 §2).** The p99≤1s budget, the bounded external-data timeout strictly inside the webhook deadline, and the cache. This is correctness-of-the-hot-path, not scale — a slow external-data call on a `failurePolicy: Fail` webhook is a production outage at *any* scale.
5. **PII-aware ingest + crypto-shred erasure for the identity fields (G7 §5, narrow).** Per-subject encryption of `sub`/`email`/agent-prompt fields at ingest + DEK-destroy erasure. This is **load-bearing the day the first design partner's legal team reads the contract** — you cannot retrofit "encrypt at ingest" onto an append-only log after it has events. Defer the full DSR pipeline, residency, legal-hold machinery.
6. **The novice authoring guardrails + mandatory sim-before-enforce (G8 §6.1, §7).** The "no runtime policy reaches enforce un-simulated" gate and the lint that catches the fail-open/overbroad footgun. This is what makes the *demo itself* (AC-2, AC-5) not blow up, and what stops design-partner #1 from mass-denying their own fleet. Cheap, high-signal, gates the actual product.
7. **Self-observability floor + the lossless-ingest SLO (G6, thin).** The "drop count MUST be 0" alert, the chain-integrity monitor, the `governance-system` self-exemption, and a ≤2-pages/shift on-call sanity. Not the full 14-service golden-signal matrix, SLO error-budget policy docs, or quarterly DR game-days.
8. **Single-tenant. Full stop (G5).** Per G5's *own* decision D-G5-1: **"The POC ships at T0 (single-tenant)."** Everything in G5 — per-tenant chains, per-tenant keys, the tenant registry, residency, RLS-under-interceptor, the six-plane isolation model — is **deferred until tenant #2 pays**.

**The one NFR I would fund first: G8 (Rego-authoring / human-factors), closely chased by the G3 circuit-breaker + G7 ingest-encryption seam.** Reasoning in §5. It is the only G-component whose failure mode kills the *POC demo* rather than a hypothetical enterprise, and the only one the THESIS reviewer independently flagged ("portability ⇐ author quality") as load-bearing for the thesis itself.

---

## 1. Per-component: genuinely-required-for-POC vs scale gold-plating

For each of the 8 G components: the POC-minimum (one line), then what is enterprise gold-plating deferrable to "when a customer pays for it."

### G1 — Scale, Performance & Capacity

The SPEC's own framing is the tell: §0 says F3 §22's "correct, not throughput" is *"the right call for a POC and the wrong basis for selling to a regulated enterprise."* Correct — but the POC **is** the POC. The entire 100×/Scale-C apparatus (125 webhook replicas, sharded hash chains, 5,000/s burst ingest, 1.5B-event cluster replay) is sizing for the enterprise that hasn't signed.

- **Genuinely required:** §2 the admission latency budget (p99≤1s, p50≤150ms) and **N-G1-102** (external-data timeout strictly < webhook deadline) + **N-G1-103** (external-data cache). This is hot-path *correctness*, not scale — it holds at 6 evals/sec just as at 100k. The single normalizer sustaining ≥200 events/s (N-G1-111) is trivially met and fine to keep.
- **Scale gold-plating (defer):** **N-G1-112/113 sharded hash chains** — at POC's 6/s avg, 50/s burst, a single chain on one node is "trivially fine on one chain" *by the SPEC's own admission* (§3.3). Defer sharding entirely. The 10×/100× capacity model (§9), the 7 load-test scenarios at scale (§8.2 LT-1..7 at 100×), the cold-tiering performance contract, the replay cost-preflight / cluster-replay-rejection (N-G1-144) — all of it is "when a customer pays for it." Cluster-wide replay at POC scale (≤15M events) just *works*; you don't need to forbid it.
- **POC-minimum:** *Admission p99≤1s with a bounded external-data timeout-inside-the-deadline and a cache; one un-sharded hash chain on one durable node; a nightly perf smoke at POC scale only.*

### G2 — Cost & Retention Economics

This is the most clearly-deferrable component for a POC, and the SPEC *proves it itself*: **"The POC is compute-bound, not storage-bound. Storage is $2/mo — a rounding error"** (§4.1). G2's entire reason to exist is the Scale-C ($490k/yr, 3.6 PB, 7-year) cost cliff — a number that is, by construction, invisible until you have a regulated enterprise.

- **Genuinely required:** essentially nothing *as a build*. The one durable insight worth keeping as a one-page note: **the cost model agrees with the THESIS** ("buy-20-build-2 is ~$7M cheaper over 3 years," §7/§9.4) — which is a *strategy input*, not a POC NFR.
- **Scale gold-plating (defer):** the entire §3 formula apparatus, the §4 three-scale TCO, the §5 hot/warm/cold/deep-archive tiering economics, the §6 per-tenant showback/chargeback, the retrieval-cost trap. None of it bites until storage compounds, which §22's 30-day window guarantees it won't.
- **POC-minimum:** *Keep 30 days hot on ordinary storage (§22 literally says "ordinary relational/document/object storage is acceptable"); write the build-vs-buy finding into the strategy memo and stop.*

### G3 — Availability, DR & Resilience

This component contains both **the single most POC-justified G-requirement** and **the single most POC-unjustified one**, and the SPEC even labels the latter ("The ≤60s async-replication window is the single most scrutinized number in this spec," §3.1).

- **Genuinely required:** (a) **Don't lose committed evidence on a single-node/disk failure** — G3 §3.1 row 1: RPO=0, restore-and-it-still-verifies, + the WORM backup (R-G3-BK-2). (b) **The circuit-breaker / infrastructure-degraded mode (§4)** — distinguishing "policy says no" from "platform is down" + the anti-self-brick carve-out (R-G3-CB-9). This is the one piece of G that a first design partner hits *immediately*: any shared-dependency hiccup mass-denies their fleet without it. (c) The restore-boundary-marker trick (§5.3) so a restore doesn't look like tampering — but only the *single-region* form.
- **Scale gold-plating (defer):** **active-active synchronous cross-region DR (P3, R-G3-RPO-3)** — this is gated on "a regulated buyer signs," which is precisely the buyer the THESIS says doesn't exist yet. The ≤60s cross-region async window, the split-brain hash-chain fork reconciliation (§7.3, CHAOS-9), the full FMEA (§6, 18 rows), the quarterly DR game-days + 8 GA-gating drills (§8.3), the per-data-class differentiated RPO/RTO matrix beyond D0. **Per-tenant DR is N/A** (single-tenant POC).
- **POC-minimum:** *Single-region multi-AZ-or-just-durable evidence store (RPO≈0 to disk loss) + daily WORM backup that restores-and-verifies; the mass-deny circuit breaker + anti-self-brick carve-out. No multi-region, no active-active, no game-day program.*

> **Is active-active DR needed before a regulated buyer signs? No.** G3's own OQ-G3-1 defaults the POC to single-region (P1). The SPEC already agrees; the gold-plating is in treating P2/P3 as in-scope build rather than "deployment elections priced later."

### G4 — Key Management & Cryptographic Lifecycle

The spine of this SPEC is *"a signature made three years ago must still verify after the key that made it has rotated."* That is a real property — for a **7-year regulated evidence store**. For a POC with a 30-day window, it is hypothetical. The component is the textbook case of *foundational-vs-hardening confusion*: META said key management is "foundational not hardening," and G4 over-corrected into a full PKI program.

- **Genuinely required (the floor G4 itself names):** **K1 audit key in KMS, non-exportable, NOT a cluster Secret** (G4-R4 — the "cluster-admin can forge the chain" defeater is real at any scale and cheap to prevent), **`key_id` in the envelope** (already in C2 §7.4) so rotation is *possible* later, and **retain old public keys** so old signatures verify. The custody split (keyless Sigstore for bundles K3/K4, which B1 already does) is free — inherit it.
- **Scale gold-plating (defer):** the **air-gapped offline trust-root hierarchy (§7.1)**, the **Key Transparency Log (G4-R8/R9)**, the **published trust bundle + standalone offline verifier (§7.2/7.3)**, **external timestamp anchoring (§6.4)**, **dual-control rotation/revocation ceremonies (G4-R26)**, **the timeline-aware revocation model (§6)**, **PQC migration planning (§9.4)**, **HSM (OQ-G4-2 already defers to "cloud KMS at POC")**. The "the key leaked, what about 3 years of checkpoints?" runbook (§9) is moot when there are 30 days of checkpoints.
- **POC-minimum:** *One KMS-backed ed25519 key, non-exportable, `key_id` recorded, old public keys retained. Rotation = "generate a new one, keep the old public key." Don't lose the key; that's the whole POC requirement.*

> **Is the full KTL needed at POC, or just "don't lose the key"? Just don't lose the key.** G4-OQ-2/4/5 already default HSM, external anchoring, and air-gapped ceremonies to "GA." The KTL itself should join them.

### G5 — Multi-Tenancy Isolation

**The cleanest deferral in the entire wave, by the SPEC's own decision.** D-G5-1: **"The POC ships at T0 (single-tenant)."** Every requirement in G5 is, by its own scope, post-POC. G5 is an *excellent* design for the moment tenant #2 arrives — and zero of it should be built before then.

- **Genuinely required for the POC:** **nothing.** Single tenant = no tenant boundary to enforce.
- **Genuinely required before *tenant #2*:** the tenant registry (§5), RLS-under-interceptor (G5-R1), per-tenant key namespace for object storage (G5-R2), the analytics-read isolation (G5-R4, the XD-5 fix). That is the "first multi-tenant release" set — real, but not POC.
- **Scale gold-plating (defer even past tenant #2):** the full T2–T4 spectrum (namespace/cluster/instance-per-tenant), residency/regional pinning (Plane 4), per-tenant signing keys + crypto-shred-per-tenant, the six-plane model in full, the per-tier conformance gate.
- **POC-minimum:** *T0 single-tenant. Do not build a tenant boundary. Freeze the D2 `ScopePredicate` library **interface** (so it lights up later without a re-cut, per MASTER-PLAN-ALT §6) and stop.*

> **Is the per-tenant hash chain needed before tenant #2? No — it's not even needed *at* tenant #2 necessarily.** G5-R26 mandates a per-tenant chain (a C2 contract change) so erasure-without-collateral works. That is a real constraint, but it is a *tenant-#2* constraint, and arguably a *first-regulated-tenant* constraint. The POC ships one global chain.

### G6 — Observability, SLOs & Day-2 Operations

This is the most *legitimately* POC-relevant of G1–G6, because the persona ("Jess, the 2–4 person SRE team") is exactly the design-partner reality. But the SPEC scales it to a 14-service golden-signal matrix, multi-window burn-rate SLO error-budget policy, and a full day-2 runbook library including the C2-schema-epoch-migration — most of which is premature for a system that hasn't shipped a v1 yet.

- **Genuinely required:** **R-G6-OBS-1** (platform telemetry ≠ audit evidence — separate pipelines; cheap and structurally load-bearing, must be right from day one), **R-G6-OBS-4/5** (`governance-system` self-exemption + monitoring-outside-the-control-loop — the watchmen trap, same idea as G3's anti-self-brick), **R-G6-OBS-9** (chain-integrity monitor + drop-count-MUST-be-0 alert), and the **Surface-2 lossless-ingest SLO** (a dropped audit event is a compliance incident, not budget burn). The ≤2-pages/shift on-call discipline (R-G6-ONCALL-1) is a good cultural default.
- **Scale gold-plating (defer):** the full §2.3 golden-signal matrix across all 14 services (you have ~6 services at POC), the three-surface SLO + error-budget *policy* framework (§3, renegotiated with the design partner *anyway*, per R-G6-SLO-1), the full day-2 runbook library (§4.1 14-service choreography, §4.2 CRD migration, §4.4 cert/secret rotation game-days), the managed-vs-self-hosted packaging analysis (§6). **The flagship C2-epoch-migration runbook (§4.3) is premature**: you cannot have a "1.0→1.0-rc breaking schema migration of a multi-year append-only log" before you have a multi-year log; it's a year-2 problem dressed as a day-2 one.
- **POC-minimum:** *Telemetry pipeline separate from evidence; chain-integrity + zero-drop alerts; self-exempt + out-of-loop monitoring; one composite health page; ≤2 pages/shift. Defer the SLO-policy framework and the runbook library until there's a system to run.*

### G7 — Data Lifecycle, Retention & Privacy

This is the **inversion case** (see §3): a chunk of G7 is *more* urgent for a first design partner than most of G1–G4, and a chunk is gold-plating.

- **Genuinely required:** **the ingest-time encryption seam (§2.1, §5.3 crypto-shred via per-subject DEK).** This is the one privacy requirement that is **architecturally irreversible if deferred**: you cannot retrofit "encrypt `sub`/`email`/agent-prompt at ingest" onto an append-only hash-chained log that already has cleartext events — the chain is computed over the envelope, so the envelope has to be right *from the first event*. The moment a design partner's DPO/legal asks "can you delete a user's data?" — and for an AI-agent/`request_object`-prompt product (F4) that is week one — you need the crypto-shred answer to already be in the data model. **G7-R3** (agent prompts never stored as one opaque cleartext blob) is a breach-magnet fix that is cheap at ingest and impossible later.
- **Scale gold-plating (defer):** the full **DSR pipeline (§7)** with SLA timers and SoD gates, the **legal-hold machinery (§4.4)**, the **archival re-notarization / crypto-agility (§4.3, G7-R11)**, the **residency hand-off to G5 (§8)** (N/A single-tenant, single-region), the **7-tier data-class TTL table at production floors**, the derived-store erasure cascade verification (G7-R19) beyond "destroy the key." You need the *capability* (per-subject DEK) and the *labels*; you don't need the orchestrated DSR workflow until a real DSR arrives.
- **POC-minimum:** *Per-subject field-level encryption of PII-DIRECT fields (`sub`, `email`, agent prompt/conversation) at ingest, keyed so a single key-destroy = erasure; the `<erased>`/`<redacted>`/`never-captured` label distinction. Defer the DSR pipeline, legal hold, residency, re-notarization.*

### G8 — Rego-Authoring & Human Factors

**The most POC-relevant component in Domain G, and the one I'd fund first.** Unlike G1–G4 (which protect a hypothetical enterprise), G8 protects the *demo and design-partner #1 directly*: the MVP acceptance criteria (F3 AC-1, AC-2, AC-5) are *authoring* and *simulation* flows, and the THESIS reviewer independently named author-quality as load-bearing for the portability thesis.

- **Genuinely required:** **the inner loop (§5: Regal lint + `opa test` + local eval, identical in CI)** — this is table-stakes DX and the difference between a demo that works and "deploy-and-hope." **G8-MUST-030 / D-G8-3: no runtime policy reaches `enforce` un-simulated** — the single most common OPA mass-deny incident, and the gate that makes design-partner #1's first policy safe. **R-LINT-FAILOPEN / R-LINT-OVERBROAD** (catch the mass-deny footgun) and **R-LINT-CLAIMS** (replay-completeness — protects the actual differentiator). The paved-road template catalog (§4.3) so authors don't start from a blank file.
- **Scale gold-plating (defer):** the **full authoring-error-rate measurement program (§11, M1–M9 with gated targets)** — you measure error rate once you have a *population* of authors; at POC you have Marcus and maybe Sam. The **novice NS-author safe-by-construction regime (§7)** is real but is gated on multi-tenant NS-delegation (G5 T2+), so it defers with G5. The cross-engine conformance suite as a hard CI gate (G8-MUST-034) matters only once you ship to a *second* engine; at POC (Gatekeeper-centric) it's a SHOULD.
- **POC-minimum:** *Regal lint (fail-open + overbroad + claims rules) + `opa test` + local eval, same in CI; mandatory differential-sim-before-enforce; a handful of paved-road templates. Defer the M1–M9 measurement program and the novice-NS regime until there's a population and multi-tenancy.*

---

## 2. Where the NFR specs CONTRADICT the wedge-first thesis

The THESIS and ALT reviewers argued: **don't build the 23-component platform; build one wedge (W1 or W5) with ~4–5 components for a named buyer who exists today.** Several G-requirements only make sense if you've *already committed to platform-first*, which both reviewers rejected. These are the contradictions:

1. **G2's entire premise is a regulated enterprise at 50M events/day × 7 years.** The wedge buyers (W5 Styra-orphans managing *bundles*; W1 GRC-adjacent ingesting foreign events) are not that buyer. W5 *"omits B2/B5 full admission flow… the customer's existing Gatekeeper/Kyverno enforces"* (ALT §2-W5) — so G1's 100× admission scale model and G2's full-population storage cliff are pricing a deployment shape the lead wedge **doesn't have**. G2 is platform-first cost modeling.

2. **G5's T2–T4 hard-tenancy spectrum assumes a multi-tenant SaaS platform.** But W5 is *"enterprise single-tenant at first"* (ALT §2-W5) and W1 is *"ingestion-only, no in-cluster footprint"* (ALT §4). The wedge buyers are single-tenant. The six-plane isolation model, the tenant registry, residency pinning — all of it presupposes the multi-tenant platform product that wedge-first explicitly defers (D2 real-scope deferred "until W2," ALT §3). Building G5 now is building for the platform the reviewers said not to build yet.

3. **G3's active-active multi-region DR is sized for "a regulated buyer signs."** That buyer is the *exact fiction* the THESIS interrogates ("Name three design partners who will sign a paid pilot in 90 days"). G3-R-RPO-3 escalates active-active to MUST "for regulated/Stack-C deployments" — i.e., the requirement only fires for the buyer whose existence is the open question. If you can't name that buyer, you can't justify the DR program.

4. **G4's KTL + offline trust-root + offline verifier is sized for the auditor of a long-horizon regulated evidence store.** But the wedge that has an auditor-adjacent buyer (W4 Vanta-compatible evidence) *"omits"* most of the heavy machinery and pairs with Vanta's *existing* trust narrative. The "verify a 2019 export offline in 2026" property is a platform-maturity feature, not a wedge feature.

5. **G6's 14-service day-2 runbook library prices the operational burden of the 14-service stack** — which is precisely META Risk #10 and the THESIS's "you are building a cathedral to sell a doorknob." A wedge has ~4–6 services. The day-2 program is sized for the platform the cost-and-strategy reviewers both argued against.

**The pattern:** G2, G5(T2+), G3(P2/P3), G4(KTL+root), and G6(full day-2) are **platform-first NFRs**. They are correct *if* the company is building the 23-component platform for a regulated enterprise. The other two reviewers argued it should not — at least not first. **You cannot adopt wedge-first sequencing and platform-first NFRs simultaneously.** Right now the corpus does exactly that.

> One honest caveat: the G-wave was authored to *answer* META's "can it scale / what does it cost" — and that answer is genuinely valuable as **de-risking knowledge** (it proves the architecture *can* reach Scale-C). The error is treating "we proved we can build it" as "we should build it now." The G-specs are excellent **reference architecture**; they are premature **work orders**.

---

## 3. The inversion risk: which NFR is UNDER-specified / more urgent than the corpus implies

The corpus orders the G-wave G1→G8, implicitly front-loading scale/performance/cost/DR and trailing privacy/human-factors. **For a first design partner, that order is inverted.** The two trailing components are the urgent ones; the leading ones are the deferrable ones.

**G8 (human-factors) and G7 (privacy ingest seam) are MORE urgent than G3 active-active DR / G2 cost / G4 KTL for a first design partner — and both are comparatively *under-built* relative to their urgency.**

- **G8 is under-weighted by position, not by content.** It is the only G-component whose failure mode is *the POC demo failing* (a mass-deny during AC-2, a non-replayable event failing AC-3/AC-5, a novice shipping a broken policy). The THESIS reviewer — writing the *kill* case — singled out author quality as the thing the portability thesis "depends on." Yet G8 sits last in the wave and its hardest requirement (M9 author-escape-rate as a gated acceptance criterion) is the right idea applied prematurely while its *cheap* requirements (the inner loop, sim-before-enforce) are the actually-urgent ones. G8 needs **re-prioritizing to first**, with its measurement program deferred.

- **G7's ingest-encryption seam is the one deferral that is architecturally irreversible.** Everything else in G can be bolted on later (that's the whole point of additive contracts). But you **cannot** add "encrypt PII at ingest into the hash chain" after the chain has cleartext events — the envelope is in `content_hash`. So the *one* privacy decision that must be made before the first event is written is buried in a component positioned as the second-to-last NFR concern. If a design partner is an AI-agent product (F4 — the market with regulatory tailwind, per THESIS §3.1), the first `request_object` prompt is a PII breach magnet on day one. **G7's §2.1/§5.3 seam is more urgent than all of G2, all of G4's KTL, and all of G3's multi-region.**

- **A genuine under-specification, not just mis-ordering:** the corpus has no component answering *"what is the minimum NFR floor for the wedge products W1/W5/W4 specifically?"* The G-specs all size to the full platform and gesture at "POC defaults" in passing. Given the company's stated lean toward wedge-first, the **missing NFR document is a per-wedge NFR floor** — and its absence means each G-spec defaults to over-building. That is the real inversion: the corpus over-specifies the enterprise NFR and under-specifies the wedge NFR.

---

## 4. The blunt POC NFR cut-line

The 5–8 NFR requirements that genuinely gate a first design partner (everything else → "when a customer pays for it"):

| # | Requirement | Source | Why it gates the design partner |
|---|---|---|---|
| 1 | Committed evidence survives single-node/disk loss + daily WORM backup that restores-and-verifies | G3 §3.1 r1, R-G3-BK-2 | If you lose the audit log, the product's one claim is a lie. Cheap, single-region. |
| 2 | KMS-backed non-exportable signing key + `key_id` + retain old public keys | G4-R4, R7 | Don't let cluster-admin forge the chain; keep rotation *possible*. Floor only — no KTL/root/HSM. |
| 3 | Mass-deny circuit breaker + anti-self-brick carve-out | G3 §4, R-G3-CB-9; G6-OBS-4 | A shared-dep hiccup mass-denies the partner's fleet without it. Hit in week one. |
| 4 | Admission p99≤1s + external-data timeout-inside-deadline + cache | G1 N-G1-100/102/103 | Hot-path correctness at any scale; a slow extdata call on fail-closed = outage. |
| 5 | Per-subject PII encryption at ingest + crypto-shred erasure | G7 §2.1, §5.3 (D-G7-01/02) | Architecturally irreversible; gated by the partner's legal team on day one. |
| 6 | Inner-loop DX + mandatory sim-before-enforce + fail-open/overbroad/claims lint | G8 §5, MUST-030, R-LINT-* | Makes the demo work and stops the partner mass-denying themselves. |
| 7 | Telemetry≠evidence pipeline + chain-integrity + zero-drop alert + self-exempt monitoring | G6-OBS-1/4/5/9, Surface-2 SLO | The "watchmen" floor; cheap and structurally must-be-right-early. |
| 8 | Single-tenant (T0); freeze the D2 ScopePredicate *interface* only | G5 D-G5-1 | Build no tenant boundary; keep the seam so tenant #2 doesn't force a re-cut. |

**Everything else is deferred to "when a customer pays for it":** all of G2; G1's sharded chains + 10×/100× capacity model + scale load-tests; G3's active-active/multi-region/split-brain/full-FMEA/game-day program; G4's KTL + offline trust-root + offline verifier + timestamp anchoring + dual-control + PQC; all of G5 T1–T4 + registry + residency + per-tenant chains/keys; G6's full golden-signal matrix + SLO-policy framework + day-2 runbook library + epoch-migration runbook; G7's DSR pipeline + legal-hold + residency + re-notarization; G8's M1–M9 measurement program + novice-NS regime + cross-engine conformance gate.

**Two irreversibility flags (the only deferrals that need a seam reserved now, not built):** (a) **G7 ingest-encryption** — the envelope must be in `content_hash` from event #1, so the *data-model seam* ships at POC even though the DSR machinery doesn't; (b) **G5 per-tenant chain identity + D2 ScopePredicate interface** — freeze the *interface/shard-key* now so tenant #2 is additive, per MASTER-PLAN-ALT §6's "freeze contracts early even in wedge-first." Build neither; reserve both seams.

---

## 5. Honest call: right-sized, over-built, or under-built — and the one to fund first

**Over-built for the POC and the first design partner; correct as reference architecture; mis-sequenced; and under-built in exactly one corner (the privacy ingest seam + author DX, which sit last but should sit first).**

- Domain G is a **high-quality answer to the wrong question.** META asked "can it scale and what does it cost"; the G-wave answered thoroughly and well. But "can it" ≠ "should we, now." Against F3 §22 ("functional validation, not throughput") and the wedge-first thesis (a 4–5-component product for a named buyer), ~70% of G1–G5's MUSTs are gated on a Scale-C enterprise that is, per the THESIS, an unproven buyer. The right disposition is: **keep the G-specs as the de-risking reference (they prove the path to scale exists), extract the ~8-requirement POC floor above, and defer the rest behind the additive contracts the corpus already mandates.**

- It is **not wrong, and not waste** — the contract-freezing discipline (per MASTER-PLAN-ALT §6) means the deferred NFR work bolts on additively. The mistake would be *building* it now, not *having designed* it.

- **The one NFR I would fund first: G8 (Rego-authoring / human-factors).** Three reasons: (1) it is the only G-component whose failure mode is the *POC demo itself* failing (AC-1/AC-2/AC-5 are authoring + simulation flows); (2) the THESIS reviewer — writing the *kill* case — independently named author quality as load-bearing for the platform's two thesis claims (portability and replay-completeness), so it's the rare NFR both the build and the kill camp agree matters; (3) its high-value requirements (inner loop, sim-before-enforce, fail-open lint) are *cheap* and its expensive requirement (the M9 measurement program) is cleanly deferrable. Funding G8 first buys a working demo and a safe design partner for the least money.

  **Immediately behind it, two seams that must ship at POC even though their machinery defers:** the **G3 mass-deny circuit breaker** (prevents the design partner's first self-inflicted outage, cheap) and the **G7 per-subject ingest-encryption seam** (architecturally irreversible; the only deferral that, if you get it wrong, you cannot fix without re-cutting the evidence log).

**The provocation, stated plainly:** the G-wave built the NFRs for the enterprise the company is being told (by two other reviewers) it may never reach, and under-prioritized the NFRs for the design partner it needs next quarter. Fund the demo and the design partner — G8, the circuit breaker, the encryption seam — and let the petabyte-scale, 7-year, active-active, multi-tenant NFR program wait for the customer who pays for it.

---

*End NFR-DEVILS-ADVOCATE. This is the scope-control opposing tree; the affirmative case is the 8 G-SPECs themselves. The decision belongs to the reader, now argued on both sides.*
