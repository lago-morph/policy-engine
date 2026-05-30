# B4 — ALT Architecture: Build on OPA Control Plane (OCP) vs. Parallel Custom Engine

**Persona:** alternative-architecture author. **Question:** should the platform's policy
lifecycle / bundle distribution / regression substrate (which B4's CRDs and B1's bundles sit on)
be **OCP-native**, or a **parallel custom engine** the platform builds and controls end-to-end?

---

## 1. The fork in the road

The primary SPEC (D-B1-01, D-B4-01) hedges: "adopt OCP behind a BundleService abstraction." That's
a *both/and* posture. This ALT forces the choice and trades them off honestly, because the hedge
has a hidden cost — building the abstraction AND understanding OCP AND keeping a fallback is more
work than committing to one path.

- **Path A — OCP-native:** Make OPA Control Plane the platform's spine. The platform's bundle build,
  multi-repo composition, global/hierarchical policy injection, HA distribution to S3/GCS/Azure, and
  **regression testing against historical decision logs** are OCP features the platform *configures*,
  not *reimplements*. B4's CRDs (PolicySimulationRun especially) become thin drivers over OCP.
- **Path B — Parallel custom engine:** The platform owns its own bundle pipeline (`opa build` + oras
  + cosign), its own distribution, its own regression-test harness over decision logs, its own
  composition. OCP is not a dependency; OPA is used only as the eval engine.

---

## 2. Path A — OCP-native

### Architecture
- `BundleService` (B1 §6.4) is implemented *as a configuration layer over OCP*. PolicySimulationRun
  (B4 §5.3) historical-replay/differential modes call OCP's regression-testing API directly.
- Global/hierarchical policy (e.g. "all prod namespaces also get baseline controls") uses OCP's
  build-time label-selector injection rather than the platform composing bundles itself.
- Policy lifecycle (A2) leans on OCP's Git-based multi-repo bundle building.

### Pros
- **Massive scope reduction.** §7 (lifecycle), §8.2 (signed/versioned OCI bundles), §14 (bundle
  behavior analytics), §17 (simulation vs historical decisions) overlap OCP "almost line-for-line"
  (market research §2). Reimplementing them is months of work for a worse result.
- **Regression-testing-against-decision-logs comes for free** — this is the engine behind E1
  differential sim (DT-49/HL-17) and the A2 promotion gate. Building it from scratch is hard.
- **CNCF-governed, ex-Styra/Apple-origin maturity.** The primitives are battle-tested commercial
  code, now open. Riding them means inheriting years of edge-case handling.
- Aligns the platform with where the OPA ecosystem is heading post-Styra; reduces "we built a worse
  Styra DAS" risk.

### Cons / risks
- **Freshly open-sourced; maturity/maintenance unproven.** Originating team is at Apple; CNCF
  stewardship is new. Breaking changes, slow patches, or de-facto abandonment are real risks (B1-AR-1).
- **API/feature coupling.** Even behind an abstraction, leaning on OCP's regression-testing and
  label-selector injection means those concepts shape the platform; truly swapping out OCP later
  would mean re-building exactly the hard parts.
- **Operational dependency** on OCP's distribution model (S3/GCS/Azure HA) — fine for cloud, awkward
  for air-gapped/on-prem unless OCP supports it well.
- Less control over the exact provenance/signing flow the platform wants (cosign keyless + in-toto)
  if OCP's signing model differs.

---

## 3. Path B — Parallel custom engine

### Architecture
- Platform owns: bundle build (`opa build` + metadata injection), OCI publish (oras), cosign signing
  + in-toto attestation, HA distribution (platform's choice of registry/CDN), and a bespoke
  regression harness that replays B1 decision logs (C2) through candidate bundles.
- PolicySimulationRun reconciles by invoking the platform's own replay harness.

### Pros
- **Full control + no external maturity risk.** Every primitive is owned; provenance/signing is
  exactly the platform's (cosign keyless + in-toto, B1-R17/18); air-gapped is straightforward.
- **No strategic dependency** on a freshly-dumped project's roadmap.
- Tighter integration with the platform's audit schema (C2) and replay schema — the regression
  harness can be purpose-built for the platform's exact decision-log format and nd_builtin_cache
  replay (B1-R26), rather than adapting to OCP's.
- Cleaner story for on-prem/regulated customers who can't depend on a CNCF sandbox project.

### Cons / risks
- **Reimplements months of mature functionality** — multi-repo composition, hierarchical injection,
  HA distribution, and especially **regression-testing-against-decision-logs**, which is genuinely
  hard to get right (deterministic replay, decision-log windowing, diff semantics).
- High risk of building "a worse Styra DAS / OCP" — exactly the trap market research §2 warns about.
- Ongoing maintenance burden the platform team owns forever.
- Slower to MVP; the differential-simulation differentiator (a key selling point) arrives later.

---

## 4. Trade-off summary

| Dimension | A: OCP-native | B: Parallel custom |
|---|---|---|
| Time-to-MVP | **Fast** | Slow |
| Scope/effort | **Low** | High |
| Regression-vs-decision-logs | **Free** | Build it (hard) |
| Strategic dependency risk | High (new project) | **Low** |
| Air-gapped / on-prem | Uncertain | **Strong** |
| Provenance/signing control | Medium | **Full** |
| "Built a worse DAS" risk | **Low** | High |
| Maintenance burden | **Low** | High |

---

## 5. Recommendation (this ALT's verdict)

**Adopt a phased hybrid that resolves the primary SPEC's hedge with a sequence, not a permanent
both/and:**

1. **MVP on Path A (OCP-native)** behind the `BundleService` abstraction, to get the
   regression/simulation differentiator and lifecycle for free and ship fast.
2. **Keep Path B's signing/provenance** (cosign keyless + in-toto) under platform control from day
   one — do NOT outsource the trust root to OCP (this is the one piece worth owning regardless).
3. **Gate a re-evaluation** at the point OCP would become a hard dependency for air-gapped customers
   or if OCP shows abandonment signals; the abstraction makes a later Path-B build for the *distribution
   + regression* pieces feasible without re-touching Rego/CRDs.

Net: ride OCP for the expensive-to-build, low-differentiation plumbing (composition, distribution,
regression harness); own the trust root and the platform's CRD/audit contracts. The primary SPEC's
"OCP behind BundleService" is right **provided** the team commits to actually using OCP for MVP
rather than building a fallback in parallel — building both is the worst option (cost of both, benefit
of neither). **Decision: commit to A for MVP, own signing, hold B in reserve behind the abstraction.**
