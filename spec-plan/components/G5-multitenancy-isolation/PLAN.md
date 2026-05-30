# G5 — Multi-Tenancy Isolation — PLAN

**Scope of this plan.** Build the per-tenant tenancy *dial* (T0→T4) as one product (per `ALT-tenancy-models.md` §5), wire the per-plane isolation requirements (SPEC §4) into the existing components (D1, D2, C2, C3/C5, F2, G1/G3/G4/G7), and deliver the **soft→hard migration path** that lets a single tenant move T1→T2→T3→T4 in place without re-architecting. Parallelism is exploited where planes are independent; the critical path is the storage/registry abstraction that everything else binds to.

---

## 1. Dependency DAG

```
                       ┌──────────────────────────────────────────────┐
                       │ G5-W0  Tenant Registry + isolation_tier dial  │  ← critical-path root
                       │ (entity, server-side store, audited, dual-ctl)│
                       └───────────────┬──────────────────────────────┘
                                       │ (every plane resolves tier+location from here)
        ┌──────────────┬──────────────┼───────────────┬───────────────┬──────────────┐
        ▼              ▼              ▼               ▼               ▼              ▼
  ┌───────────┐  ┌───────────┐  ┌───────────┐   ┌───────────┐   ┌───────────┐  ┌───────────┐
  │ W1 Store  │  │ W2 Compute│  │ W3 Identity│  │ W4 Analytics│ │ W5 Residency│ │ W6 Network│
  │ abstraction│ │ abstraction│ │ integrity  │  │ isolation   │ │ pinning     │ │ isolation │
  │ (T1 RLS +  │  │(quota+fair │  │(claim↔reg, │  │ (XD-5: scope│ │(region tag, │ │(NetPolicy │
  │ obj prefix │  │ queue over │  │ issuer bind│  │ analytics + │ │ pin writes/ │ │ per-tenant│
  │ + per-ten  │  │ serial E1  │  │ G5-R13/14) │  │ global view │ │ replicas/   │ │ ns; T2+)  │
  │ key)       │  │ core)      │  │            │  │ G5-R5/R7)   │ │ restores)   │ │           │
  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘   └─────┬─────┘   └─────┬─────┘  └─────┬─────┘
        │              │              │               │               │              │
        └──────────────┴──────────────┴───────────────┴───────────────┴──────────────┘
                                       ▼
                       ┌──────────────────────────────────────────────┐
                       │ W7  Operator tenant provisioning (T1, T2)     │  ← turns the dial physical
                       │ (Tenant CR / registry reconcile; idempotent;  │
                       │  provisions ns/quota/netpol/schema/bucket/key)│
                       └───────────────┬──────────────────────────────┘
                                       ▼
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
  ┌───────────┐                 ┌───────────┐                  ┌───────────┐
  │ W8 Lifecycle│               │ W9 Migration │                │ W10 Conformance│
  │ onboard/    │               │ T1→T2→T3→T4  │                │ suite (DT-55   │
  │ suspend/    │               │ in place     │                │ × every backend│
  │ offboard+G7 │               │ (no re-arch) │                │ + analytics +  │
  │ deletion    │               │              │                │ forged claim + │
  │             │               │              │                │ noisy-neighbor)│
  └───────────┘                 └───────────┘                  └───────────┘
                                       │
                                       ▼
                       ┌──────────────────────────────────────────────┐
                       │ W11  T3/T4 (cluster/instance) — on demand only │
                       └──────────────────────────────────────────────┘
```

---

## 2. Workstreams

### W0 — Tenant registry + the dial (CRITICAL PATH ROOT)
The single server-side entity (SPEC §5) every plane resolves against. Build first; everything binds to it.
- Registry schema (`tenant_id, isolation_tier, trusted_issuers, residency, storage, quotas, lifecycle, scope_defaults`).
- Server-side store; **audited, dual-controlled edits** (G5-R17, via D3).
- `isolation_tier` and `residency`/`trusted_issuers` are **upgrade-only with migration** (G5-R18).
- **Blocks:** all of W1–W10. **Independent of:** nothing.

### W1 — Storage abstraction (`TenantStore`) + T1 isolation
- `TenantStore(tenant_id) → (relational handle scoped, object-store handle scoped)` resolving location from registry.
- **RLS-under-interceptor mandatory** on relational tables (G5-R1; resolves D2-A1/A13/XD-5). CI/startup check: a tenant table without RLS = build break.
- Object-store: per-tenant key prefix + per-request tenant-scoped credential (G5-R2/OQ-G5-4).
- Per-tenant encryption key wiring (G5-R3, depends on G4 key abstraction).
- **Per-tenant C2 hash chain + per-tenant signing key** (G5-R26/OQ-G5-3) — *imposes a C2 contract change; coordinate with C2 reconciliation early.*
- **Parallel with:** W2–W6. **Depends on:** W0, G4 (key refs).

### W2 — Compute abstraction + noisy-neighbor controls
- `TenantCompute` (shared pool w/ quota at T1; namespace at T2).
- Per-tenant quotas/concurrency via **G1** mechanism (G5-R8).
- **Per-tenant fair queue + max-in-flight over the serial E1 replay core** (G5-R9) — the single most important noisy-neighbor control (serial core = narrowest resource).
- **Parallel with:** W1, W3–W6. **Depends on:** W0, G1 (rate/quota primitive).

### W3 — Tenant identity integrity (binds tenancy to the registry, not the claim)
- D1: validate tenant claim against registry (G5-R13); reject if unknown → `incomplete`/fail closed.
- **Issuer→tenant binding** (G5-R14): reject cross-tenant assertions; `auth_denied{tenant_issuer_mismatch}`.
- Physical partition keyed on *registry* tenant, not the raw claim (G5-R15).
- **Parallel with:** W1/W2/W4/W5/W6. **Depends on:** W0, D1.

### W4 — Analytics/reporting isolation (XD-5 — the largest escape surface)
- C3/C5 read paths link the **same** ScopePredicate (G5-R4); no unscoped storage path.
- Default **per-tenant** analytics at T2+ (G5-R7).
- Cross-tenant **global view** as an explicitly-privileged, audited, k-anonymized, residency-respecting path (G5-R5); preserves HL-13.
- Per-tenant evidence-quality denominator (G5-R6, XD-6 cross-ref).
- **Parallel with:** W1–W3/W5/W6. **Depends on:** W0, W1 (TenantStore), C3/C5.

### W5 — Residency / regional pinning
- Residency attribute enforced at write/replicate/restore (G5-R10); fail closed on violation.
- Region-pinned spokes/storage in F2 hub-and-spoke (G5-R11); control plane sees metadata only.
- Cross-region aggregation exclusion (G5-R12).
- **Parallel with:** W1–W4/W6. **Depends on:** W0, F2 topology, G3 (backups respect region).

### W6 — Network isolation (T2+)
- Per-tenant NetworkPolicy/mesh authz (G5-R16) so a leaked credential can't move laterally.
- **Depends on:** W0, F2; **Parallel with** W1–W5; **only materializes at W7 (T2 provisioning).**

### W7 — Operator tenant provisioning (T1 + T2)
- Declarative, idempotent provisioning (`Tenant` CR / registry reconcile) per tier (G5-R21): T1 = RLS+prefix+key; T2 = namespace+quota+NetworkPolicy+schema+bucket+key.
- Tenant stays `provisioning` until **all** planes for the tier confirm (never half-isolated).
- **Depends on:** W0–W6 contracts, F2 operator.

### W8 — Lifecycle: onboard / suspend / offboard + deletion
- Onboard (§7.1) wires W7; suspend (§7.2) freezes writes, preserves chain; offboard (§7.3) two-phase, retention-aware, **crypto-shred via G7+G4** (G5-R23/24/25), no cross-tenant collateral (G5-R26), signed deletion certificate.
- **Depends on:** W7, G7 (erasure machinery), G4 (key destruction), G3 (suspended≠deleted).

### W9 — Migration T1→T2→T3→T4 in place (the soft→hard path)
- See §4. **Depends on:** W1/W2/W7 abstractions; W0 upgrade-only tier rule.

### W10 — Conformance suite
- DT-55 × every backend (relational + **object-store log**) + analytics path + forged/confused-claim vectors + noisy-neighbor saturation (G5-R27); per-tier acceptance gate (G5-R28). P0 on regression.
- **Depends on:** W1–W6 (things to test); runs continuously thereafter.

### W11 — T3/T4 (cluster/instance) — on demand
- Dedicated cluster/node-pool / separate install behind the same abstractions. Built only when a contract requires physical isolation. **Depends on:** W7, W9.

---

## 3. Critical path

```
W0 (registry/dial)
  → W1 (TenantStore + RLS + per-tenant chain/key)        ← longest pole (C2 contract change + RLS + obj-store)
    → W7 (operator provisioning T1/T2)
      → W8 (lifecycle incl. crypto-shred deletion)
        → W10 conformance (per-tier acceptance gate)  ⇒ first multi-tenant release (T1 default + T2 for regulated)
```
W2–W6 run **in parallel** off W0 and rejoin at W7. W9 (migration) and W11 (T3/T4) are post-first-release. The two riskiest items on the critical path: **(a) the per-tenant C2 hash chain + key** (OQ-G5-3 — a C2 contract change, must be agreed in C2 reconciliation *before* W1 hardens), and **(b) object-store evidence isolation** (OQ-G5-4 — no RLS to lean on; the per-tenant credential/prefix design must be pen-tested, not assumed — G-3 falsifier).

---

## 4. Migration path — soft (T1) → hard (T2/T3/T4) without re-architecting

The dial is only worth building if a *single tenant* can move up without a rewrite. Mechanism (per `ALT` §5.5): the inner `ScopePredicate` is identical at every tier; outer layers are *added*. Migration is therefore a **data move + a reconcile**, never a code fork.

### 4.1 T1 → T2 (shared → namespace-per-tenant) — the important one
1. Dual-controlled registry edit: `isolation_tier: T1 → T2` (G5-R18, upgrade-only, audited).
2. Operator (W7) provisions the tenant's namespace + ResourceQuota + NetworkPolicy + **new DB schema** + **new bucket** + **new per-tenant key** (G4) — tenant stays `migrating`, still served by T1 paths.
3. **Copy-with-verify:** copy the tenant's rows (shared schema → tenant schema) and blobs (shared prefix → tenant bucket), re-encrypt under the new per-tenant key, **re-chain the tenant's per-tenant hash chain** in the new location, verify the copied chain end-to-end (this is *why* the chain is per-tenant from day one — OQ-G5-3 — so the move is a per-tenant operation, not a global re-chain).
4. **Cutover:** flip `TenantStore(tenant)` to resolve the new location (registry); the `ScopePredicate` and all callers are unchanged (they ask the abstraction, not the tier).
5. **Verify + reap:** run the T2 conformance gate (W10) against the new partition; only on pass mark `active@T2`; then delete the tenant's rows/blobs from the shared store (crypto-shred the old slice). Rollback = keep the T1 slice until the T2 gate passes.

> No application code changes. The migration is data-plane only because the storage/compute/key abstractions hide the tier from callers (`ALT` §5).

### 4.2 T2 → T3 (namespace → dedicated cluster/node-pool)
Registry edit; operator provisions a dedicated cluster/node-pool; the tenant's schema/bucket/key (already separate from T1→T2) are *relocated* to the dedicated store; NetworkPolicy → dedicated network. Because storage/key were *already* per-tenant at T2, this move relocates an already-isolated unit — strictly easier than T1→T2.

### 4.3 T3 → T4 (dedicated cluster → separate install)
Stand up a separate hub for the tenant; relocate the (already-dedicated) store/key; repoint the tenant's spoke. The tenant exits the shared control plane; only metadata (that the tenant exists, in which region) optionally remains, or is removed for data-sovereign tenants.

### 4.4 Why this is not a re-architecture
Each step *adds* a separation boundary outside an already-correct inner boundary. At no point is the `ScopePredicate` removed or the data model changed. The migration cost is **data movement + verification**, which scales with one tenant's data, not the platform. The hardest engineering — per-tenant chain/key and the three abstractions — is paid *once*, in W0/W1/W2, for *all* tiers. Downgrades (T2→T1) are also defined but are the highest-risk op (weaken isolation) and require the same dual-controlled migration + a data-co-mingle step that the conformance gate must bless (G5-R18).

---

## 5. What can be built concurrently / what blocks what

| Can run in parallel | Blocks / blocked by |
|---|---|
| W1, W2, W3, W4, W5, W6 (all off W0) | All blocked by **W0** (registry); W4 also needs W1 (TenantStore); W5 needs F2/G3; W6 materializes only at W7 |
| W10 conformance authored alongside W1–W6 | Executes after the thing it tests exists |
| C2 per-tenant-chain agreement (OQ-G5-3) | **Must precede** W1 hardening — coordinate in C2 reconciliation immediately (off critical path otherwise) |
| W9 migration + W11 T3/T4 | After W7/W8 (first release); not on the first-release critical path |

**Coordination flags raised to cross-cutting reconciliation:**
1. **C2 contract change** — per-tenant hash chain + per-tenant signing key (OQ-G5-3). Without it, deletion-without-collateral (G5-R26) and per-tenant key-blast-radius (G5-R3) are impossible. **Highest-priority external dependency.**
2. **D2 RLS promotion** — RLS-under-interceptor from "optional/later" (D2-OQ-1) to **mandatory** (G5-R1) — already the XD-5 resolution; G5 makes it normative for tenancy.
3. **G1 per-tenant quota** — G5 configures, G1 implements (G5-R8/R9).
4. **G4 per-tenant keys** — G5 requires per-tenant keys at T1+ (not just at T2) so crypto-shred works (G5-R3/R25).
5. **G7 erasure** — G5's offboard invokes G7; the retention-hold interaction (G5-R23) must be jointly owned.
6. **F2 region-pinned spokes** (G5-R11) and **Tenant CR / declarative provisioning** (G5-R21).

---

## 6. Milestones

- **M0:** Registry + dial (W0) + decision ratified; C2 per-tenant-chain agreed.
- **M1:** T1 hard floor — RLS mandatory, object-store per-tenant credential, per-tenant key+chain, scoped analytics (W1, W4 core); DT-55 passes against *both* backends + analytics path (W10 subset).
- **M2:** T2 first-class — operator provisioning, namespace/quota/NetworkPolicy/schema/bucket/key (W2/W6/W7); per-tier conformance gate green; **first multi-tenant release: T1 default + T2 for regulated.**
- **M3:** Lifecycle complete — onboard/suspend/offboard + crypto-shred deletion + signed certificate (W8); residency enforced (W5).
- **M4:** Migration path proven — a real tenant promoted T1→T2 in staging with chain verified end-to-end and zero code change (W9).
- **M5:** T3/T4 on first regulated/sovereign contract (W11).

## 7. Test strategy
- **Per-tier acceptance gate** (G5-R28): no tenant set to a tier until conformance passes for that tier against the *deployed* backend.
- **Continuous DT-55** × {relational, object-store log} × {CRUD, aggregate/analytics, forged-claim, noisy-neighbor saturation} (G5-R27). Any regression = **P0**.
- **Migration test** (M4): copy-with-verify must produce a per-tenant chain that verifies independently and matches pre-migration content hashes; cutover must have **no window** of cross-boundary readability.
- **Deletion proof test:** after offboard, the tenant's at-rest evidence is undecryptable (key destroyed) and a signed deletion certificate verifies; tenant B's chain/analytics are intact (no collateral).
- **Residency test:** a write/replica/restore that would cross the region boundary fails closed; `residency_violation_blocked` emitted.
