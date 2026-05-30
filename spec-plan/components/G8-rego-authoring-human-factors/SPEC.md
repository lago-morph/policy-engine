# G8 — Rego-Authoring & Human Factors — SPEC

**Component ID:** G8 · **Domain:** G — Operational / NFR · **Spec source:** the NFR gap surfaced by
`cross-cutting/META-ADVERSARIAL-SECOND-OPINION.md` §3 ("No Rego-authoring / human-factors owner — the
#1 field-failure mode of OPA products", Risk #7) and `THESIS-DEVILS-ADVOCATE.md` (the platform-portability
thesis depends on people actually writing good Rego).
**Status:** DRAFT v1 · **Date:** 2026-05-30 · **Author persona:** developer-experience / human-factors owner
(cooperative; writes for the median engineer, not the expert).

> **One-line scope.** G8 owns the *authoring experience* as a first-class non-functional requirement: the
> people who write policy (Marcus the expert; Sam the novice namespace-author), the toolchain that makes them
> succeed (Regal lint, the unit-test framework, the local Conftest loop, the §26.1 Gemara→Rego generation path
> and its hand-off to human-authored templates), the guardrails that stop authoring footguns (the
> fail-closed/typo that mass-denies a fleet — ties to **G3**), the review/approval of policy *code*
> (CODEOWNERS, DT-71, ties to **A2**/**D3**), the onboarding + paved-road templates + PDP-library reuse
> (**E3**), the docs/DX requirements, and — the thing nobody else owns — the **measurement of authoring error
> rate**. The platform thesis ("one Rego decides everywhere") is only true if a *population* of authors can
> produce correct, performant, portable Rego. G8 is the component that makes the median engineer succeed, not
> just the expert.

---

## 0. Normative language & decisions

Requirement IDs `G8-MUST-NNN` / `G8-SHOULD-NNN` / `G8-MAY-NNN`. Unattended decisions are tagged
**[D-G8-n]** and collected in §14. MUST = build-blocking; SHOULD = strongly recommended, deviation
recorded; MAY = optional.

---

## 1. Why this component exists (the problem statement)

Every other component in the corpus assumes a supply of fluent Rego authors who attach correct
`__control_id__` / `__required_claims__` / `# METADATA` blocks (B1 §4), express decisions purely and
deterministically (B1-R10/R26), and carry the join-key metadata the entire lineage graph (E2),
replay (E1/C4), and analytics (C3) depend on. **No component staffs, trains, equips, or measures those
authors.** The result is the canonical field-failure mode of OPA governance products:

- The **compliance team (Priya, Daniel)** owns the control language but cannot write Rego.
- The **platform team (Marcus)** can write Rego but is six people and will not own every compliance control.
- The **namespace author (Sam)** is a competent Kubernetes user who explicitly *does not want to learn Rego*
  (persona 4) — yet §16.3 / D2 hand him a Namespace Authoring view and let him write policy that can deny his
  team's deploys (or, worse, *permit* something that should be denied).

The platform's two load-bearing claims both rest on authoring quality:

1. **Portability** ("one Rego decides everywhere", `MASTER-PLAN §5.1`) is only real if authors write Rego that
   evaluates identically across OPA REST/Wasm/Gatekeeper/Conftest (B1-R30 conformance). A non-portable
   construct authored by a novice silently breaks the thesis at one engine.
2. **Authoritative replay** (the differentiator) is only real if authors surface every input they read
   (B1-R11 `evidence`, R26 nd-builtins). An author who reads `input` fields without listing them in
   `__required_claims__` produces events that replay `insufficient` — the "declared vs verified" sin the
   product exists to prevent.

G8's mandate: **drive the authoring error rate down to a measured, gated target**, and make the *median*
author — not just Marcus — capable of producing a correct, portable, replay-complete, control-traceable
policy on the paved road.

### 1.1 The two author personas G8 designs for

| Author | Who | Scope | Skill | Primary risk G8 must contain |
|---|---|---|---|---|
| **Marcus** (expert) | Platform Security Engineer | Central library, cluster-wide controls | Writes Rego fluently | Footgun at scale: a typo/fail-closed default mass-denies a fleet (§6, ties to G3) |
| **Sam** (novice) | App developer / Namespace Policy Author | NS-scoped policy for his own namespace (§16.3 V5, D2) | Does **not** want to learn Rego | (a) gives up → no policy; (b) authors a **too-permissive** NS policy → security hole (§7) |

> **[D-G8-1]** G8 treats **novice NS-authoring as a security surface, not a convenience feature.** A bad NS
> policy that is *too permissive* is a silent control gap, and the corpus's whole D2 story (scope-isolated
> authoring) implicitly delegates authorization decisions to people who cannot evaluate whether their policy
> is safe. G8 owns the guardrails that make novice NS-authoring safe-by-construction (§7). Rationale: the
> META-ADVERSARIAL pass calls this out (Risk #7) and the ADVERSARIAL-REVIEW of *this* component (its #2
> finding) makes it the second-biggest human-factors risk after expert footguns.

---

## 2. Scope

**In scope (G8 owns):**

1. The **authoring toolchain contract** (§5): Regal lint config + platform custom rules, the unit-test
   framework + coverage floor, the local Conftest/OPA inner loop, editor integration (LSP), pre-commit.
2. The **Gemara→Rego generation strategy** (§4) — the §26.1 full-vs-template decision *from the human-factors
   side*: what the generator covers, where it must hand off to a human, and the **paved-road template library**
   that catches the hand-off (§4.3). (A2 owns the lifecycle/provenance entity `GenerationDecision`; G8 owns the
   *generator quality bar and the template content*.)
3. The **authoring guardrails** (§6) that prevent footguns: the fail-closed/typo mass-deny class, the
   non-portable-construct class, the replay-incompleteness class, the metadata-drift class.
4. The **novice NS-author guardrails** (§7): constrained authoring surface, too-permissive detection, mandatory
   simulation before promote, mandatory review.
5. **Policy-code review & approval** (§8): CODEOWNERS over policy repos, the code-review gate distinct from the
   A2 promotion approval, DT-71 code-owner approval, the four-eyes principle for policy code.
6. **Onboarding & paved-road templates** (§9): the new-author golden path, the template catalog, PDP-library
   reuse (E3) as the starting point, the "first correct policy in < N minutes" target.
7. **Docs & DX requirements** (§10): error-message quality, the remediation-link contract, the
   author-facing documentation surface.
8. **Authoring-error-rate measurement** (§11): the metrics, the instrumentation, the gated targets — *the thing
   no other component owns*.

**Out of scope (owned elsewhere, G8 *consumes/constrains*):**

- The Rego metadata schema and bundle/decision contract — **B1** (G8's lint *enforces* B1-R1..R11).
- The lifecycle state machine, promotion gates, and `GenerationDecision`/`template_todos` entity — **A2**
  (G8 supplies the generator quality bar + template content + the G-AUTHOR-feeding lint/test gates).
- The Conftest CI/pre-commit *mechanics* — **B3** (G8 specifies the *authoring* use of the local loop, not the
  CI gate).
- The simulation/differential engine — **E1** (G8 *requires* sim-before-promote and consumes its output).
- The console Rego Explorer / Namespace Authoring *views* — **E2** (G8 specifies the *constraints* those views
  must enforce for novices, §7; E2 renders them).
- The PDP/Action library catalog content — **E3** (G8 *reuses* it as the paved-road starting point and adds the
  template/scaffold layer on top).
- Fail-closed/availability blast-radius mechanics (staged rollout, last-known-good, break-glass) — **G3** (G8
  owns the *authoring-side* causes of a mass-deny; G3 owns the *runtime-side* containment). This is a shared
  contract, §6.1.
- Scope-isolation enforcement in storage/GUI — **D2** (G8 constrains the *authoring* surface; D2 enforces the
  read/write boundary).
- Approval *runtime* (webhooks/CRDs) — **D3** (G8 *requires* a code-review approval; D3 realizes it).

---

## 3. The authoring lifecycle (the human's journey)

```
 Control (A1)  ──►  GENERATE (§4, A2 :generate)  ──►  full Rego  ─────────────────┐
   intent             §26.1 decision                                              │
                          │                                                       ▼
                          └─►  template + template_todos  ──►  HUMAN AUTHORS  ──► CANDIDATE POLICY
                                (§4.3 paved-road template)        (Marcus or Sam)      │
                                                                                       ▼
        ┌──────────────────────────── INNER LOOP (§5, local, seconds) ──────────────────────────┐
        │  edit ──► Regal lint (§5.1) ──► opa test (§5.2) ──► local Conftest/OPA eval (§5.3)      │
        │           ▲ red squiggles, control-id check, portability lint, replay-completeness     │
        └───────────┴──────────────────────────────────────────────────────────────────────────┘
                          │  (all green)
                          ▼
   PRE-COMMIT (§5.5) ──► PR  ──► CODE REVIEW (§8: CODEOWNERS, DT-71, four-eyes on code)
                                    │  (lint+test+coverage CI green; novice extra gates §7)
                                    ▼
                          A2 G-AUTHOR gate (§3 A2): compiles, metadata matches, tests green,
                          NO template_todos remain, lint clean  ──►  dry-run ──► sim (E1) ──► warn ──► enforce
```

- **G8-MUST-001** Every candidate policy MUST pass the §5 inner loop (lint + test + local eval) **before** it is
  eligible to open a PR, and the same checks MUST run in CI as the merge gate. The inner loop and the CI gate
  MUST run the **identical** Regal ruleset and `opa test` invocation (no "passes locally, fails in CI" drift).
- **G8-MUST-002** The A2 **G-AUTHOR** gate (A2 §3) MUST consume G8's lint+test+coverage results and the
  "no `template_todos` remain" condition as its evidence. G8 does not duplicate A2's gate; it *feeds* it. A
  promotion out of `draft` MUST be refused if any G8 inner-loop check is red.

---

## 4. Gemara → Rego generation (§26.1) from the human-factors angle

A1 expresses a control as Gemara intent; A2 `:generate` produces an implementation and records a
`GenerationDecision{mode: full|template, template_todos[], prefilled_metadata, test_scaffold}` (A2 §2.5).
G8 owns the **quality bar for the generator** and the **content of what it hands a human**.

### 4.1 What the generator covers (the "easy cases")

- **G8-MUST-010** The generator MUST emit `mode: full` **only** when the control is **complete, deterministic,
  and safe** to generate end-to-end — concretely (the gate, normative):
  - the control maps to a **catalogued PDP decision point** (E3) with a known input/replay schema;
  - the rule is expressible as a **pure** decision over `input` + bundle `data` (no `http.send`, no wall-clock,
    no unbounded iteration — B1-R10);
  - every claim/field the rule reads is **declared** (so the generated `__required_claims__` /
    `# METADATA custom.required_claims` is provably complete — B1-R5/R11);
  - a **golden test per intended outcome** (allow/deny/warn/require_approval) can be auto-derived from the
    control's evidence examples.
- **G8-MUST-011** A `full`-generated policy MUST itself pass the entire §5 inner loop (lint, test, conformance
  smoke) before it is offered to the human. The generator MUST NOT emit Rego that its own lint rejects.

### 4.2 Where generation fails (the "hard cases" → human hand-off)

The generator MUST fall back to `mode: template` (never silently emit incomplete `full`) when **any** of:

| Hard case | Why generation can't finish it | What the template carries |
|---|---|---|
| Env-specific **claim mapping** (which JWT group ⇒ which tenant) | Org-specific; not in the control | `template_todo` + a typed slot + the D1 claim-mapping doc link |
| **Non-deterministic** intent (time windows, rate, external lookups) | Breaks pure/replayable (B1-R10/R26) | `template_todo` + the nd-builtin-capture pattern + a warning |
| **Multi-resource / cross-object** logic | Needs author judgment on join keys | scaffold + `template_todo` + an example from E3 |
| **Ambiguous severity / action** | Action taxonomy choice (B4) is a human call | `template_todo` forcing an explicit action |
| Control has **no catalogued decision point** (E3 miss) | No hook/replay schema to target | `template_todo` + "file an E3 library gap" link |

- **G8-MUST-012** The generator MUST NOT downgrade a hard case to a permissive default. A template with an
  unfilled `template_todo` MUST default the decision to **fail-closed for runtime classes** (deny / require
  review), never allow, so an unfinished template cannot ship a hole. (Ties to §6.1 and G3.)
- **G8-MUST-013** Every `template_todo` MUST block promotion until resolved (A2 G-AUTHOR; A2 AC-5 / DT-09).
  G8's responsibility is that each `template_todo` is **actionable**: it MUST carry (a) a one-line description
  of the decision the human must make, (b) a link to the relevant doc/template/E3 entry, and (c) a typed slot
  the lint can verify is filled (not just a `# TODO` comment a human can delete).

> **[D-G8-2]** **Generation is an accelerator, not an author.** The corpus's own adversarial pass (this
> component's ADVERSARIAL-REVIEW #3, echoing the THESIS doc) warns that a generator which "covers the easy
> cases and dumps the hard ones on humans with no support" is worse than no generator — it creates a false
> sense of coverage. G8 therefore measures the **template hand-off rate** (§11 metric M4) and treats a high
> hand-off rate on common controls as a *generator defect*, not an author problem: if 70% of controls fall to
> template, the generator and the E3 catalog are under-built, and that is G8's backlog, not the author's burden.

### 4.3 The paved-road template library (the hand-off catch)

The thing that makes the hard-case hand-off survivable is a **curated template library** so a human never starts
from a blank file.

- **G8-MUST-014** G8 MUST maintain a **template catalog** keyed by `(control archetype × PDP/product)` — e.g.
  "image-signing for k8s admission", "RBAC drift for Keycloak", "branch-protection for GitLab" — each template
  built **from the E3 PDP-library entry** for that product (E3 §4 template) and pre-wired with: the package
  path (B1-R6), the `# METADATA` + `__control_id__` block (B1-R1, slot for the control id), the canonical
  `decision` rule skeleton (B1-R8), the input normalization shim usage (B3-R10), a `policy_test.rego` scaffold
  with one fixture per outcome, and `template_todo` markers at every decision the author must make.
- **G8-MUST-015** Each template MUST be **lint-clean and test-green as shipped** (with TODOs unfilled producing
  fail-closed defaults, per G8-MUST-012). A template that doesn't pass its own §5 loop is a G8 defect.
- **G8-SHOULD-010** The template catalog SHOULD cover **≥ 80% of the controls observed in the POC corpus**
  (§22 sizing: 25–100 controls) so the median new control starts from a paved-road template, not a blank file.
- **G8-MUST-016** Templates MUST be **versioned and signed** like E3 libraries (E3 §7, §23) and pinned by
  authors, so "the template I generated from" is reproducible provenance (feeds A2 `generated_from=template`).

---

## 5. The authoring toolchain (the inner loop)

The inner loop must be **fast (seconds), local, and identical to CI**. This is the single highest-leverage
DX investment: an author who gets a correct red/green signal in their editor in seconds writes correct policy;
one who must push to a dev cluster and wait writes buggy policy (Marcus persona "before": deploy-and-hope).

### 5.1 Regal lint (the correctness front line)

Per `market research §2`: Regal (the OPA linter) is now open-sourced into CNCF. The platform adopts it
(B1-D-B1-02, B1-R7) and G8 owns the **platform ruleset**.

- **G8-MUST-020** G8 MUST ship a **platform Regal configuration** that, in addition to Regal's upstream rules,
  enforces the platform's authoring contract via **custom Regal rules**:
  - **R-LINT-META** — every governance package declares `__control_id__` **and** a `# METADATA custom.control_id`
    with the **same** value, both non-empty (B1-R1).
  - **R-LINT-CTRLID** — `custom.control_id` resolves against the Gemara catalog snapshot (B1-R2); unknown ⇒ error.
  - **R-LINT-PKG** — package path matches `governance.<product>.<capability>` (B1-R6).
  - **R-LINT-DECISION** — package exposes a canonical `decision` rule conforming to the B1 §5 shape (B1-R8);
    `allowed` is derived, never hand-authored (B1-R9).
  - **R-LINT-PURE** — no `http.send` at decision time, no unbounded iteration (B1-R10); flags wall-clock reads
    not surfaced for nd-capture (B1-R26) — the **portability + replay** lint.
  - **R-LINT-CLAIMS** — every `input` field the rule reads is declared in `__required_claims__` /
    `custom.required_claims` (B1-R5/R11) — the **replay-completeness** lint (DT-25).
  - **R-LINT-PORT** — flags constructs that are **non-portable across the four engines** (B1-R30) — e.g.
    Gatekeeper-incompatible builtins, reliance on Conftest's bare-`input` instead of the normalization shim
    (B3-R10). This is the lint that protects the portability thesis at author-time.
  - **R-LINT-FAILOPEN** — flags a `decision` whose default for a runtime-class control is `allow` (the
    footgun: a policy that defaults-open silently never denies — §6.2).
- **G8-MUST-021** Regal lint MUST run (a) in-editor via the Regal **language server (LSP)** with inline
  diagnostics, (b) in the pre-commit hook, and (c) in CI as a hard gate — the **same ruleset** in all three
  (G8-MUST-001). A package with any lint **error** MUST NOT be bundleable (B1-R7) and MUST fail G-AUTHOR.
- **G8-SHOULD-020** Lint **warnings** (style, complexity, suggested simplifications) SHOULD surface but not
  block, with an explicit per-rule escalation path so a class of warning can be promoted to error per-org.

### 5.2 Unit-test framework + coverage floor

- **G8-MUST-022** Every governance package MUST ship co-located `*_test.rego` unit tests (B1 §6.1 layout,
  B1-R31), runnable with `opa test --coverage`. G8 owns the **test ergonomics**: the test scaffold the
  generator/template emits (one fixture per intended outcome — A2 §2.5 `test_scaffold`), helper assertions for
  the canonical `decision` shape, and golden-fixture management.
- **G8-MUST-023** CI MUST enforce a **branch-coverage floor** (default **80%** of decision branches, B1-R31).
  Below floor ⇒ G-AUTHOR fails. The floor is configurable up, never silently down.
- **G8-MUST-024** Where E1/§17.5 produces **audit-derived fixtures** (real production events captured as test
  cases), G8 MUST provide the workflow to pin them to the package as regression tests (A2 `tests` /
  `TestCaseRef`), so a real incident becomes a permanent test (Marcus persona: "regression suites built from
  past incidents" — the thing that "doesn't get done" today).

### 5.3 Local Conftest / OPA eval loop

- **G8-MUST-025** G8 MUST provide a **local evaluation harness** so an author can run the candidate policy
  against a sample input and see the full canonical `decision` object **without a cluster** — using `conftest`
  (B3 local path, B3-R15) for IaC/manifest inputs and `opa eval` for raw inputs. This is the "what would this
  decide?" loop that replaces deploy-and-hope.
- **G8-MUST-026** The local harness MUST use the **same input normalization shim** (B3-R10) the engines use, so
  a policy that passes locally evaluates identically at admission/CI (no envelope-shape surprises). Divergence
  here is a P1 DX bug.
- **G8-SHOULD-021** The harness SHOULD let an author pull **one real (redacted) production event** for their
  control (via E1/C2) as a local fixture — closing the loop between "what happens in prod" and "what I'm
  authoring" without giving the author un-redacted access (D4/G7 redaction applies).

### 5.4 Editor / LSP integration

- **G8-MUST-027** G8 MUST ship a documented **editor setup** (Regal LSP + recommended extensions) for at least
  VS Code, providing: live lint diagnostics (§5.1), go-to-control-definition (jump from `__control_id__` to the
  Gemara catalog entry), hover docs for the canonical `decision` schema, and snippet completion seeded from the
  template catalog (§4.3). Target: a correct package skeleton from a snippet in **one keystroke**.

### 5.5 Pre-commit

- **G8-MUST-028** G8 MUST ship a **pre-commit hook bundle** (reusing B3-R15) running lint + `opa test` +
  local-eval smoke on changed packages, fast (incremental, changed-files-only — B3-R16), degrading gracefully
  offline. Pre-commit is advisory acceleration; the CI gate is authoritative (B3-R17) — but pre-commit is where
  the median author *first* sees red, so its quality is load-bearing for the error rate (§11).

---

## 6. Guardrails — preventing authoring footguns (expert path)

These are the guardrails that stop a competent author from causing a large-blast-radius incident. They split
into four footgun classes. Class 1 (mass-deny) is the headline and is a **shared contract with G3**.

### 6.1 Footgun class 1 — the fail-closed/typo that mass-denies a fleet (shared with G3)

The scenario: an author writes (or generates) a policy whose `decision` denies far more than intended — a typo
in a matcher, an inverted condition, a too-broad selector, or a fail-closed default that fires on a population
nobody simulated — and it is promoted to `enforce` across a fleet, blocking every deploy. (Marcus persona
"before": the legacy-signer canary trips the new policy at 2 a.m.)

**G8 owns the author-time *causes*; G3 owns the runtime *containment*. The contract:**

- **G8-MUST-030 (author-time prevention).** No runtime-class policy may reach `enforce` without **mandatory
  differential simulation** (E1 §17.4) over the EvidenceRequirement window, surfaced to the author, with **zero
  un-triaged "newly blocked" rows** (this is A2's G-DIFF gate, A2 §3 — G8 makes it non-bypassable for *every*
  runtime author including experts, and makes the diff the **primary promotion artifact the human must read**).
  An author MUST explicitly classify each newly-blocked row Intended/FalsePositive before promote.
- **G8-MUST-031 (author-time prevention).** The **R-LINT-FAILOPEN** rule (§5.1) and a companion
  **R-LINT-OVERBROAD** check MUST flag a `decision` rule whose denial predicate has no narrowing condition
  (e.g. denies on `true`, or a selector matching all namespaces) — a likely mass-deny — as a lint **error**
  requiring an explicit, audited `# regal-allow: overbroad-decision <reason>` override that is itself recorded
  in provenance.
- **G8-MUST-032 (G3 contract — runtime containment).** G8 REQUIRES that G3 provide: **staged/canary rollout**
  of any `enforce` promotion (B1-R23), **instant rollback to last-known-good digest** (B1-R23, A2 demotion
  ungated/sub-2-min, A2-MUST-060 / DT-06), and a **deny-rate circuit breaker** that auto-demotes a newly
  promoted policy whose observed deny rate exceeds the simulation forecast by a configured tolerance. G8 owns
  the *requirement*; G3 owns the *mechanism*. (If G3 does not yet exist, this requirement is the contract that
  the G-domain index/G3 must satisfy.)
- **G8-MUST-033.** The **soak forecast vs. actual** comparison (A2 G-SOAK) MUST be shown to the author as part
  of promotion, and a soak whose actual warn/deny volume exceeds the dry-run forecast beyond tolerance MUST
  block warn→enforce until re-simulated. This is the second net that catches a footgun the diff missed.

> **[D-G8-3]** **No runtime policy reaches `enforce` un-simulated, ever, regardless of author seniority.** The
> single most common OPA mass-deny incident is an expert who "knew it was fine" and skipped the dry-run. G8
> removes the skip option for the `enforce` transition. Rationale: the cost of a forced simulation (minutes)
> is trivial against the cost of a fleet-wide deny (an outage); the corpus's whole E1 differential-sim
> investment exists precisely to be this gate, and G8 makes it mandatory rather than optional.

### 6.2 Footgun class 2 — non-portable constructs (protects the portability thesis)

- **G8-MUST-034** The **R-LINT-PORT** rule (§5.1) MUST flag, at author time, every construct that would make a
  package evaluate differently across OPA REST/Wasm, Gatekeeper-embedded OPA, and Conftest (B1-R30). The
  **cross-engine conformance suite (B1-R30)** MUST run in CI for every changed package as a hard gate — a
  package that lint-passes but conformance-fails (REST ≠ Wasm ≠ Gatekeeper ≠ Conftest) MUST NOT merge. This is
  the only author-time defense of the "one Rego decides everywhere" claim; without it, portability is a runtime
  surprise. (See ADVERSARIAL-REVIEW #1.)

### 6.3 Footgun class 3 — replay-incompleteness (protects the differentiator)

- **G8-MUST-035** The **R-LINT-CLAIMS** rule MUST flag any `input` field read by the decision that is not
  declared in `__required_claims__` (B1-R5/R11), and any wall-clock/random read not surfaced for nd-capture
  (B1-R26). A package whose declared inputs don't match its actually-read inputs produces events that replay
  `insufficient` — G8 catches this at author time, not at audit time. **G8-SHOULD-030** A "read-vs-declared"
  static analysis SHOULD be run and any drift reported as a lint warning escalatable to error per-org.

### 6.4 Footgun class 4 — metadata drift / control-id rot

- **G8-MUST-036** The dual-metadata problem (B1 §4.1 var **and** §4.2 `# METADATA`, which the B1 adversarial
  review flags as a drift risk, AR-11) MUST be resolved on the **single-source-of-truth** principle: authors
  edit the `# METADATA` block; the `__control_id__` Go var is **generated** from it by a pre-commit/CI step, and
  R-LINT-META verifies they agree. Authors never hand-maintain two copies. **G8-MUST-037** A package whose
  `control_id` references a control that A1 has **deprecated/retired** MUST surface a lint warning (B1-R2 / F7)
  so dead policies are visible to their authors, feeding the A2 reconciler's `zombie_enforcement` finding.

---

## 7. Novice NS-author guardrails (Sam — the security surface)

Sam authors namespace-scoped policy in the E2 Namespace Authoring view (§16.3 V5), hard-locked to his claimed
namespaces by D2. He does not want to learn Rego. The risk is **not** that he writes a policy that denies too
much (that hurts only his team and is self-correcting); the risk is that he writes one that is **too
permissive** — a NS policy that *allows* something the org intends to deny, creating a silent control gap that
the central team cannot see. G8 makes novice NS-authoring **safe-by-construction.**

- **G8-MUST-040 (constrained surface).** The Namespace Authoring view MUST NOT present Sam a blank Rego editor
  as the primary path. The primary path MUST be **template-and-fill** (§4.3): Sam picks a template from the
  catalog (filtered to NS-legal archetypes), fills typed `template_todo` slots through guided form fields, and
  the Rego is rendered from the filled template. Raw Rego editing MAY be available as an "advanced" affordance
  but is **lint-gated identically** (§5.1) and **review-gated** (§8).
- **G8-MUST-041 (can only tighten, never loosen).** A namespace-scoped policy MUST be **additive-restrictive
  only**: it MAY add a denial that the central policy does not, but it MUST NOT *override* or *weaken* a
  central control. Concretely, the composition rule (enforced by lint **R-LINT-NS-RESTRICT** + the engine's
  bundle composition, B1-R12 explicit roots) is **most-restrictive-wins**: a NS policy's `allow` cannot
  un-deny what a cluster policy denies. **G8-MUST-042** Lint MUST reject a NS policy that attempts to define a
  `decision` for a control owned by the central library (a NS author cannot re-author `SC-IMG-001`); NS authors
  define policy only in the **NS-delegated control namespace** A1/D2 grants them.
- **G8-MUST-043 (too-permissive detection).** Before a NS policy can be promoted, G8 MUST run a
  **permissiveness check**: differential simulation (E1) over the namespace's recent events showing what the
  new NS policy would **newly allow** that is *not* otherwise covered, flagged for review. A NS policy whose net
  effect is to *permit* (rather than restrict) MUST be flagged `permissive_ns_policy` and **routed to central
  review** (§8) — a novice cannot self-approve a policy that opens a gap.
- **G8-MUST-044 (mandatory sim + mandatory review).** Every NS-author promotion (even to dry-run) MUST require
  (a) a passing E1 simulation the author has seen, and (b) a **CODEOWNERS review by a central reviewer** (§8) —
  NS authors do **not** have self-promote authority to `enforce`. This is stricter than the expert path on
  purpose. **[D-G8-4]** rationale: D2 gives Sam *write* scope; G8 withholds *unreviewed enforce* authority,
  because the corpus's threat model (META-ADVERSARIAL Risk #7, this component's ADVERSARIAL-REVIEW #2) is a
  novice silently shipping a permissive control, and unreviewed novice enforcement is exactly that hole.
- **G8-SHOULD-040** The NS authoring experience SHOULD give Sam **plain-language outcomes** ("this will block
  deploys that use an unsigned image in your namespace; last 30 days that's 3 deploys") rather than Rego diffs,
  so a non-Rego author can reason about effect. The Rego stays generated/hidden unless he opts into advanced mode.

---

## 8. Policy-code review & approval (CODEOWNERS, DT-71, four-eyes on code)

There are **two distinct human gates** and G8 insists they are not conflated:

1. **Code review** (G8 owns the requirement) — is this *Rego* correct, portable, replay-complete, and not a
   footgun? Reviewed by someone who can read Rego.
2. **Promotion approval** (A2 G-APPROVE / D3 §17B owns) — should this control *enforce* in production? A
   separation-of-duties governance approval (A2-MUST-040), possibly by a non-author who cannot read Rego.

- **G8-MUST-050** Policy repositories MUST use **CODEOWNERS** (or equivalent) so every governance package has a
  designated code-owner reviewer, and a PR touching a package MUST require that owner's approval before merge
  (DT-71 code-owner approval pattern; E3 GitLab library §5). Central-library packages are owned by the platform
  team (Marcus); NS-delegated packages are owned by **central review** for promotion despite NS write access
  (§7, G8-MUST-044).
- **G8-MUST-051** The code-review gate MUST be **distinct from and precede** the A2 promotion approval. A policy
  may pass code review (the Rego is correct) and still be held at the promotion gate (the org isn't ready to
  enforce), and vice-versa. Conflating them lets a governance approver who can't read Rego rubber-stamp a
  footgun, or lets a Rego reviewer authorize production enforcement they have no authority over.
- **G8-MUST-052 (four-eyes on code).** No author may merge their own governance package without a second
  party's code-owner approval (four-eyes), and — because the *code* reviewer and the *promotion* approver are
  different gates — the author MAY be neither, but MUST NOT be **both** the sole code reviewer and the sole
  promotion approver (this composes A2's separation-of-duties for `enforce`, A2-MUST-040, with code-review SoD).
- **G8-MUST-053** The code-review checklist MUST be **machine-assisted**: the PR MUST surface the lint result,
  the coverage delta, the conformance-suite result, the differential-simulation summary (newly allowed / newly
  blocked counts), and any `# regal-allow` overrides (§6.1) — so a reviewer reviews *evidence*, not just diff
  text. **G8-SHOULD-050** A reviewer SHOULD be required to explicitly acknowledge any overbroad-decision or
  fail-open override (§6.1) before approving.

---

## 9. Onboarding & paved-road templates

- **G8-MUST-060** G8 MUST define a **golden onboarding path** for a new author that takes them from zero to a
  merged, correct, simulated first policy, measured: target **first correct policy in < 60 minutes** for an
  expert (Marcus) and **< 30 minutes via template-and-fill** for a novice (Sam). The path MUST be: pick a
  template (§4.3) → fill TODOs guided by docs (§10) → inner loop green (§5) → open PR → review (§8) → dry-run +
  sim (E1). No step requires tribal knowledge or a Slack ping.
- **G8-MUST-061 (reuse-first).** The onboarding path MUST start from **E3 PDP-library reuse** (E3 §1, §4
  template): the author never writes a hook/replay schema from scratch; they compose from the catalogued
  decision points for their product. G8's template layer (§4.3) sits directly on top of E3, turning a library
  entry into a fillable policy skeleton. This is the concrete realization of the platform's "reusable building
  blocks" claim for *authoring*, not just for the catalog.
- **G8-SHOULD-060** G8 SHOULD ship a small **example-driven tutorial corpus** ("author SC-IMG-001 from scratch",
  "add a NS rate-limit policy as Sam") that doubles as integration tests for the toolchain (if the tutorial
  breaks, the DX broke).
- **G8-SHOULD-061** A **policy cookbook** of vetted patterns (claim mapping, nd-builtin capture, the canonical
  `decision` aggregation of `deny[msg]` rules, the input shim usage) SHOULD exist so authors copy a *correct*
  pattern rather than inventing a footgun. The cookbook patterns MUST themselves be lint-clean and tested.

---

## 10. Docs & DX requirements

- **G8-MUST-070 (error-message contract).** Every author-facing failure — a lint error, a failed test, a
  conformance mismatch, a blocked promotion, a denied-at-CI Conftest result (B3) — MUST emit an **actionable**
  message containing: the `control_id` (when known), a one-line human cause, a **remediation link** to the
  relevant doc/cookbook/template, and (for a CI/admission denial) a reproducible local fixture (B3 / E1 §17.5).
  The Sam persona's whole "before" pain (`denied by gatekeeper: violates K8sPSPPrivilegedContainer` with no
  path forward) is a **G8-MUST-070 violation** and the explicit thing this requirement abolishes.
- **G8-MUST-071** The author-facing documentation surface MUST cover, versioned alongside the platform: the
  authoring contract (metadata, canonical `decision`, purity/replay rules — B1), the template catalog (§4.3),
  the lint ruleset with per-rule rationale and remediation, the inner-loop setup (§5), the review process (§8),
  and the novice NS-authoring guide (§7). Docs are a **build deliverable**, not an afterthought.
- **G8-SHOULD-070** Lint rules SHOULD link from the in-editor diagnostic directly to the rule's doc page
  (`R-LINT-PORT` diagnostic → the portability doc), so remediation is one click from the error.

---

## 11. Authoring-error-rate measurement (the thing nobody else owns)

The META-ADVERSARIAL pass's deepest charge is that **no one measures the authoring error rate**. A platform
that can't measure whether its authors succeed cannot claim the median engineer succeeds. G8 owns the metrics.

- **G8-MUST-080** G8 MUST instrument and report the following authoring-quality metrics, per author cohort
  (expert/novice) and per product, on the governance console (E2) and in reporting (C5):

  | ID | Metric | Definition | Why it matters | Gated target (default) |
  |---|---|---|---|---|
  | **M1** | Inner-loop catch rate | % of defects caught at lint/test/local-eval vs. caught later (CI, review, sim, prod) | High = the cheap loop is working | ≥ 90% caught before PR |
  | **M2** | Promotion-blocked rate | % of promotion attempts blocked by a gate (lint/diff/coverage/TODO) | Gates are firing, not bypassed | track; sudden drop ⇒ bypass alarm |
  | **M3** | Mass-deny near-miss rate | # of `enforce` candidates whose differential sim showed > X% newly-blocked, caught pre-enforce | Proves §6.1 is catching footguns | 0 reach enforce un-triaged |
  | **M4** | Template hand-off rate | % of controls falling to `template` vs `full` generation | High ⇒ generator/E3 under-built (a G8 defect, §4.2) | track; reduce over time |
  | **M5** | Replay-completeness at author-time | % of merged packages passing R-LINT-CLAIMS with zero read-vs-declared drift | Protects the replay differentiator | ≥ 95% |
  | **M6** | Conformance pass rate | % of merged packages passing B1-R30 cross-engine suite first try | Protects the portability thesis | ≥ 95% |
  | **M7** | Permissive-NS-policy rate | # of NS policies flagged `permissive_ns_policy` and routed to central review | Surfaces the novice security risk (§7) | track; all reviewed |
  | **M8** | Time-to-first-correct-policy | onboarding metric (§9) | DX health | < 60m expert / < 30m novice |
  | **M9** | Author escape rate | # of policy defects that reached **enforce** and were later rolled back / caused an incident | The ground-truth error rate | trend to zero; reviewed each retro |

- **G8-MUST-081** M9 (author escape rate) MUST be the **gating acceptance criterion** for the authoring-layer
  claim: the platform may not assert "the median engineer succeeds" until M9 is measured and below an agreed
  threshold over a real corpus. (Mirrors the META-ADVERSARIAL demand that the differentiator claims be gated by
  measurement, not asserted.)
- **G8-SHOULD-080** A sudden drop in M2 (promotion-blocked rate) or a spike in M9 SHOULD raise an alert — a
  bypass or a toolchain regression is eroding the guardrails.

---

## 12. Failure modes & handling

| # | Failure | Required behavior |
|---|---|---|
| F1 | Generator emits incomplete `full` (silent hole) | Forbidden (G8-MUST-010/012): hard cases MUST fall to `template` with fail-closed TODOs; a `full` that fails its own §5 loop is a build defect |
| F2 | Author deletes a `# TODO` to bypass a `template_todo` | Mitigated by typed slots verified by lint (G8-MUST-013), not free-text comments; lint fails until the slot is filled |
| F3 | Expert skips dry-run on an `enforce` promotion | Impossible: G8-MUST-030/D-G8-3 makes differential sim non-bypassable for `enforce`; G3 circuit breaker (G8-MUST-032) is the backstop |
| F4 | Novice ships a too-permissive NS policy | Caught by permissiveness check (G8-MUST-043) + mandatory central review (G8-MUST-044); cannot self-promote to enforce |
| F5 | Lint passes but engines disagree (portability break) | B1-R30 conformance suite is a hard CI gate (G8-MUST-034); merge blocked |
| F6 | Package reads inputs it doesn't declare (replay gap) | R-LINT-CLAIMS error (G8-MUST-035); merge blocked |
| F7 | Dual metadata drifts | Single-source-of-truth: edit METADATA, generate the var, lint verifies (G8-MUST-036) |
| F8 | Local loop ≠ CI ("works on my machine") | Identical ruleset/invocation contract (G8-MUST-001/021/026); divergence is a P1 DX bug |
| F9 | Code reviewer and promotion approver are the same person rubber-stamping | Distinct gates + four-eyes (G8-MUST-051/052) compose with A2 SoD |
| F10 | Author can't read the error, pings Slack, stalls (the §1 field-failure) | G8-MUST-070 actionable-message contract abolishes the dead-end error |
| F11 | High template hand-off rate blamed on authors | Treated as generator/E3 defect (D-G8-2, M4); G8 backlog, not author burden |

---

## 13. Dependencies

| Depends on / contracts with | For | Direction |
|---|---|---|
| **B1 (OPA/Rego)** | the authoring contract G8's lint enforces (metadata, canonical `decision`, purity/replay, R30 conformance) | B1 → G8 (G8 enforces) |
| **A2 (lifecycle)** | G-AUTHOR/G-DIFF/G-SOAK/G-APPROVE gates G8 feeds; `GenerationDecision`/`template_todos` entity; demotion path for the circuit breaker | G8 ↔ A2 |
| **B3 (Conftest)** | the local pre-commit/CI loop mechanics G8 wraps for authoring; input normalization shim | B3 → G8 |
| **E1 (Simulation)** | mandatory differential sim before enforce (footgun + NS permissiveness checks); audit-derived fixtures | E1 → G8 |
| **E2 (Console)** | Rego Explorer + Namespace Authoring views render the guardrails G8 specifies; metrics dashboards | G8 → E2 (G8 constrains) |
| **E3 (PDP libraries)** | the reuse substrate the template catalog (§4.3) is built from | E3 → G8 |
| **D2 (scoped RBAC/storage)** | the NS write-scope boundary G8's NS guardrails compose with | D2 → G8 |
| **D3 / §17B (approvals)** | the promotion-approval runtime distinct from code review (§8) | D3 → G8 |
| **G3 (availability/DR)** | the runtime mass-deny containment (staged rollout, rollback, circuit breaker) G8 requires (§6.1) | G3 ↔ G8 (shared contract) |
| **C5 (reporting) / E2** | surfaces the §11 authoring-quality metrics | G8 → C5/E2 |
| **A1 (Gemara)** | control catalog the generator targets and lint resolves control_ids against | A1 → G8 |
| **D1 (Keycloak/JWT)** | claim-mapping docs the template TODOs link to (§4.2) | D1 → G8 |

**Consumed by:** every author (Marcus, Sam); the A2 lifecycle (gate evidence); the platform thesis (portability
+ replay quality is produced here).

---

## 14. Decisions log

- **D-G8-1** Novice NS-authoring is a **security surface**, not a convenience — too-permissive NS policy is the
  threat; G8 owns safe-by-construction NS guardrails (§7).
- **D-G8-2** Generation is an **accelerator, not an author**; high template hand-off rate is a *generator/E3
  defect* (G8 backlog), measured by M4 — not an author burden (§4.2).
- **D-G8-3** **No runtime policy reaches `enforce` un-simulated**, regardless of author seniority; G8 removes the
  skip option for the `enforce` transition (§6.1).
- **D-G8-4** NS authors get **write scope (D2) but not unreviewed-enforce authority**; mandatory central review +
  mandatory sim for every NS promotion (§7).
- **D-G8-5** **Code review and promotion approval are distinct gates** that MUST NOT be conflated; four-eyes on
  code composes with A2 separation-of-duties on enforce (§8).
- **D-G8-6** The **inner loop and CI run the identical ruleset/invocation** — "works on my machine" is a P1 bug,
  not a quirk (§5, F8).
- **D-G8-7** **Authoring error rate (M9) is a gated acceptance criterion** for the "median engineer succeeds"
  claim; the platform may not assert it until M9 is measured below threshold (§11).

---

## 15. Traceability

- **NFR source:** META-ADVERSARIAL §3 "Human factors — who actually writes the Rego" / Risk #7;
  THESIS-DEVILS-ADVOCATE (portability ⇐ author quality).
- **Spec / components:** B1 §4–§5, §10 (metadata, canonical decision, R30 conformance); A2 §2.5/§3 (§26.1
  generation, G-AUTHOR/G-DIFF gates); B3 §7 (local/pre-commit loop); E1 §17.4/§17.5 (differential sim,
  audit-derived fixtures); E2 §16.3 (Rego Explorer, Namespace Authoring views); E3 §17D (PDP-library reuse);
  D2 §17A.5 (NS write scope); D3 §17B (approval runtime); G3 (mass-deny containment); C5/§17E (metrics).
- **Personas:** Marcus (expert author — footgun prevention), Sam (novice NS-author — safe-by-construction),
  Priya/Daniel (can't write Rego — generation + template + plain-language outcomes serve them indirectly).
- **Scenarios:** DT-09 (generation full-vs-template), DT-05/DT-06 (promotion gates / sub-2-min rollback),
  DT-25 (replay completeness), DT-40 (test coverage), DT-43 (NS-scoped authoring), DT-71 (code-owner approval),
  HL-02 (image-signing rollout end-to-end), HL-04 (developer onboarding through gates), HL-17 (authoring a new
  policy version in the Rego Explorer).
