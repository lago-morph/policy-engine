# G3 — ALT: Disaster-Recovery Topology Trade Study

**Alternative-architecture deliverable (high-value).** Date: 2026-05-30. Pairs with `SPEC.md` §7 (multi-region) and §3 (RPO/RTO). This document presents the **three genuinely different DR topologies** for the D0 evidence store — the durability-critical, RPO-sensitive tier — and the trade between **RPO × RTO × cost × complexity**. It coordinates conceptually with **G2** (cost model — G2 owns the dollar figures; this doc owns the *shape* and the *relative* cost) and **G4** (key custody — each topology has a different key-availability requirement at the failover/restore moment).

> **Why this is an ALT, not just SPEC §7.** The choice of DR topology is the single largest cost and complexity lever in the whole platform (each full D0 copy is a growing, 7-year-retained, cryptographically-signed evidence store), and it directly determines whether the product's headline durability promise (RPO=0 on the audit log) is *true*. Getting it wrong in either direction — over-building active-active for a dev tenant, or under-building backup-only for a regulated bank — is fatal commercially or for compliance. This deserves a first-class trade study, not a buried default.

---

## 0. The decision being made

For the **D0 evidence store** (and only D0 — D1/D2/D3 follow it or are rebuildable, SPEC §2), choose one of:

- **T0 — Backup-restore only** (single active region, no standby): the cheapest, the SPEC's *minimum*.
- **T1 — Active-passive warm standby** (cross-region async replication): the SPEC's **default recommendation (profile P2)**.
- **T2 — Active-active multi-region** (synchronous cross-region quorum): the SPEC's **regulated profile (P3)**, RPO=0 even on region loss.

The primary SPEC picks **T1/P2 as the general default** and **T2/P3 as a SHOULD-escalate-to-MUST for regulated buyers**. This ALT pressure-tests that choice and — informed by the ADVERSARIAL-REVIEW A-3a finding — **argues the default is wrong for the actual target market** and proposes a **fourth, blended option (T3)** as the recommendation.

---

## 1. The three primary topologies

### T0 — Backup-restore only (single region, WORM backup off-region)

```
   Region A (active)                         Off-region object storage
   ┌─────────────────────┐                   ┌──────────────────────────┐
   │ D0 quorum (≥3 AZ)    │ ──checkpoint──▶   │ WORM / Object-Lock backup │
   │ RPO 0 to zone loss   │   segments (≤15m) │ (immutable, separate cred)│
   └─────────────────────┘                   └──────────────────────────┘
   Region loss ⇒ rebuild a new region from WORM backup (restore-to-checkpoint).
```

- **RPO:** 0 to zone loss; **≤15 min** (last checkpoint) to region loss.
- **RTO:** **hours** (provision a new region, restore the full chain, run §5.7 validation gate). Grows with chain size (ADVERSARIAL A-7).
- **Cost (relative):** **1.0×** baseline (one live store + cheap cold WORM backup). The WORM backup is cheap object storage, not a running second cluster.
- **Complexity:** **Lowest.** No replication, no failover orchestration, no split-brain risk (one writer, ever).
- **Split-brain risk:** **None** — there is only ever one active region. This is a real, underrated advantage: T0 is the only option that *cannot* fork the chain (ADVERSARIAL A-1 doesn't apply).
- **Key custody (G4):** simplest — keys needed only at restore time, can be re-fetched from G4's own restore.
- **Verdict:** correct for **POC and C-LOW / non-regulated** data. **Unacceptable for regulated D0** because a multi-hour RTO means the system of record is *down* (no enforcement evidence captured for new decisions during the hours-long rebuild — though spokes buffer, the buffer is finite) and the ≤15-min RPO loses evidence.

### T1 — Active-passive warm standby (cross-region async) — *SPEC default P2*

```
   Region A (active, primary)        async replication (≤60s lag)      Region B (warm standby)
   ┌─────────────────────┐ ════════════════════════════════════▶ ┌─────────────────────┐
   │ D0 quorum (≥3 AZ)    │                                        │ D0 replica (warm)    │
   │ serves all writes    │                                        │ promotable on failover│
   └─────────────────────┘                                        └─────────────────────┘
   Region A loss ⇒ FENCE A, promote B (head = last replicated checkpoint), restore_boundary for lag gap.
```

- **RPO:** 0 to zone loss; **≤60 s** (async lag) to region loss — a *disclosed, reconcilable* gap (SPEC R-G3-RPO-2).
- **RTO:** **≤4 h** — mostly fence + promote + restore-tail reconciliation + validation gate; faster than T0 because the standby is already warm (no provision-and-full-restore).
- **Cost (relative):** **~1.8–2.0×** (a second, continuously-running replica region — full second copy of the growing store + cross-region egress for replication). G2 prices the egress + second-store growth.
- **Complexity:** **High.** Requires failover orchestration, fencing, lag monitoring, and — per ADVERSARIAL A-1a — **a real fencing mechanism (external epoch arbiter), not just a runbook**, or it risks split-brain. This is the costly hidden complexity the SPEC under-specified.
- **Split-brain risk:** **Real and is the default-topology hole (ADVERSARIAL A-1a)** — async means the standby is a *separate* quorum that *can* commit independently after promotion, so a gray-failure/zombie-primary can fork the chain. **Mitigable only with the epoch-arbiter fix.**
- **Key custody (G4):** keys must be available in *both* regions for the standby to sign continuation checkpoints post-promotion; G4 must replicate key material (or at least continuation-signing capability) to Region B.
- **Verdict:** the SPEC's general default. **But its RPO>0 makes it invalid for the regulated buyer (A-3a), and its split-brain exposure is the most expensive complexity in the platform.** It is the "middle" option that carries *most* of the cost of T2 (a second running region) while *not* delivering T2's RPO=0 or T2's split-brain immunity. **The classic worst-of-both-worlds risk.**

### T2 — Active-active multi-region (synchronous quorum) — *SPEC regulated P3*

```
   Region A ◀══════ synchronous write-quorum spans regions ══════▶ Region B  (+ optional C)
   ┌─────────────────────┐                                        ┌─────────────────────┐
   │ D0 quorum member(s)  │   a write commits only when a majority │ D0 quorum member(s)  │
   │                      │   across regions acknowledges          │                      │
   └─────────────────────┘                                        └─────────────────────┘
   Region loss ⇒ surviving majority keeps serving; RPO=0; no promotion step (already active).
```

- **RPO:** **0 even on region loss** (commit requires cross-region majority, so any committed event survives any single region's loss). The one topology that makes the headline promise true.
- **RTO:** **≤15 min** (no promotion — the surviving members already serve; just shed the dead region from quorum).
- **Cost (relative):** **~2.5–3.0×** (≥3 regions for a cross-region majority that survives one region loss, full copies in each, plus the *latency tax*: every D0 commit pays cross-region round-trip — which at production write volume is also a *throughput* cost, G1/G2).
- **Complexity:** **Highest at write time, lowest at failover time.** No failover orchestration (nothing to promote), no split-brain (the single cross-region quorum *is* the linearization — a minority partition genuinely cannot commit, which is exactly why ADVERSARIAL A-1a says the split-brain guarantee *only* holds here). The complexity moves from "failover runbook" to "tolerate cross-region commit latency on the hot path."
- **The latency tension (ADVERSARIAL A-3b):** synchronous cross-region commit on the evidence path adds tens-to-hundreds of ms per commit. The admission hot path (B5-R6 ≤2s) does **not** wait for D0 commit (evidence emission is parallel/non-blocking, B5 §2 t5), so this latency hits *ingest throughput and the buffer-drain rate*, not admission latency — an important nuance: **T2's latency tax is on evidence durability, not on enforcement speed**, which makes it more affordable than it first appears for *this* product (the hot path is already decoupled from D0 commit).
- **Key custody (G4):** keys / continuation-signing must be live in every region (any region may be the surviving writer); strongest G4 coupling.
- **Verdict:** the **correct topology for regulated D0** and, per ADVERSARIAL A-3a, what the regulated *default* should be. RPO=0, no split-brain, and the latency tax lands on the decoupled ingest path rather than admission. Its cost (~3×) is the price of the product's central promise being true for the buyer who pays for it.

---

## 2. The trade table

| Axis | **T0 backup-only** | **T1 warm standby (async)** | **T2 active-active (sync)** |
|---|---|---|---|
| **RPO, zone loss** | 0 | 0 | 0 |
| **RPO, region loss** | ≤15 min | ≤60 s (disclosed gap) | **0** |
| **RTO, region loss** | hours (provision+restore) | ≤4 h (promote+reconcile) | ≤15 min (shed dead region) |
| **Relative cost** | **1.0×** | ~1.8–2.0× | ~2.5–3.0× |
| **Write-path latency tax** | none | none (async) | cross-region RTT *on ingest, not admission* |
| **Operational complexity** | lowest | **highest** (failover + fencing + lag) | high-at-write, low-at-failover |
| **Split-brain risk** | **none** (single writer) | **real** (needs epoch arbiter — A-1a) | **none** (single cross-region quorum) |
| **Regulated-buyer acceptable?** | No (RPO>0 + slow RTO) | **No** (RPO>0 — A-3a) | **Yes** (RPO=0) |
| **POC / C-LOW acceptable?** | **Yes** | overkill | overkill |
| **Key custody (G4) coupling** | restore-time only | keys in 2 regions | keys live in all regions |
| **Makes the headline "RPO=0 audit log" true?** | only intra-region | only intra-region | **yes, end-to-end** |

---

## 3. The recommendation — a blended T3, not a single pick

The SPEC defaults to T1 and offers T2 as an election. The ADVERSARIAL review (A-3a) shows **T1's RPO>0 is invalid for the target (regulated) buyer**, and the trade table shows **T1 is the worst-of-both-worlds**: it carries ~2× cost (a full running second region) yet delivers neither T2's RPO=0/split-brain-immunity nor T0's simplicity/zero-split-brain. **T1's only real win is RTO (≤4h vs T0's hours), bought at high cost and high complexity for a still-non-zero RPO.**

### T3 (recommended) — data-class-tiered, criticality-routed topology

Apply **different topologies to different data, by criticality**, rather than one topology for the whole platform:

- **D0 evidence in C-CRITICAL / regulated scopes → T2 (active-active sync, RPO=0).** This is the *only* data that needs it, and it is the data the compliance thesis depends on. The ~3× cost applies only to the regulated subset, not the whole estate.
- **D0 evidence in C-STANDARD / C-LOW / non-regulated scopes → T0 (backup-only).** Cheap, simple, no split-brain. RPO ≤15m and multi-hour RTO are fine for non-regulated decision logs.
- **D1 config / D2 derived → T0 everywhere** (config is GitOps-recoverable; derived is rebuildable from D0).
- **Skip T1 entirely as a *standalone* default.** Offer it only as a deliberate election for the narrow buyer who wants cross-region resilience but genuinely cannot pay the T2 latency/cost *and* can tolerate a disclosed ≤60s RPO — a real but small segment. It is never the *default*.

**Why T3 beats the SPEC's "T1-default, T2-election":**
1. **Cost lands where the value is.** The expensive RPO=0 (~3×) is paid only for the regulated D0 that justifies it; the bulk non-regulated estate stays at 1.0× (T0). This is materially cheaper than T1-everywhere (~2×) while being *more* correct for the regulated subset.
2. **Eliminates the split-brain exposure for most data.** T0 cannot fork; T2's single cross-region quorum cannot fork. **T3 removes the platform's split-brain risk (A-1a) almost entirely** by routing away from T1, the only topology that has it.
3. **The default is correct for the buyer.** Regulated data defaults to RPO=0 (fixes A-3a); non-regulated data defaults to cheap. No buyer is silently mis-served by a single global default.
4. **It honors the data-class taxonomy (SPEC §2) that already exists** — the criticality class already drives the breaker (§4.1); reusing it to drive DR topology is consistent and requires no new classification act.

**The cost of T3:** more operational *heterogeneity* (two topologies to run instead of one) and a routing layer that places each source-chain in the right topology by its scope's criticality. That heterogeneity is the price; it is justified because the alternative (one global topology) is either too expensive (T2-everywhere), wrong for the buyer (T1-everywhere, A-3a), or too weak (T0-everywhere, fails regulated).

---

## 4. Coordination seams (explicit)

- **→ G2 (cost):** G2 prices the relative multipliers (1.0× / ~2× / ~3×) into absolute dollars at G1's production volume × 7-year retention, *per topology*, and prices T3's tiered blend (regulated-subset-at-3× + bulk-at-1×). The recommendation that T3 is cheaper-than-T1-everywhere-yet-correct **must be confirmed by G2's actual numbers** — this ALT asserts the *shape* of the saving; G2 owns the magnitude. (ADVERSARIAL A-8b: the default must not be finalized before G2's model exists.)
- **→ G4 (key custody):** each topology has a different key-availability requirement at failover (T0: restore-time; T1: 2-region; T2: all-region). G4 must support **multi-region availability of the continuation-signing capability** for T2, and the **public verification keys must be embedded in every D0 backup** (ADVERSARIAL A-8a) so restore-verify is self-contained even if G4's live store shares the disaster.
- **→ G1 (scale):** restore RTO and the cross-region commit latency tax are both **volume-sensitive** (A-7). G1's production volume model sizes them; the T2 latency-vs-throughput trade and the T0/T1 restore-RTO-vs-chain-size curve are G1 inputs to this choice.
- **→ SPEC §7 / §3:** this ALT *replaces* the SPEC's "T1-default" recommendation with **T3 (criticality-tiered)**; SPEC R-G3-MR-1's three profiles (P1/P2/P3) remain the deployable *primitives*, and T3 is the recommended *composition* of them by data class. The SPEC's default (OQ-G3-1 = P1) is correct *as the POC default*; T3 is the *production* recommendation.

---

## 5. One-paragraph bottom line

The primary SPEC's instinct — RPO=0 intra-region, a cross-region story, and an explicit profile choice — is right, but its **default (T1 warm-standby) is the worst-of-both-worlds**: ~2× the cost for a still-non-zero RPO that the regulated buyer (the actual customer) cannot accept, plus the platform's only split-brain exposure. The better architecture is **T3: route DR topology by data criticality** — **T2 active-active (RPO=0) for regulated D0** (the data that earns the cost and where RPO=0 is the product promise), **T0 backup-only for everything else** (cheap, simple, fork-proof), and **T1 only as a niche election, never the default.** This is cheaper than T1-everywhere, correct for the regulated buyer, and removes the split-brain risk for the vast majority of the estate. The dollar magnitudes must be confirmed by G2 and the key-availability and scale curves by G4/G1, but the *shape* — pay for RPO=0 exactly where compliance requires it and nowhere else — is the defensible choice.
