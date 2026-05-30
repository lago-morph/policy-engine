# E1 — ALT architecture: Reuse OPA decision logs vs. re-execute bundles

**Component:** E1 · **Type:** Alternative architecture (high-value tree) · **Date:** 2026-05-30
**Question framed:** For replay-based modes (M2, M5–M9), should E1 **reuse the OPA decision logs** already emitted at runtime, or **re-execute the bundles** against reconstructed input?

The primary SPEC chooses **re-execute bundles** (OQ-1) and treats decision logs only as a cross-check oracle. This doc develops the decision-log-reuse alternative and weighs it.

---

## 1. The alternative: decision-log reuse

**Architecture.** OPA can emit a **decision log** for every evaluation: the input, the policy path, the result, and (with `nd_builtin_cache`) non-deterministic builtin results. The ALT proposes: for the *previous* bundle outcome, **read the recorded decision log entry** instead of re-running bundle P; only re-execute the *new* bundle N.

```
previous outcome  ◄── decision log lookup (recorded at runtime)
new outcome       ◄── re-execute bundle N on reconstructed input
diff              = classify(recorded_prev, recomputed_new)
```

---

## 2. Trade-off analysis vs. primary (re-execute both)

| Dimension | Re-execute both (primary) | Decision-log reuse (this ALT) |
|---|---|---|
| **Fidelity of "previous" outcome** | Re-derived; must trust input reconstruction | **Exactly what production decided** — ground truth, no reconstruction error for prev |
| **Non-determinism (time, http.send, rand)** | Must pin/stub; risk of divergence | Decision log can record `nd_builtin_cache` ⇒ replays the *actual* non-deterministic values |
| **Cost** | 2× evaluations | ~1× (only new bundle) + a lookup |
| **Coverage** | Any bundle pair, incl. bundles never deployed | **Previous must have been the deployed bundle that produced the log** — cannot diff two *hypothetical* bundles |
| **Engine scope** | Works for OPA, Gatekeeper, Kyverno, Conftest uniformly | **OPA-specific**; Gatekeeper/Kyverno/Conftest don't emit the same decision-log structure |
| **Audit-schema dependency** | Needs C2 reconstructed input | Needs decision logs *and* still needs C2 input for the *new* bundle |
| **Detecting reconstruction bugs** | Reconstruction errors hide in both sides equally | **Diffing recorded-prev vs re-executed-prev reveals reconstruction bugs** (powerful oracle) |
| **Differential matrix correctness** | Both sides on identical reconstructed input ⇒ apples-to-apples | prev=real-input, new=reconstructed-input ⇒ **apples-to-oranges**: a diff could be caused by input reconstruction drift, not the policy change |

---

## 3. The decisive flaw in pure reuse

**Apples-to-oranges inputs.** If `previous` comes from a decision-log entry (the *real* runtime input) and `new` comes from re-execution on the *reconstructed* input, any difference between the real input and the reconstructed input shows up as a phantom newly-blocked/newly-allowed result that is actually an artifact of imperfect reconstruction — **not** a policy change. The §17.4 matrix would be polluted. For a *differential* simulation whose entire value is attributing a behavior change to the *policy* change, this is disqualifying.

Pure reuse also **cannot diff two hypothetical bundles** (e.g., v13 vs v14 where neither was ever deployed), which DT-49/HL-17-style "before promotion" analysis frequently needs.

---

## 4. Where reuse genuinely wins (and how the SPEC absorbs it)

- **Ground-truth oracle for input reconstruction.** Comparing the decision log's recorded result to a re-execution of the *same* (previous) bundle on the *reconstructed* input is the single best test that C2's replay schema is faithful. Divergence ⇒ the audit schema is missing a field the policy used ⇒ downgrade to `replay_completeness=partial/insufficient` (exactly §17.3's intent).
- **Non-deterministic builtins.** Where a policy uses `time.now_ns`, `http.send`, or `rand`, decision-log `nd_builtin_cache` records the actual values, enabling faithful replay of otherwise-irreproducible decisions.

The SPEC therefore adopts re-execute-both as the **decision substrate**, and uses decision logs as a **validation oracle and non-deterministic-builtin source** — capturing reuse's fidelity benefits without its apples-to-oranges and engine-scope penalties.

---

## 5. Verdict

**Re-execute both bundles** chosen (SPEC OQ-1). Decision-log reuse rejected as the primary substrate because (a) it pollutes the differential matrix with input-reconstruction artifacts, (b) it is OPA-only, and (c) it cannot compare two never-deployed bundles. Reuse is retained in a *supporting* role: (1) as the oracle that validates C2 input reconstruction and drives `replay_completeness` downgrades, and (2) as the source of `nd_builtin_cache` values for faithful replay of non-deterministic policies.
