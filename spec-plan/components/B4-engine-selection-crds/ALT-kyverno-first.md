# B4 — ALT Architecture: Kyverno-First vs. OPA/Gatekeeper-First

**Persona:** alternative-architecture author. **Question:** for the Kubernetes enforcement layer,
should the platform be **Kyverno-first** (YAML-native engine as primary, OPA as cross-product brain)
or **OPA/Gatekeeper-first** (the primary SPEC's posture)?

---

## 1. Why this is a live question now (market reality)

Per market research §3: Kyverno matured significantly — Nirmata's Nov 2025 **AI Platform Engineering
Assistant** (multi-agent policy authoring/detection/remediation, copilot, signed-PR workflows,
rollback safety, SSO, granular RBAC, tamper-proof audit, evidence export) is essentially a competing
product to the platform's console + authoring + approval-workflow (§16/§17A/§17B) — *but Kyverno-only*.
Meanwhile Gatekeeper is commoditized (all three hyperscalers ship it) but its UX for K8s teams is
weaker than Kyverno's YAML-native model. The spec already uses both; this ALT asks which is *primary*.

---

## 2. Path A — Kyverno-first (this ALT's alternative)

### Architecture
- **Kyverno is the primary Kubernetes engine** for admission validate/mutate/generate/cleanup/
  image-verify. K8s teams author YAML-native ClusterPolicy/Policy resources.
- **OPA/Rego remains the cross-product decision brain** for non-K8s PDPs (CI/CD, identity, scanners,
  apps, replay) and for genuinely complex/identity-aware K8s decisions, surfaced to Kyverno via its
  **external-data / API-call** mechanism or Kyverno's emerging CEL/JSON logic.
- Gatekeeper is **optional/secondary**, used only where a customer already runs it (hyperscaler-managed)
  or needs Rego-centric admission specifically.
- B4's action taxonomy + CRDs are unchanged; Kyverno becomes the dominant *effector*.

### Pros
- **Best UX for Kubernetes teams** (Sam/Marcus personas): YAML-native, no Rego required for the
  common 80% of K8s policies (require labels, restrict registries, mutate defaults, generate
  NetworkPolicies, verify signatures).
- **Native operational effects** the spec explicitly routes to Kyverno (§17C.1/2): generate, mutate,
  cleanup, image-verify are first-class — no custom controllers needed for those (shrinks B4's
  PolicyRemediationAction surface).
- **Native policy reports** (Kyverno PolicyReport CRDs) → normalized evidence (C2) with less glue.
- **Rides Kyverno's AI tooling momentum** (Nirmata assistant) — the market is investing here.
- Single-engine simplicity for the K8s layer (one engine to operate, not Gatekeeper+Kyverno+OPA).

### Cons / risks
- **Fractures the cross-product-consistency thesis.** The platform's core claim (§17C: one decision
  model across K8s/CI/identity/scanners/apps) weakens if K8s decisions live in Kyverno YAML/CEL and
  everything else lives in Rego. Now there are *two* policy languages and two evaluation semantics;
  "the same control everywhere" becomes "the same control, expressed twice."
- **Identity-aware + complex decisions** are weaker in Kyverno than Rego (the matrix in SPEC §3.2
  scores Kyverno 1 on complex logic / identity). Pushing those into Kyverno external-data calls to
  OPA reintroduces the OPA dependency anyway — and on the admission hot path (latency, availability).
- **Replay/retrospective** (§19) is Rego's strength; Kyverno policies don't replay over normalized
  audit logs the way Rego does. K8s decisions made in Kyverno would need *re-expression in Rego* for
  replay — duplicating logic and risking divergence (the exact thing the platform exists to prevent).
- Risk of being a thin wrapper over Nirmata's product for the K8s layer, with unclear added value.

---

## 3. Path B — OPA/Gatekeeper-first (the primary SPEC's posture)

### Architecture (as SPEC §3, D-B4-01)
- OPA/Rego is the decision brain *and* the K8s admission decision logic (via Gatekeeper); Kyverno is
  used **only** for the native effects it's uniquely good at (generate/mutate/cleanup/image-verify),
  selected by the rubric (§3.2). One language (Rego) for *decisions* everywhere.

### Pros
- **Cross-product consistency is real, not aspirational:** one decision language (Rego), one replay
  story, one conformance suite (B1-R30) — the platform's central differentiator holds.
- **Identity-aware + complex + replay** decisions are first-class (Rego's strengths), uniformly.
- Kyverno is still used where it shines (effects), so no capability is lost — the rubric (§3.2)
  routes effects to Kyverno explicitly.
- Aligns with the spec as written; less re-architecture.

### Cons / risks
- **Worse K8s-team UX** for simple policies: Rego + ConstraintTemplates are heavier than Kyverno YAML
  for "require a label." Risks adoption friction with the very teams (Sam) who'd prefer Kyverno.
- **Gatekeeper UX/tooling lags Kyverno's** (esp. the new AI-assisted authoring) — the platform must
  build console/authoring (§16/§17A) that Kyverno's ecosystem increasingly provides off-the-shelf.
- Two engines (Gatekeeper + Kyverno) to operate for K8s anyway (effects still need Kyverno), so the
  "one engine" simplicity benefit of Kyverno-first is partially available but not taken.

---

## 4. Trade-off summary

| Dimension | A: Kyverno-first | B: OPA/Gatekeeper-first (SPEC) |
|---|---|---|
| K8s-team authoring UX | **Strong (YAML)** | Weaker (Rego/templates) |
| Native effects (gen/mutate/cleanup) | **Native** | Via Kyverno anyway (rubric) |
| Cross-product consistency | Weak (two languages) | **Strong (one Rego)** |
| Identity-aware/complex decisions | Weak (→ external OPA) | **Strong** |
| Retrospective replay (§19) | Weak (re-express in Rego) | **Strong** |
| Conformance "one rule everywhere" | Broken for K8s | **Holds** |
| Rides market/AI momentum | **Yes (Nirmata)** | Partial |
| Engine count to operate (K8s) | Lower (mostly Kyverno) | Higher (GK+Kyverno) |

---

## 5. Recommendation (this ALT's verdict)

**Keep OPA/Gatekeeper-first for *decisions*, but adopt Kyverno-first for *effects*, and make the
decision/effector split (R-B4-2) the load-bearing architectural commitment** — which is exactly
what the primary SPEC's D-B4-01 says. The honest conclusion of stress-testing Kyverno-first is:

- The platform's **entire reason to exist** is cross-product consistency + replay (the thing GRC
  suites and CNAPPs and Kyverno-only products *don't* do). Kyverno-first sacrifices precisely that
  to gain K8s-team UX. That's a bad trade for *this* platform's thesis.
- **But** the UX concern is real and the SPEC under-weights it. Mitigation: invest in
  Rego/ConstraintTemplate *authoring ergonomics* (generators, the console §16, Regal) so K8s teams
  don't hand-write Rego for simple policies, and **use Kyverno aggressively for effects** so the
  "Kyverno does the satisfying operational stuff" experience is still there.
- **Watch item:** if Kyverno's CEL/JSON logic + AI authoring closes the complex/identity-aware gap
  AND grows a replay story, re-evaluate. Today (May 2026) it has not, so Kyverno-first would trade
  the platform's differentiator for ergonomics it can otherwise mitigate.

**Decision: OPA-first for decisions, Kyverno-first for effects (= the SPEC's split), with a
deliberate, funded investment in Rego authoring UX to neutralize Kyverno-first's main advantage.**
