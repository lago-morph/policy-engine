# G2 — Cost & Retention Economics — SPEC

**Component ID:** G2 · **Domain:** G — Operational / Non-Functional Architecture
**Spec sources:** §22 (POC scale), §24–25 (deployment stack = compute driver), and the C2 audit schema (event size + retention/integrity = storage driver).
**Companion docs:** `cross-cutting/META-ADVERSARIAL-SECOND-OPINION.md` (NFR gap #2 "no production-scale or cost model exists"; "audit-log retention economics undefined"), `cross-cutting/THESIS-DEVILS-ADVOCATE.md` (build-vs-buy economics — this model is the cost input to that debate), `components/C2-audit-schema/SPEC.md`, `components/F2-deployment-extensibility/SPEC.md`, `policy engine market research.md`.
**Status:** Authored (NFR wave, G-domain). **Date:** 2026-05-30.
**One-line purpose:** A concrete, parameterized **cost model** for running and retaining the platform's evidence, so build-vs-buy and "is full-population audit affordable at scale?" can be answered with numbers, not adjectives.

---

## 0. Why this component exists

The cross-cutting review's blunt verdict is that the corpus is "a *functional* architecture with no *operational* architecture," and names **two** cost holes by name (META §3, §4 Risk 2):

1. *"No production-scale or cost model exists"* — the signed/chained/CAS evidence store and the ~14-service stack are unquantified at 1000× the POC.
2. *"Audit-log retention economics undefined"* — multi-year regulated retention of raw external-data values + before/after state + request objects, content-addressed and signed, is "a large and unmodeled cost — and XD-1's resolution *increases* it by making value capture a MUST. **No owner, no cost model.**"

The devil's-advocate thesis turns this into a business-case attack: the platform's *core differentiator* is **full-population, replay-capable audit** (every decision preserved with enough input to re-run it). G2's job is to find the price tag on that differentiator and ask whether it survives contact with a 7-year regulated retention regime and a 50M-events/day enterprise — **before** contracts are signed.

This SPEC does three things: (A) states explicit unit-cost assumptions; (B) gives formulas parameterized by events/day, retention, clusters, tenants; (C) computes worked TCO at three scales and a build-vs-buy comparison.

---

## 1. Scope

### 1.1 In scope
- The **itemized monthly cost-driver model**: compute (the ~14-service stack), audit storage across hot/warm/cold tiers, egress, KMS/signing operations, managed dependencies (Postgres, object store, optional search index).
- **Formulas** parameterized by `E` (events/day), `R` (retention days), `K` (clusters/spokes), `T` (tenants), `S` (avg event size), with stated tiering policy.
- **Three worked TCOs**: POC (single team), Mid (5 clusters / 500k events-day / 1-yr retention), Regulated (7-yr retention, multi-region).
- **Storage-tiering economics** (hot→warm→cold→delete) and the **tamper-evidence/signing overhead** cost.
- A **build-vs-buy cost comparison** vs adopting OPA Control Plane (OCP) + OCSF + Vanta for the same outcome, including the eng-years to build 23 components.
- A short **cheaper-architecture** section (§9).

### 1.2 Out of scope (delegated)
- *Throughput/latency engineering* and the serial-E1 scale ceiling → **G1 (scale & performance)**. G2 prices what G1 sizes.
- *DR/backup RPO·RTO* mechanics → **G3**; G2 prices the backup/replica storage G3 mandates (§4.6).
- *KMS/HSM key-lifecycle design* → **G4**; G2 prices the KMS ops G4 specifies (§4.4).
- *Per-tenant isolation mechanics* → **G5**; G2 supplies the cost-per-tenant attribution model (§6, PLAN).
- The **business/market decision** (build vs buy, wedge vs platform) is owned by `THESIS-DEVILS-ADVOCATE.md` and `MASTER-PLAN-ALT.md`; G2 supplies the cost half of that decision only.

### 1.3 Design tenets
- **List-price, single-region, on-demand, no committed-use discount** is the default quoted figure, so numbers are an honest *ceiling*; committed-use / reserved discounts (≈30–55%) are shown as a separate line, never baked silently into the headline.
- **Storage is the cost that compounds; compute is the cost that's roughly flat.** The model must make that visible (it is the whole point — §8 cost cliff).
- **Every figure carries its assumption inline.** A number without its unit-cost basis is forbidden (the same honesty tenet C2 applies to evidence, applied to dollars).

---

## 2. Unit-cost assumptions (the price book)

All prices are **AWS us-east-1 on-demand list, mid-2026**, rounded to the stated precision, chosen because they are public and conservative (GCP/Azure are within ±15%). Stated so every downstream number is reproducible; substitute your own price book and the formulas hold.

### 2.1 Compute (per vCPU-month, fully-loaded)
| Resource | List basis | Assumed unit |
|---|---|---|
| EKS worker vCPU (on-demand `m6i`-class, ~$0.0462/vCPU-hr) | $0.0462 × 730 hr | **$33.7 / vCPU-month** |
| Memory (GB-month, `m6i` ratio 4 GB/vCPU, priced into above) | bundled | counted via vCPU |
| EKS control-plane fee | $0.10/hr/cluster | **$73 / cluster-month** |
| Managed Postgres (RDS `db.m6i`, ~$0.18/vCPU-hr incl. storage overhead) | per vCPU | **$140 / vCPU-month** (Multi-AZ ≈ 2×) |

A "service replica" in this model = **0.5–1.0 vCPU + 1–2 GB** unless noted (most of the ~14 services are light; the heavy ones — E1 simulation workers, C3 analytics, Keycloak, Postgres — are called out).

### 2.2 Storage (per GB-month)
| Tier | AWS product | List unit |
|---|---|---|
| **Hot** (queryable, indexed) | gp3 EBS / RDS storage / OpenSearch hot | **$0.125 / GB-month** (EBS gp3 $0.08 + index/replica overhead) |
| **Warm** (object, standard) | S3 Standard | **$0.023 / GB-month** |
| **Warm-infrequent** | S3 Standard-IA | **$0.0125 / GB-month** (+ $0.01/GB retrieval) |
| **Cold** | S3 Glacier Flexible Retrieval | **$0.0036 / GB-month** (+ retrieval fee, hours latency) |
| **Deep cold** | S3 Glacier Deep Archive | **$0.00099 / GB-month** (+ $0.02/GB retrieval, 12-hr latency) |

### 2.3 Egress, requests, KMS
| Item | List unit |
|---|---|
| Internet egress (first 10 TB/mo) | **$0.09 / GB** |
| Cross-region replication transfer | **$0.02 / GB** |
| S3 PUT/POST (per 1k) | **$0.005 / 1k** |
| S3 GET (per 1k) | **$0.0004 / 1k** |
| KMS API request (sign/verify/encrypt) | **$0.03 / 10k requests** |
| KMS key (CMK) | **$1.00 / key-month** |
| CloudHSM (if HSM-backed signing, per G4) | **$1.45 / hr ≈ $1,058 / month** per HSM (2 for HA ≈ $2,116) |

### 2.4 Event-size assumptions (the storage multiplier)
From C2 §3 (36 fields) and F2 R-F2-SCALE-3 ("~2–5 KB/event"):

| Event profile | What it carries | Assumed stored size |
|---|---|---|
| **Lean** | decision + scope + subject, no large bodies | **2 KB** |
| **Typical** (model default) | + `request_object`, `jwt_claims`, `external_data_refs` (refs only) | **5 KB** |
| **Heavy / replay-complete** | + `before_state`/`after_state` + captured external-data **values** (XD-1 MUST) | **15 KB** (event 5 KB + CAS blobs ~10 KB amortized) |

> **Key driver:** XD-1's resolution (capture external-data *values*, not just refs) and `before_state`/`after_state` push the regulated profile from 5 KB toward **15 KB/event**. The "honest replay" differentiator is a **3× storage multiplier** over a lean SIEM event. This single fact dominates the regulated TCO (§8).

### 2.5 Compression & dedup
- Cold/warm tiers store canonicalized JSON; **gzip ≈ 6–8×** on C2 events (highly repetitive structure). Model assumes **6× compression** on warm+cold (so 15 KB → ~2.5 KB on disk), **none** on hot (indexed/queryable).
- CAS blobs (`before_state`/`after_state`/external-data values) **dedup** strongly (same image-signature-status response shared across thousands of events): assume **5× dedup** on the blob portion. Both factors are applied in §3 formulas and are the main reason cold retention is survivable.

---

## 3. The cost model (formulas)

Parameters: `E` = events/day, `R` = retention days, `K` = clusters/spokes, `T` = tenants, `S` = avg raw event size (KB), `Hd` = hot-window days, `Wd` = warm days, `Cd` = cold days (`Hd+Wd+Cd = R`).

### 3.1 Storage volume
```
Raw ingested/day        Vday   = E × S            (KB/day)
Hot stored (indexed)    Vhot   = E × S × Hd                      (no compression, +1.0× index overhead already in $0.125)
Warm stored (compressed) Vwarm = E × S × Wd / 6
Cold stored (compressed) Vcold = E × S × Cd / 6
Plus CAS blob store (heavy profile only):
   Vblob = E × Bsize × R / (6 × 5)   where Bsize ≈ 10 KB amortized
```

### 3.2 Storage cost / month
```
$hot   = (Vhot  in GB) × $0.125
$warm  = (Vwarm in GB) × $0.0125      (Standard-IA)
$cold  = (Vcold in GB) × $0.00099     (Deep Archive) + retrieval reserve
$blob  = (Vblob in GB) × blended($0.023 hot-ish, $0.0036 cold)
$storage = $hot + $warm + $cold + $blob
```
(1 GB = 1e6 KB here for round numbers; the spreadsheet uses 2^30 — within rounding.)

### 3.3 Tamper-evidence / signing overhead
```
Checkpoints/day        = E / Ncadence     (C2 §7.4 default N=10,000)  + time-based (96/day at 15-min)
$kms_sign  = (checkpoints/day × 30) / 10,000 × $0.03      (ed25519 sign per checkpoint)
$kms_verify= (verify ops: chain audits + auditor exports) × $0.03/10k
$kms_keys  = #CMKs × $1.00         (+ CloudHSM $2,116 if G4 mandates HSM)
$hash_cpu  = negligible compute (sha256 per event) — folded into ingest worker sizing, NOT a separate line
$signing = $kms_sign + $kms_verify + $kms_keys (+ $hsm)
```
> **Finding (counter-intuitive):** at checkpoint cadence (1 signature / 10k events), **KMS signing is nearly free** — even 50M events/day = ~5,000 checkpoints/day = $0.45/mo in sign ops. The expensive part of "tamper-evidence" is **not the crypto, it's the storage**: append-only + raw-retention + CAS forbids deletion/overwrite, so the integrity model is what *forces* the 3× heavy-profile volume and the long retention. **Signing overhead ≈ $1–$2,200/mo (HSM-dominated); storage overhead from the integrity model ≈ the entire storage bill.** The cost of tamper-evidence is a retention cost, not a compute cost.

### 3.4 Compute cost / month
The ~14-service control plane (F2 §2.2). Per-replica vCPU at POC HA (≥2 replicas) → production:

| Service (F2) | POC vCPU (HA) | Mid (5-cluster) vCPU | Driver |
|---|---|---|---|
| Governance API (F1) | 2×0.5 = 1 | 3×1 = 3 | QPS / GUI users |
| Lineage/governance (A1/A2) | 1 | 2 | metadata, flat |
| **Simulation E1 + workers** | 2 + 2 = 4 | 2 + 8 = 10 | **bursty replay; scales with dataset (G1)** |
| **Analytics C3** | 2 | 6 | streams all events |
| Audit store iface C2 | 1 | 4 | ingest workers ∝ E |
| Authz/identity D1/D2 broker | 1 | 2 | |
| **Keycloak** (if in-cluster) | 2 | 4 | IdP |
| Console/E2 backend | 1 | 2 | |
| Operator + 9 CRD controllers | 2 | 2 | flat |
| Privateer | 2 | 3 | |
| **Postgres (index/read-model)** | 2 (Multi-AZ) | 8 (Multi-AZ) | ∝ hot index + queries |
| **Per-spoke** (Gatekeeper+OPA+shippers) | 1.5 × K | 1.5 × K | ∝ clusters; runs in customer's existing cluster |
| **Total control-plane vCPU** | ~21 + 1.5K | ~50 + 1.5K | |

```
$compute = (control-plane vCPU) × $33.7
         + (Postgres vCPU) × ($140 − $33.7 already counted ⇒ add $106 premium)
         + K × $73                                    (EKS control-plane/cluster)
```
Spoke enforcement (Gatekeeper/OPA) runs **inside the customer's existing clusters** — its compute is arguably *not* incremental platform cost (the customer already pays for those nodes). Model counts it separately and lets the reader include/exclude it.

### 3.5 Egress
```
$egress = (events exported to SIEM/auditor + cross-region replication) × unit
        ≈ (E × S × export_fraction × $0.09/GB)  + (cross-region: Vday × $0.02/GB if multi-region)
```
Egress is small unless the customer streams 100% to an external SIEM (then it's a real line — §8 note).

### 3.6 Total
```
$monthly = $storage + $signing + $compute + $egress + $managed_deps
$TCO_annual = 12 × $monthly  + amortized build cost (build-vs-buy, §7)
```

---

## 4. Worked TCO at three scales

> Convention: figures are **list-price, on-demand, single-region** unless the scenario says multi-region. Committed-use discount line shown separately. Spoke-enforcement compute **excluded** from headline (it lives in the customer's clusters) and shown as a memo line. Rounding to 2 sig figs.

### 4.1 Scale A — POC (single team)
`E = 100k/day` (mid of §22's 10k–500k), `S = 5 KB typical`, `R = 30 days` (all hot), `K = 1`, `T = 1`, non-HA-to-light-HA.

| Driver | Calc | $/month |
|---|---|---|
| Hot storage | 100k × 5KB × 30d = 15 GB × $0.125 | **$2** |
| Signing (KMS) | ~96 checkpoints/day, 1 CMK | **$1** |
| Compute (control plane ~21 vCPU) | 21 × $33.7 | **$710** |
| Postgres premium | 2 vCPU × $106 | **$210** |
| EKS cluster fee | 1 × $73 | **$73** |
| Egress (light) | minimal | **$10** |
| **Total** | | **≈ $1,000 / month** |
| Memo: spoke enforcement (in customer cluster) | 1.5 vCPU | +$50 |
| **POC annual TCO** | | **≈ $12k/yr** (infra only) |

**The POC is compute-bound, not storage-bound.** Storage is $2/mo — a rounding error. The platform's real POC cost is the **idle cost of running ~14 services HA** (~$1k/mo), i.e. you pay for the stack's *complexity*, not its data. This is the cross-cutting review's "day-2 operational burden of a 14-service stack" (Risk 10) showing up as dollars.

### 4.2 Scale B — Mid (5 clusters / 500k events-day / 1-yr retention)
`E = 500k/day`, `S = 5 KB typical`, `R = 365 days` (`Hd=30, Wd=90, Cd=245`), `K = 5`, `T ≈ 10`, full HA.

Volumes: 500k × 5KB = **2.5 GB/day raw**.
- Hot (30d): 75 GB × $0.125 = **$9.4**
- Warm (90d, /6): 500k×5KB×90/6 = 37.5 GB × $0.0125 = **$0.47**
- Cold (245d, /6, Deep Archive): 102 GB × $0.00099 = **$0.10**
- Total **storage ≈ $10/month** (!).

| Driver | $/month |
|---|---|
| Storage (hot+warm+cold) | **$10** |
| Signing (5×96 checkpoints/day ~ still trivial; HSM **not** assumed) | **$5** |
| Compute (control plane ~50 vCPU) | 50 × $33.7 = **$1,685** |
| Postgres premium (8 vCPU Multi-AZ) | 8 × $106 = **$850** |
| EKS cluster fees | 5 × $73 = **$365** |
| Egress (modest SIEM forwarding 20%) | 500k×5KB×0.2×30/1e6 ×$0.09 ≈ 15GB | **$1.4** |
| **Total** | **≈ $2,900 / month ≈ $35k/yr** |
| With 40% committed-use discount on compute | | **≈ $24k/yr** |
| Memo: 5 spokes' enforcement (customer clusters) | +$370/mo |

**Even at 500k/day for a full year, storage is $10/mo and the bill is still ~95% compute.** §22's POC volumes are simply too small for storage economics to bite. The retention cliff only appears when **both** volume **and** retention period **and** the heavy/replay-complete profile rise together — which is Scale C.

### 4.3 Scale C — Regulated vertical (7-yr retention, multi-region, production volume)
This is the scenario META Risk 2 demands: *"Project storage + signing + replay cost at 50M events/day for 7-year retention."* We model the regulated buyer (financial services, SOC2 + SEC 17a-4 + 7-yr WORM).

`E = 50,000,000/day`, `S = 15 KB heavy/replay-complete` (XD-1 value capture MUST + before/after state), `R = 2,555 days (7 yr)` (`Hd=30, Wd=335, Cd=2,190`), `K = 20`, `T = 50`, **2 regions** (active + DR), WORM/Object-Lock.

Raw volume: 50M × 15KB = **750 GB/day = 274 TB/yr raw**.

**Storage (per region; ×~1.7 for 2-region with cross-region cold replication):**
| Tier | Volume | Unit | $/month |
|---|---|---|---|
| Hot (30d, indexed, no compress) | 50M×15KB×30 = **22.5 TB** | $0.125 | **$2,810** |
| Warm (335d, /6, Standard-IA) | 50M×15KB×335/6 = **419 TB** | $0.0125 | **$5,230** |
| Cold (2,190d ≈ 6yr, /6, Deep Archive) | 50M×15KB×2190/6 = **2,738 TB** | $0.00099 | **$2,710** |
| CAS blob store (dedup 5×, blended) | ~10KB×50M×2555/(6×5) = **426 TB** | ~$0.005 blended | **$2,130** |
| **Storage subtotal (1 region)** | **~3.6 PB** | | **≈ $12,900/month** |
| **× 2 regions (DR cold replica + cross-region transfer)** | | | **≈ $22,000/month** |

**Signing / KMS:**
- 50M/day ÷ 10k = 5,000 checkpoints/day × 30 = 150k sign ops/mo → **$0.45**.
- Auditor verify + continuous chain-verification reads: ~10M KMS verify/mo → **$30**.
- HSM-backed signing (G4 likely mandates for regulated) 2× CloudHSM → **$2,116**.
- **Signing subtotal ≈ $2,200/month** (HSM-dominated; the crypto itself is $30).

**Compute:**
| Driver | $/month |
|---|---|
| Control plane (production ~120 vCPU: C3 analytics + C2 ingest scale with 50M/day; E1 replay workers heavy) | 120 × $33.7 = **$4,040** |
| Postgres/read-model (Multi-AZ, large, ~32 vCPU + 22TB hot index storage already in hot line) | 32 × $106 = **$3,390** |
| 20 EKS cluster fees | 20 × $73 = **$1,460** |
| Multi-region active control-plane duplication (~0.6×) | **$5,300** |
| **Compute subtotal** | **≈ $14,200/month** |

**Egress:** if 100% streamed to a central SIEM: 50M×15KB×30/1e6 = 22.5 TB/mo × $0.09 = **$2,025/mo**. Cross-region replication of cold tier: ~$0.02/GB on 750GB/day×30 = 22.5TB = **$450/mo**. **Egress ≈ $2,500/month** (and a real line here, unlike A/B).

| **Scale C total** | **$/month** | **$/year** |
|---|---|---|
| Storage (2-region) | $22,000 | $264k |
| Signing (HSM) | $2,200 | $26k |
| Compute (2-region) | $14,200 | $170k |
| Egress | $2,500 | $30k |
| **Total infra** | **≈ $41,000/month** | **≈ $490k/yr** |
| With 50% committed-use on compute+storage | | **≈ $330k/yr** |

**Now storage dominates** (54% of the bill) and it is *structurally* driven by the three multipliers that are the platform's differentiators: (1) 7-yr retention, (2) 15 KB heavy/replay-complete events (XD-1 value capture), (3) full-population capture (every one of 50M/day). Drop any one and the bill collapses (§8, §9).

### 4.4 Summary table — the three scales

| | POC (A) | Mid (B) | Regulated (C) |
|---|---|---|---|
| Events/day | 100k | 500k | 50M |
| Event size | 5 KB | 5 KB | 15 KB |
| Retention | 30 d | 1 yr | 7 yr |
| Clusters / Tenants | 1 / 1 | 5 / 10 | 20 / 50 |
| Regions | 1 | 1 | 2 |
| Raw retained volume | 15 GB | ~190 GB effective | **~3.6 PB/region** |
| **Storage $/mo** | $2 | $10 | **$22,000** |
| Compute $/mo | $1,000 | $2,900 | $14,200 |
| Signing $/mo | $1 | $5 | $2,200 |
| Egress $/mo | $10 | $1.4 | $2,500 |
| **Total $/mo (list)** | **~$1,000** | **~$2,900** | **~$41,000** |
| **Total $/yr (list)** | **~$12k** | **~$35k** | **~$490k** |
| **Total $/yr (committed)** | ~$10k | ~$24k | ~$330k |
| Dominant driver | **compute** | **compute** | **storage** |

---

## 5. Storage-tiering economics (hot→warm→cold→delete)

The lifecycle policy is what keeps Scale C survivable. Without tiering, holding 7 years on hot ($0.125/GB) instead of Deep Archive ($0.00099/GB) is a **126×** difference on the cold portion — the difference between a $490k/yr product and a multi-million one.

### 5.1 Lifecycle policy (normative, R-G2-TIER)
- **R-G2-TIER-1 (MUST):** Hot tier holds the **active replay/analytics window** only (default 30 d, configurable; this is the window E1 replays and C3 streams over). Hot is indexed and queryable.
- **R-G2-TIER-2 (MUST):** At hot-expiry, events transition to **warm object storage** (Standard-IA), compressed (§2.5), still individually retrievable by digest with seconds–minutes latency. Index retained as a manifest, not a hot DB row.
- **R-G2-TIER-3 (SHOULD):** At warm-expiry (default 1 yr total), events transition to **cold/Deep Archive** with WORM/Object-Lock for the regulated regime (SEC 17a-4 / FINRA require non-rewriteable, non-erasable storage — Object Lock in Compliance mode satisfies this).
- **R-G2-TIER-4 (MUST):** Deletion only at retention-expiry, as a **signed tombstone** event (C2 N-C2-400), so the deletion is itself auditable. Tombstones are tiny and retained beyond the data.
- **R-G2-TIER-5 (MUST):** The **hash chain and checkpoint signatures must survive tiering** — moving an event to cold MUST NOT break verifiability. Checkpoints and the chain manifest stay in warm/hot (they are tiny) even when the payloads are in Deep Archive, so an auditor can prove the chain without thawing petabytes. **This is the load-bearing tiering invariant** (see ADVERSARIAL — restoring 7 yr to verify is the cost cliff).

### 5.2 Retrieval-cost trap
Deep Archive is cheap to *hold* and expensive to *read*: retrieval is **$0.02/GB + 12-hr latency**. Thawing all 2.7 PB of Scale C cold tier for a full re-verification = 2.7M GB × $0.02 = **$54,000 one-time** plus standard egress if it leaves the region. **An auditor demand to "re-verify the entire 7-year chain" is a $54k–$300k event**, not a $/month line. Mitigation (R-G2-TIER-5): keep the *chain + checkpoints* hot so verification reads metadata, not payloads; thaw payloads only for the specific events under dispute. This converts a PB-scale thaw into a GB-scale one. **If the architecture cannot verify the chain without rehydrating payloads, Scale C has a five-figure-per-audit hidden cost** — flagged as the #2 cliff in ADVERSARIAL.

### 5.3 Tiering decision table
| Retention regime | Hot | Warm | Cold | Effective $/GB-month blended |
|---|---|---|---|---|
| POC (30d) | 30d | — | — | $0.125 |
| Mid (1yr) | 30d | 90d | 245d | ~$0.005 |
| Regulated (7yr) | 30d | 335d | 2,190d | **~$0.0018** (94% in Deep Archive) |

The blended cost per GB **falls 70×** from POC to Regulated *because* the long tail is almost entirely Deep Archive. Tiering is not an optimization here; it is the thing that makes 7-yr full-population audit financially possible at all.

---

## 6. Cost attribution (cost-per-tenant / cost-per-control)

(Mechanics in PLAN; the model here.) Because storage is the compounding driver and it is naturally `scope`-tagged (C2 `scope.{cluster,namespace,tenant}` is on every event), **cost-per-tenant is directly measurable**: sum stored bytes (× tier price) and event counts per `scope.tenant`. Compute is shared and allocated by **event-share** (a tenant emitting 40% of events bears ~40% of C2-ingest/C3-analytics compute) plus a flat **platform-overhead allocation** (operator, Keycloak, API base) split evenly or by seat. Cost-per-control rolls up the same bytes/compute by `control_id`. This makes **showback/chargeback** a query over the evidence store the platform already maintains — a rare case where the product instruments its own cost for free.

---

## 7. Build-vs-buy cost comparison

The devil's-advocate thesis says the defensible novelty is ~2 components and the other ~20 are re-implementations of free/funded projects. G2 prices both sides.

### 7.1 Build cost (the 23-component platform)
| Item | Assumption | Cost |
|---|---|---|
| Engineering to GA | 23 components; MASTER-PLAN ~14 MVP + 9 later. Blended fully-loaded senior eng = **$250k/yr**. MVP est. **18–25 eng-years** (operator+CRDs, C2 store+integrity, E1 differential engine, C3/C4/C5, D1/D2, E2 console, F-stack). | **$4.5M–$6.3M to MVP** |
| Full 23-component GA | +8–12 eng-years | **+$2M–$3M** |
| Ongoing maintenance | ~30% of build/yr (chasing OPA/Kyverno/Headlamp/OCSF upstream breaks — META G-4, devil's-advocate §2) | **~$1.5M/yr** |
| Run cost (infra) | from §4: $12k (POC) → $490k/yr (regulated, per deployment) | **per-deployment, above** |
| **3-yr build TCO** (MVP + 2yr maint, single regulated deployment) | | **≈ $6.3M build + $3M maint + $1.5M run = ~$10.8M** |

### 7.2 Buy/adopt cost (same outcome, assembled)
| Outcome layer | Adopt | List cost |
|---|---|---|
| Policy control plane / lifecycle / bundles / OPA-replay | **OPA Control Plane (OCP)** | **$0 license** (CNCF, free; was $50k+/yr as Styra DAS). Self-host run cost ~$30–80k/yr. |
| Audit event schema / SIEM normalization | **OCSF** (+ Security Lake) | **$0 schema**; Security Lake storage ≈ the same S3 economics as §4 (no premium). |
| Continuous compliance evidence / GRC / auditor-facing | **Vanta** (or Drata) | **~$25k–$75k/yr** typical mid-market; enterprise $100k+. (15k customers; list not public, these are market-observed bands.) |
| CNAPP / posture (if needed) | **Wiz / Prisma** | **$100k–$500k/yr** enterprise (orthogonal; only if posture is in scope). |
| Glue / integration eng to wire OCP+OCSF+Vanta to the org | bespoke | **2–4 eng-years one-time ≈ $0.5M–$1M**, ~0.5 eng-year/yr maint. |
| **3-yr buy TCO** (OCP self-host + Vanta + glue, no CNAPP) | | **≈ $1M glue + $0.15M/yr OCP run + $0.15M/yr Vanta + $0.4M maint ≈ $2.3M** |

### 7.3 The comparison
| | Build (23-component) | Buy/adopt (OCP+OCSF+Vanta) |
|---|---|---|
| 3-yr TCO (one regulated deployment) | **~$10.8M** | **~$2.3M** |
| Upfront capital at risk before first dollar | $4.5M–$6.3M | <$1M |
| Get the **5 novel differentiators** (differential cross-engine sim, lineage graph, replay schema, PDP catalog, approval-mesh)? | **Yes** | **No** (OCP gives OPA-only replay; no cross-engine diff, no lineage graph, no governance-traceable replay schema) |
| Recurring exposure to upstream churn | High (own all 14 services) | Lower (foundations carry it) |

**The honest cost reading:** buying is **~4.7× cheaper over 3 years** for the *commodity* outcome. The ~$8.5M delta is the price of the **2 components of genuine novelty** the devil's-advocate identifies. That reframes the build decision exactly as the thesis does: *don't build 23 to get 2.* G2's contribution is the multiplier — **the differentiators cost ~$8.5M of build premium over assembling them from free/funded parts**, and the business case has to clear that bar. The cheapest path to the differentiators (§9) is to build only those 2 *on top of* the bought 20, turning $10.8M into roughly **$2.3M (buy) + $1.5M (build the 2 novel) ≈ $3.8M** — saving ~$7M while keeping the moat.

---

## 8. The biggest cost cliff (decision summary)

**The cost cliff is the joint product of {retention period} × {per-event size} × {full-population capture}, and it is invisible at POC/Mid scale and dominant at regulated scale.**

- Storage is **$2/mo at POC, $10/mo at Mid, $22,000/mo at Regulated** — a **>2,000× jump** that is *not* linear in volume; it is super-linear because the regulated case simultaneously raises (a) retention 85× (30d→7yr), (b) event size 3× (5KB→15KB via XD-1 value capture), and (c) volume 100× (500k→50M/day). 85 × 3 × 100 ≈ the 25,000× hot-equivalent blowup, which tiering compresses back down to ~2,200×.
- **The single sharpest edge:** XD-1's "capture external-data *values*" MUST, combined with 7-yr retention, is what turns a cheap SIEM-event product into a PB-scale one. The differentiator (authoritative replay) **is** the cost cliff. They are the same decision.
- **The hidden five-figure cliff:** a full 7-yr chain re-verification thaws ~2.7 PB from Deep Archive = **~$54k one-time per full audit** unless R-G2-TIER-5 keeps the chain+checkpoints hot. This is the cost most likely to surprise a regulated buyer mid-contract.

**Decision (decide-document-continue):** G2 ratifies the C2 tiering contract and the heavy-profile event size, and **flags to F3/MASTER-PLAN that full-population × 7-yr × value-capture is affordable only with aggressive tiering (R-G2-TIER) + chain-metadata-stays-hot (R-G2-TIER-5)**. At a typical regulated deployment of ~$330k–$490k/yr infra, the model is affordable *for an enterprise buyer* — but it is **not** affordable to offer as a flat-rate SaaS tier without metering storage, which is why §6 cost-per-tenant showback is a MUST, not a nicety.

---

## 9. Cheaper-architecture section

Four levers, each trading a differentiator slice for cost:

1. **Tier the *replay fidelity*, not just the storage age (biggest lever).** Capture heavy/replay-complete events (15 KB) **only for governed/high-assurance namespaces** (the ~5–15% where replay actually matters and PII redaction is already mandatory), and lean 2 KB events elsewhere. This cuts Scale C storage by ~60–70% (most events never needed authoritative replay). The honesty model already supports it: non-captured events are `best_effort`/`insufficient` *by design*, which is truthful, not a regression. **Estimated Scale C saving: ~$13k/mo → ~$160k/yr.**

2. **Sample-and-prove instead of capture-everything for cold.** Keep full events hot+warm; in cold, retain **100% of `deny`/`require_approval`/governed-control events** (the audit-relevant minority) and a **statistical sample of `allow`** with the chain proving completeness of the sample. SEC 17a-4 cares about the records of *consequence*; a defensible sampling policy for routine allows can cut cold volume 10×. (Legal sign-off required — flagged as risk, not assumed.)

3. **Don't self-host the cold tier integrity at all — externalize to a WORM object store with native Object-Lock + a third-party timestamping authority (RFC 3161 / a transparency log).** Replaces self-run Merkle checkpoint infra for the cold tier with the cloud's WORM guarantee + an external notarization, removing a chunk of C2/G4 build and run cost. Trades "we own the integrity primitive end-to-end" for "we anchor to a public/standard one" — the Sigstore-style play the devil's-advocate notes (and which OCSF/foundations make natural).

4. **Adopt the buy-20-build-2 architecture (§7.3).** The cheapest architecture overall: OCP substrate + OCSF schema + Vanta auditor-facing, and build only the cross-engine differential-sim wrapper and the lineage graph on top. ~$3.8M 3-yr vs ~$10.8M, keeps the moat, and inherits foundations' upstream-maintenance burden. This is a cost finding that **agrees with the devil's-advocate's strategic finding from an independent (cost) direction** — the rare case where the economics and the strategy point the same way.

---

## 10. Decisions (decide-document-continue)

| ID | Decision | Rationale |
|---|---|---|
| **D-G2-01** | Quote **list, on-demand, single-region** as headline; show committed-use as separate line. | Honest ceiling; no hidden discounts. |
| **D-G2-02** | Adopt the **15 KB heavy/replay-complete** event size for regulated-profile modeling (XD-1 value-capture MUST). | The differentiator's true storage cost must be visible, not hidden behind the 5 KB POC figure. |
| **D-G2-03** | **Tiering (R-G2-TIER) is mandatory, not optional**, for any retention > 1 yr; chain+checkpoints stay hot (R-G2-TIER-5). | Without it, 7-yr full-population audit is financially infeasible; with it, ~$330–490k/yr. |
| **D-G2-04** | **Cost-per-tenant/control showback is a MUST** (PLAN), not a reporting nicety. | Storage compounds per-tenant; flat-rate SaaS without metering is a margin trap. |
| **D-G2-05** | Recommend the **buy-20-build-2** cheaper architecture (§9.4) as the cost-optimal path; defer the business call to the thesis docs. | Cost evidence independently corroborates the devil's-advocate; ~$7M 3-yr saving while keeping the moat. |
| **D-G2-06** | KMS/crypto signing is **not** a material cost line (≈$30/mo); HSM (if G4 mandates) is the only signing cost that matters (~$2.1k/mo). | Prevents over-engineering "signing cost" worry; the integrity cost is *storage*, not crypto. |

## 11. Open questions (with decided defaults)
- **OQ-G2-1: Cloud price book?** *Default:* AWS us-east-1 list (§2); formulas portable. Revisit per target cloud.
- **OQ-G2-2: Does G4 mandate HSM-backed signing?** *Default:* assume **yes for regulated** (Scale C includes 2× CloudHSM); **no for POC/Mid** (CMK only). Confirm with G4.
- **OQ-G2-3: Sampling of routine `allow` events in cold tier (cheaper-arch lever 2)?** *Default:* **off** until legal sign-off; modeled as full-capture. The saving is real but needs a compliance ruling.
- **OQ-G2-4: SaaS vs self-hosted offering?** *Default:* self-hosted/BYO-cloud assumed (customer pays infra directly); a managed-SaaS offering inherits the cost cliff and **requires** §6 metering. Owned by product.

## 12. Dependencies
- **Consumes:** C2 (event size, retention, integrity model = storage driver), F2 (service inventory = compute driver), §22 (POC volumes), G1 (scale sizing it prices), G4 (KMS/HSM decision), G3 (backup/replica volume), G5 (tenant model for attribution).
- **Consumed by:** F3/MASTER-PLAN (affordability gate), `THESIS-DEVILS-ADVOCATE.md` (build-vs-buy cost input), product/pricing (showback, SaaS metering), G3 (DR storage cost), G5 (per-tenant chargeback).
- **Cross-cutting contract:** the §3 formulas + §2 price book + §5 tiering policy are the surface other NFR components and the business case cite for any cost figure.
