# G4 — Key Management & Cryptographic Lifecycle — SPEC

**Component ID:** G4 · **Domain:** G — Operational / Non-Functional Architecture (NFR wave)
**Spec sources:** §23 (Security / Evidence integrity), §26.1 (signing intentionally unspecified — the gap this component closes), with normative inputs from C2 §7 (per-source hash chain, signed Merkle checkpoints, signed export envelope, ed25519, one export-signing primitive), B1 §6.2 (signed OCI Rego bundles, cosign keyless), C1 §4 / C5 §3.4 (signed evidence-package exports), D1 §4 (JWKS trust / issuer pinning), D4 §2 + §3 (the security floor + the secret/key inventory).
**Resolves:** **D4-OQ-1** ("signing technology unspecified") and the META-ADVERSARIAL Risk #3 ("key-management lifecycle is unowned; all tamper-evidence rests on one ed25519 key with no rotation/custody/compromise-recovery plan").
**Gates:** D4 (promotes SEC-2/SEC-8/SEC-21/SEC-23 from "interface only" to a decided implementation), C2 §7.4/§7.6 (the signer behind every checkpoint and export), B1 §6.2 (the bundle-signing trust policy), and the **auditor scenario HL-18** (offline, independent, long-horizon signature verification).
**Scenarios exercised:** HL-18 (independent offline replay + signature verification), DT-24 (signed evidence export), HL-05 (SOC2 Type II), and transitively every signing/verification step across B1, C1, C2, C5, D1, D3, D4.
**Status:** AUTHORED.
**Author persona:** Marcus (Platform Security Engineer) + a cryptographic-lifecycle architect.

---

## 0. Why this component exists (the one-paragraph thesis)

The entire product is a **tamper-evidence claim**: "an independent auditor can verify, offline, that this evidence was produced by enforcement and was not altered." Every word of that claim reduces to a key. If the key is mismanaged, the differentiator is theatre (META-ADVERSARIAL G-6). Yet the functional corpus left signing technology "unspecified" (§26.1, D4-OQ-1) and named key compromise as explicitly unmodeled (C2 ADVERSARIAL, XD-19). G4 owns the unified cryptographic story end-to-end: **every key, its type, generation, custody, rotation, revocation, the trust root, how a verifier obtains trusted public keys, and how the platform recovers from compromise** — with a decided default and the one property that the rest of the corpus assumed but never specified: **a signature made three years ago must still verify after the key that made it has rotated.** That property — long-lived signature verification across key rotation — is the spine of this spec.

---

## 1. Scope

### 1.1 In scope
- A complete **inventory** of every cryptographic key / signing / trust-anchor use across the platform (§2).
- For each: **key type/algorithm, generation, custody, rotation policy, revocation, trust root, verifier key acquisition, and compromise recovery** (§3–§10).
- The decided resolution of **D4-OQ-1** (signing tech): a per-use default (§2 table, §11).
- The **rotation ↔ long-lived-signature** reconciliation: key history / transparency so historical signatures keep verifying (§5, §6 — the load-bearing section).
- The **trust-root model** and the **verifier key-distribution** mechanism that makes HL-18 offline verification possible (§7).
- **Compromise recovery** runbooks per key class, including the "the audit-checkpoint key leaked — what about 3 years of signed checkpoints?" scenario (§9).
- Separation-of-duties / custody controls for the signers (§8), satisfying D4 SEC-11/SEC-12/SEC-23.

### 1.2 Out of scope (delegated)
- The *content* of what is signed (C2 §7 owns the event/checkpoint/export structure; B1 §6 owns the bundle structure; C1/C5 own export assembly). G4 owns the **keys and the signing/verification lifecycle**, not the payload schema.
- JWT *verification logic* (D1 owns `iss`/`aud`/`alg` validation). G4 owns only the **trust-anchor lifecycle** for the JWKS/issuer set (how the trusted-issuer list is governed and rotated) — see §2 row K7.
- TLS/mTLS *transport mechanics* (D4 SEC-19/SEC-20, F2 deployment). G4 owns the **PKI lifecycle** behind mTLS certs (the CA, issuance, rotation) — row K6.
- HSM/KMS *procurement and infra wiring* (F2 deployment). G4 specifies the **requirement and interface**; F2 wires it.

### 1.3 Design tenets (normative posture)
- **Verifier-offline-first.** Any signature the platform asks an external auditor to trust MUST be verifiable with materials the auditor can hold *outside* the platform — a public key (or transparency-log inclusion proof + root), never a live API call to the platform being audited (HL-18 success criterion: "an independent third party can re-verify the signature offline"). A verification path that requires the auditee's server to be up and honest is not independent verification.
- **Key history is append-only, like the evidence it protects.** Keys rotate; **public keys are never retired from the trust store** — they move from `active` to `retired` and remain verifiable forever. The verifier resolves *which* key signed *which* artifact by a stable `key_id` recorded inside the signature envelope (C2 §7.4 already carries `key_id`). This is the single mechanism that makes rotation safe for long-lived signatures (§5).
- **Custody beats algorithm.** A perfectly chosen algorithm with the private key sitting in a cluster Secret defeats the tamper-evidence thesis: anyone with `cluster-admin` can forge the audit chain (META-ADVERSARIAL, this ADVERSARIAL §A-3). The custody decision (where the private key lives, who can invoke it) is more security-load-bearing than the algorithm choice, and is decided per use class (§2, §4).
- **Separate trust domains for separate harms.** The key that signs *audit checkpoints* (the tamper-evidence root) MUST NOT be the same key, the same custody, or the same operator-reachable surface as the key that signs *Rego bundles* (the enforcement root) or the key that signs *platform images* (the supply-chain root). Compromise of one must not forge the others (D4 §3 "separate trust domain"; §8 here).
- **Decide-document-continue.** D4-OQ-1 is resolved with a concrete default per use (§11). The POC floor and the GA hardening target are both stated, so the posture is a deliberate trajectory, not an accident (mirrors D4 §1.3 / §5).

---

## 2. The key & signing inventory (the canonical list)

This is the authoritative enumeration the META-ADVERSARIAL pass said no document owned. Every place the platform signs, hashes-for-trust, or anchors trust is one row. **"Decided default" resolves D4-OQ-1 for that use.**

| ID | Use | What is signed/trusted | Key type (default) | Custody model (decided default) | Trust root / verifier obtains key via | Rotation cadence (default) | Long-lived? |
|----|-----|------------------------|--------------------|----------------------------------|----------------------------------------|----------------------------|-------------|
| **K1** | **Audit-chain checkpoint signing** (C2 §7.4 signed Merkle checkpoints) | The Merkle root + `signed_at` over each chain segment, per source | **ed25519** (matches C2 §7.4; fast, small, deterministic) | **KMS-backed signing key, private key non-exportable** (cloud KMS / HSM-backed KMS). Signing is an API call; the key never leaves the HSM/KMS boundary. **Not a cluster Secret.** | **Platform Key Transparency Log + published trust bundle** (§7). Verifier obtains the historical public keyset offline; resolves by `key_id`. | **180 days** (long checkpoint cadence ⇒ slow rotation; every public key retained forever) | **YES — primary long-lived concern** |
| **K2** | **Evidence-package export signing** (C2 §7.6 detached signature over `manifest.json`; called by C1 §4, C5 §3.4) | The export manifest embedding the Merkle root, `key_id`, `signed_at` | **ed25519** (one export primitive, C2 §7.6) | **Same KMS custody class as K1**, but **MAY use the K1 key** (an export is a re-attestation over already-checkpointed events) *or* a dedicated export key — **default: a dedicated `export-signing` key** so export-volume key use is separable from the checkpoint root (blast-radius separation). | Same as K1 (published trust bundle + transparency log, resolve by `key_id`). | 180 days | **YES** (an export signed for a 2019 audit must verify in 2026 — DT-24, HL-18) |
| **K3** | **Signed Rego/OCI policy bundles** (B1 §6.2 B1-R17) | The OCI bundle digest + provenance attestation | **Sigstore keyless (Fulcio short-lived cert + Rekor transparency)** default; **KMS key** enterprise option (B1 D-B1-03 — *adopted as-is*) | **Keyless: no long-lived private key** (ephemeral Fulcio cert bound to a CI workload identity, logged in Rekor). KMS option: KMS-backed, non-exportable. | **Fulcio root + Rekor public key + a pinned signer-identity policy** (B1-R20 cert-identity allow-list). Verifier checks the Rekor inclusion proof. | **Keyless: per-build (ephemeral, ~10 min cert)**; KMS option: 90 days | **Bundle signatures are short-horizon** (you verify at load; old bundles are superseded) — see §5.4 |
| **K4** | **Platform container-image signing** (D4 SEC-21 dogfooding; F2 deploy-time verify) | Platform's own images + SBOM attestation | **Sigstore keyless (Fulcio/Rekor)** default; KMS option | Keyless, same as K3, bound to the **platform release CI** identity (isolated CI per D4 §3). | Fulcio/Rekor root + pinned release-CI identity. | Per-release (ephemeral) | No (verify at deploy) |
| **K5** | **Approval-callback signing** (D3-R9; D4 SEC-20, §3 row) | Approval callbacks, per-approver bound | ed25519 *or* HMAC (per-approver binding) | **KMS-backed (asymmetric) preferred**; HMAC key in a secret manager if symmetric. **Per-approver key binding** (D3 A5). | Internal trust (no external verifier); platform JWKS-style endpoint. | 90 days | No (callbacks are short-lived; expiry-checked, D3 A8) |
| **K6** | **mTLS service PKI** (D4 SEC-19/SEC-20: subject + approval-callback channels) | Service identities (x509) for the trust-bearing internal channels | **x509, ECDSA P-256** leaf certs from an internal CA | **cert-manager + an internal issuing CA whose root is KMS/HSM-backed**; private keys per-pod, short-lived, auto-rotated. | Internal mesh trust bundle (not an external-verifier concern). | **Leaf: 24–90 days (auto)**; issuing CA: 1–2 yr; root: 5–10 yr | No (transport, ephemeral) |
| **K7** | **JWT verification trust anchors** (D1-R1/R2: JWKS + issuer allow-list) | *Not a platform-held signing key* — the **set of trusted external issuers** and their JWKS | n/a (trust-config, not a key the platform holds) | **G4 owns the lifecycle of the trusted-issuer set**: the `iss` allow-list (D1-R2) and the JWKS-pin material (D4 SEC-13/SEC-23) are an **authz-relevant, versioned, signed-as-config artifact** (governed like D1's mapping config, §15). Keycloak rotates its own JWKS; the platform refreshes via the JWKS client (D1 §4 `out` edge) with pinned `alg`. | Keycloak's published JWKS; platform-side issuer allow-list is the trust root for *which* JWKS to trust. | Issuer JWKS: per IdP (hours–days); allow-list: change-controlled | n/a |
| **K8** | **At-rest / redaction salts** (C2 §7.5 salted-hash redaction; D4 OQ-4 hash-at-rest for `sub`/`email`) | Salts for redaction-preserving hashes inside `content_hash` | HMAC/salt material (32B random) | Secret manager, **per-deployment salt, never rotated within a retention window** (rotating a redaction salt would break the deterministic `content_hash` contribution — C2 §7.5). Rotation only at a generational boundary with re-hash. | n/a (internal); the *holder of cleartext* can prove a match (C2 §7.5). | **Per generation, not periodic** (see §5.5) | Tied to evidence retention |

**Decision summary (resolves D4-OQ-1):**
- **Tamper-evidence roots (K1, K2): KMS/HSM-backed ed25519, non-exportable, with a Key Transparency Log + published trust bundle for offline historical verification.** This is the decided default — *not* keyless — because **audit signatures are long-lived (5–10 yr) and must verify offline against a stable keyset**, which is the one thing keyless-per-build is bad at (a 2019 ephemeral Fulcio cert + a Rekor lookup is a fine integrity record but a worse *offline, decades-later, single-public-key* verification story for an auditor's workpapers — §3 of ALT). See the ALT for the full three-way trade study.
- **Enforcement / supply-chain roots (K3, K4): Sigstore keyless (Fulcio/Rekor)** — *adopted from B1 D-B1-03 unchanged* — because bundle/image signatures are **short-horizon, high-frequency, CI-produced**, exactly Sigstore's sweet spot, and keyless removes a long-lived private key from CI.
- **Internal trust (K5, K6, K7, K8): KMS-backed or mesh/secret-manager**, no external-verifier requirement.

This split — **keyless for the supply chain, KMS+transparency for the audit root** — is the central architectural decision of G4 and is justified in detail in §3 and the ALT.

---

## 3. Per-key: type, algorithm, generation

### 3.1 Algorithm choices (and why)
- **ed25519** for K1/K2 (audit roots): chosen to match C2 §7.4's already-frozen `"alg": "ed25519"`. Deterministic signatures (no nonce-reuse foot-gun), small (64B sig, 32B pubkey — cheap to embed in every export manifest and to ship in a trust bundle), fast to verify (an auditor verifying 250 events in HL-18 verifies instantly), broad library support (Go `crypto/ed25519`, every KMS).
  - **GA hardening note (PQC):** ed25519 is not post-quantum. For 5–10-yr evidence, a store-now-verify-later quantum adversary is a *theoretical* long-horizon risk. **Decision:** ed25519 at POC/GA-v1; the envelope's `alg` field (C2 §7.4) is the version lever — a future migration to a hybrid/PQC scheme (e.g. ML-DSA / dual-sign) is an *additive `alg` value* + a re-checkpoint-forward operation, **not** a re-sign of history (history stays valid under ed25519 as long as ed25519 isn't broken; if it is, see §9.4). G4-R-PQC tracks this as a named GA+ item, not POC scope.
- **Sigstore/ECDSA P-256** for K3/K4 (Fulcio default leaf is P-256 / ECDSA): inherited from the Sigstore stack; not a G4 choice.
- **x509 ECDSA P-256** for K6 mTLS leaves: standard, cert-manager-native, short-lived.

### 3.2 Generation
- **G4-R1 (MUST):** All long-lived private keys (K1, K2, and the K6 issuing-CA/root) are **generated inside the KMS/HSM** and are **non-exportable** — there is no point in time at which the private key exists in cluster memory, a Secret, a file, or a CI variable. Generation is an auditable KMS operation (D4 SEC-23: "managed outside source, never logged").
- **G4-R2 (MUST):** Keyless signers (K3, K4) generate **no persistent private key**; the ephemeral key is created in-memory in the CI job, certified by Fulcio against the CI's OIDC workload identity, used once, and discarded — the trust is in the **Rekor transparency record + the pinned signer identity**, not a stored key (B1-R20).
- **G4-R3 (MUST):** Every key has a stable, globally unique **`key_id`** (recommended: `sha256` of the SPKI public key, truncated, prefixed by use-class, e.g. `auditckpt:ed25519:ab12…`). The `key_id` is what binds a signature to a key across rotation (C2 §7.4 already records `key_id` in the signature envelope). `key_id` is **immutable** and **never reused** across keys.

---

## 4. Custody (KMS/HSM vs Sigstore-keyless vs in-cluster) — the decided model

The three custody models and where each is used. (Full trade study in `ALT-key-custody-models.md`.)

| Custody model | Used for | Why here | Why NOT elsewhere |
|---|---|---|---|
| **KMS/HSM-backed, non-exportable** | K1, K2 (audit roots), K6 root/issuing CA | Audit roots are long-lived and the single point of forgeability for the whole tamper-evidence thesis; the private key must be unreachable even to `cluster-admin` (§A-3). | Overkill + ops cost for per-build supply-chain signatures (K3/K4). |
| **Sigstore keyless (Fulcio cert + Rekor log)** | K3 (bundles), K4 (images) | High-frequency, CI-produced, short-horizon; no long-lived private key to steal; transparency log gives independent provenance; aligns with B1 D-B1-03 + Kyverno/Gatekeeper image-verify posture. | Bad fit for the *audit root*: offline, decades-later, single-pubkey verification is weaker with per-build ephemeral certs (see ALT §3). |
| **In-cluster (Secret / secret manager)** | K5 (HMAC option), K7 JWKS-pin material, K8 salts | Internal-only trust, no external verifier, short-lived or non-rotating-by-design; convenience acceptable because a compromise here does **not** forge audit evidence. | **Explicitly forbidden for K1/K2** — an in-cluster audit-signing key means anyone with `cluster-admin` can forge the audit chain, which **defeats the entire tamper-evidence thesis** (ADVERSARIAL §A-3; this is the single most important custody rule in the spec). |

- **G4-R4 (MUST):** The audit-chain checkpoint key (K1) and export-signing key (K2) **MUST NOT** be stored as Kubernetes Secrets, in a config file, in CI variables, or anywhere a cluster/namespace administrator can read the private key material. They MUST be KMS/HSM-backed and invoked only via an authenticated signing API. *(This is the direct fix for the "in-cluster keys = cluster-admin forges the audit chain" defeater.)*
- **G4-R5 (MUST):** Invocation of the K1/K2 signing API is **least-privilege and audited**: only the C2 checkpoint/export signer workload identity may call it (D4 SEC-22 least-priv); every signing call is itself logged to C2 (a signing-service audit trail), so anomalous signing volume is detectable (§9 compromise detection).
- **G4-R6 (SHOULD→MUST@GA):** The K1/K2 signer enforces **rate / context limits** (e.g. checkpoints only at the configured cadence; exports only via the export API with a caller scope) so that a stolen *signing capability* (as opposed to a stolen key — KMS keys can't be exfiltrated, but the *ability to call sign* can be hijacked) cannot mass-forge. This is the residual threat KMS-custody introduces and §9.3 addresses.

---

## 5. Rotation policy AND its interaction with long-lived signatures (THE load-bearing section)

> The hard requirement, stated once, plainly: **An audit checkpoint signed in 2023 with key K1-gen-1 must still verify in 2030 after K1 has rotated to gen-2, gen-3, gen-4.** Rotation must never invalidate a past signature. This is the property the rest of the corpus assumed and never specified (META-ADVERSARIAL Risk #3; this is *the* reason the component exists).

### 5.1 The mechanism: stable `key_id` + append-only key history + transparency
Rotation does **not** mean "replace the key everyone trusts." It means "start signing *new* artifacts with a new key, while every *old* public key remains in the trust store forever, indexed by `key_id`." Concretely:

- **G4-R7 (MUST):** Each signed artifact records the `key_id` of the signing key inside its signature envelope (already true: C2 §7.4 `signature.key_id`). A verifier **resolves the public key by `key_id`**, not by "the current key." Therefore a verifier presented with a 2023 checkpoint looks up `auditckpt:ed25519:<gen-1-id>`, finds the (retired-but-retained) gen-1 public key, and verifies. Rotation is invisible to historical verification.
- **G4-R8 (MUST):** The platform maintains a **Key History / Key Transparency Log (KTL)**: an append-only, itself-tamper-evident log of `{key_id, public_key (SPKI), use_class, status, valid_from, valid_until, retired_at, supersedes, superseded_by, generation_attestation}` for every K1/K2 key ever used. **Public keys are never removed** — `status` transitions `pending → active → retired → revoked`, but the entry and the public key persist forever so historical signatures stay verifiable. (`revoked` ≠ removed — a revoked key's *future* signatures are distrusted, but pre-revocation signatures' validity is determined by the revocation timeline, §6.3.)
- **G4-R9 (MUST):** The KTL is **published** the same way evidence is: it is content-addressed, hash-chained, and the *current head* of the KTL is itself signed (by a higher tier — the trust-root, §7.1) and distributed in the **trust bundle** (§7.2). A verifier can hold the entire key history offline.

### 5.2 Rotation cadence (decided defaults)
| Key | Cadence | Trigger |
|---|---|---|
| K1 audit-checkpoint | **180 days** scheduled + on-demand on compromise | Time-based; slow because checkpoints are infrequent and each rotation grows the KTL |
| K2 export-signing | 180 days + on-demand | Time-based |
| K3/K4 keyless | **Per-build (ephemeral)** | Every signature is a fresh cert — "rotation" is automatic and continuous |
| K5 approval-callback | 90 days | Time-based; short-lived artifacts so old key retirement is cheap |
| K6 mTLS leaf / issuing CA / root | 24–90 d / 1–2 yr / 5–10 yr | cert-manager automatic; staggered tiers |
| K7 JWKS pin | Follows IdP | IdP rotates; platform refreshes via JWKS client |
| K8 salts | Per generation only | Never within a retention window (would break `content_hash`) |

- **G4-R10 (MUST):** Scheduled rotation is **overlap-based, never break-before-make**: gen-N+1 is generated, attested, published to the KTL, and distributed in the trust bundle **before** any artifact is signed with it; signing cuts over only after a propagation grace window; gen-N stays `active` for verification of in-flight artifacts and `retired` (still trusted for verification) thereafter. There is **no instant** at which a freshly-signed artifact references a key the trust bundle hasn't published.

### 5.3 The "historical verification across rotation" acceptance test (gating)
- **G4-R11 (MUST):** A gating acceptance test (mirrors HL-18 success criteria): sign a checkpoint and an export with K1-gen-1; rotate to gen-2 and gen-3; **with the platform offline**, hand an external verifier only the trust bundle and the 3 artifacts; the verifier MUST validate the gen-1 export's signature using only the trust bundle. If it cannot, rotation has broken historical verification and the build does not ship. *(This is the test the META-ADVERSARIAL pass said no one owned: "the signing key leaked yesterday — what do we do with 3 years of signed checkpoints?" — answered concretely in §9, tested here.)*

### 5.4 Why bundle/image signatures (K3/K4) are NOT a long-lived concern
Bundle and image signatures are verified **at load/deploy time** against the currently-trusted Fulcio/Rekor roots and the pinned signer identity (B1-R19). A superseded bundle is never re-verified years later; the *audit record* that a given bundle was active is captured in C2 (which K1 protects). So K3/K4 inherit Sigstore's per-build ephemerality with no long-horizon penalty — the long-horizon evidence is the C2 chain (K1), not the bundle signature. This is precisely why the custody split in §2 is correct: long-lived → KMS+KTL; short-lived → keyless.

### 5.5 Why salts (K8) don't rotate within a retention window
C2 §7.5 makes the redaction hash contribute deterministically to `content_hash`. Rotating the salt would change every future hash and break the chain's reproducibility. **G4-R12 (MUST):** the redaction salt is fixed per *generation*; a salt change is a generational boundary that requires re-hashing a sealed segment forward (a planned, signed migration — §9.4 / G2 retention), never an in-place change.

---

## 6. Revocation

Rotation is routine; revocation is "this key is *bad* (compromised/weak), distrust it." Revocation must distrust the key **without retroactively voiding honest historical signatures made before the compromise** — otherwise a single key leak destroys years of valid evidence (ADVERSARIAL §A-1).

- **G4-R13 (MUST):** Revocation is recorded in the KTL as a `revoked` status with a **`revoked_at` timestamp and `revocation_reason`**, signed by the trust-root (§7.1). The KTL head is re-signed and re-published. Revocation is **append-only** (you never delete the key; you mark it revoked).
- **G4-R14 (MUST):** Verification semantics for a revoked key are **timeline-aware**: a signature whose artifact carries a trustworthy `signed_at` (and, at GA, a transparency-log timestamp, §6.4) **before** `revoked_at` is treated as **valid-but-flagged** (it was made while the key was honest); a signature dated **after** `revoked_at` is **invalid**. This is the difference between "the key leaked so re-sign everything" (catastrophic) and "the key leaked, so we distrust post-leak signatures and prove the rest were earlier" (recoverable) — see §9.
- **G4-R15 (MUST):** Because `signed_at` is attacker-controllable if the attacker holds the signing capability, the *trustworthy* timestamp for the timeline test is **the audit chain's own ordering** (a checkpoint's position in the hash chain + the *previous* checkpoint's independently-trusted `signed_at`) and, at GA, an **external timestamp anchor** (§6.4). A backdated forgery cannot insert itself before a genuinely-earlier checkpoint without forging that checkpoint's signature too (C2 §7.4 threat model). This is what makes the timeline test sound.
- **G4-R16 (SHOULD):** A lightweight **revocation feed** (the revoked-`key_id` set + `revoked_at`) ships in the trust bundle so offline verifiers apply revocation without contacting the platform (HL-18 offline-first).

### 6.4 External timestamp anchoring (GA hardening)
- **G4-R17 (SHOULD→MUST@GA):** Periodically anchor the KTL head and/or checkpoint roots to an **independent timestamp authority** (RFC 3161 TSA, a public transparency log, or a public-ledger commitment) so the platform cannot *retroactively* backdate. This converts "trust the platform's `signed_at`" into "trust an external clock," closing the single-root self-attestation gap (ADVERSARIAL §A-2). POC: the chained-checkpoint ordering is the floor; GA: an external anchor.

---

## 7. Trust root & verifier key acquisition (how an auditor gets trusted public keys)

This is what makes HL-18 step 8 ("manifest signature verifies against the platform key… independently verifiable outside the platform") real.

### 7.1 The trust-root hierarchy
A small, rarely-used **offline trust-root** sits above the operational signing keys:

```
        ┌─────────────────────────────────────────────┐
        │  G4 TRUST ROOT (offline, HSM, 5–10 yr)       │  ← signs the KTL head + key attestations only
        └─────────────────────────────────────────────┘
              │ attests / authorizes
   ┌──────────┴───────────┬──────────────────────┐
   ▼                      ▼                      ▼
K1 audit-ckpt (KMS)   K2 export-sign (KMS)   K6 mTLS issuing-CA (KMS)
   │ signs                │ signs                │ issues
checkpoints           export manifests       leaf certs
```

- **G4-R18 (MUST):** The trust root is **offline / air-gapped HSM**, used only to (a) attest a new operational key into the KTL and (b) sign the KTL head. It signs nothing high-frequency, so its attack surface is minimal and its rotation is rare (5–10 yr). Compromise of an *operational* key (K1/K2) is recoverable under the trust root (§9); compromise of the trust root is the catastrophic case (§9.5).
- **G4-R19 (MUST):** Operational keys (K1/K2) are valid **only** while attested in a KTL signed by the trust root. A verifier's chain of trust is: *trust-root public key (the one bootstrapped artifact)* → KTL signature → `key_id` entry → artifact signature.

### 7.2 The trust bundle (what the verifier holds)
- **G4-R20 (MUST):** The platform publishes a **trust bundle**: `{ trust_root_pubkey, KTL (full key history with all retired public keys), revocation_feed, [external timestamp anchors] }`, content-addressed and itself signed by the trust root. It is distributable as a file the auditor downloads once and keeps (e.g. in the signed export package itself, or from a published, separately-hosted location — *not* solely from the audited platform's live API, to preserve independence).
- **G4-R21 (MUST):** The **export package (C2 §7.6) MAY embed the relevant slice of the trust bundle** (the trust-root pubkey + the specific `key_id` entries needed to verify *that* package + revocation status), so a verifier can validate a 5-year-old export with nothing but the package on disk and a one-time-pinned trust-root key. *(This is the offline-decades-later property that pushed K1/K2 to KMS+KTL over keyless — ALT §3.)*
- **G4-R22 (MUST):** The **trust-root public key** is the single bootstrap of trust. It is published through multiple independent channels (documentation, a `.well-known` endpoint, the signed export, ideally a third-party/notary mirror) so an auditor can corroborate it out-of-band and pin it. Everything else chains from it.

### 7.3 Verifier tooling
- **G4-R23 (MUST):** Ship a **standalone, offline verifier** (a small static binary / library, no platform dependency) that takes a signed artifact or export package + a pinned trust-root key and outputs valid/invalid with the resolved `key_id`, the key's KTL status at `signed_at`, and the revocation/timeline determination (§6). This is the concrete instrument behind HL-18's "independent third party can re-verify offline." It also verifies the C2 hash chain + Merkle inclusion (C2 §10.6 `/verify` is the *online* counterpart; G4's verifier is the *offline* one).
- **G4-R24 (SHOULD):** The verifier also validates Sigstore (K3/K4) signatures against Fulcio/Rekor roots + the pinned signer-identity policy, so one tool covers both trust domains.

---

## 8. Custody separation of duties (who can sign, who can rotate)

Satisfies D4 SEC-11/SEC-12/SEC-23 and the "separate trust domains" tenet.

- **G4-R25 (MUST):** **Separate trust domains, separate custody:** K1/K2 (audit root), K3/K4 (supply chain), K6 (mesh PKI) live under **distinct KMS key rings / Sigstore identities / CAs** with distinct access policies. No single principal can sign in two trust domains. Compromise of the bundle-signing identity (K3) cannot forge an audit checkpoint (K1), and vice-versa.
- **G4-R26 (MUST):** **Rotation and revocation are dual-control** (two authorized operators / a `KeyRotation` approval gate reusing D3's approval-gated mechanism): no single operator can rotate or revoke an audit-root key, mirroring D4 SEC-11 (requester ≠ approver) and SEC-12 (privileged actions approval-gated). The trust-root operations (§7.1) are dual-control + offline-ceremony.
- **G4-R27 (MUST):** The principal that *operates* the platform (deploys, has `cluster-admin`) is **not** the principal that controls the audit-root KMS key policy. Otherwise the operator can both forge an event and sign over it — collapsing SoD. (KMS IAM is owned by a separate security-operations role; the cluster operator can *invoke* the signer via the least-priv signer workload identity but cannot *export, rotate, or change policy* on the key.)
- **G4-R28 (MUST):** Every key lifecycle event (generation, attestation, activation, rotation, revocation, trust-root ceremony) is **audited to C2** with actor, timestamp, and `key_id` (D4 SEC-16; the key log is itself evidence).

---

## 9. Key compromise recovery (the runbooks)

The META-ADVERSARIAL test: *"The signing key leaked yesterday — what do we do with 3 years of signed checkpoints?"* Answered per key class.

### 9.1 General principle
Compromise recovery is **never** "re-sign all of history." Re-signing history is impossible *with authority* (the new signer wasn't present 3 years ago; a fresh signature over old data proves nothing about *when* it was made) and unnecessary if the timeline model (§6) is sound. Recovery is: **revoke the bad key with a `revoked_at`, prove which signatures predate it, re-establish a new active key, and re-checkpoint forward.**

### 9.2 K1/K2 audit-root key compromise (the headline case)
1. **Revoke** the compromised `key_id` in the KTL with `revoked_at = T_leak_lower_bound` (the earliest time the key could have been exposed; if unknown, the conservative bound is when it was generated, but see step 3). Re-sign the KTL head (trust root) and re-publish the trust bundle + revocation feed.
2. **Generate gen-N+1** under the trust root (KMS, §7.1), attest into the KTL, distribute.
3. **Establish the honesty boundary:** every checkpoint signed by the compromised key *before* `revoked_at` is **valid-but-flagged** *iff* its position in the (independently-trusted) chain ordering and the **previous, differently-keyed or externally-anchored checkpoint** establish it was made before the leak (§6.2, §6.4). Checkpoints whose timing can't be independently established fall back to **`best_effort` trust** (honesty tenet — never silently "complete"). **The 3 years of checkpoints are NOT discarded; they are re-anchored under the new key by issuing a fresh checkpoint (gen-N+1) over the existing, unaltered chain head** — the chain content never changed, only the top-level attestation key did. The Merkle history is intact; you are re-attesting the *same* root with a *trusted* key going forward.
4. **External timestamp anchors (§6.4)**, if present, are what make step 3 strong: they prove pre-leak checkpoints existed before the leak, independent of any platform key. **This is the single most valuable GA hardening item** and the reason §6.4 is escalated.
5. **Disclose:** an auditor is told "key K1-gen-1 was compromised on date D; checkpoints before D are independently time-anchored and trustworthy; verification tooling flags them; nothing was silently re-signed." This is a *defensible* answer — the previous corpus had none.

### 9.3 Signing-capability hijack (KMS key not stolen, but call-ability abused)
KMS keys can't be exfiltrated, but an attacker who hijacks the signer workload identity can request signatures. Detection (G4-R5 signer audit trail + G4-R6 rate/context limits): anomalous signing volume or out-of-cadence checkpoints trigger an alert; response is to revoke the *workload identity's* KMS-invoke permission (fast, reversible) and rotate the key. Because the key never left the HSM, the blast radius is "signatures made during the hijack window," bounded by the timeline model.

### 9.4 ed25519 algorithm break (PQC long-horizon)
If ed25519 is ever broken, *all* ed25519 signatures become forgeable, including historical ones — the timeline model doesn't save you because an attacker could now forge a pre-dated signature. Mitigation is the **external timestamp anchor (§6.4)** (a forgery still can't insert itself before a genuine external timestamp) plus a planned **`alg` migration**: dual-sign forward with ed25519 + a PQC scheme, and rely on the external anchors for pre-migration history. Tracked as G4-R-PQC (GA+). This is why §6.4 is the keystone hardening.

### 9.5 Trust-root compromise (catastrophic)
The offline HSM trust root is the one key whose compromise has no in-band recovery (it can attest fraudulent operational keys into a fraudulent KTL). Mitigations: offline/air-gapped custody (§7.1), dual-control ceremonies (G4-R26), and **out-of-band trust-root distribution** (G4-R22) so a forged KTL signed by a stolen root is caught when its trust-root key doesn't match the auditor's independently-pinned copy. Recovery is a manual trust-root rotation ceremony + re-attesting the (unchanged) operational keys + re-publishing through all channels. This is accepted residual risk, minimized by keeping the root tiny, offline, and rarely used.

### 9.6 K3/K4 keyless compromise
No long-lived key to steal; "compromise" means the CI workload identity was abused to sign a malicious bundle/image. Detection: Rekor transparency log shows a signature from the legitimate identity at an unexpected time/for an unexpected artifact (the whole point of the transparency log). Response: the pinned signer-identity policy (B1-R20) + Rekor monitoring; rotate the CI workload identity; superseded bundles are re-built and re-signed; consumers fail-closed on the bad one once its identity/time is distrusted. No history-re-sign problem because bundle signatures are short-horizon (§5.4).

---

## 10. Failure modes

| Failure | Effect | Control |
|---|---|---|
| KMS/HSM unavailable at checkpoint time | Cannot sign new checkpoints | Buffer events in the (still hash-chained) log; checkpoint catches up when KMS returns — **the chain is intact without a signature; the signature is an additional attestation, not the chain itself** (C2 §7.3 vs §7.4). Fail-degraded, not fail-open. |
| KMS unavailable at export time | Cannot produce a signed export | Export fails closed (don't emit an unsigned package masquerading as signed); retry. |
| Trust bundle stale at verifier | Verifier can't resolve a *new* `key_id` | Overlap-based rotation (G4-R10) guarantees no artifact references an unpublished key; verifier refreshes bundle; offline verifier uses the embedded slice (G4-R21). |
| `key_id` collision / reuse | Wrong key resolved | G4-R3 forbids reuse; `key_id` derived from the public key so collision ⇒ same key. |
| Revoked key's old signatures mass-invalidated | Years of evidence wrongly voided | Timeline-aware revocation (§6) — pre-`revoked_at` signatures stay valid-but-flagged. |
| Salt rotation breaks `content_hash` | Chain reproducibility lost | G4-R12: salts are generational, never rotated in-window. |

---

## 11. Open questions — decided defaults (resolves D4-OQ-1)

| # | Question | Decided default | Rationale |
|---|---|---|---|
| **G4-OQ-1** | **Signing technology for the audit root (the D4-OQ-1 resolution)?** | **KMS/HSM-backed, non-exportable ed25519 (K1/K2), with an append-only Key Transparency Log + offline-distributable trust bundle.** Keyless (Sigstore) is used for the *supply-chain* roots (K3/K4) only. | Audit signatures are long-lived (5–10 yr) and must verify **offline, decades later, against a single stable public key** — the one thing per-build keyless is weakest at. Custody must exclude `cluster-admin` (tamper-evidence thesis). See full trade study in ALT. |
| **G4-OQ-2** | KMS vs full HSM at POC? | **Cloud KMS (HSM-backed key protection level) at POC; dedicated HSM optional at GA** for the highest-assurance buyers. | KMS gives non-exportable keys + audit + IAM without HSM procurement cost (POC-budget concern, ADVERSARIAL §A-4); the interface is identical, so GA hardening to a dedicated HSM is a config change. |
| **G4-OQ-3** | Reuse the K1 checkpoint key for K2 exports? | **No — a dedicated `export-signing` key by default**, same custody class. | Blast-radius separation; export volume/exposure differs from the checkpoint root. Cheap (another KMS key). |
| **G4-OQ-4** | External timestamp anchoring at POC? | **Chained-checkpoint ordering at POC (the floor); external RFC 3161 / public-log anchor at GA (MUST@GA).** | Anchoring is what makes compromise recovery and PQC mitigation strong (§6.4, §9.2/§9.4); the chain ordering is a sufficient POC floor. |
| **G4-OQ-5** | Trust-root custody at POC? | **Offline KMS key with dual-control + restricted IAM at POC; air-gapped HSM ceremony at GA.** | Keeps the root small and rare; POC doesn't need an air-gapped ceremony but must already separate the root from operational keys. |
| **G4-OQ-6** | PQC now? | **No — ed25519 now; `alg`-versioned migration path documented (G4-R-PQC, GA+).** | Quantum is a long-horizon theoretical risk; external timestamp anchors (§6.4) cover the store-now-verify-later gap without a premature PQC dependency. |

---

## 12. Dependencies

- **Consumes:** a KMS/HSM (F2 deployment wires it); Sigstore (Fulcio/Rekor) for K3/K4 (B1 already depends on it); cert-manager + internal CA for K6 (F2); Keycloak JWKS for K7 (D1).
- **Consumed by / gates:**
  - **D4** — resolves SEC-2 (bundle signing impl), SEC-8 (checkpoint key), SEC-21 (image signing key), SEC-23 (key management); promotes them from "interface only" to a decided implementation. **Resolves D4-OQ-1.**
  - **C2 §7.4/§7.6** — provides the signer + key lifecycle behind every checkpoint and export; C2 owns the envelope, G4 owns the key.
  - **B1 §6.2** — adopts B1's cosign-keyless decision (D-B1-03) unchanged and adds the signer-identity-policy lifecycle (B1-R20).
  - **C1 §4 / C5 §3.4** — their signed exports use K2 via the C2 primitive.
  - **D1 §4** — G4 owns the trusted-issuer-set lifecycle (K7); D1 owns the verification logic.
  - **D3-R9** — approval-callback key (K5).
- **Cross-domain contract:** the §2 inventory + the `key_id`-resolves-key rule (G4-R7) + the trust bundle (§7.2) + the offline verifier (§7.3) are the surface other components verify against. **The single invariant every consumer relies on: a signature's `key_id` resolves, via the append-only KTL, to a public key that is never removed — so historical signatures verify forever.**
