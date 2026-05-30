# G5 — ALT Architecture — Tenancy Models (row-level vs namespace-per-tenant vs cluster/instance-per-tenant)

**Question this ALT explores.** The SPEC decides a *per-tenant dial* (T0–T4) rather than a single global tenancy model. This document does the honest comparison the decision rests on: it evaluates the **three canonical multi-tenancy architectures** as if each were chosen as *the* model, with isolation-strength / cost / ops trade-offs, then shows why a single one is wrong and the dial is right — and, critically, **why the dial is buildable as an evolution rather than three products.**

The three canonical models (mapped to the SPEC tiers):

- **ALT-1 — Row-level / shared (pooled).** One shared compute + one shared store; the tenant boundary is a column + predicate (+ RLS). = SPEC **T1**.
- **ALT-2 — Namespace-per-tenant (bridge / hard-logical).** Shared cluster + shared operator; per-tenant K8s namespace, DB schema/bucket, key, NetworkPolicy. = SPEC **T2**.
- **ALT-3 — Cluster/instance-per-tenant (silo).** Dedicated cluster or a fully separate install per tenant. = SPEC **T3/T4**.

(Industry terms: pooled / bridge / silo. SaaS literature, AWS SaaS lens.)

---

## 1. ALT-1 — Row-level / shared (pooled)  ·  = T1

### Design
Single install. Every tenant-bearing row/object carries `tenant` (D2 `authz` block). Reads/writes pass the D2 ScopePredicate interceptor; relational tables also carry **RLS** (mandatory per G5-R1); object-store evidence uses a per-tenant key prefix + tenant-scoped credential (G5-R2). Compute (ingest, simulation, analytics) is shared, governed by per-tenant quotas (G5-R8/R9). One signing key cluster, but a **per-tenant key id** in the chain (G5-R3) so crypto-shred works.

### Isolation strength
**Weakest.** The boundary is logical. Defense-in-depth (interceptor + RLS + per-tenant key prefix + per-tenant encryption key) makes a *single* bug non-catastrophic, but the substrate is shared: a sufficiently deep platform bug, a privileged-credential leak, or a side-channel can cross tenants. **A compliance auditor will not accept "your audit log shares a database with other customers" for a regulated workload** — this is the board's "unsellable to a regulated buyer" finding, and it is correct *for this model in isolation*.

### Cost
**Lowest.** One store, one compute pool, one key infra, one upgrade. Marginal cost of tenant N+1 ≈ storage + quota. This is why it's the right *floor* and the right model for SMB / non-regulated / design partners.

### Ops
**Simplest.** One install to run, patch, back up, monitor. Day-2 burden does not scale with tenant count. This is the model that keeps G6/G2 sane at high tenant counts.

### Failure mode
Forgotten predicate / bad aggregate (XD-5) / forged claim → **all-tenant** exposure of *evidence*. Mitigated, not eliminated, by RLS + per-tenant keys + registry-bound claims. Residual = catastrophic-if-it-fails, low-probability-with-defense-in-depth.

### Verdict
**Correct as the floor (T1) and for non-regulated tenants; wrong as the only model.** Sellable to SMB, never to regulated.

---

## 2. ALT-2 — Namespace-per-tenant (bridge / hard-logical)  ·  = T2

### Design
One cluster, one operator, one control-plane codebase — but each tenant gets:
a dedicated **K8s namespace** (per-tenant analytics/replay pods, ResourceQuota, LimitRange, NetworkPolicy), a dedicated **DB schema** (or database) and **object-store bucket**, a dedicated **signing/encryption key** (G4), and a registry record (`isolation_tier: T2`). The D2 ScopePredicate still runs *inside* each tenant's partition (defense in depth), but a forgotten predicate now hits an **empty schema/bucket**, not another tenant. Compute is bounded per-tenant by ResourceQuota; the global analytics view is the only deliberately cross-tenant path.

### Isolation strength
**Strong-logical.** Storage is physically separated (schema/bucket); compute is namespace-bounded; network is NetworkPolicy-bounded; keys are per-tenant. No *single* layer is the boundary. A predicate bug is contained; a credential leak is NetworkPolicy-contained; a key leak decrypts one tenant. **This is the minimum a regulated mid-market buyer accepts** — "my data is in my own schema/bucket, encrypted with my own key, in my own namespace." It answers the security questionnaire honestly with "yes, isolated" at the logical-but-physically-separated level.

### Cost
**Moderate, sub-linear.** Shared cluster + shared operator + shared control-plane code amortize across tenants; per-tenant cost = a schema + a bucket + a key + namespace quota headroom. Scales to ~hundreds of tenants per cluster before namespace/schema sprawl and noisy-neighbor-at-the-cluster-level bite. Materially cheaper than a cluster each.

### Ops
**Moderate.** Still **one** install to upgrade (the big win vs ALT-3 — no per-tenant upgrade choreography). Day-2 burden grows with tenant count but only in *config objects* (namespaces, quotas, schemas), not in *running stacks*. Per-tenant provisioning is the operator's job (G5-R21). Schema/bucket migrations must fan out per tenant — a real but bounded cost.

### Failure mode
A bug crosses at most a misconfigured tenant pair (e.g. a wrong schema binding), not all tenants. Cluster-level resource exhaustion can still affect co-tenants if ResourceQuota is mis-set (G5-R8 mitigates). Operator bug during provisioning could leave a tenant half-isolated (G5-R21 idempotence + "stay `provisioning` until all planes confirmed" mitigates).

### Verdict
**The keystone tier.** This is the model that makes the product *sellable to regulated buyers* without the cost/ops explosion of ALT-3. **The single most important architectural target of G5.** If only one hard tier could be built, it is this one.

---

## 3. ALT-3 — Cluster/instance-per-tenant (silo)  ·  = T3/T4

### Design
Each tenant gets a **dedicated cluster / node-pool (T3)** or a **fully separate install** of the whole 14-service stack (T4), optionally self-hosted / air-gapped / customer-managed keys (BYOK). No shared substrate. The global control plane, if any, holds only tenant *metadata* (which tenants exist), never tenant evidence.

### Isolation strength
**Strongest — physical.** No shared store, compute, network, or key. A bug in tenant A's install cannot reach tenant B by construction. This is what data-sovereign, top-tier-financial, and air-gapped buyers require. Side-channels are closed physically.

### Cost
**Highest, linear (or worse).** Tenant N+1 = a whole new stack: its own Keycloak, OPA, Gatekeeper, operator, evidence store, analytics, simulation, console, API (meta-adversarial Risk #10's 14 services). Run-cost and $/tenant balloon. This is the F3 "build/run everything per tenant" trap if used as the *default*.

### Ops
**Heaviest.** Upgrade choreography across N independent 14-service stacks is the board's Risk #10 made N times worse. On-call surface, SRE headcount, and upgrade runbooks scale linearly with tenant count. **Cross-tenant analytics (HL-13 "platform-wide attack patterns") becomes impossible** — there is no shared plane to aggregate over (you'd have to ship metadata out of each silo, which a data-sovereign tenant may forbid).

### Failure mode
Isolation failures are near-impossible by construction; the failure mode shifts to **operational** — a missed upgrade on one silo, config drift across silos, an un-patched tenant. The risk moves from "data leak" to "inconsistent / stale / unpatched fleet."

### Verdict
**Right for the top tier (T3/T4) on demand; catastrophic as the default.** Reserve for tenants whose regulatory/sovereignty posture *requires* physical isolation and who pay for it.

---

## 4. Comparison

| Dimension | ALT-1 Row-level (T1) | ALT-2 Namespace (T2) | ALT-3 Cluster/Instance (T3/T4) |
|---|---|---|---|
| Isolation strength | Logical (weakest) | Strong-logical (physical store/key/net) | **Physical (strongest)** |
| Storage isolation | shared store + RLS + key prefix | **per-tenant schema/bucket + key** | dedicated store |
| Compute isolation | shared + quota | namespace ResourceQuota | **dedicated** |
| Key isolation | per-tenant key id, shared signer | **per-tenant key** | dedicated/HSM/BYOK |
| Network isolation | shared (mTLS) | per-tenant NetworkPolicy | **dedicated network** |
| Blast radius of a scope bug | **all tenants' evidence** | one tenant pair (contained) | none (no shared plane) |
| Noisy-neighbor exposure | high (quota-only) | medium (ResourceQuota) | **none** |
| Side-channel (shared OPA/analytics) | present (disclosed) | reduced (per-tenant pods) | **none** |
| Cross-tenant analytics (HL-13) | native (and a risk) | native via privileged global view | **impossible** |
| Data residency pinning | per-row region tag (fragile) | per-bucket/schema region (**clean**) | per-install region (**cleanest**) |
| Provable deletion | crypto-shred only | crypto-shred + schema/bucket drop | **physical teardown** |
| Cost / tenant | **lowest** | moderate (sub-linear) | highest (linear) |
| Ops / upgrade burden | **one install** | one install + per-tenant config | **N installs** (worst) |
| Scales to | thousands (cost-bound) | hundreds/cluster | tens (ops-bound) |
| Sellable to regulated buyer | **No** | **Yes (minimum bar)** | Yes (premium) |
| Maps to SPEC tier | **T1** | **T2** | **T3 / T4** |

---

## 5. Why a single model is wrong — and the dial is right

- Pick **ALT-1 only** → unsellable to the buyer the product exists for (regulated compliance). The board's headline.
- Pick **ALT-3 only** → cost/ops explosion (Risk #10 × N), cross-tenant analytics impossible (HL-13 dead), F3 "everything on spec" trap.
- Pick **ALT-2 only** → closest to right, but you still can't serve the SMB/design-partner long tail economically, and you can't serve the data-sovereign top tier — and you've hard-coded "namespace" as the boundary, so a tenant that later demands a dedicated cluster forces a re-architecture.

**The dial (D-G5-1) is right because tenancy strength is a *per-customer regulatory* attribute, not a *platform* attribute.** Different customers of the *same* product legitimately need different isolation, and a single customer's needs change (a tenant gets acquired by a bank → T1→T3). A single global model forces a re-architecture every time; the dial makes it a per-tenant migration.

### What makes the dial buildable as one product, not four (the load-bearing claim)

The dial only works if **the inner layers are identical at every tier and the outer layers are *added*, never *swapped*.** Three indirections make this true:

1. **The D2 `ScopePredicate` is the innermost layer at *every* tier.** At T1 it's the boundary; at T2–T4 it's defense-in-depth inside an already-separated partition. It is never removed, so code paths don't fork by tier.
2. **A single `TenantStore` storage abstraction** resolves `(tenant_id) → physical location` from the registry: at T1 it returns `(shared_db, schema=public, predicate)`; at T2 `(shared_db, schema=tenant_x, bucket=…)`; at T3/T4 `(tenant_x_instance)`. Callers never branch on tier — they ask the abstraction for "tenant X's store" and get a correctly-isolated handle. The same applies to a `TenantCompute` (worker pool vs namespace vs cluster) and `TenantKey` (G4) abstraction.
3. **The operator provisions the tier declaratively** (`isolation_tier` in the registry / a `Tenant` CR). Promoting a tenant is a registry edit + a reconcile + a data migration — not a code change.

This is the same evolve-in-place pattern D2's own ALT uses for its enforcement mechanism (`ScopePredicate` indirection so interceptor→RLS→OPA is a swap, not a rewrite). G5 applies the identical discipline one layer up, at the tenancy-realization layer.

---

## 6. Recommendation (matches SPEC D-G5-1)

1. **POC: T0** (single-tenant) — don't pay for any tenancy machinery before there's a second tenant.
2. **First multi-tenant release: T1 default + T2 first-class.** Build the registry, the `TenantStore`/`TenantCompute`/`TenantKey` abstractions, RLS-mandatory, the scoped analytics path (XD-5), and the operator provisioning for T1 and T2. Sell T1 to SMB, **T2 to regulated**.
3. **T3/T4 on demand**, behind the same abstractions, for data-sovereign / top-financial tenants — built when the first such contract is signed, not on spec.
4. **Never** expose T1 to a buyer as "isolated" (G5-R19/R20); the registry-derived questionnaire answer enforces honesty.

**Bottom line.** The three models are not competitors; they are *positions on one dial*. Build the dial (registry + three storage/compute/key abstractions + operator provisioning), ship T1+T2 first because that pair covers SMB-through-regulated at sane cost/ops, and reach for T3/T4 only when a contract demands physical isolation. The `ScopePredicate`-inside-everything + `TenantStore`-abstraction discipline is what makes this an evolution rather than four products.
