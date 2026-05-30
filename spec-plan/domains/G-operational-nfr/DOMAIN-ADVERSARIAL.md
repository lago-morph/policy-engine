# Domain G — Operational / NFR — ADVERSARIAL RECONCILIATION

Domain-level reconciliation of the eight per-component adversarial reviews (`G1…G8/ADVERSARIAL-REVIEW.md`). Purpose: consolidate the CRITICAL/HIGH findings into a single ranked Domain-G register (XG-*), mark each as a **correctness fix** (the design is wrong / the claim is false until fixed) or **hardening** (the design is right; close the residual hole), and surface the items that are really **C2 `v1.0-rc` ratification asks** so they fold into the already-open C2 un-freeze pass (META M-1) rather than rotting as G-backlog.

---

## 1. The convergent finding (every reviewer, independently, lands here)

**Almost every Domain-G defect is a contract change on C2 or D2 that the NFR component does not own — and the dangerous failure mode is that the fix never propagates into the owning SPEC.** This is the META M-1 propagation failure ("the flag sits in the index but the load-bearing doc still says X") reproduced eight times, one level down. G5's review states it as a first-class defect (G5-A14: "G5 is a cross-cutting *contract* dressed as a component"); G6's as dangling runbook pointers (G6-D4); G7's as a key-grain fait accompli (A-3); G1/G3 as un-accepted C2 chain changes. **Reconciled domain decision:** each NFR mandate MUST be propagated into the owning component's SPEC in Wave-3, and the five correctness-class C2 changes (§3) MUST be folded into the C2 `v1.0-rc`. A lint or an abstraction-by-convention is not the enforcement; the owning SPEC is.

---

## 2. The recurring themes (where independent reviews converge)

### Theme A — "Declared vs verified" (XD-6) re-appears at every NFR layer
The platform's cardinal sin recurs as the domain's most common defect class: a label that doesn't mean what it says, or partial coverage presented as total.

| Finding | The lie |
|---|---|
| G2-A1 | the cheapest cost lever ships `best_effort` replay for 90% of traffic under a "full-population replay" banner |
| G7-A1 | a D-G7-03-backed replay carries an unqualified `complete` label that is actually "trust our cached boolean" |
| G5-A9 | the "platform-wide" attack view silently excludes residency-opted-out (often highest-value) tenants |
| G5-A2 | a signed deletion certificate asserts erasure that crypto-shred didn't achieve (derived/backup residue) |
| G6-D1 | the epoch seal launders a pre-existing chain break into a signed "verified" Epoch-0 root |
| G7-A5 | the `erased_input` honesty label launders a genuine capture defect off the SLO |

**Reconciled rule:** every NFR label must be backed by a verifiable signal (a tombstone + key-destruction cert for `erased_input`; a clean pre-flight for the epoch seal; a coverage denominator for the global view; a certificate scoped to what was *actually* destroyed). Partial coverage surfaces its own denominator.

### Theme B — the safety control that is also the attack tool / kill switch
Three independent reviews find a mechanism whose *protective* function is also its *bypass*: the G3 circuit breaker is a remotely-triggerable enforcement kill switch (XG-2); the G3 `restore_boundary` marker is a signed license to truncate the chain (XG-6); the G7 legal-hold is an unbounded permanent erasure-defeat switch (XG-12). **Reconciled rule:** every "downgrade / restore / suspend / hold" primitive needs a *bound* (max-degraded-duration, proportionality + expiry), *symmetric dual-control* (manual open as well as force-close), and an *independent cross-check* (WORM-backup digest, pre-disaster spoke signature, the D0 log) — never a single signed assertion accepted on its face.

### Theme C — POC controls are GA-deferred exactly where the malicious-operator threat lives
G4's three top defects share one root cause: the controls that defend against a *malicious operator* — signer-invoke identity separation, operator-independent root publication, external timestamp anchoring — are all GA-deferred, while the framing implies POC-grade auditor independence. The same pattern: G3's RPO=0 is left as an *election* not the regulated default; G5's T2 is positioned as sales guidance not a correctness rule. **Reconciled rule:** for the *regulated buyer the product is actually sold to*, the malicious-operator-grade control is the **default**, and the POC's honest limitation is stated plainly, not implied away.

---

## 3. The C2 `v1.0-rc` ratification asks (fold into the open C2 un-freeze, do NOT bolt on later)

These five are **correctness-class** and **C2-owned**. Each per-component review explicitly routes them to the C2 reconciliation pass; the domain ratifies that routing. They are the domain's headline deliverable to the cross-cut wave.

| C2-rc ask | From | What C2 must add | Why it can't wait |
|---|---|---|---|
| **Per-source chain *sharding*** by `(source.system, cluster)` + cross-shard roll-up super-checkpoint | G1 (XD-G1-4/5) | chain identity = shard identity; a periodic Merkle-of-shard-roots checkpoint so whole-shard deletion stays detectable | the single chain is the #1 scale ceiling; un-sharded, the evidence spine cannot absorb 100× multi-cluster ingest |
| **`infrastructure_degraded` disposition + `degraded_session_id` + `chain.restore_boundary` / `fork_reconciliation` event types** | G3 (A-8c) | the degraded-mode disposition and the restore/fork markers as ratified (not assumed-additive) schema | the breaker and chain-continuous DR don't exist without them; G3 asserts additivity it doesn't own |
| **Per-tenant hash chain** (global→per-tenant re-architecture) + global signed chain-head meta-log | G5 (G5-A13/A3) | chain identity scoped per tenant so erasure has no collateral; a meta-log so tenant existence + head stays globally tamper-evident | crypto-shred deletion is unsafe on a shared chain; this is a C2 *re-architecture*, the breaking change META Risk #8 flagged |
| **`insufficient:erased_input`** (+ `complete:memoized_post_erasure`) completeness sub-states | G7 (A-1/A-5) | new completeness values, with `erased_input` required to reference an in-chain tombstone + key-destruction cert | erasure regresses replay; without honest sub-states the `complete` label becomes the declared-vs-verified sin |
| **`key_id` resolution + append-only Key Transparency Log (KTL)** the verifier consults | G4 (A-1) | `key_id`→public-key resolution against a never-removing KTL; KTL gets its own DR/RPO (from G3); per-package embedded trust slice | rotation silently revokes offline verifiability of old packages; KTL loss makes ALL history unverifiable |

> **Note for the C2-rc editor:** items 1–3 change *chain identity*; sequencing them together avoids three separate re-chain migrations. Item 4 is a completeness-enum extension (additive-ish but enum-breaking for strict consumers). Item 5 adds a verifier-side index. The chain-epoch migration mechanism (G6) is the *vehicle* that carries all of these across the 1.0→1.0-rc boundary without rewriting history.

---

## 4. Ranked Domain-G defect register (XG-*)

Severity is the higher of the component severities involved. **Fix type:** *Correctness* = the design/claim is wrong until fixed; *Hardening* = design is right, residual hole.

| Rank | ID | Sev | Components | Fix type | Defect & reconciled fix |
|---|---|---|---|---|---|
| **1** | **XG-1** | **CRITICAL** | G1, (C2) | **Correctness** | **Un-sharded per-source hash chain is a serialization bottleneck overwhelmed at 100× multi-cluster ingest** (XD-G1-4). Fix: shard by `(source.system, cluster)` + cross-shard roll-up checkpoint (XD-G1-5) so deletion stays detectable. **C2-rc ask #1.** |
| **2** | **XG-2** | **CRITICAL** | G3, B1/B2/B5 | **Correctness** | **The circuit breaker is a remotely-triggerable enforcement kill switch** (G3-A4); defense rests entirely on correct human criticality classification + post-hoc catch-up. Fix: default-deny-on-ambiguity (regulated/safety controls = C-CRITICAL regardless of namespace), fast-path in-window re-eval for net-new degraded admits, absolute max-degraded-duration, dual-control on manual *open*. |
| **3** | **XG-3** | **CRITICAL** | G3, (C2) | **Correctness** | **RPO>0 on D0 is unacceptable to the regulated buyer** (disclosure ≠ evidence) and the SPEC leaves it as the *default* (G3-A3); even P3 has an irreducible commit-latency RPO floor. Fix: mandate RPO=0 (P3/sync-D0) as the regulated *default*; state the floor = commit latency; confront the local-buffer/RPO=0/survive-correlated-loss trilemma. Pairs with **C2-rc ask #2** (`infrastructure_degraded`/`restore_boundary`). |
| **4** | **XG-4** | **CRITICAL** | G5, G7, (C2) | **Correctness** | **Crypto-shred + a signed deletion certificate is a false attestation if derived stores / backups / KMS-escrow residue isn't handled** (G5-A2). Fix: key every derived store under the tenant/subject key; define backup-erasure with G3; KMS destruction covers escrow/backups; certificate scopes its claim to what was actually destroyed. Inline PII must move out-of-line to CAS so GC stays chain-safe (G7-A4). |
| **5** | **XG-5** | **CRITICAL** | G5, (C2) | **Correctness** | **T1 cannot protect the raw evidence object-store log to a regulated standard** (no RLS; falls back to the condemned per-request credential minter) (G5-A1). Fix: "regulated ⇒ T2 (per-bucket) minimum" is a *correctness* rule, not positioning. Drives **C2-rc ask #3** (per-tenant chain) for T2+. |
| **6** | **XG-6** | **CRITICAL→HIGH** | G4, D4, (C2) | **Correctness** | **At POC, a cluster-admin with signer-invoke rights can still drive K1 to forge the chain** (G4-A3); KTL has no DR contract so its loss makes ALL history unverifiable; offline verify returns "clean" without bounding revocation freshness (G4-A1). Fix: promote signer-invoke-identity-separation + rate/context limits to **MUST@POC**; give the KTL its own RPO; add `revocation_freshness` to the verifier. **C2-rc ask #5** (`key_id`/KTL). |
| **7** | **XG-7** | **CRITICAL** | G7, B1/B5, (C2) | **Correctness** | **Crypto-shred breaks authoritative replay and the D-G7-03 mitigation is oversold** — a memoized boolean ≠ re-executable replay, yet carries `complete` (G7-A1); per-subject keying can't erase third-party PII in agent `request_object` or `unknown`-disposition events (G7-A2). Fix: `complete:memoized_post_erasure` sub-state; B1 input-fact capture as a hard upstream MUST or drop the claim; per-(content,embedded-subject) erasure path or scope the GDPR claim honestly. **C2-rc ask #4** (`erased_input`). |
| **8** | **XG-8** | **CRITICAL** | G2, (product) | **Correctness** | **The cheapest cure for the cost cliff deletes full-population replay = the differentiator** (G2-A1); thaw/retrieval mis-labeled "one-time" hides ~$100–300k/yr recurring (G2-A4); build column under-counts the 8 G-domain components (real build ≈ $14–18M) (G2-A7). Fix: surface the cliff=moat as a strategic constraint; add the recurring thaw + index-growth lines; re-cost build (buy advantage *grows* to ~6–7×). |
| **9** | **XG-9** | **CRITICAL** | G6, (C2) | Correctness | **The epoch seal can launder a pre-existing chain break into a signed "verified" root** (G6-D1); the seal assumes one global instant but the topology is per-source + buffered spokes (G6-D3). Fix: promote the clean-chain pre-flight from SHOULD to **MUST** (refuse to seal over a gap); make the seal **per-source rolling**, verifier tolerates mixed epochs. |
| **10** | **XG-10** | **CRITICAL→HIGH** | G6, (product) | Hardening | **The ≤2-pages/shift budget defines un-runnability away** and can be met by under-alerting a silently-broken chain (G6-D2). Fix: exempt chain-integrity/ingest-drop alerts from the budget (they always page); honest "managed tier for sub-floor teams; self-hosted not recommended below N SREs." |
| **11** | **XG-11** | **HIGH** | G1, (C2) | **Correctness** | **The eval:recorded-event ratio is unstated and swings ingest sizing by 170×** (~580/s vs ~100k/s); compliance may force "record all" (XD-G1-12). Fix: make it a defended assumption with a knob; it changes the entire §3/§4 sizing and the storage economics. |
| **12** | **XG-12** | **HIGH** | G7 | Correctness | **Unbounded/over-broad legal hold is a permanent erasure-defeat switch** (G7-A7); restore can un-erase a destroyed DEK (G7-A10b). Fix: bound hold scope+duration + proportionality + expiry; "on restore, re-apply the tombstone log to re-destroy any post-backup-tombstoned key." |
| **13** | **XG-13** | **HIGH** | G3, (C2) | Hardening | **`restore_boundary` is a signed license to truncate** the chain; dual-control insufficient vs the named malicious-admin (G3-A2). Reconciliation (failover + fork) is an **event-injection channel** (G3-A6). Fix: marker references + checkable against the immutable WORM digest, cross-checked vs buffers/replica; recovered events must carry a pre-disaster spoke-buffer signature or be rejected. |
| **14** | **XG-14** | **HIGH** | G4, D4 | Hardening | **POC tamper-evidence is self-referential** (platform signs+timestamps+publishes root); the one operator-independent root channel is "ideally" optional; timeline-aware revocation is circular on a single-keyed chain (G4-A2/A5). Fix: make ≥1 operator-independent root channel a MUST; label POC as "outsider/mistake-grade, not malicious-operator-grade", or pull a minimal external anchor into POC. |
| **15** | **XG-15** | **HIGH** | G5, D2/C2/D1/F2/G1/G3/G4/G7 | **Correctness (process)** | **G5 is a cross-cutting tenancy *contract*, not a component** (G5-A14); it imposes a breaking per-tenant C2 re-architecture (G5-A13) and rests on three abstractions every consumer must adopt (G5-A15). Fix: reframe as a tenancy contract; propagate each mandate into the owning SPEC; enforce no-direct-store/compute/key-handle architecturally. |
| **16** | **XG-16** | **HIGH** | G8, E1/B1/G3/E3 | **Correctness** | **The load-bearing guardrail (sim-before-enforce) and 5 other G8 MUSTs are subcontracted to components that may slip** (E1, B1-R30, G3, E3) (AR-4); portability is defended by a heuristic lint + an unbuilt conformance suite, and Kyverno isn't Rego-portable (AR-1). Fix: B1-R30 is a *blocking prerequisite*; scope portability to Rego-executing engines; a lint cannot substitute for the sim gate — say so. |
| **17** | **XG-17** | **HIGH** | G8, A2/D2 | Correctness | **NS "most-restrictive-wins" is asserted but owned by no component**, and the permissiveness check is blind to latent (unexercised) gaps (AR-2); mandatory central review re-centralizes the bottleneck NS-authoring existed to remove (AR-6/AR-7). Fix: pin the B-layer combinator (or restrict NS authoring to disjoint additive denials); add static latent-permissiveness analysis; tier review by permissiveness result. |
| **18** | **XG-18** | **HIGH** | G3, D1/D3 | Hardening | **Restore re-opens closed bypasses** (revoked exceptions, post-backup denials), not just expired ones (G3-A5). Fix: the D0 append-only security-decision log is authoritative over the D1 snapshot; replay grants/revokes forward from the backup point; never restore to a more-permissive state than D0 supports. |
| **19** | **XG-19** | **HIGH** | G6, G3/G4/G1/G5 | **Correctness (process)** | **G6's day-2 runbooks hand off DR/keys/capacity/tenancy to G3/G4/G1/G5 — unbuilt at authoring**; the highest-severity branches dangle (G6-D4). Fix: turn each pointer into a *named MUST-requirement on the other component* so it surfaces in cross-cut reconciliation. |
| **20** | **XG-20** | HIGH | G2 | Correctness | **The cost model is an optimistic point estimate dressed as a quote** (G2-A2/A3/A5/A6): scale points hide the storage knee; steady-state conflated with cumulative growth; index/read-model over 128B events stays warm/hot (unpriced); 6×/5× compression/dedup asserted. Fix: intermediate scale points, per-year ramp table, an index-growth line, best/expected/worst sensitivity bands. |
| **21** | **XG-21** | HIGH | G1, B2/B3/D1 | Correctness | **The admission hot path makes a synchronous external-data (cosign) call that saturates during deploy storms** (0% cache hit) and, with `failurePolicy: Fail`, bricks deploys cluster-wide at the worst moment (XD-G1-1). Fix: mandate **build-time cache-seeding** (reuse the B3/CI verification result) so admission is never the first place a signature is checked; pre-fetch JWKS out-of-band (XD-G1-3). |
| **22** | **XG-22** | HIGH | G1, C2/E1 | Hardening | **The 9-index OLTP event store is a co-equal (arguably worse) bottleneck** under write-amplification + analytical-scan contention (XD-G1-10); fleet-wide 30-day differential replay (the differentiator) is O(events×iterations) and namespace-scoping amputates the feature (XD-G1-8). Fix: CQRS / columnar analytics mirror; promote the ALT sampling/stratified-materialized-sample. |
| **23** | **XG-23** | MED | G6, B5/C2 | Hardening | **The admission SLO (p99<250ms) collides with synchronous evidence-value capture** (G6-D6); "audit every op" (RUN-0) vs "no op noise in chain" (OBS-3) (G6-D7). Fix: split the SLI — decide fast, enrich evidence async/lossless; scope governed C2 events to the enforcement/integrity-affecting subset. |
| **24** | **XG-24** | MED | G4/G7/G1/G2 | Hardening | **The millions-of-DEK population + per-event KMS sign/verify is unsized on the hot path** (G4-A4/D-G4-8, G7-A9). Fix: hand G1/G2 a real key-count + per-write-unwrap-latency + KMS-ops-cost model; document the trust-root upgrade as a ceremony, not a config flip. |
| **25** | **XG-25** | MED | G5 | Hardening | **Side-channel/lifecycle gaps:** serial-core fairness ≠ isolation (timing side-channel leaks tenant activity — a T3 driver) (G5-A6); suspend conflates incident-quarantine with billing (G5-A11); global/cross-tenant work has no quota attribution (G5-A7); residency exposes even data-sovereign tenants' existence in the global registry (G5-A8). Fix per finding (T3 for timing-sensitive; add a `quarantine` state; global-path quota; regionally-partitioned registry). |
| **26** | **XG-26** | MED | G7 | Correctness | **PII leaks into the non-erasable skeleton** (`resource_id`, free-text `outcome_reason`) which is in `content_hash` and can't be crypto-shredded (G7-A12); warm-tier GC of agent prompts weakens the AI-governance evidence F4 sells (G7-A11); rectification retains inaccurate PII forever encrypted (G7-A6). Fix: PII-detection/scrub on structural + free-text fields at ingest; per-tenant TTL override for agent bodies; decide rectification auto-shred. |
| **27** | **XG-27** | MED | G3, G4 | Hardening | **Restore-verify needs G4 keys that may die in the same disaster** (G3-A8a); drill cadence (quarterly, staging) manufactures false DR confidence and the prod-scale restore is the one not drilled (G3-A7). Fix: embed public verification keys in the WORM backup; tie drill cadence to data-growth; continuously *project* restore-RTO; ≥1 prod-scale drill/year. |
| **28** | **XG-28** | MED | G8, B1 | Correctness | **"Inner loop identical to CI" is false on the data/bundle plane** (stale cache, redacted fixtures) (AR-8); one fail-closed template default DoS's build-time/detective pipelines (AR-9); generate-the-var contradicts B1-R1's must-both-present (AR-10). Fix: scope the claim to ruleset+invocation + data-stale warning; enforcement-class-aware template defaults; pin where `__control_id__` generation runs and reconcile with B1-R1. |
| **29** | **XG-29** | MED | G6, F2 | Hardening | **Plugins (unbounded via F2 §5) have no golden-signal contract; nobody watches the monitoring stack itself** (G6-D9/D10). Fix: make `/metrics` a plugin SPI load condition; an external dead-man's-switch heartbeat terminates the who-watches-the-watchmen recursion at a deliberately-external watcher. |
| **30** | **XG-30** | LOW | G1/G2/G4/G7/G8 | Hardening | **Record-keeping & calibration:** don't optimize the signer (not the bottleneck — the chain-verify full scan is the real crypto cost) (XD-G1-7); capacity-model constants need ±error bands (XD-G1-11); name the component that breaks first past 100× (XD-G1-13); salt fate / reveal budget / DSR-SLA-as-MUST / cross-subject lineage erasure (G7-A13–16); template/cookbook rot (AR-12); metric-validation owner (AR-11). |

---

## 5. Contradictions *between* G components (resolve before integration)

| # | Tension | Resolution |
|---|---|---|
| C-1 | G1 wants C2's chain *sharded by cluster*; G5 wants it *sharded by tenant*; G6 must *epoch-migrate* it | All three are chain-identity changes → **sequence them together in one C2-rc re-chain** (epoch boundary is the vehicle). Shard identity = `(tenant, source.system, cluster, epoch)`; the roll-up checkpoint commits all shard roots. |
| C-2 | G3 default DR profile (P1/P2, cheap, RPO>0) vs G3's own regulated need (P3, RPO=0); set before G2 cost is visible (G3-A8b) | Regulated default = **P3 (RPO=0)**; don't finalize the default before G2's cost trade is published. |
| C-3 | G4 KTL has no DR owner; G3 owns DR; G7 + G5 depend on per-subject/per-tenant keys G4 hasn't agreed to | **G3 gives the KTL an RPO ≥ the evidence store; G4 + G7 co-design the per-(subject,tenant) DEK grain now**, not as a G7-imposed fait accompli. |
| C-4 | G6 "no customer content in telemetry" vs the SRE's need to debug a specific slow event (G6-D5) | Boundary-crossing is a **scoped, audited break-glass** (emits its own C2 event), on labels/timings by default, not payload. |
| C-5 | G7 erasure cascade is a MUST over an unbounded surface (G6 traces, SIEM exports, pre-encryption backups) (G7-A8) | Invariant: **PII never exists in cleartext outside the envelope**, enforced at the G6/SIEM/backup boundaries; already-delivered exports can't be cascaded — state it. |

---

## 6. Verdict

The eight NFR architectures are **structurally sound** — no reviewer found a reason to redesign the crux mechanisms (sharded chain, chain-epoch migration, crypto-shred, KTL, circuit breaker, the per-tenant dial, the build-vs-buy model, the Regal-ruleset authoring layer). The domain's risk is concentrated in three places, all of them *contract* and *honesty* problems rather than architecture rewrites:

1. **Propagation (XG-15/XG-19, Theme A / §1):** these NFRs are cross-cutting contracts; if they're filed as G-backlog they never land in C2/D2/D4/B1. The five **C2-rc ratification asks (§3)** are the gate item — they must fold into the open C2 un-freeze.
2. **Fail-direction for the regulated buyer (XG-3/XG-5/XG-6/XG-14, Theme C):** the malicious-operator-grade control is GA-deferred or left as an election, while the framing implies POC-grade independence. For the buyer the product is sold to, that control is the *default*.
3. **Honesty of NFR labels (XG-4/XG-7/XG-8/XG-9, Theme A):** crypto-shred certificates, post-erasure `complete`, the cost lever's "full-population", and the epoch seal each risk the declared-vs-verified sin at the operational layer — the exact failure the platform exists to prevent.

**Gate before integration:** the five C2-rc asks (§3) and the three CRITICAL fail-direction items (XG-2/XG-3/XG-5) must be resolved in the SPECs before any regulated go-live; XG-4/XG-6/XG-7/XG-8/XG-9 before GA.

---

## 7. Top-8 Domain-G defects (the executive list)

1. **XG-1** — un-sharded C2 hash chain = the #1 scale ceiling (C2-rc).
2. **XG-2** — the circuit breaker is a remotely-triggerable enforcement kill switch.
3. **XG-3** — RPO>0 is unacceptable to the regulated buyer yet left as the default.
4. **XG-4** — crypto-shred + signed deletion certificate = false attestation if residue isn't handled (C2-rc).
5. **XG-5** — T1 can't protect the raw evidence log to a regulated standard; "regulated ⇒ T2" is correctness (C2-rc).
6. **XG-6** — POC cluster-admin can still forge the chain; KTL has no DR; offline verify ignores revocation freshness (C2-rc).
7. **XG-7** — crypto-shred breaks replay; the `complete`-after-erasure label is the declared-vs-verified sin (C2-rc).
8. **XG-8** — the cost cliff *is* the differentiator; the cheapest cure deletes the moat.

### The C2 `v1.0-rc` ratification asks (fold into the open C2 un-freeze)
- **Per-source chain *sharding*** by `(source.system, cluster)` + cross-shard roll-up super-checkpoint — **G1** (XG-1).
- **`infrastructure_degraded` disposition + `chain.restore_boundary`** (+ `degraded_session_id` / `fork_reconciliation`) event types — **G3** (XG-3).
- **Per-tenant hash chain** + global signed chain-head meta-log — **G5** (XG-5).
- **`insufficient:erased_input`** (+ `complete:memoized_post_erasure`) completeness sub-states — **G7** (XG-7).
- **`key_id` resolution + append-only Key Transparency Log (KTL)** (with its own DR) — **G4** (XG-6).
