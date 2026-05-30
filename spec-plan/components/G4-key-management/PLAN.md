# G4 — Key Management & Cryptographic Lifecycle — PLAN

**Component ID:** G4 · **Domain:** G — Operational / NFR wave
**Gates:** D4 (SEC-2/SEC-8/SEC-21/SEC-23, D4-OQ-1), C2 §7.4/§7.6 (the signer behind checkpoints + exports), B1 §6.2 (bundle signing trust policy), and the **auditor scenario HL-18** (offline, long-horizon, independent signature verification). G4 is on the critical path for any claim of tamper-evidence.
**Status:** AUTHORED.

---

## 1. Build philosophy

G4 is mostly **lifecycle and trust-distribution plumbing around existing crypto primitives** — it does not invent algorithms. The risk is not "can we sign?" (trivial) but "does a signature survive rotation, revocation, and 5 years offline, and is the key unreachable to a cluster admin?" Therefore the plan front-loads the two load-bearing artifacts — the **Key Transparency Log (KTL)** and the **offline verifier** — because they are what HL-18 and compromise recovery actually depend on, and they are what the rest of the corpus assumed without building.

**Sequencing principle:** build the *verification* path before the *production* path. A signer with no verifier is unfalsifiable; a verifier with a stub signer is testable. So WS-3 (verifier) and WS-1 (KMS signer) co-develop, with the offline historical-verification acceptance test (G4-R11) as the gate that closes the loop.

---

## 2. Workstreams

| WS | Name | Owns | Depends on | Parallelizable? |
|----|------|------|-----------|-----------------|
| **WS-1** | **KMS/HSM integration & the audit-root signer** | K1/K2 key generation in KMS, non-exportable custody (G4-R1/R4), the least-priv signer workload identity (G4-R5), the C2 §7.4/§7.6 signing call behind checkpoints+exports, KMS-unavailable fail-degraded behavior (§10). | F2 (KMS provisioned); C2 §7 envelope contract (frozen). | Core — gates everything signed. |
| **WS-2** | **Rotation automation & key lifecycle** | The KTL data model + append-only store (G4-R8), overlap-based rotation (G4-R10), revocation timeline semantics (G4-R13/R14), dual-control rotation gate reusing D3 approval (G4-R26), salt-generation rule (G4-R12), lifecycle audit to C2 (G4-R28). | WS-1 (a key to rotate); D3 approval mechanism. | After WS-1 has a key; KTL schema can start day-1. |
| **WS-3** | **Transparency log (KTL) + trust bundle + verifier tooling** | The published KTL + trust bundle (G4-R9/R20/R21), the trust-root hierarchy (§7.1), the **standalone offline verifier** (G4-R23/R24), the embedded-trust-slice in exports (G4-R21), revocation feed (G4-R16). | KTL schema (WS-2); C2 export envelope; Sigstore roots (for K3/K4 verify). | The verifier is independently buildable against fixtures; **highest-value, build early.** |
| **WS-4** | **Supply-chain keyless signing (adopt B1)** | Wire B1 D-B1-03 cosign keyless (K3/K4): Fulcio/Rekor signing in CI, the pinned signer-identity policy lifecycle (B1-R20), Rekor monitoring (§9.6), platform-image signing (K4, D4 SEC-21). | B1 §6.2 (already specified); CI workload OIDC identity (F2). | Independent of WS-1/2/3 (different trust domain) — fully parallel. |
| **WS-5** | **Internal trust lifecycle (mesh PKI, JWKS, salts, approval-callback)** | K6 cert-manager + internal CA tiers, K7 trusted-issuer-set governance (the `iss` allow-list + JWKS pin as signed config), K5 approval-callback key, K8 salt management. | F2 (cert-manager); D1 (JWKS client); D3 (callback). | Parallel; low risk. |
| **WS-6** | **External timestamp anchoring (GA hardening)** | RFC 3161 / public-log anchoring of KTL head + checkpoint roots (G4-R17, §6.4) — the keystone for compromise recovery + PQC mitigation. | WS-2 (KTL head), WS-3 (verifier checks anchor). | GA-phase; not POC-blocking but design-reserved now. |

---

## 3. Dependency DAG

```
F2 (KMS provisioned) ──────────────┐
                                    ▼
C2 §7 envelope (frozen) ──▶ WS-1 KMS signer ──▶ WS-2 rotation+KTL schema ──┐
                                    │                                       ▼
                                    │                              WS-3 KTL + trust bundle + OFFLINE VERIFIER
                                    │                                       │
                                    │                                       ▼
                                    └──────────────────────────▶ G4-R11 GATE: offline historical-verification test
                                                                            │
                                                                            ▼
                                                                  WS-6 external timestamp anchor (GA)

B1 §6.2 (specified) ──▶ WS-4 keyless supply-chain signing (PARALLEL, separate trust domain)
F2 cert-manager / D1 JWKS / D3 callback ──▶ WS-5 internal trust lifecycle (PARALLEL)
```

**Critical path:** `F2 KMS → WS-1 signer → WS-2 KTL+rotation → WS-3 verifier+trust-bundle → G4-R11 offline-verification gate`. Everything that claims tamper-evidence (C2 checkpoints/exports, C1/C5 signed packages, HL-18) is blocked on this path closing.

**Off critical path (parallel):** WS-4 (keyless supply chain) and WS-5 (internal trust) share no key material with the audit root (by design, §8 SoD) and proceed independently. WS-6 is GA, design-reserved.

---

## 4. Milestones

| Milestone | Definition of done | Gates |
|---|---|---|
| **M-G4-KMS** | K1/K2 generated in KMS, non-exportable (G4-R1/R4 verified: attempt to export the private key fails); C2 checkpoint/export calls the KMS signer; `key_id` recorded in envelope (G4-R3/R7). | C2 §7.4/§7.6 can produce a *real* (not stubbed) signature. |
| **M-G4-KTL** | Append-only KTL stores key history with all retired public keys; trust-root signs the head; revocation records timeline (G4-R8/R13/R14). | Rotation can happen without losing a public key. |
| **M-G4-VERIFY** *(keystone)* | Standalone offline verifier validates a signed export against a pinned trust-root + KTL slice, with revocation/timeline determination, **platform offline** (G4-R23). | **HL-18** offline-verification success criterion. |
| **M-G4-ROTATE** *(keystone, gating)* | **G4-R11 acceptance test passes:** sign with gen-1, rotate to gen-2/gen-3, verify gen-1 artifact offline using only the trust bundle. | The long-lived-signature-survives-rotation property. **This is the gate D4/HL-18 depend on.** |
| **M-G4-SUPPLY** | K3/K4 keyless signing live; consumers fail-closed on bad signatures (B1-R19); Rekor monitoring on (§9.6). | D4 SEC-21; B1 §6.2. |
| **M-G4-SOD** | Custody SoD enforced (G4-R25/R27): cluster-admin cannot read/rotate K1/K2; rotation is dual-control (G4-R26). | D4 SEC-11/SEC-12/SEC-23; ADVERSARIAL §A-3. |
| **M-G4-GA** | External timestamp anchoring (WS-6) + dedicated-HSM option + PQC migration design (G4-R-PQC). | GA hardening. |

---

## 5. What can be built concurrently / what blocks what

- **Concurrent from day 1:** WS-4 (keyless supply chain — entirely separate trust domain), WS-5 (internal trust), and the **KTL schema design + offline-verifier-against-fixtures** (WS-3 can develop against hand-made fixtures before WS-1's real signer exists — this is the "build verification before production" principle, and it de-risks the keystone early).
- **Blocked until KMS exists (WS-1):** real checkpoint/export signing; C2's §7.4/§7.6 stay stubbed until M-G4-KMS, but C2's *chain* (§7.3) works without a signature, so C2 is not fully blocked — only its *signed* attestation is (§10 failure-mode reasoning).
- **Blocked until M-G4-ROTATE:** any external-facing claim that "evidence verifies after rotation" — i.e. the HL-18 demo and the D4 sign-off. Do **not** ship the auditor scenario before this gate.
- **GA-deferred:** WS-6 anchoring, dedicated HSM, PQC — design-reserved (the `alg` field is the lever) but not POC-blocking.

---

## 6. Test strategy

- **Unit:** `key_id` derivation determinism (G4-R3); canonical-serialization + sign + verify round-trip per `alg`; KTL append-only enforcement (no in-place edit/delete).
- **Lifecycle:** overlap rotation produces no window where a signed artifact references an unpublished key (G4-R10); revocation marks `revoked_at` and re-signs the KTL head; timeline-aware verification returns valid-but-flagged for pre-`revoked_at`, invalid for post (G4-R14).
- **The gating acceptance tests (mirror HL-18):**
  1. **G4-R11 (rotation × historical verification):** sign gen-1 → rotate ×2 → **offline** verify gen-1 artifact using only the trust bundle. MUST pass to ship.
  2. **Offline independence:** verifier validates a full export package with the platform **network-isolated** (no live API), using only the embedded trust slice + pinned trust-root (HL-18 success criterion).
  3. **Custody (G4-R4):** as a cluster-admin, attempt to read/exfiltrate the K1 private key from any Secret/file/CI var — MUST fail (the tamper-evidence-thesis test; ADVERSARIAL §A-3).
  4. **Compromise drill:** simulate K1 leak → revoke → re-anchor forward → confirm pre-leak checkpoints remain independently verifiable and post-leak signatures are rejected (§9.2).
- **Negative:** unsigned/wrong-key artifact rejected; backdated forgery without a valid prior checkpoint rejected (relies on chain ordering, G4-R15); revoked-key future signature rejected.

---

## 7. POC floor vs GA hardening (mirrors D4 §5 trajectory)

| Item | POC floor | GA target |
|---|---|---|
| K1/K2 custody | Cloud KMS (HSM-protected), non-exportable | Dedicated HSM option for highest-assurance buyers (G4-OQ-2) |
| Timestamp trust | Chained-checkpoint ordering | External RFC 3161 / public-log anchor (G4-R17, MUST@GA) |
| Trust root | Offline KMS key, dual-control IAM | Air-gapped HSM ceremony (G4-OQ-5) |
| Rotation | Overlap-based, manual-triggered + scheduled | Fully automated with dual-control approval gate |
| PQC | ed25519, `alg`-versioned migration documented | Dual-sign / hybrid migration if/when warranted (G4-R-PQC) |
| Verifier | Standalone offline binary (ed25519 + KTL) | + Sigstore verification (G4-R24), packaged for auditor distribution |

---

## 8. Risks to the plan (see ADVERSARIAL-REVIEW.md for the full red-team)

- **R-PLAN-1:** HSM/KMS cost vs POC budget (ADVERSARIAL §A-4) — mitigated by KMS-not-HSM at POC (G4-OQ-2); the interface is identical so GA hardening is config, not rebuild.
- **R-PLAN-2:** The KTL is itself a tamper-evidence-critical store; building it correctly (append-only, trust-root-signed head) is the long pole. Front-loaded into WS-3 with fixture-driven dev to de-risk early.
- **R-PLAN-3:** Coordinating the custody SoD (G4-R27) requires an org boundary (security-ops owns KMS IAM, separate from cluster operators) that is a *process*, not code — must be agreed before M-G4-SOD or the control is theatre.
