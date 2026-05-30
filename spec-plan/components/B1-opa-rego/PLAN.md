# B1 — OPA / Rego Integration & Signed Bundles — PLAN

**Component:** B1 · **Pairs with:** SPEC.md · **Date:** 2026-05-30

---

## 1. Dependency DAG

```
[A1 Gemara catalog API] ──┐
[D1 JWT claim contract] ───┼──► W1 Rego conventions & metadata schema ──► W2 Authoring/lint (Regal rules)
[B4 action taxonomy] ──────┘                                                │
                                                                            ▼
[OCP eval / D-B1-01] ──► W3 BundleService (build/sign/publish/verify) ──► W4 Distribution & activation (hot-reload, rollback)
                                            │                                       │
                                            ▼                                       ▼
                                   W5 Decision-log pipeline ──► [C2 audit]   W6 Cross-engine conformance suite
                                            │                                       ▲
                                            ▼                                       │
                                   W7 Replay capture (nd builtins) ──► [C4]   (consumes B2/B3 builds)
```

Critical path: **W1 → W3 → W4 → W6**. W2, W5, W7 parallelize off W1/W3.

---

## 2. Parallelizable workstreams

| WS | Title | Deps | Can start | Parallel with |
|---|---|---|---|---|
| W1 | Rego conventions + metadata schema (R1–R6) | A1 catalog stub, B4 taxonomy stub | immediately (with stubs) | — |
| W2 | Regal custom ruleset + authoring CI (R7) | W1 | after W1 schema frozen | W3 |
| W3 | BundleService over OCP: build/sign/publish/verify (R12–R20, R24) | OCP eval, D4 keys | OCP spike done | W2 |
| W4 | Distribution, atomic activation, staged rollout, rollback (R21–R23) | W3 | after W3 publish path | W5 |
| W5 | Decision-log schema + sink + redaction (R25–R28) | W1 entrypoint contract, C2 sink stub | after W1 | W4, W6 |
| W6 | Cross-engine conformance suite (R29–R31) | W1, B2/B3 minimal | after B2/B3 have stubs | W5 |
| W7 | Deterministic replay capture (R26) | W5 | after W5 | — |

Stubs/contract-first lets W1 begin before A1/B4/C2 are complete; freeze the **interfaces**
(catalog lookup, taxonomy enum, log sink) on day 1 so downstream teams parallelize.

---

## 3. Critical path & milestones

- **M1 — Conventions frozen (W1):** metadata schema + entrypoint `decision` contract published;
  Regal rules drafted. Unblocks every Rego author across B2/B3/E3.
- **M2 — Signed bundle round-trip (W3):** build → cosign sign → publish OCI → verify → activate,
  with provenance attestation. Demonstrated on `SC-IMG-001`. (DT-10.)
- **M3 — Hot-reload + rollback (W4):** atomic activation, `bundle.activated` events, instant
  rollback to prior digest. (DT-06.)
- **M4 — Decision logs flowing to C2 (W5):** with redaction + correlation_id + nd capture.
- **M5 — Conformance green (W6):** REST = Wasm = Gatekeeper-OPA = Conftest on the golden corpus.
  **This milestone is the gate** for claiming "shared cross-product semantics."
- **M6 — Regression testing wired (W3/OCP):** candidate bundles diffed against historical
  decision logs; feeds A2 promotion + E1 differential sim.

---

## 4. Test strategy

1. **Unit (per package):** `opa test --coverage`, floor 80% of decision branches (R31).
2. **Metadata/lint:** Regal ruleset enforces R1–R6; CI fails on control-id mismatch / unknown control.
3. **Bundle integrity:** build is reproducible; manifest roots/revision asserted; signature +
   attestation verified in a clean verifier (no build creds).
4. **Verification fail-closed (F1):** inject a tampered bundle → assert last-good retained +
   `bundle.verification_failed` emitted, no activation.
5. **Cold-start (F3):** start agent with no bundle → assert deny for runtime class, advisory fail-open.
6. **Cross-engine conformance (R30):** the golden corpus run on all four engines; **any disagreement
   fails the build.** This is the highest-value test in the domain.
7. **Replay determinism (R26):** decision with `time.now_ns` → capture nd cache → replay later →
   identical result.
8. **Regression/differential:** OCP regression test over a decision-log window; assert flip-diffs surfaced.
9. **Redaction (R27):** inject JWT/PII in input → assert masked in exported decision log.
10. **Performance:** p99 eval latency budget for the admission path (B5) — Wasm vs REST measured;
    feeds B2/B5 latency budget.

---

## 5. What blocks what

- **Blocks B2:** Gatekeeper embeds OPA; B2 cannot claim conformance until W6 corpus exists.
- **Blocks B3:** Conftest runs the same Rego; B3 evidence normalization depends on W1 entrypoint shape.
- **Blocks B5:** realtime flow depends on signed-bundle activation (W4) + decision logs (W5).
- **Blocked by A1:** needs control catalog to resolve control_ids (use a stub catalog until A1 lands).
- **Blocked by B4:** action taxonomy enum (use frozen enum stub day 1).
- **Blocked by D4:** signing keys / cosign identity policy.

---

## 6. Risks & mitigations (plan-level)

| Risk | Mitigation |
|---|---|
| OCP immaturity/API churn | BundleService abstraction (R24); spike OCP in week 1; keep a minimal `opa build`+oras fallback path |
| Conformance gaps (Wasm vs REST builtins) | Restrict to a vetted builtin allowlist; conformance test gates every release |
| Decision-log volume/cost | Sampling for advisory class; full capture for runtime/deny; redaction reduces size |
| Cosign keyless dependency on public Rekor/Fulcio | KMS + private Rekor option for air-gapped envs (OQ2) |
| Authors bypass `decision` entrypoint | Regal rule forbids consumers querying non-`decision` paths; conformance corpus only tests `decision` |

---

## 7. Estimated sequencing (relative)

Week 1: W1 + OCP spike. Week 2: W2 + W3 build/sign. Week 3: W3 verify + W4 activation. Week 4:
W5 logs + W7 replay. Week 5: W6 conformance (needs B2/B3 stubs) + M5 gate. Week 6: regression/differential + hardening.
