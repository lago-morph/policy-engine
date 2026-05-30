# G5 — Multi-Tenancy Isolation — SPEC

**Domain:** G · Operational / Non-Functional (NFR wave) · **HIGH-VALUE (has ALT tree)**
**Spec sources / drivers:**
- `META-ADVERSARIAL-SECOND-OPINION.md` §3 ("Multi-tenancy isolation beyond query scoping" — *no owner*) and §4 Risk #9 ("Multi-tenant isolation = a WHERE clause").
- `CROSSCUT-ADVERSARIAL.md` XD-5 (aggregate/analytics reads bypass the D2 scope interceptor).
- D2 `SPEC.md` / `ALT-opa-rls-spicedb.md` / `ADVERSARIAL-REVIEW.md` (A1, A2 — the *only* isolation today is an app-layer query rewrite enforced by a lint).
- D1 `SPEC.md` §1.3, §5.2, §8 (the tenant claim is normalized from attacker-influenced upstream claims; tenancy rests on claim integrity).
- C2 `SPEC.md` (per-tenant tamper-evident audit data; the system of record).
- F2 `SPEC.md` §2 (hub-and-spoke topology, deployment table, CRDs).
- Scenarios: HL-13 (cross-tenant detection), DT-54 (cross-tenant admin audit), DT-55 (scope-isolation pen-test), §20.2 (multi-tenant operation).

**Status:** AUTHORED (G5 author agent, NFR wave)
**Author persona:** Marcus (Platform Security Engineer) + an SRE/platform-operations engineer.
**Alt-architecture:** `ALT-tenancy-models.md` (row-level vs namespace-per-tenant vs cluster/instance-per-tenant).

**Sibling NFR components this SPEC binds to (authored in parallel — referenced by contract, not by content):**
G1 (scale/performance — owns the rate/quota mechanism G5 *configures per tenant*), G2 (cost/retention economics — owns the per-tenant storage cost model), G3 (availability/DR — owns per-tenant RPO/RTO and restore-and-prove-chain), G4 (key management — owns the per-tenant signing-key custody G5 *requires*), G6 (observability/day-2 ops — owns per-tenant operational dashboards), G7 (data lifecycle / privacy — owns the deletion/erasure machinery G5's offboard step *invokes*), G8 (Rego authoring — orthogonal).

---

## 1. Scope

### 1.1 The one-sentence problem

> Today the *entire* tenant boundary of this platform is a single `WHERE authz.tenant ∈ subject.tenants` predicate (D2-R5), appended in **one** app-layer interceptor, guarded by **one** lint (D2-A1), over a **shared** evidence log (C2) whose tenant label is derived from an **attacker-influenced** upstream JWT claim (D1 §1.3). One missing predicate, one bypassing read path (XD-5: analytics/reporting), one forged/confused tenant claim, or one saturating tenant workload, and one tenant sees, corrupts, or starves another tenant's **compliance evidence** — the single most catastrophic failure a compliance-evidence product can have.

G5 owns the **tenant isolation model** end-to-end across *every plane* — not just the read path D2 covers. G5 decides, per plane and per customer tier, **how** isolation is *enforced* (a mechanism with a failure mode), not merely *queried* (a predicate with a bypass).

### 1.2 In scope

1. The **tenancy model spectrum** (soft / logical → hard / physical) and a **decided position per customer tier** (POC/internal/SMB vs regulated/financial).
2. For each of the **six planes** (§4): the isolation *mechanism*, its *blast radius* when it fails, and its *cost*.
3. **Noisy-neighbor controls** — per-tenant rate/quota/concurrency limits, tied to the G1 rate-limit/quota mechanism, across the eval edge, the ingest path, the simulation/replay worker pool, and the analytics path.
4. **Data residency / regional pinning** — how a tenant's evidence is constrained to a region, and what that forbids (cross-region aggregation, global admin reads).
5. **The aggregate/analytics-read isolation problem (XD-5)** — making C3 analytics, C5 reporting, and every cross-tenant counter (HL-13) tenant-isolated, not just CRUD.
6. **Tenant lifecycle** — onboard, suspend, offboard + provable data deletion (tied to G7), and the **migration path** soft→hard without re-architecting (in PLAN.md).
7. The **tenancy-isolation conformance test suite** (DT-55 against *every* backend and the analytics path, not a Postgres demo).
8. The **tenant identity integrity** chain — binding the D1 tenant claim to a server-side tenant registry so tenancy does not rest on a claim alone.

### 1.3 Out of scope (owned elsewhere, consumed here)

- **The authorization decision and the scope predicate itself** — D2. G5 *strengthens* D2's enforcement (RLS, namespace, instance) and *extends* it to the planes D2 doesn't cover (compute, key, network), but D2 owns the predicate's semantics.
- **Producing the tenant claim** — D1. G5 owns *binding* the claim to a server-side registry and the *blast radius* of a forged claim; D1 owns normalization.
- **The rate-limit/quota primitive** — G1. G5 owns the *per-tenant* configuration and the tenant-fairness semantics; G1 owns the mechanism.
- **The deletion machinery** — G7. G5 owns the *offboard orchestration*; G7 owns crypto-shredding / erasure / retention-hold interaction.
- **Per-tenant key custody mechanics** — G4. G5 owns *requiring* per-tenant key separation for hard tenants; G4 owns rotation/HSM/KMS.
- **Per-tenant DR/RPO/RTO** — G3. G5 owns the *isolation* requirement on restore (a restore must not co-mingle tenants); G3 owns the durability numbers.

### 1.4 Load-bearing invariant (G5's signature)

> **G5-INV — Tenant isolation is a defense-in-depth stack, not a single predicate. Every plane that touches tenant data (read, write, compute, key, network, lifecycle) enforces the tenant boundary with a mechanism whose *failure mode* is documented, and no single failure (one missing predicate, one bypassing reader, one forged claim, one saturating workload, one leaked key) crosses a tenant boundary for the *regulated* tenancy tier.** For the *soft* tier, the boundary is logical and the residual cross-tenant blast radius is explicitly disclosed to (and accepted by) the buyer — soft tenancy is **never silently sold as hard**.

---

## 2. Definitions — the tenancy spectrum

| Tier | Name | Compute | Storage (evidence/SoR) | Signing key | Network | Sold to |
|---|---|---|---|---|---|---|
| **T0** | **Single-tenant / internal** | shared (1 logical tenant) | shared | shared | shared | POC, internal, design partner #1 |
| **T1** | **Soft / logical multi-tenancy** | shared workers; per-tenant quota | shared store; **row-level scope + RLS-under-interceptor** (mandatory) | shared signer, **per-tenant key-id in the chain** | shared; per-tenant rate limit | SMB / non-regulated SaaS |
| **T2** | **Namespace-per-tenant (hard-logical)** | per-tenant K8s namespace + ResourceQuota + NetworkPolicy; shared cluster | **per-tenant DB schema / per-tenant bucket prefix or bucket** | **per-tenant signing key** (G4) | per-tenant NetworkPolicy / mesh authz | mid-market regulated |
| **T3** | **Cluster-per-tenant** | dedicated cluster / node-pool | dedicated store instance | dedicated key (own HSM partition) | dedicated VPC/network | large regulated / financial |
| **T4** | **Instance-per-tenant (dedicated / single-tenant SaaS or self-host)** | entirely separate install (own hub) | entirely separate store | own key custody, optionally customer-managed (BYOK) | own network boundary | top-tier regulated, data-sovereign, air-gapped |

**Soft tenancy (T1)** = "logical, shared compute + row-level scope" — the current D2 design.
**Hard tenancy (T2–T4)** = progressively stronger physical/namespace isolation.

---

## 3. The decided position

### 3.1 Decision (D-G5-1)

> **The POC ships at T0 (single-tenant). The first multi-tenant product release ships T1 (soft) as the default *and* makes T2 (namespace-per-tenant) a first-class, supported, *sold* tier reachable without re-architecting. We do NOT sell T1 to a regulated buyer; the regulated buyer is sold T2 minimum, T3/T4 on demand.** The architecture is built so a single tenant can be *promoted* T1→T2→T3→T4 in place (see PLAN.md migration path), because tenancy strength is a per-tenant attribute, not a global build-time choice.

**Rationale.**
1. The meta-adversarial board's verdict (and the market memos) is blunt: **"soft tenancy is unsellable to a regulated buyer"** for a product whose entire premise is per-tenant evidence integrity. A WHERE-clause boundary fails the first FINOS / SOC2 / financial-services security questionnaire ("is my audit data physically isolated from other customers?"). So T1-only is a non-starter for the buyer the product is *for*.
2. But building T3/T4 for everyone (full instance-per-tenant) is the F3 "platform-first builds everything on spec" trap at the tenancy layer — it makes the per-tenant run-cost (G2) and day-2 ops (G6) explode before there's a paying tenant.
3. **Therefore: a tenancy-strength *dial*, not a tenancy *fork*.** One codebase, one storage *abstraction*, one identity edge — and a per-tenant `isolationTier` attribute (T0–T4) that selects the *physical* realization. The same `ScopePredicate` (D2) is the *innermost* layer at every tier; outer layers (schema, namespace, cluster, instance) are added as the dial turns up. This is the only way to satisfy "regulated buyers need hard isolation" *and* "don't build 4 products" *and* "migrate a tenant up without re-architecting."

### 3.2 What this explicitly rejects

- **Rejected: T1-only ("a WHERE clause is enough").** The XD-5 / Risk-9 critique stands; row-level scope over a shared log is one bug from cross-tenant disclosure of *audit data*, which is catastrophic for a compliance product. T1 is a floor, not a ceiling.
- **Rejected: T4-only ("just give everyone a dedicated instance").** Defensible for isolation, but the per-tenant cost (G2), the day-2 upgrade choreography across N instances of a 14-service stack (G6, meta-adversarial Risk #10), and the cross-tenant analytics impossibility (you can never answer "show me cross-tenant attack patterns," HL-13) make it wrong as the *only* model.
- **Rejected: choosing tenancy strength globally at build time.** That forces a re-architecture every time a tenant's regulatory posture changes. Tenancy strength MUST be per-tenant runtime config.

---

## 4. Per-plane isolation — how the boundary is *enforced*, not queried

For each plane: the mechanism per tier, the **blast radius** of a cross-tenant bug, and the noisy-neighbor control. This section is the heart of G5: the meta-adversarial finding is precisely that prior specs covered **only Plane 1 (read), partially**, and left Planes 2–6 unowned.

### 4.1 Plane 1 — Storage / read path (the WHERE clause + below it)

| Tier | Mechanism |
|---|---|
| T0 | n/a (one tenant) |
| T1 | **D2 ScopePredicate (app interceptor) + RLS-under-interceptor MANDATORY for relational** (XD-5 resolution, D2-A1/A13). Object-store evidence (C2's recommended backend, meta-adversarial G-3): **per-tenant key prefix** + a tenant-bound access credential per request, so a missing predicate cannot list another tenant's blobs. |
| T2 | T1 **+ per-tenant DB schema / per-tenant bucket** — the tenant predicate is now *also* a connection/namespace boundary; a forgotten predicate hits an empty schema, not another tenant's rows. |
| T3 | T2 **+ dedicated store instance** — cross-tenant read requires crossing a network + credential boundary. |
| T4 | dedicated install — no shared store exists. |

- **G5-R1 (MUST)** For T1, RLS-under-interceptor is **mandatory** on every relational table holding tenant data (promotes D2 OQ-1 from "optional/later" — resolves D2-A1, A13, XD-5). A forgotten interceptor call MUST still fail closed at the engine.
- **G5-R2 (MUST)** For the **object-store evidence log** (C2 system of record), tenant isolation MUST NOT rely on a query predicate alone (object stores have no RLS — meta-adversarial G-3 falsifier). It MUST use a **per-tenant key namespace** (bucket-per-tenant for T2+, or strict key-prefix + a tenant-scoped, least-privilege storage credential minted per request for T1). A pen-test (DT-55) MUST be run **against the object-store backend**, not a Postgres demo (G-3 falsifier; D2 left "which backend" unstated).
- **G5-R3 (MUST)** Tenant data at rest is **encrypted with a tenant-scoped key** for T2+ (per-tenant DEK wrapped by the per-tenant key from G4). Loss/leak of one tenant's key MUST NOT decrypt another tenant's data (limits the G4 / XD-19 key-compromise blast radius to one tenant).

**Blast radius of a Plane-1 bug:** T1 = *all tenants' evidence readable* (catastrophic — this is the WHERE-clause failure the board names). T2 = bug confined to a misconfigured tenant pair at worst (schema/bucket still separates). T3/T4 = none (no shared store).

### 4.2 Plane 2 — Aggregate / analytics / reporting reads (XD-5, the largest real escape surface)

This is the plane D2-A2 / XD-5 call out as **likely bypassing the interceptor entirely**, and the richest data (per-subject cross-tenant counters, HL-13 step 4) is exactly here.

- **G5-R4 (MUST)** C3 analytics and C5 reporting read paths MUST link the **same** D2 ScopePredicate library (the XD-5 resolution: "never reimplemented"). An analytics worker has **no** direct, unscoped storage path. Enforced by architecture (the storage client is only reachable via the scoped accessor) **and** by RLS underneath (T1+) / schema separation (T2+) so a forgotten scope in an aggregate query returns empty, not cross-tenant.
- **G5-R5 (MUST)** A **genuinely cross-tenant** aggregate (e.g. "platform-wide attack patterns," HL-13; the security operator's global view) is a **distinct, explicitly-privileged operation**, available only to a global-scope subject, and:
  (a) emits a `boundary_crossing` audit event per tenant touched (D2-R9, DT-54);
  (b) runs over a **pre-aggregated, k-anonymized / tenant-id-tokenized** materialized view — it MUST NOT expose one tenant's *raw* rows to another tenant's report, only platform-level counts;
  (c) is **forbidden entirely** for any tenant pinned to a residency region that disallows cross-region aggregation (§4.4, G5-R10).
- **G5-R6 (MUST)** Every C5 rollup carries the per-tenant **evidence-quality denominator** (XD-6 / DC-12 cross-ref) *within* the tenant; cross-tenant rollups never aggregate one tenant's `best_effort` into another's `complete`.
- **G5-R7 (SHOULD)** For T2+, analytics runs **per-tenant** (a worker scoped to one schema/bucket) by default; the global view is the exception, gated as in G5-R5. This makes the *default* analytics path physically tenant-bound, leaving only the explicitly-privileged global path as a cross-tenant surface.

**Blast radius:** an unscoped aggregate at T1 leaks *cross-tenant evidence in a report an auditor reads* — worse than a CRUD leak because it's exfiltrated into a durable artifact. G5-R4/R5/R7 collapse this surface.

### 4.3 Plane 3 — Compute / worker isolation & noisy neighbors

D2 says nothing about compute. The meta-adversarial finding: "no blast-radius bound when one tenant's policy evaluation or replay job saturates shared workers."

| Tier | Mechanism |
|---|---|
| T0 | n/a |
| T1 | shared simulation/analytics/ingest workers with **per-tenant concurrency caps + fair-share scheduling + per-tenant rate/quota** (configured via **G1**); a runaway tenant is throttled, not allowed to starve others. |
| T2 | T1 + per-tenant K8s **ResourceQuota + LimitRange** in the tenant namespace; CPU/mem bounded per tenant. |
| T3 | dedicated node-pool / cluster — physical compute isolation. |
| T4 | dedicated install. |

- **G5-R8 (MUST)** Every shared worker pool (E1 simulation/replay, C3 analytics, C2 ingest) enforces **per-tenant quotas and concurrency limits** tied to G1's rate/quota mechanism. A single tenant's replay/simulation burst MUST NOT consume more than its configured share; excess is **queued or shed for that tenant only** (per-tenant backpressure), never global.
- **G5-R9 (MUST)** The simulation/replay path (E1) — explicitly a **serial, single-author core** that the master plan forbids parallelizing (meta-adversarial Risk #2) — is a **per-tenant fairness hazard**: a serial core is a global chokepoint. G5 requires either (a) per-tenant replay worker pools (T2+), or (b) a **per-tenant fair queue with a max in-flight per tenant** (T1) so one tenant's large replay cannot monopolize the serial core. This is the single most important noisy-neighbor control because the serial replay core is the platform's narrowest resource.

**Blast radius:** without G5-R8/R9, one tenant's nightly full-window replay (G2/retention can make this enormous) makes *every* tenant's console/replay hang — a shared-fate availability incident. Side-channel: shared OPA/analytics compute is a timing/cache side-channel between tenants (meta-adversarial "shared OPA/analytics compute = side-channel + noisy neighbor"); for T3/T4 this is closed physically; for T1/T2 it is an accepted, disclosed residual (data is already scoped; the side-channel leaks *metadata*, e.g. that another tenant ran a large job — disclosed in the T1/T2 isolation statement, §6).

### 4.4 Plane 4 — Data residency / regional pinning

No prior component owns residency. For a financial/regulated buyer (EU data-sovereignty, FINOS), "where does my evidence physically live" is a buying gate.

- **G5-R10 (MUST)** Each tenant carries a **residency attribute** (`region`, `allowCrossRegionAggregation: bool`) in the tenant registry (§5). All of that tenant's evidence (C2 log), replay datasets, exports, and backups (G3) MUST be **pinned to the declared region**; a write/replicate/restore that would place tenant data outside its region MUST fail closed.
- **G5-R11 (MUST)** The hub-and-spoke topology (F2 §2.1) MUST support **region-pinned spokes and region-pinned storage**: a tenant in `eu` is served by an `eu` hub/store; the global control plane MUST NOT pull that tenant's raw evidence cross-region. Cross-region behavior is limited to **control metadata** (which tenants exist, health) — never tenant evidence content.
- **G5-R12 (MUST)** If `allowCrossRegionAggregation=false`, the global analytics path (G5-R5) **excludes** that tenant entirely (it cannot appear even in a count that crosses its region).

**Blast radius of a residency bug:** a single mis-routed write/replica = a **regulatory breach** (data left its jurisdiction), independent of any read leak. This is why residency is fail-closed at write time, not audited after the fact.

### 4.5 Plane 5 — Tenant identity integrity (tenancy rests on the claim — D1)

The adversarial thesis: **the tenant claim is attacker-influenced (D1 §1.3 derives it from upstream `tenant`/`org_id`); tenancy rests on claim integrity.** If an attacker can forge/confuse their tenant claim, every downstream boundary (which is keyed on that claim) opens.

- **G5-R13 (MUST)** The tenant value in the normalized subject (D1) MUST be **bound to a server-side tenant registry** (§5): a presented tenant claim is *validated against* the registry (does this issuer/realm legitimately map to this tenant?), not trusted as asserted. An unknown or unauthorized tenant ⇒ `degraded`/`incomplete` and fail closed (extends D1 §5.2 "unknown tenant ⇒ degraded" into a *hard* registry check). This reuses D2 OQ-6 ("server-side grant store authoritative; token is an assertion") and elevates it to a MUST for the *tenant* dimension specifically.
- **G5-R14 (MUST)** The **issuer→tenant binding** is part of the registry: realm/issuer `kc.example/realms/payments` may assert `tenant=payments` and nothing else. A token from one tenant's realm asserting another tenant's id MUST be rejected (defeats the multi-IdP confusion attack the D1 normalization could otherwise mask). Cross-tenant assertion attempt ⇒ `auth_denied{reason: tenant_issuer_mismatch}` audit event.
- **G5-R15 (MUST)** The tenant boundary MUST NOT rest on the claim *alone* at any tier ≥ T2: the physical layer (schema/bucket/cluster) is selected from the **registry-resolved** tenant, so even a (hypothetically) forged claim that slipped past G5-R13/R14 lands in its registry-bound partition, not an arbitrary one. (Defense in depth: claim integrity is the first gate, physical partition keyed on the *registry* record is the second.)

**Blast radius:** without G5-R13/R14, a forged/confused tenant claim is a **total cross-tenant compromise at T1** (the claim *is* the boundary). With them, a forged claim is caught at the registry gate (first line) and, even if it weren't, lands in the wrong-but-still-isolated partition only if the registry record itself is wrong (second line).

### 4.6 Plane 6 — Network / control-plane isolation

| Tier | Mechanism |
|---|---|
| T0/T1 | shared network; tenant data isolated at app+storage layer; mTLS between services (D4). |
| T2 | per-tenant **NetworkPolicy** (tenant namespace cannot reach another tenant's pods/PVCs) + mesh authz scoped per tenant. |
| T3 | dedicated VPC/network per tenant. |
| T4 | fully separate network boundary / customer-controlled. |

- **G5-R16 (MUST)** For T2+, a tenant's workloads (per-tenant analytics/replay pods, T2 §4.3) are constrained by NetworkPolicy so they cannot reach another tenant's storage endpoint or pods, even if a credential leaked. (Closes the lateral-movement path that app-layer scope alone cannot.)

---

## 5. Tenant registry (the new shared entity G5 introduces)

The single server-side source of truth that binds claim → physical isolation → residency → lifecycle state. Referenced by D1 (validate claim), D2 (resolve scope), F2 controllers (place workloads), G1 (per-tenant quota), G3 (per-tenant DR), G7 (offboard).

```json
{
  "tenant_id": "payments",
  "display_name": "Payments Co.",
  "isolation_tier": "T2",                       // T0..T4 — the dial (D-G5-1)
  "trusted_issuers": ["https://kc.example/realms/payments"],  // G5-R14 binding
  "residency": { "region": "eu-west-1", "allowCrossRegionAggregation": false }, // G5-R10/12
  "storage": {                                   // resolved per tier
    "relational_schema": "tenant_payments",      // T2+
    "evidence_bucket": "evidence-eu-payments",   // per-tenant key namespace (G5-R2)
    "kms_key_ref": "kms://eu/payments-signing"   // per-tenant signing key (G4, G5-R3)
  },
  "quotas": {                                    // G1-configured per-tenant limits (G5-R8/R9)
    "eval_rate_per_sec": 200, "ingest_events_per_day": 100000,
    "replay_concurrency": 2, "analytics_concurrency": 1,
    "max_replay_window_days": 90, "storage_quota_gb": 200
  },
  "lifecycle": { "state": "active", "onboarded_at": "...", "suspended_at": null,
                 "offboard_requested_at": null, "retention_hold": false },
  "scope_defaults": { "namespaces": ["payments-*"], "policy_domains": ["*"] }
}
```

- **G5-R17 (MUST)** The tenant registry is itself a **scoped, audited governance artifact** (lifecycle per §7); creating/editing a tenant record is a Platform Governance Admin operation under **dual control** (D3 approval; cross-ref D2-A6 — high-impact admin verbs need approval, not just audit). A registry edit *is* a privilege/boundary change.
- **G5-R18 (MUST)** `isolation_tier`, `residency.region`, and `trusted_issuers` are **append/upgrade-only with migration** — they cannot be silently weakened (e.g. T2→T1, or region change) without an explicit, dual-controlled, audited migration (PLAN.md) that includes a data-move/erasure step. Weakening isolation or moving a region is the highest-risk tenant operation.

---

## 6. The honesty contract — never sell soft as hard

- **G5-R19 (MUST)** Each tenant's **isolation statement** (tier, planes, residual side-channels, blast radius at this tier) is a first-class, exportable artifact the buyer signs off (mirrors C2's honesty-over-coverage thesis at the tenancy layer). A T1 tenant's statement explicitly says: "compute and storage are *shared*; isolation is logical (row-level + RLS); a platform bug could expose your data to another tenant; metadata side-channels exist." A T1 tenant MUST NOT be told they have "isolated" data. (Directly answers the board: "soft tenancy is unsellable to a regulated buyer" — so we don't *mis-sell* it; we sell regulated buyers T2+.)
- **G5-R20 (MUST)** The product's compliance questionnaire answers ("is customer data isolated?") are **derived from the registry `isolation_tier`**, not hand-asserted — so the sales answer cannot drift from the deployed reality.

---

## 7. Tenant lifecycle

### 7.1 Onboard

1. Create tenant registry record (G5-R17, dual-controlled) with `isolation_tier`, residency, issuer binding, quotas.
2. **Provision physical isolation for the tier** (idempotent, operator-driven — F2 `governance-operator`): T1 = ensure RLS policies + per-tenant bucket prefix/credential; T2 = create tenant K8s namespace + ResourceQuota + NetworkPolicy + DB schema + bucket + per-tenant signing key (G4); T3/T4 = provision cluster/instance.
3. Bind issuer→tenant in D1 (G5-R14) and scope defaults in D2.
4. Emit `tenant_onboarded` audit event; produce the isolation statement (G5-R19).

- **G5-R21 (MUST)** Onboarding is **declarative and idempotent** via the operator (a `Tenant` CR or registry-driven reconcile); partial onboarding must be safely re-runnable and must not leave a tenant with *some* planes isolated and others shared.

### 7.2 Suspend

- **G5-R22 (MUST)** Suspend **freezes writes** (no new evidence/eval ingest accepted, returns a clear suspended error) but **preserves read + the immutable evidence chain** (a suspended tenant's auditor can still read; the compliance record is never destroyed by suspension). Suspend is reversible. Quotas drop to zero for active work; the tenant's data and chain remain verifiable (cross-ref G3 — suspended ≠ deleted).

### 7.3 Offboard + provable deletion (tied to G7)

- **G5-R23 (MUST)** Offboard is a **two-phase, retention-aware** operation orchestrated by G5, executed by G7:
  - **Phase 1 (grace + hold check):** if `retention_hold` or an active legal/regulatory retention obligation exists (a compliance product *must* honor retention holds), deletion is **blocked** and the tenant moves to `retained` (read-frozen, not erased) until the hold clears. Deleting evidence under a hold would itself be a compliance violation.
  - **Phase 2 (erasure):** when no hold remains, G7 performs **crypto-shredding** — destroy the tenant's per-tenant key (G4) so the encrypted-at-rest evidence (G5-R3) is unrecoverable — plus best-effort blob/row deletion, namespace/schema/bucket teardown (operator), and registry tombstone.
- **G5-R24 (MUST)** Deletion is **provable**: a signed `tenant_offboarded` certificate records what was erased, the key-destruction proof, and any retained-under-hold residue, **scoped to that tenant** and verifiable independently. (A compliance product must prove deletion the way it proves everything else.)
- **G5-R25 (MUST)** Crypto-shredding (key destruction) is the **primary** erasure guarantee precisely because T1/T2 share storage substrate — physically scrubbing one tenant's rows from a shared append-only log is impractical, but destroying the tenant key renders that tenant's at-rest evidence unrecoverable (this is *why* G5-R3 mandates per-tenant keys even at T1's shared store). For T3/T4, physical store teardown is added.
- **G5-R26 (MUST)** Offboard MUST guarantee **no cross-tenant collateral**: erasing tenant A must not break tenant B's hash chain, shared catalog, or analytics. (Per-tenant chains/keys make this hold; a *shared* chain across tenants would make deletion impossible — so the C2 evidence chain MUST be **per-tenant**, see §8 dependency.)

---

## 8. Dependencies & the contracts G5 imposes on other components

| Depends on | For | Contract G5 imposes |
|---|---|---|
| D1 | normalized subject + tenant claim | **G5-R13/R14**: validate tenant against registry; bind issuer→tenant; reject cross-tenant assertions |
| D2 | ScopePredicate + interceptor | **G5-R1**: RLS-under-interceptor mandatory (T1); **G5-R4**: analytics/reporting link the same predicate |
| C2 | evidence log (system of record) | **per-tenant hash chain + per-tenant signing key** (G5-R3/R25/R26): the chain MUST NOT span tenants, or deletion + key-blast-radius break. This is a **C2 contract change** G5 requires (flag for C2 reconciliation). |
| C3 / C5 | analytics / reporting | **G5-R4/R5/R7**: route through scoped accessor; default per-tenant; global view explicitly privileged + k-anonymized |
| F2 | topology + operator + CRDs | **G5-R11/R16/R21**: region-pinned spokes/storage; per-tenant namespace+quota+NetworkPolicy; idempotent declarative tenant provisioning (a `Tenant` CR) |
| G1 | rate/quota primitive | **G5-R8/R9**: per-tenant quotas/concurrency, per-tenant backpressure, fair queue over the serial replay core |
| G3 | DR/RPO/RTO | restore MUST be per-tenant and MUST NOT co-mingle tenants or violate residency (G5-R10) |
| G4 | key management | **per-tenant signing/encryption keys** (G5-R3); key destruction = crypto-shred (G5-R25) |
| G7 | data lifecycle / privacy | executes offboard erasure (G5-R23/R24/R25); honors retention holds |
| D3 | approval gating | dual control on tenant registry edits + isolation/residency changes (G5-R17/R18) |

| Depended on by | For |
|---|---|
| Every read/write/analytics path | tenant boundary at the correct tier |
| Sales / compliance questionnaire | registry-derived isolation answers (G5-R20) |
| G2 cost model | per-tenant storage/compute attribution (the registry's per-tenant resources make cost attributable) |

---

## 9. Failure modes

| Failure | Behavior |
|---|---|
| Tenant claim unknown / not in registry (G5-R13) | Fail closed: `incomplete` subject; tenant-scoped reads return empty; no fallback to "default tenant." |
| Issuer asserts a tenant it isn't bound to (G5-R14) | Reject token; `auth_denied{tenant_issuer_mismatch}`; alert (active attack signal). |
| RLS policy missing on a relational table (T1, G5-R1) | Startup/CI check fails closed; the table is unreachable until RLS is present (a table without RLS is a tenancy hole — treat as a build break). |
| Analytics worker attempts unscoped aggregate (G5-R4) | No unscoped storage path exists; query is rejected at the accessor; alert. |
| Residency would be violated by a write/replica/restore (G5-R10) | Fail closed; the operation is refused; `residency_violation_blocked` audit event. |
| One tenant saturates the serial replay core (G5-R9) | Per-tenant fair queue sheds/queues *that tenant's* excess only; other tenants unaffected; tenant sees backpressure. |
| Per-tenant key unavailable (G4) for a T2+ tenant | That tenant's writes fail closed (cannot sign/encrypt); other tenants unaffected (blast radius = one tenant). |
| Offboard requested but retention hold active (G5-R23) | Deletion blocked; tenant → `retained`; explicit operator visibility; never silently deleted under hold. |
| Tenant onboard partially completed (G5-R21) | Reconcile is idempotent; tenant stays `provisioning` (not `active`) until *all* planes for the tier are confirmed; never serves traffic half-isolated. |
| Migration T1→T2 mid-flight (PLAN.md) | Dual-write/cutover with verification; never a window where the tenant is readable cross-boundary; rollback to T1 defined. |

---

## 10. Conformance / test strategy (the DT-55 extension)

- **G5-R27 (MUST)** The tenancy conformance suite runs DT-55 (scope-isolation pen-test) against **every storage backend in use** (relational *and* the object-store evidence log — G-3 falsifier), **and** against the **analytics/reporting aggregate path** (XD-5), **and** against the **forged/confused tenant-claim** vectors (G5-R13/R14), **and** a **noisy-neighbor saturation test** (one tenant floods replay; assert others meet SLO — G5-R9). A regression in any is a **P0** (matches D2 §6's "scope escape = P0").
- **G5-R28 (MUST)** A **per-tier acceptance gate**: a tenant cannot be set to a tier until the conformance suite passes for that tier's mechanisms against the actual deployed backend. (No "T2 on paper, T1 in reality.")

---

## 11. Open questions — decided defaults

| # | Question | Decided default | Rationale |
|---|---|---|---|
| OQ-G5-1 | Global tenancy choice vs per-tenant dial? | **Per-tenant `isolation_tier` (T0–T4); a dial, not a fork.** | Satisfies regulated-needs-hard + don't-build-4-products + migrate-in-place (D-G5-1). |
| OQ-G5-2 | Default tier for a new non-regulated tenant? | **T1 (soft) + RLS mandatory**, isolation statement disclosed. Regulated ⇒ **T2 minimum**. | Soft is a floor, never sold as hard (G5-R19). |
| OQ-G5-3 | Is the C2 evidence hash chain per-tenant or global? | **Per-tenant chain + per-tenant key.** | Required for deletion-without-collateral (G5-R26) and per-tenant key-blast-radius (G5-R3). Imposes a C2 contract change. |
| OQ-G5-4 | Object-store evidence isolation mechanism? | **Per-tenant key namespace (bucket/prefix) + per-request tenant-scoped credential**, not a predicate (object stores have no RLS). | G-3 falsifier; predicate-only over object store is the worst escape surface. |
| OQ-G5-5 | How is deletion proven on a shared append-only log? | **Crypto-shredding (destroy per-tenant key) is the primary guarantee; physical teardown added at T3/T4.** | Physically scrubbing a shared append-only log is impractical; key destruction is provable + bounded. |
| OQ-G5-6 | Cross-tenant analytics — allowed at all? | **Only as an explicitly-privileged, audited, k-anonymized, residency-respecting global view (G5-R5); per-tenant analytics is the default.** | Preserves HL-13's "platform-wide attack patterns" use case without making it the default leak surface. |
| OQ-G5-7 | Does tenancy rest on the JWT tenant claim? | **No — claim is validated against the server-side registry (G5-R13) and issuer-bound (G5-R14); physical partition keyed on the registry record (G5-R15).** | Tenancy must not rest on an attacker-influenced claim alone. |
| OQ-G5-8 | Can a tenant be downgraded (T2→T1) or region-changed? | **Only via dual-controlled, audited migration with a data-move/erasure step; never silently.** | Weakening isolation/moving region is the highest-risk tenant op (G5-R18). |
