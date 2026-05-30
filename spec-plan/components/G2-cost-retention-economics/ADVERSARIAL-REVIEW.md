# G2 — Cost & Retention Economics — ADVERSARIAL REVIEW

**Component ID:** G2 · **Date:** 2026-05-30 · **Reviewer role:** hostile FinOps/CFO + skeptical infra architect attacking the cost model itself.
**Mandate:** find where the retention economics make the platform's *core differentiator* (full-population, replay-capable audit) financially infeasible at scale; where 7-yr regulated retention dwarfs everything; where the build cost destroys the business case the devil's-advocate raised. Then attack the model's own optimism.

**One-line verdict:** The SPEC is directionally right and its headline cliff is real, but it **systematically flatters the build case** in three places and **under-prices two failure modes** (retrieval/thaw, and metadata/index growth) that can each add a five- to six-figure surprise. The differentiator-is-the-cost-cliff finding (SPEC §8) is correct and is the single most important sentence in the whole NFR wave — but the model's "with tiering it's only ~$330–490k/yr" reassurance hides where it breaks.

---

## 1. Where retention economics make full-population audit infeasible at scale — the core attack

This is the brief's #1 question and the SPEC half-answers it. The honest answer is harsher than SPEC §8.

### A-1 (CRITICAL): Full-population × value-capture × 7-yr is the differentiator *and* the bankruptcy, and SPEC §9 lever 1 quietly concedes the differentiator doesn't survive.
The SPEC's own cheaper-architecture **lever 1** says: capture the heavy 15 KB replay-complete event **only for ~5–15% of namespaces** and lean 2 KB elsewhere. Read that against the product thesis. The *entire* differentiator (META G-2, C2 §0) is **"authoritative replay of the decision from the enforcement event itself, for the whole population."** Lever 1 says that's affordable only if you **stop doing it for 85–95% of the population.** That is not a tiering optimization — it is **abandoning full-population replay**, which is the product. So the cost model, followed to its own cheapest conclusion, **deletes the differentiator to afford the storage.** The SPEC frames this as "truthful, not a regression" (because un-captured events are honestly `best_effort`) — but a buyer told "we have authoritative replay" who then learns it's `best_effort` for 90% of traffic is exactly the "declared vs verified" sin (XD-6) the platform exists to prevent. **The cost model's own cure for the cliff is the marketing claim's death.** This must be surfaced to product as a first-order strategic constraint, not buried in a cheaper-arch appendix. *Priority: P0.*

### A-2 (HIGH): "Storage is only $22k/mo at Scale C" is true *and* misleading — the cliff is super-linear and the SPEC's own scale points hide it.
The three scales (100k / 500k / 50M events-day) are spaced so that the **middle scale shows storage at $10/mo** — making storage look like a non-issue right up until it's 54% of the bill. A buyer reading A→B concludes "storage is free." There is **no scale point between 500k/day-1yr and 50M/day-7yr** — a 25,000× hot-equivalent gap — so the model never shows *where* the knee is. The dangerous zone is exactly the unmodeled middle: a **5M events/day, 3-yr-retention, heavy-profile** mid-market regulated customer is where storage *first* overtakes compute, and that customer is more likely than the 50M/day giant. **Add intermediate scale points (5M/day×3yr, 10M/day×5yr) so the knee is visible**, or the model will be used to reassure exactly the customers it shouldn't. *Priority: P1.*

### A-3 (HIGH): The model prices *steady-state* retained volume, not *cumulative growth*, and conflates them at long retention.
At 7-yr retention the store is still *filling* for 7 years before it reaches steady state. The SPEC §4.3 figures are the **steady-state** (full 7-yr) numbers. But the buyer pays a **growing** bill years 1–7: storage ramps from ~$3k/mo (year 1, mostly hot/warm) to $22k/mo (year 7, full cold tail). The SPEC presents the terminal figure as if it's the run-rate from day one — **over-stating early-year cost and under-stating the compounding nobody budgeted for in year 5.** A CFO who provisioned $22k/mo in year 1 over-bought; one who provisioned year-1 actuals under-bought for year 5. **The model needs a per-year ramp table, not a single steady-state column.** *Priority: P1.*

---

## 2. Where 7-yr regulated retention dwarfs everything

### A-4 (CRITICAL): The thaw/retrieval cliff is under-priced and mis-categorized as "one-time."
SPEC §5.2 prices a full re-verification thaw at **~$54k** (2.7 PB × $0.02/GB) and calls it "one-time." Three errors:
1. **It's not one-time.** Regulated audits recur **annually**; litigation holds and regulator exams (SEC, FINRA) can each demand production. Budget **$54k+ per exam, multiple per year** for a large regulated entity — call it **$100–300k/yr** of thaw, an entire extra cost category absent from the §4.3 total.
2. **Deep Archive retrieval has a 12-hr latency** and **bulk-retrieval request fees** the SPEC omits ($0.025/1k requests on millions of objects = thousands of dollars in *request* charges alone, on top of the $/GB).
3. **Egress on thaw if data leaves the region** (auditor in another region, or export to the auditor's system) adds $0.02–0.09/GB on top. A 2.7 PB cross-region thaw is **another ~$54k–$240k.**
SPEC §5.2's mitigation (keep chain+checkpoints hot, thaw only disputed events) is correct and **must be promoted from a mitigation note to a hard MUST (R-G2-TIER-5 already exists — good — but the cost of *not* honoring it is the real number)**. If E1 replay or any auditor workflow *ever* needs the payload (not just the chain) for more than a handful of events, the thaw cost is real and recurring. **The model must include a "regulated audit/exam thaw reserve" line (~$100–300k/yr) at Scale C, or it understates regulated TCO by 20–60%.** *Priority: P0.*

### A-5 (HIGH): Index / metadata / read-model growth is unpriced, and it does NOT tier to cold.
The SPEC tiers the *event payloads* to Deep Archive, but the **secondary indexes** (C2 §8.1: `correlation_id`, `control_id`, `scope.*`, `resource_id`, `policy_version`, `replay_completeness`, `timestamp`) and the **Postgres read-model** that make the cold tier *findable* **cannot all go cold** — you must be able to locate a 5-year-old event to thaw it. Even at a lean 200 bytes/event of retained index, **50M/day × 7yr = 128 billion events × 200 B = 25 TB of index that stays warm/hot.** At hot prices that's **~$3k/mo of index alone**, growing every day, unbudgeted. The SPEC's "chain+checkpoints stay hot" (R-G2-TIER-5) is tiny; the **searchable index over 128B events is not**, and it's the thing that lets you find what to thaw. **Add an index-growth line; it may rival the cold-payload line at long retention.** *Priority: P1.*

### A-6 (MEDIUM): Compression and dedup factors are optimistic and load-bearing.
The whole Scale C affordability rests on **6× compression** and **5× blob dedup** (SPEC §2.5). These are asserted, not measured. If real-world C2 events compress **3×** (plausible once `request_object`/`before_state` carry high-entropy YAML/base64) and blobs dedup **2×** (heterogeneous external-data values), the §4.3 storage line **roughly doubles to ~$44k/mo / ~$530k/yr** for storage alone. The model is a *point estimate dressed as a quote*. **Every multiplier needs a sensitivity row (best/expected/worst); the headline should be a range, not a number.** *Priority: P1.*

---

## 3. Where the build cost destroys the business case (the devil's-advocate's point, costed)

### A-7 (CRITICAL): The build-vs-buy table flatters the build by under-counting eng-years and omitting the cost of the things the model itself says are unowned.
SPEC §7.1 estimates **18–25 eng-years to MVP**. That is optimistic against this corpus's own evidence:
- The cross-cutting review (META §3) lists **seven entirely unowned capabilities** (scale/cost, DR, key-management, retention, multi-tenant isolation, Rego human-factors, day-2 ops) — the **G-domain itself (G1–G8) is eight new components** the §7.1 estimate doesn't fully price. Building the *operational* architecture the review demands is **easily +10–15 eng-years** the build column omits.
- The "differential cross-engine sim" (E1) is the longest pole and the corpus's own ALT calls cross-engine replay "apples-to-oranges… disqualifying." Building the *hard* version of the headline feature is a multi-year research risk, not a line item. **Schedule risk = cost risk; the $4.5–6.3M MVP figure has no contingency.**
- A realistic loaded senior-eng cost in a competitive 2026 market is **$300–400k/yr fully loaded** (the SPEC uses $250k). At 30+ eng-years that alone moves the build to **$9–12M to MVP**, before maintenance.
**Corrected 3-yr build TCO is plausibly $14–18M, not $10.8M** — which *widens* the buy advantage (SPEC §7.3's 4.7×) to **~6–7×**. The SPEC's own conclusion (buy-20-build-2) gets *stronger* under honest accounting, but the SPEC under-sells it by under-counting the build. *Priority: P0 — fix to strengthen the very finding the SPEC is making.*

### A-8 (HIGH): The "buy" column is optimistic too — but less so, and the asymmetry matters.
The buy side assumes Vanta at **$25–75k/yr** and OCP self-host at **$30–80k/yr**. Vanta enterprise (the regulated buyer) is **$100k+**, and OCP self-host at regulated scale (HA bundle distribution, multi-region) is more like **$100–150k/yr** run. Glue at "2–4 eng-years" is light if you're wiring OCP+OCSF+Vanta+CNAPP into a regulated org's existing stack — call it **4–6 eng-years.** Corrected buy 3-yr TCO is **~$3.5–4.5M, not $2.3M.** **Both columns were optimistic; correcting both keeps buy ~3–4× cheaper.** The directional conclusion survives, but the SPEC should not present either number as precise. *Priority: P1.*

### A-9 (MEDIUM): "Build only the 2 novel on top of bought 20" (SPEC §9.4 / D-G2-05) under-prices integration tax.
Building the cross-engine differential-sim wrapper and lineage graph **on top of OCP+OCSF** means continuously tracking *their* schema/API changes — the lineage graph's value is "function of how many sources feed it" (devil's-advocate §1.1) and **every source is bespoke and breaks on its own schedule.** The "$1.5M to build the 2 novel" assumes a stable substrate; OCP is a young CNCF project and OCSF ships quarterly extensions. **Integration-maintenance on a buy-20-build-2 architecture may be a larger recurring line than the build-2 itself.** The cheapest architecture is real but its *recurring* cost is under-modeled. *Priority: P2.*

---

## 4. Attacks on the model's internal assumptions

### A-10 (MEDIUM): Spoke-enforcement compute is excluded from the headline — convenient, and sometimes wrong.
SPEC §3.4/§4 push Gatekeeper/OPA spoke compute to a memo line ("customer already pays for those clusters"). True for in-cluster admission — but **centralized OPA** (F2 §2.2 offers sidecar *or* centralized) and the **per-spoke decision-log shippers** *are* incremental, and at 20 spokes (Scale C) the excluded memo line is **+$370–$1,000+/mo** that a customer comparing to a SaaS competitor *will* count. **Don't exclude it from the headline at multi-cluster scale; show it in-band.** *Priority: P2.*

### A-11 (MEDIUM): KMS-is-free is right for checkpoints but wrong if per-event signing is ever turned on.
SPEC §3.3 / D-G2-06 correctly find checkpoint signing is ~$30/mo. But C2 §3.6 allows **per-event signature** on exported events, and a regulated auditor may demand **per-event** (not just per-checkpoint) signatures on an exported dataset. 50M events × $0.03/10k = **$150 per full-population signed export** — small per export, but if exports are frequent/large it's a line. More importantly, **per-event *verify*** on chain audits over 128B events is not free. The "signing is free" headline is true only under the checkpoint-cadence assumption; state that assumption as a *condition*, not a fact. *Priority: P2.*

### A-12 (LOW): Cost-per-tenant compute allocation by "event-share" is gameable and imprecise.
SPEC §6 / PLAN §2.2 allocate shared compute by event-share. But **E1 replay** is wildly uneven — one tenant running a full-cluster 7-yr differential sim consumes more compute than a thousand tenants' ingest, yet event-share would barely move. The PLAN notes job-attribution as a fallback-to-better; make it the **default for E1**, or a single tenant's replay job silently subsidized by everyone else becomes a chargeback dispute. *Priority: P3.*

### A-13 (LOW): The price book has no FX / multi-cloud / sovereign-cloud column.
Regulated multi-region (Scale C) often means **EU/sovereign regions** where egress and storage list prices are **15–40% higher** and Deep Archive equivalents may not exist (Azure/GCP archive tiers differ). The "±15% across clouds" claim (SPEC §2) is too tight for sovereign/gov regions. *Priority: P3.*

---

## 5. Prioritized defect list

| # | Pri | Defect | Fix |
|---|---|---|---|
| **A-1** | **P0** | The cheapest cure for the cliff (lever 1) **deletes full-population replay = the differentiator**. | Surface to product as a strategic constraint; the cost cliff and the moat are the same decision — decide it explicitly, don't bury it. |
| **A-4** | **P0** | Thaw/retrieval cost mis-labeled "one-time"; recurring annual exams + request fees + cross-region egress add **$100–300k/yr** at Scale C, omitted from the total. | Add a recurring "audit/exam thaw reserve" line; price request fees + cross-region egress on thaw. |
| **A-7** | **P0** | Build column under-counts eng-years (omits the 8 G-domain ops components + schedule risk + real loaded rates). Real build ≈ **$14–18M**, not $10.8M. | Re-cost build at $300–400k/eng-yr incl. G-domain + contingency; the buy advantage *grows* to ~6–7×. |
| **A-2** | P1 | Scale points spaced to hide the storage knee (mid shows $10/mo). | Add 5M/day×3yr and 10M/day×5yr intermediate scales so the knee is visible. |
| **A-3** | P1 | Steady-state vs cumulative-growth conflated at 7yr. | Add a per-year ramp table years 1–7. |
| **A-5** | P1 | Searchable index/read-model over 128B events stays warm/hot, unpriced (~$3k+/mo growing). | Add an index-growth line; it scales with cumulative event count, not retained payload. |
| **A-6** | P1 | 6× compression + 5× dedup asserted, not measured; affordability rests on them. | Add best/expected/worst sensitivity rows; quote a range. |
| **A-8** | P1 | Buy column also optimistic (Vanta enterprise, OCP regulated run, glue). | Correct to ~$3.5–4.5M; note both columns are bands. |
| **A-9** | P2 | Buy-20-build-2 under-prices recurring integration-maintenance on young/churning substrates. | Add a recurring integration-tax line to the cheaper-arch option. |
| **A-10** | P2 | Spoke/centralized-OPA compute excluded from headline at 20 spokes. | Show spoke compute in-band at multi-cluster scale. |
| **A-11** | P2 | "KMS free" holds only at checkpoint cadence; per-event signing/verify on 128B events isn't free. | State the checkpoint-cadence condition explicitly. |
| **A-12** | P3 | Event-share allocation lets a heavy E1 replay tenant be subsidized. | Default E1 replay to job-attribution, not event-share. |
| **A-13** | P3 | No sovereign/multi-cloud price column; ±15% too tight for EU/gov. | Add a sovereign-region multiplier (≈1.2–1.4×). |

---

## 6. Bottom line

The model gets the **shape** right — *compute-bound at POC, storage-bound at regulated scale, and the differentiator (full-population value-capture replay × 7-yr) is itself the cost cliff* — and that shape is the most important thing the NFR wave needed to establish. But the model is an **optimistic point estimate** where a **conservative range** is required, and it **omits two recurring cost categories** (audit/exam thaw, searchable-index growth) that at Scale C are jointly worth **$150–400k/yr** — enough to move the regulated TCO from ~$490k to **~$700k–900k/yr**.

The three P0s share a theme: **everywhere the model rounds, it rounds in the direction that makes the platform look more affordable and more differentiated than it is** — except A-7, where rounding *against* the build actually weakens the SPEC's own correct "buy, don't build" conclusion. Fix the P0s and the SPEC's two headline findings (the cliff is the differentiator; buy-20-build-2 is cost-optimal) **both get stronger and more honest** — which is the right direction for a document whose entire job is to be the honest cost half of the build-vs-buy decision.
