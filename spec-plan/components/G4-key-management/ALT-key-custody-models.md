# G4 — ALT: Key Custody Models — KMS/HSM vs Sigstore-keyless vs Self-managed in-cluster

**Component ID:** G4 · **Doc type:** Alternative-architecture trade study (high-value).
**Question under study:** *Where do the platform's signing keys live, and how does that choice trade off trust, cost, ops, and — critically — offline long-horizon verification?* This is the decision that resolves D4-OQ-1, and it is genuinely contested, so it gets a full ALT rather than a one-line default.

**The SPEC's chosen answer (for reference):** a **split** — KMS/HSM-backed for the *audit roots* (K1/K2) and *internal CA root* (K6), Sigstore-keyless for the *supply-chain roots* (K3/K4), in-cluster only for *non-verifier-facing internal material* (K5/K7/K8). This ALT defends that split by examining all three models as if each were applied to *all* signing, then showing why no single model wins everywhere and the split is the right shape.

---

## 1. The three models

### Model A — KMS/HSM-backed (centralized, non-exportable keys)
The private key is generated inside and never leaves a Key Management Service / Hardware Security Module. Signing is an authenticated API call (`KMS.Sign(key_id, digest)`); the key material is never in cluster memory, a Secret, a file, or CI. Verification uses the corresponding public key, distributed out-of-band. Rotation = a new KMS key version; old public keys retained in a key history.

- **Examples:** AWS KMS / GCP Cloud KMS / Azure Key Vault (HSM-protection tier), CloudHSM, on-prem PKCS#11 HSM, HashiCorp Vault Transit.

### Model B — Sigstore keyless (ephemeral keys + transparency log)
No persistent private key. At signing time the workload obtains a short-lived (~10 min) X.509 cert from **Fulcio**, bound to its OIDC workload identity, signs once with an in-memory ephemeral key, and records the signature + cert in the **Rekor** public/append-only transparency log. Verification checks: the cert chains to the Fulcio root, the signer identity matches a pinned policy, and the signature is present in Rekor (inclusion proof).

- **Examples:** cosign keyless (the B1 D-B1-03 default for bundles), GitHub Actions OIDC → Fulcio, npm/PyPI provenance.

### Model C — Self-managed in-cluster keys
The private key is a long-lived secret the platform holds itself: a Kubernetes Secret, a mounted file, a secret-manager entry the app reads into memory, or a key the application generates and stores. Signing happens in-process. Verification uses a public key the platform publishes.

- **Examples:** an ed25519 key in a `Secret`, a JWT-style signing key in app config, sealed-secrets, a Vault static secret read into the pod.

---

## 2. The evaluation axes (what actually matters for THIS product)

The product is a **long-horizon (5–10 yr), auditor-trustable, offline-verifiable evidence platform**. That weights the axes unusually:

| Axis | Why it dominates here |
|---|---|
| **Offline long-horizon verification** | HL-18's success criterion is literally "an independent third party can re-verify the signature offline," for evidence that may be 5+ years old. This axis is the product's reason to exist and weights heaviest. |
| **Trust / forgeability under insider threat** | The threat model (C2 §7.1) is the *operator/insider* deleting or editing evidence. A custody model that lets `cluster-admin` sign defeats the thesis. |
| **Cost (incl. POC budget)** | F3 §22 is a small functional POC; an expensive model gets skipped, landing back in the worst model (C). |
| **Ops complexity / day-2 burden** | META-ADVERSARIAL Risk #10: nobody owns day-2. A model that adds an HSM ceremony to the on-call surface is a real cost. |
| **Rotation safety for long-lived signatures** | The G4 spine: rotation must not break historical verification. |
| **Independence from the audited party** | Auditor-grade verification must not depend on the auditee's live, honest API. |

---

## 3. Head-to-head on the dominating axis: offline long-horizon verification

This is where the models genuinely diverge and where the SPEC's split is decided.

### Model A (KMS) — STRONG for long-horizon offline
The verifier needs only the **public key**, resolved by a stable `key_id` from an append-only key history. A 2019 signature verifies in 2026 with a 32-byte ed25519 public key the auditor can hold in a file, pin, and corroborate out-of-band. No live service required. Rotation retains old public keys, so history stays verifiable. **This is exactly the property HL-18 demands.** The cost is that *you* must run the key-history/transparency infrastructure (the KTL) — Sigstore gives you Rekor for free; with KMS you build the KTL yourself (G4 WS-3).

### Model B (Sigstore keyless) — WEAKER for long-horizon offline, despite being "transparency-native"
This is the counterintuitive finding and the crux of the ALT. Sigstore is *excellent* for supply-chain provenance, but its long-horizon **offline** story for *audit evidence* is worse than KMS:
1. **Verification requires Rekor (a live external service) or a downloaded Rekor snapshot.** Offline verification of a keyless signature means the auditor must have obtained a Rekor inclusion proof and the Fulcio/Rekor roots *at signing time or via a bundle*. It is doable (cosign supports offline bundles) but it is **more material to carry, from more parties, than a single pinned public key** — and the trust roots (Fulcio CA, Rekor key) themselves rotate, so a 5-year-old keyless signature requires the *historical* Fulcio/Rekor roots, which the auditor must have archived. The dependency surface for "verify this in 2031" is larger.
2. **The signing identity is a CI workload OIDC identity, not a durable institutional key.** An auditor verifying a 5-year-old evidence package wants "signed by *the platform's audit-evidence authority*," a stable, nameable trust anchor — not "signed by an ephemeral cert issued to `github-actions[bot]@release-pipeline` in 2021," whose meaning requires reconstructing the CI trust context years later.
3. **Per-build ephemerality is a feature for bundles (short-lived) and a liability for evidence (long-lived).** Every audit checkpoint and export would be a *distinct ephemeral identity*; there is no single durable public key the auditor pins once. The thing that makes keyless great for CI (no long-lived key) is the thing that makes it awkward for "pin one authority and verify a decade of its signatures."

**This is why the SPEC does NOT use keyless for K1/K2** — not because keyless is insecure (it isn't), but because the *audit root* needs a **durable, single, offline-pinnable authority**, which is precisely Model A's shape and precisely *not* keyless's shape.

### Model C (in-cluster) — verification fine, trust catastrophic
Offline verification with a published public key works *mechanically*. But the trust is void: the signature attests nothing because anyone with `cluster-admin` could have made it (§A-3 in the adversarial review). For *audit* evidence this is disqualifying. For *internal* trust (K5/K7/K8), where the verifier is the platform itself and the threat isn't "prove to an external auditor," it's acceptable.

---

## 4. Full comparison matrix

| Axis | A · KMS/HSM | B · Sigstore keyless | C · In-cluster |
|---|---|---|---|
| **Offline long-horizon verify** | **STRONG** — single pinnable public key, KTL retains history | MODERATE — needs archived Fulcio/Rekor roots + inclusion proofs; larger dependency surface | Mechanically works, but trust is void (C) |
| **Forgeability under insider** | **STRONG** — key non-exportable; cluster-admin can't steal it (residual: invocation hijack, §9.3) | STRONG — no persistent key to steal; abuse is visible in Rekor | **FAILS** — cluster-admin signs at will; defeats the thesis |
| **Durable, nameable authority** | **YES** — "the platform audit-evidence key, `auditckpt:…`" | NO — ephemeral CI identity per signature | Yes but meaningless (untrusted) |
| **Transparency / public auditability** | Self-hosted KTL (you build it) | **NATIVE** — Rekor public log is the headline feature | None |
| **Cost (POC)** | **LOW** — managed KMS key ≈ cents/mo + pennies/op | **LOWEST** — free public Sigstore infra | Lowest infra, highest *risk* cost |
| **Cost (prod volume)** | Per-op KMS cost at 10–100M events/day is real, unmodeled (ADV D-G4-8) | Free, but per-build only — N/A for high-freq audit signing | Free |
| **Ops / day-2 burden** | MODERATE — IAM, rotation, KTL; HSM ceremony at GA | LOW — CI-integrated, no key to manage | LOW infra / HIGH incident risk |
| **Rotation safety (long-lived sigs)** | **STRONG** — overlap rotation + retained public keys | N/A (per-build, no rotation concept) — but root rotation is a real archive concern | Manual, error-prone |
| **Independence from audited party** | **STRONG** — public key + KTL slice held by auditor | STRONG — Rekor is third-party — *if* it's the public instance | **FAILS** — verify against the auditee's own published key |
| **Fit for: audit roots (K1/K2)** | ✅ **chosen** | ❌ ephemeral, awkward offline | ❌ defeats thesis |
| **Fit for: supply chain (K3/K4)** | ⚠️ works, more ops | ✅ **chosen** (short-lived, CI-native, Rekor provenance) | ❌ long-lived key in CI |
| **Fit for: internal (K5/K7/K8)** | ⚠️ overkill | ❌ wrong shape | ✅ **chosen** (no external verifier) |

---

## 5. Pure-strategy alternatives (and why each loses)

### ALT-1: "Everything KMS/HSM"
Use Model A for *all* signing, including bundles and images.
- **Pro:** one custody model; one mental model; everything has a durable authority.
- **Con:** loses Sigstore's free public transparency log for the supply chain (the exact thing the broader ecosystem — Kyverno/Gatekeeper image-verify, B1 D-B1-03 — has standardized on); reintroduces a long-lived bundle-signing key in CI (a steal target); more per-build ops; swims against the supply-chain industry current. **Rejected** because supply-chain signing is short-horizon and high-frequency — Sigstore's sweet spot — and B1 already decided keyless.

### ALT-2: "Everything Sigstore keyless"
Use Model B for *all* signing, including audit checkpoints/exports.
- **Pro:** no long-lived keys anywhere; native transparency; lowest cost; fashionable.
- **Con:** the §3 finding — **bad fit for long-horizon offline audit verification.** Audit evidence needs a durable, single, offline-pinnable authority, not a per-signature ephemeral CI identity whose verification in 2031 requires archived historical Fulcio/Rekor roots. Also: signing 10–100M audit events/checkpoints through Fulcio (rate-limited public infra) is operationally wrong; you'd self-host Sigstore, at which point you've taken on *more* ops than KMS for a worse audit-verification story. **Rejected** as the audit root — this is the single most important rejection in the ALT and the direct answer to "why not just use the keyless thing B1 already uses everywhere?"

### ALT-3: "Everything in-cluster (the do-nothing / POC-shortcut path)"
Use Model C for everything because it's free and simple.
- **Pro:** zero infra, fastest to demo.
- **Con:** **defeats the entire product.** A tamper-evidence platform whose audit key any operator can read is integrity theatre (META-ADVERSARIAL G-6; ADV §A-3). This is the path the POC budget pressure (ADV §A-4) tempts teams toward, and the SPEC's KMS-at-POC decision (G4-OQ-2) exists specifically to make rejecting it cheap. **Rejected, hard** — and named explicitly so a future budget-pressured implementer can't quietly choose it.

---

## 6. Why the SPEC's split wins

No single model dominates all axes because **the platform signs two fundamentally different kinds of thing:**

1. **Long-horizon institutional attestations** (audit checkpoints, evidence exports) that an *external auditor* verifies *offline* *years later* → needs a **durable, single, offline-pinnable, non-exportable authority** → **Model A (KMS/HSM)**.
2. **Short-horizon supply-chain provenance** (Rego bundles, platform images) produced by *CI*, verified *at load/deploy time* → needs **ephemerality, transparency, no long-lived CI key** → **Model B (Sigstore keyless)**.
3. **Internal-only trust** (approval callbacks, JWKS pins, redaction salts) with *no external verifier* → convenience acceptable → **Model C (in-cluster), explicitly excluded from anything an auditor verifies.**

The split is not fence-sitting; it is recognizing that "audit evidence" and "supply-chain artifact" have **opposite** optimal custody shapes (durable-single-authority vs ephemeral-transparent), and forcing one model onto both makes one of them worse. The SPEC's split also keeps the **trust domains separated** (a §8 requirement): a compromise of the keyless CI identity (K3) physically cannot forge an audit checkpoint (K1) because they live in different custody systems entirely — a property neither pure-strategy alternative provides as cleanly.

---

## 7. The one place the choice is genuinely close (and the tiebreaker)

For **K2 (evidence-package export signing)** specifically, keyless is more tempting than for K1: exports are discrete artifacts (like a release), Rekor-logging each export would give nice public provenance, and the volume is lower than per-event checkpoints. The tiebreaker that keeps K2 on KMS: **an auditor verifying a 5-year-old export wants the *same* pinned authority they used for the checkpoints inside it** — a uniform "platform audit-evidence authority" public key — not a different (keyless) trust path for the wrapper than for the contents. Uniform offline verification (one pinned key resolves both the checkpoints and the export signature) beats per-artifact transparency for *this* product. Hence K2 = KMS, same custody class as K1, but a *separate key* for blast-radius (G4-OQ-3).

---

## 8. Recommendation

**Adopt the SPEC's split.** It is the correct shape, and this ALT's value is in making the *rejections* explicit and defensible:
- **Do not** unify on keyless (ALT-2) — it's the wrong shape for long-horizon offline audit verification, even though B1 already uses it for bundles. The instinct "we use Sigstore for bundles, just use it for everything" is the most likely wrong turn, and §3 is the argument against it.
- **Never** use in-cluster (ALT-3) for anything an auditor verifies — it's the budget-pressure trap that silently voids the product.
- **KMS-not-dedicated-HSM at POC** (G4-OQ-2) removes the cost excuse for taking the ALT-3 shortcut.

The residual risks this ALT surfaces for the SPEC/adversarial to own: KMS per-operation cost at production volume (unmodeled — ADV D-G4-8), the self-hosted KTL you take on by choosing KMS over Rekor (it's the price of the durable-authority property — WS-3), and the fact that *both* A and B ultimately bottom out in an external-anchor/out-of-band-root question (ADV §A-2/§A-5) that the SPEC defers to GA.
