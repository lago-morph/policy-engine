# Domain B — Policy Engines & Enforcement — SUMMARY

**Domain lead deliverable.** Date: 2026-05-30. ~2 pages.

---

## 1. What Domain B is

Domain B is the platform's **enforcement spine**: it executes governance controls (from Domain A /
Gemara) as real decisions and effects across engines and enforcement points, and emits the audit
evidence that Domains C/E/F depend on. Five components:

- **B1 (OPA/Rego + signed bundles)** — the decision substrate. One Rego language, one canonical
  `decision` entrypoint, signed/versioned OCI bundles, decision logs. Everything else evaluates *this*.
- **B2 (Gatekeeper)** — the Kubernetes admission effector for Rego decisions; four enforcement modes;
  the 17-field audit contract; the deny-with-approval-required pattern.
- **B3 (Conftest)** — the build-time (shift-left) effector; same controls evaluated in CI + pre-commit;
  normalized evidence.
- **B4 (engine selection, taxonomy, CRDs)** — the decision/extension layer: the rubric for which
  engine, the closed 13-action vocabulary, and the six CRDs that hold durable state for things engines
  can't do synchronously (approvals, exceptions, simulation, remediation).
- **B5 (real-time flow)** — the integration spec: the precise admission→decision→audit sequence, the
  correlation_id contract, the latency budget, and the §17B.4 invariant.

---

## 2. The shared data model (consolidated)

```
Gemara control (A1)
   ▲ control_id
RegoPackage ──has──► canonical `decision` { action ∈ 13-taxonomy, allowed, severity, messages,
   │                                         obligations[], approval?, evidence{}, policy{} }
   │ bundled into
PolicyBundle (semver + digest) ──signed(cosign)+attested(in-toto)──► OCIArtifactRef
   │ activated by
OPA agents / Gatekeeper / Conftest ──emit──► DecisionLog / GatekeeperDecision(17 fields) / NormalizedEvidence
                                                   │ all keyed by
                                              correlation_id ──► C1 Privateer / C2 audit / C3 analytics / C4 §19
   when action == require_approval/exception:
PolicyApprovalRequest CRD  /  PolicyException CRD  (B4) ◄── consulted at admission (B2) via bounded external-data
```

**Three things bind the whole domain together** (the contract surface): the **canonical `decision`
object** (B1), the **closed 13-action taxonomy** (B4), and the **correlation_id** (B5). If any of
these three drifts between components, the platform's cross-product-consistency thesis fails.

---

## 3. The hardest decisions (5)

1. **OPA decides, engines effect (decision/effector separation, R-B4-2).** The central
   architectural commitment: keep all *decisions* in Rego (cross-product consistency + replay) and
   route *effects* (mutate/generate/cleanup/image-verify) to Kyverno, admission to Gatekeeper,
   build-time to Conftest. Both B4 ALTs (OCP-substrate, Kyverno-first) stress-tested this and
   *confirmed* it is right for *this* platform's thesis — but only if Rego authoring UX is funded
   (else teams resent it and drift to Kyverno-first).

2. **Long-running approvals must never block admission (§17B.4).** Resolved uniformly as
   **deny-with-approval-required + PolicyApprovalRequest CRD + bounded retry** (B2-R17/B4-R6/B5-R5).
   This is the strongest idea in the domain and the riskiest to implement (durability of the
   deny→CRD hand-off, single-use idempotency, no held webhook). It is the keystone integration
   spanning B2+B4+B5 and should be built as one vertical slice.

3. **Adopt OCP (OPA Control Plane) as the bundle/lifecycle/regression substrate, behind an
   abstraction.** Post-Styra/Apple, OCP gives bundle build/distribution/regression-vs-decision-logs
   nearly for free (the differential-simulation differentiator). Decided: commit to OCP for MVP, own
   the trust root (cosign+in-toto) regardless, hold a custom fallback in reserve (B4 ALT-ocp verdict).

4. **The input-normalization contract (admission-envelope canonical).** For "one Rego everywhere" to
   be true, the *same package* must evaluate Gatekeeper's `input.review.object`, Conftest's bare
   document, and an app PDP's input. Decided: normalize all to the admission envelope (B3-R10) so
   Gatekeeper needs no shim. **This is the most under-specified cross-cut and the adversarial reviews
   flag it hard (B1-AR-2, B2-AR-6, B3-AR-2) — see DOMAIN-ADVERSARIAL.**

5. **Fail-closed defaults vs. cluster availability.** Runtime decisions fail closed (deny) on JWT-
   unresolved / decision-error / engine-down — secure but a cluster-bricking and self-amplifying-
   outage risk (B1-AR-6, B5-AR-7). Mitigated by the system-namespace carve-out (B2-R4) which is
   *itself* a §19-monitored bypass surface. The tension is real and only partially resolved.

---

## 4. Consolidated open questions (decided defaults; revisit triggers in parens)

| # | Question | Decided default | Revisit if |
|---|---|---|---|
| 1 | Build pipeline: OCP vs custom? | OCP behind BundleService; own signing | OCP shows abandonment / air-gapped need |
| 2 | K8s engine: OPA-first vs Kyverno-first? | OPA decisions + Kyverno effects | Kyverno closes complex/identity/replay gap |
| 3 | Canonical input shape? | Admission envelope (Conftest/app wrap) | author ergonomics prove unworkable |
| 4 | Action set: closed vs extensible? | Closed 13 (B4-R5) | AI-gov / new effects needed (B4-AR-6) |
| 5 | Action model: linear precedence vs disposition+obligations? | **OPEN — adversarial says split** (B4-AR-7) | resolve before C2 bakes it in |
| 6 | Cold-start / fail-closed posture? | Fail-closed runtime + system-ns carve-out | brick-guard test fails |
| 7 | Replay-exactness for external-data decisions? | **OPEN — must capture external-data values** (B5-AR-5) | flagship replay claim depends on it |
| 8 | Approval `requiredApproval` source? | **MUST be controller-derived, not author-writable** (B4-AR-1) | security blocker — pre-GA |
| 9 | Latency budget realism? | ≤2s p99 default — **needs perf model** (B5-AR-1) | deploy-storm load test |
| 10 | Approval w/o external workflow system? | **OPEN — need built-in default approver path** (B4-AR-11) | §17B.3 is out-of-scope but feature depends on it |

OQ5, OQ7, OQ8, OQ10 are **carried as decided-but-contested** — the adversarial reviews argue they need
revision before downstream components (C2, E1) bake in assumptions. See DOMAIN-ADVERSARIAL.

---

## 5. Build order (domain-internal)

Freeze **B4's action taxonomy** (closed 13) + **B1's decision contract** + **B5's correlation_id
contract** first — they're the shared vocabulary. Then B1 signing/bundles, B2 admission + 17 fields,
B3 build-time, in parallel. The **approval vertical slice (B2+B4+B5)** is the keystone and highest
risk. B5 integrates last; C4/§19 depends on B5's expected-decision-set export. The **cross-engine
conformance suite (B1-R30)** — proving REST=Wasm=Gatekeeper=Conftest — is the single highest-value
test and the gate for claiming "shared cross-product semantics."
