# G2 — Cost & Retention Economics — PLAN

**Component ID:** G2 · **Domain:** G — Operational / Non-Functional Architecture
**Status:** Authored · **Date:** 2026-05-30
**Purpose:** How cost is *measured, attributed, and governed* over time — FinOps tagging, cost-per-tenant / cost-per-control / showback, and the workstreams that produce and maintain the cost model in SPEC.md.

---

## 1. What "implementing G2" means

G2 is not a runtime service; it is (a) a **cost model** (SPEC §3–4), (b) a **measurement/attribution pipeline** that keeps the model honest against reality, and (c) a set of **FinOps controls** (tagging, budgets, alerts, showback). The deliverable lifecycle is: *model → instrument → attribute → report → govern → re-baseline*. Because the platform already stores `scope`-tagged, `control_id`-tagged evidence, most of the attribution data exists for free — the work is wiring it to billing data, not building new collection.

---

## 2. Cost measurement & attribution

### 2.1 The three measured quantities
| Quantity | Source of truth | Method |
|---|---|---|
| **Bytes stored per scope** | C2 store metadata (every event carries `scope.{cluster,namespace,tenant}` + size + tier) | Sum stored bytes × current tier unit-price, grouped by `scope.tenant` / `control_id`. Direct, exact. |
| **Compute consumed** | K8s metrics (per-namespace/pod CPU·mem) + cloud billing | Allocate shared services by **event-share** (tenant's % of ingested events) + flat **platform-overhead** split. |
| **Egress / KMS / managed deps** | Cloud billing + tags | Tag-based; egress attributed to the exporting tenant/SIEM adapter. |

### 2.2 Cost-per-tenant (the primary attribution)
```
cost(tenant) = Σ_tier (bytes_in_tier(tenant) × $tier)              # storage, exact
             + event_share(tenant) × shared_compute_$              # C2 ingest + C3 analytics + E1 replay
             + platform_overhead_$ / N_tenants                     # operator, Keycloak, API base, EKS fees
             + egress_$(tenant) + kms_$(tenant)
```
- **Storage is exact** (per-event `scope.tenant`); compute is **allocated** (event-share is the best proxy because C2-ingest and C3-analytics cost scales ~linearly with events processed).
- E1 replay compute is **job-attributable** (a replay job names its dataset's tenant/scope) — better than event-share; use job attribution where available, fall back to event-share.

### 2.3 Cost-per-control
Same rollup keyed by `control_id` instead of tenant. Answers "what does enforcing `SC-IMG-001` cost us in evidence storage + analytics?" — directly useful to the governance team deciding whether a low-value control's full-population evidence is worth its storage tail.

### 2.4 Showback / chargeback
- **Showback (default, POC→Mid):** a monthly per-tenant cost report rendered by C5 (reuses the reporting/export surface), no money moves — visibility only.
- **Chargeback (Regulated/SaaS):** the same numbers feed a metering record per tenant per month; required if the platform is offered as managed SaaS (SPEC D-G2-04, OQ-G2-4), because the storage cliff (SPEC §8) makes flat-rate margin-negative for a heavy tenant.
- Both are **queries over data the platform already holds** — the only new artifact is the join to cloud billing.

---

## 3. FinOps tagging

### 3.1 Tagging contract (R-G2-FINOPS)
- **R-G2-FINOPS-1 (MUST):** Every cloud resource the platform provisions carries tags: `platform=governance`, `component=<id>` (C2/C3/E1/...), `tier=hot|warm|cold`, `environment`, and where the resource is tenant-dedicated, `tenant=<id>`. This is what makes cloud-billing-side attribution possible without guesswork.
- **R-G2-FINOPS-2 (MUST):** Object-storage lifecycle rules are tagged by `tier` and `retention_class` so tiering transitions (SPEC §5) are auditable and their cost-effect measurable.
- **R-G2-FINOPS-3 (SHOULD):** Shared (non-tenant-dedicated) resources carry `cost_allocation=shared` so the allocator knows to split them by event-share, not assign them.
- **R-G2-FINOPS-4 (SHOULD):** A monthly reconciliation compares **modeled** cost (SPEC §3 formulas applied to actual `E`, `R`, `S`) against **billed** cost; drift > 15% triggers a model re-baseline (workstream WS-5). This keeps the SPEC's numbers from rotting.

### 3.2 Budgets & alerts
- Per-deployment cloud **budget alerts** at 80/100/120% of the modeled monthly figure for the deployment's scale tier.
- A **storage-growth-rate alert**: if `bytes_stored` growth/day exceeds the modeled `E × S`, it means event size or volume drifted up (often: heavy-profile capture turned on more broadly than planned) — the early-warning for the SPEC §8 cliff.

---

## 4. Workstreams (parallelizable)

| WS | Name | Depends on | Parallel? | Output |
|---|---|---|---|---|
| **WS-1** | Price-book + formulas (SPEC §2–3) | none | yes | the model itself (done in SPEC) |
| **WS-2** | Three worked TCOs + cliff analysis (SPEC §4–8) | WS-1 | yes | scenario numbers (done in SPEC) |
| **WS-3** | FinOps tagging contract + lifecycle tagging | C2 store, F2 operator | yes | R-G2-FINOPS reqs → operator applies tags |
| **WS-4** | Attribution pipeline (storage-exact + compute-allocated) + showback report | C2 metadata, C5 reporting, cloud billing export | after WS-3 | cost-per-tenant / cost-per-control report |
| **WS-5** | Model-vs-billed reconciliation + re-baseline loop | WS-2, WS-4 | after WS-4 | quarterly re-baseline; drift alerts |
| **WS-6** | Build-vs-buy + cheaper-arch brief (SPEC §7, §9) | WS-2 | yes | input to F3/MASTER-PLAN + thesis docs |

**Critical path:** WS-1 → WS-2 → WS-6 (the decision-relevant path; deliverable in SPEC already). **Independent path:** WS-3 → WS-4 → WS-5 (the operational/instrumentation path; needs the C2 store + C5 reporting + a cloud billing export). WS-3/WS-4 can start as soon as C2's `scope` tagging is frozen (it is) and F2's operator can apply tags.

### 4.1 What blocks what
- WS-4 (attribution) **blocks** any SaaS/chargeback offering and the SPEC D-G2-04 metering MUST.
- WS-6 (build-vs-buy) **blocks** nothing technically but is an **input to the funding/sequencing decision** — it should land before MASTER-PLAN's "commit to platform-first vs wedge-first" gate.
- WS-3 tagging **blocks** WS-4 attribution (can't attribute cloud cost without tags).

---

## 5. Test / validation strategy

- **Model-reproducibility test:** the SPEC §3 formulas, applied to the SPEC §4 parameters, must reproduce the SPEC §4 tables (a spreadsheet/notebook asserting each line). Guards against the numbers drifting from the formulas as either is edited.
- **Attribution-conservation test:** `Σ cost(tenant) + unallocated_overhead == total_billed` (within rounding). Money must not be created or lost in allocation.
- **Tiering-saving test:** in a staging deployment, force a tier transition and assert the billed $/GB for the moved data drops to the target tier's unit (proves lifecycle rules actually fire — the SPEC §5 economics depend on transitions *happening*, not just being configured).
- **Drift test:** inject a synthetic 2× event-size increase; assert WS-5 reconciliation flags >15% drift and the storage-growth alert fires (validates the §8 cliff early-warning).
- **Cliff-projection test:** parameterize the model at `E=50M, R=7yr, S=15KB` and assert it lands in the SPEC §4.3 band (~$41k/mo) — a regression guard on the headline finding.

## 6. Milestones
- **M-G2-MODEL:** SPEC formulas + 3 TCOs published (done).
- **M-G2-TAG:** FinOps tagging live in the operator (WS-3) — gates attribution.
- **M-G2-SHOWBACK:** cost-per-tenant report rendered from real data (WS-4).
- **M-G2-BASELINE:** first model-vs-billed reconciliation; SPEC numbers ratified or re-baselined (WS-5).
- **M-G2-DECISION:** build-vs-buy brief delivered to the sequencing gate (WS-6).

## 7. Ownership & cadence
- Owner: the NFR/SRE owner the cross-cutting review asks for (META Risk 2 "a new NFR/SRE owner"), jointly with product (pricing).
- Cadence: re-baseline **quarterly** and on any change to retention policy, event-profile defaults, or cloud price book. The cliff (SPEC §8) is sensitive to exactly those three, so they are the change-triggers.
