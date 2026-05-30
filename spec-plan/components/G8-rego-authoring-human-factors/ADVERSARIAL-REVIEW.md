# G8 — Rego-Authoring & Human Factors — ADVERSARIAL REVIEW

**Component:** G8 · **Domain:** G — Operational / NFR · **Status:** DRAFT v1 · **Date:** 2026-05-30
**Reviewer persona:** red-team — a CTO who has watched three OPA governance rollouts stall in the field, plus a
security lead who assumes the novice author is the attacker. Attacks the G8 SPEC and, through it, the platform
thesis it props up.

> **One-line verdict.** G8 correctly *names* the #1 OPA field-failure mode and writes a thorough spec — but the
> spec is **mostly a pile of lint rules and process gates whose load-bearing parts are subcontracted to
> components that may not deliver** (E1 simulation, B1-R30 conformance, G3 circuit breaker, E3 catalog). G8's
> guarantees are only as real as those four dependencies, and the spec writes them as MUSTs as if it controlled
> them. Worse, the deepest charge from the META-ADVERSARIAL pass — *most engineers can't write good Rego, and a
> generator that covers the easy cases dumps the hard ones on humans* — is **acknowledged but not solved**: G8's
> answer is "templates + TODOs + measure the hand-off rate," which is a plan to *observe* the failure, not
> prevent it. The biggest unanswered question is whether the platform thesis ("one Rego decides everywhere")
> can survive a population of median authors *at all*, or whether the honest conclusion is that humans should
> not be authoring Rego directly for this product.

---

## 1. The attack on the thesis G8 is supposed to defend

The whole platform rests on a population of authors producing **correct, performant, portable, replay-complete,
control-traceable** Rego. G8 is the component that's supposed to make that population exist. The adversarial
question is not "are G8's mechanisms reasonable?" (they are) but **"do they actually produce that population, or
do they produce the *appearance* of one?"** Five attacks:

### AR-1 (CRITICAL) — Portability is *asserted* by a lint, but *proven* only by a conformance suite that doesn't exist yet
The "one Rego decides everywhere" claim is the load-bearing differentiator (THESIS / META-ADVERSARIAL G-1). G8's
defense is **R-LINT-PORT** (§6.2) at author-time and the **B1-R30 cross-engine conformance suite** as the CI
gate. But:
- **R-LINT-PORT is a heuristic blacklist of "known non-portable constructs."** A linter cannot prove semantic
  equivalence across OPA-REST / OPA-Wasm / Gatekeeper-embedded-OPA / Conftest; it can only flag patterns
  someone already knew were unsafe. The first *novel* non-portable construct sails through. The lint is a
  speed-bump masquerading as a guarantee.
- **B1-R30 is unbuilt.** The META-ADVERSARIAL pass (G-1) explicitly names B1-R30 as the proof of portability
  and notes it is "listed as highest-value test still to be built." G8-MUST-034 makes it a hard CI gate — but a
  hard gate on a suite that doesn't exist is a hard gate on nothing. And the corpus *also* selects **Kyverno as
  the effector for complex cases** (B-domain), which does **not** execute OPA Rego at all. So even a perfect
  conformance suite proves portability across the *three* engines that run Rego and is silent on the *fourth*
  enforcement path the product depends on. **G8 cannot defend a portability thesis the architecture itself
  partially abandons.** G8 should state plainly: *portability is guaranteed only across the Rego-executing
  engines, and Kyverno controls are re-authored, not portable* — and stop the lint from implying otherwise.

### AR-2 (HIGH) — Novice NS-authoring "safe by construction" leans on a most-restrictive-wins property that may not hold over the real backend
§7's entire safety argument is **G8-MUST-041 "can only tighten, never loosen"** — a NS policy can add denials
but can't un-deny a central control, enforced by lint **R-LINT-NS-RESTRICT** + bundle composition (B1-R12
explicit roots). Two problems:
- **Composition semantics are asserted, not proven.** "Most-restrictive-wins" across independently-authored
  bundles is a *property of how the decisions are combined at the PDP*, not something a lint over one file can
  guarantee. If two NS policies and a central policy all expose a `decision` for overlapping inputs, the actual
  combined outcome depends on the aggregation logic at the enforcement point — which G8 doesn't own (B-layer
  does) and the SPEC never specifies. A NS author who defines a `decision` that *returns allow with high
  specificity* could, under a "most-specific-wins" or "first-match" aggregation, override a broad central deny.
  G8 *asserts* most-restrictive-wins but **points at no component that guarantees the combinator is
  most-restrictive-wins.** This is the same "it's a lint, not a control" flaw the META-ADVERSARIAL pass nailed
  D2 on (Risk #9, G-3) — and G8 has imported it.
- **The permissiveness check (G8-MUST-043) only sees what the namespace's *recent events* exercised.** A NS
  policy that opens a gap for an input pattern that simply *hasn't occurred in the last 30 days* shows "newly
  allows: 0" and sails through review as benign. The check measures historical effect, not latent
  permissiveness. A novice can ship a dormant hole that the differential sim, by construction, cannot see.

### AR-3 (HIGH) — The generator is the headline solution and the spec admits it mostly punts
The META-ADVERSARIAL charge is that the §26.1 generator "covers the easy cases and dumps the hard ones on humans
with no support." G8's rebuttal (D-G8-2, §4.3) is: templates catch the hand-off, typed TODOs are actionable, and
M4 measures the hand-off rate so a high rate is treated as a generator defect. **This is honest but it is not a
solution — it is a measurement plan for an unsolved problem.** Attack it:
- If the hard cases are the *common* cases (env-specific claim mapping is in **every** real control; non-trivial
  controls are usually multi-resource), then "full" generation covers toy controls and the median real control
  is template-with-TODOs — i.e., the median author is **still hand-writing the hard part of the Rego**, which is
  exactly the population the META-ADVERSARIAL pass says doesn't exist. Templates reduce the blank-page problem;
  they do not reduce the *judgment* problem (which action? which join key? is this deterministic?). A typed slot
  the author must fill correctly is still a place the median author fills *incorrectly*.
- D-G8-2 reframes a high hand-off rate as a "generator/E3 defect, G8's backlog." That's a way to take the blame
  off the author, but it doesn't help the author who is blocked **today** on a control the generator can't do.
  "We'll improve the generator later" is not authoring support; it's a backlog item wearing a guardrail's badge.

### AR-4 (HIGH) — Every hard guarantee is subcontracted; G8 controls almost none of its own MUSTs
Count what G8 actually *owns* vs. *requires from others*: the lint ruleset and templates G8 owns; but the
**mandatory differential sim** (§6.1, the #1 guardrail) is E1's engine, the **conformance gate** (§6.2) is B1's
suite, the **mass-deny circuit breaker / rollback** (§6.1, G8-MUST-032) is G3's mechanism, the **reuse
substrate** (§4.3) is E3's catalog, the **NS write boundary** is D2, the **NS view constraints** are E2, the
**promotion approval** is A2/D3. G8 is a **coordination spec** that writes MUSTs against six other components.
If any of E1 / B1-R30 / G3 / E3 slips (and the corpus flags E1 as the longest pole, B1-R30 as unbuilt, and G3
as a sibling that doesn't exist yet — its SPEC isn't written), the corresponding G8 guarantee is vapor. The PLAN
half-admits this ("if E1 slips, WS-E ships the lint halves") — but the lint halves are precisely the
*non-load-bearing* parts. **The guardrail that matters most (sim-before-enforce) is the one most likely to be
missing, and a lint can't substitute for it.**

### AR-5 (MEDIUM) — No measurement of error rate *before* the platform claims success, and M9 is defined circularly
G8-MUST-081 makes **M9 (author escape rate)** the gated acceptance criterion for "the median engineer
succeeds." Good. But:
- M9 is "defects that reached **enforce** and were rolled back / caused an incident." That only counts the
  failures the *other guardrails let through*. If the sim gate (§6.1) is working, M9 is near-zero **by
  construction** — not because authors are good, but because the gate caught their mistakes. M9 measures
  *guardrail leakage*, not *authoring competence*. The competence signal is actually **M1 (inner-loop catch
  rate)** and the **raw defect-injection rate** (how often does a fresh author's first draft fail lint/test?) —
  which G8 doesn't directly measure. A platform can have a near-zero M9 and a population that cannot write Rego;
  the gates are just doing all the work. That might be *fine as a product outcome* — but it falsifies the claim
  "the median engineer succeeds at authoring," replacing it with "the median engineer's mistakes are caught."
  Those are different claims and G8 conflates them.
- There is **no baseline**. Without measuring authoring error rate *before* the toolchain, none of the M1–M9
  targets are calibrated — "≥ 90% caught before PR" of *what denominator*? The metrics are unfalsifiable as
  written.

---

## 2. Gaps, inconsistencies, and unhandled cases

### AR-6 (HIGH) — The "first correct policy in < 30 min for a novice" target is fantasy for anyone who hits a TODO
§9 G8-MUST-060 targets < 30 min novice via template-and-fill. But §7 routes **every** novice promotion through
mandatory central review (G8-MUST-044) — which is a *human in another team's queue*. The wall-clock to a
*merged, dry-run* novice policy is therefore bounded below by **central-reviewer turnaround**, not by the 30-min
authoring loop. The onboarding metric measures the easy half and ignores the half that actually blocks Sam (the
same review-latency wall his persona's "before" state complained about — a Jira plus three Slack threads). G8
re-creates the bottleneck it set out to abolish and then measures around it.

### AR-7 (MEDIUM) — Code review four-eyes assumes a second Rego-literate reviewer exists; for novice/NS it assumes central capacity that the org explicitly lacks
G8-MUST-052 (four-eyes on code) and G8-MUST-044 (mandatory central review of NS policies) both assume a supply
of Rego-literate central reviewers with spare capacity. The *premise of the whole component* is that this supply
is scarce (Marcus is a team of six; that's why §1 exists). G8's solution to "novices can't write safe Rego" is
"make experts review all of it" — which **does not scale** and re-centralizes the bottleneck the NS-authoring
feature (D2) existed to decentralize. Either NS-authoring decentralizes authoring (then central review can't be
mandatory on all of it) or central review is mandatory (then NS-authoring didn't decentralize anything). G8
hasn't resolved this tension; it's written both MUSTs and let them collide.

### AR-8 (MEDIUM) — "Inner loop identical to CI" is asserted but the local-eval harness uses a *cached, possibly stale* bundle and *redacted* fixtures
§5.3/§5.5 (and B3-R16) let the local loop run against a cached signature-verified bundle offline, and §5.3
pulls *redacted* production events. Both legitimately diverge from CI: a stale local bundle evaluates against
old `data`/external_data; a redacted fixture has different field presence than the real event. So "identical to
CI" (G8-MUST-001, D-G8-6) is true for the *ruleset/invocation* but **not for the inputs/data**, and the spec
elides that. An author can pass locally against stale data and fail CI against current data — the exact F8 the
spec claims to abolish, re-entering through the data plane G8-MUST-026 doesn't cover.

### AR-9 (MEDIUM) — Templates "lint-clean as shipped with fail-closed TODO defaults" can't both be true for a build-time/detective control
G8-MUST-012/015 say an unfilled template defaults **fail-closed (deny/require-review)** for runtime classes and
the template is lint-clean+test-green as shipped. But for a **Build-Time (Conftest) or Detective** control (A2
§4.2, no dry-run/warn ladder, `enforce` = CI-hard-fail or audit-emit), a "fail-closed default" shipped template
means **a template instance that, if merged un-filled, fails every CI build or emits a violation on everything**.
The fail-closed default that's safe for runtime admission is a denial-of-service for the build pipeline. G8
applied a one-size default across enforcement classes A2 explicitly distinguishes. Templates need
*class-aware* defaults, and the spec doesn't say so.

### AR-10 (MEDIUM) — Single-source-of-truth metadata "generate the var from METADATA" contradicts B1's MUST that both be hand-present
G8-MUST-036 resolves the dual-metadata drift (B1 AR-11) by making authors edit only `# METADATA` and
**generating** `__control_id__`. But **B1-R1 says every package MUST *declare* both** and the build rejects a
package where either "is absent." A generated var satisfies "present at build time" only if the generation step
runs *before* the B1 build check — a sequencing dependency the spec asserts but doesn't pin (which CI stage
generates? does pre-commit? what about a package edited outside the toolchain?). If an author edits METADATA and
pushes without the generation step, B1-R1 rejects it for a *missing* var the author was told not to write. G8
and B1 give the author contradictory instructions unless the generation step is guaranteed to run everywhere —
which §6.4 assumes but doesn't enforce.

### AR-11 (LOW) — Measurement (M1–M9) is itself an authoring/instrumentation burden with no owner-of-the-owner
§11 mandates nine metrics across two cohorts and every product, on E2 + C5. That's a non-trivial analytics
build (it overlaps C3/C5) with its own correctness risk — a mis-instrumented M3 ("mass-deny near-miss") could
report 0 because it's miscounting, not because the guardrail works. The spec mandates the metrics but doesn't
own *validating* them, and a falsely-green safety metric is worse than none.

### AR-12 (LOW) — Cookbook / templates rot exactly like the conformance corpus the B1 review warned about
B1-AR (line ~99) warns that golden cases without an active generation strategy *rot*. G8's template catalog,
cookbook, and tutorial corpus are the same liability at larger scale — they must be regenerated/re-verified
against every B1 contract change, every E3 library bump, every Regal rule addition. §6 PLAN makes them CI tests
(good) but doesn't budget the *maintenance* — and a stale template that ships a now-wrong pattern teaches the
whole author population the wrong thing at once.

---

## 3. Prioritized defect list

| # | Sev | Defect | Fix |
|---|---|---|---|
| **AR-1** | **CRITICAL** | Portability defended by a heuristic lint + an unbuilt conformance suite; Kyverno path isn't Rego-portable at all | State the honest scope: portability guaranteed only across Rego-executing engines (OPA-REST/Wasm/Gatekeeper/Conftest); Kyverno controls are re-authored. Make B1-R30 a *blocking prerequisite*, not a co-built MUST. Stop the lint implying semantic-equivalence proof. |
| **AR-4** | **CRITICAL** | The load-bearing guardrail (sim-before-enforce) and 5 other MUSTs are subcontracted to components that may slip (E1, B1-R30, G3, E3, D2) | Add an explicit "external-contract status" gate: G8 may not claim the mass-deny guardrail until E1's gate interface + G3's circuit breaker exist. A lint cannot substitute for the sim gate; say so. |
| **AR-2** | **HIGH** | NS "most-restrictive-wins" is asserted but no component guarantees the decision combinator; permissiveness check is blind to latent (unexercised) gaps | Pin the combinator: name the B-layer component that guarantees most-restrictive aggregation across central+NS bundles, or downgrade NS-authoring to "additive denials in a disjoint control namespace only, no overlapping decisions." Add latent-permissiveness analysis (static, not event-driven). |
| **AR-3** | **HIGH** | Generator solution is a measurement plan, not a solution; hard cases ≈ common cases; median author still hand-writes the judgment-heavy part | Be explicit that for controls outside the catalogued-easy set, **direct Rego authoring by novices is not supported** — those go to experts. Don't pretend templates close the competence gap; scope which controls novices may author at all. |
| **AR-5** | **HIGH** | M9 measures guardrail leakage, not authoring competence; no pre-toolchain baseline; metrics unfalsifiable | Add a raw first-draft-defect-rate metric and a baseline measurement before claiming success. Distinguish "median engineer succeeds" (competence) from "median engineer's mistakes are caught" (containment) — and decide which the product actually claims. |
| **AR-6** | **HIGH** | < 30-min novice onboarding ignores mandatory central-review latency — the real bottleneck | Either measure *time-to-merged-dry-run* including review, or provide a fast-path (auto-approve template-only NS policies that pass permissiveness + lint, reserve human review for advanced/raw-Rego). |
| **AR-7** | **MEDIUM** | Mandatory central review of all NS policy re-centralizes the bottleneck NS-authoring existed to remove; assumes scarce expert capacity | Tier review: template-and-fill NS policies with a benign permissiveness result get lightweight/automated review; only `permissive_ns_policy` or raw-Rego NS policies require an expert. |
| **AR-8** | **MEDIUM** | "Inner loop identical to CI" is false on the data/bundle plane (stale cache, redacted fixtures) | Scope the claim to ruleset+invocation; add a freshness check + a "data-stale" warning; mark redacted-fixture results advisory. |
| **AR-9** | **MEDIUM** | One fail-closed template default DoS's build-time/detective pipelines | Make template defaults **enforcement-class-aware** (runtime ⇒ fail-closed deny; build-time ⇒ no-op/skip until filled; detective ⇒ emit-nothing until filled). |
| **AR-10** | **MEDIUM** | Generate-the-var contradicts B1-R1's MUST-both-present unless the generation step is guaranteed everywhere | Pin where generation runs (pre-commit + a CI pre-build stage) and make B1-R1 check run *after* it; or change B1-R1 to require METADATA-only and treat the var as derived. Reconcile with B1. |
| **AR-11** | **LOW** | Safety metrics (M1–M9) can be falsely green; no validation owner | Add metric-validation tests (synthetic streams with known outcomes) and treat a green safety metric without a passing validation as unverified. |
| **AR-12** | **LOW** | Templates/cookbook/tutorials rot against contract changes | Budget maintenance; CI-gate the corpus against every B1/E3/Regal change; version + sign templates (already in SPEC G8-MUST-016) and alert on staleness. |

---

## 4. "This will not survive contact with production because…"

1. …**the median author hits a template TODO they fill wrong, and nothing downstream catches a *semantically*
   wrong-but-lint-clean policy except a simulation that only sees the last 30 days.** The dormant-hole and
   wrong-judgment cases (AR-2, AR-3) pass every gate. The platform ships a control that's syntactically perfect
   and semantically wrong, with a green dashboard.
2. …**the sim gate that prevents the mass-deny depends on E1, the corpus's longest pole, and the circuit breaker
   depends on G3, whose SPEC isn't even written.** The one incident G8 exists to prevent is gated by the two
   components least likely to be ready (AR-4). In the gap, the lint is all that stands between an expert and a
   fleet-wide deny — and a lint can't simulate.
3. …**mandatory central review collides with the scarce-expert premise** (AR-6/AR-7). Either reviews pile up and
   authoring stalls (the original field-failure, re-created), or reviews get rubber-stamped to clear the queue
   (the four-eyes gate becomes theater). Both are the documented OPA-rollout death spiral.

---

## 5. Biggest human-factors risk to the platform thesis (the bottom line)

**The honest, uncomfortable conclusion the SPEC circles but never states: the platform thesis may require that
humans largely *not* author Rego directly.** Every G8 mechanism is an admission that direct Rego authoring is
dangerous (lint to catch footguns, sim to catch mass-deny, templates to avoid blank pages, review to catch the
rest, metrics to watch for the failures that slip). When the entire toolchain exists to prevent the author from
doing the thing the platform asks the author to do, the design is telling you the abstraction is wrong. The
defensible end-states are:
- **(a)** Narrow direct Rego authoring to the expert team (Marcus), make *everything else* generated-from-Gemara
  or template-and-fill with **no raw-Rego affordance for novices at all**, and accept that controls the
  generator can't do are *expert work, full stop* — not "novice work with TODOs." This is smaller, honest, and
  probably what survives contact with production.
- **(b)** Keep broad authoring but accept that **M9-near-zero comes from the gates, not from author competence**,
  and market the product as *"we catch your Rego mistakes,"* not *"anyone can write Rego."* — a containment
  claim, not a competence claim.

What G8 must not do is what the current SPEC half-does: claim the median engineer *succeeds at authoring* while
building a six-layer apparatus whose existence proves the median engineer *fails at authoring* and is only saved
by the apparatus. That gap between the claim and the apparatus is the biggest human-factors risk to the thesis —
because the moment a customer's median author ships a semantically-wrong-but-green policy (AR-2/AR-3) and a real
control gap reaches production, the differentiator ("authoritative, traceable, correct policy everywhere")
becomes the liability ("your platform vouched for a policy that was wrong"). The product that vouches for human
Rego it can't fully verify is exposed exactly where it claims to be strongest.

---

## 6. Build-vs-buy: the authoring layer (no full ALT required, per brief)

The brief asks for a note rather than a full alternative architecture. The question: **build the authoring layer,
or lean on OPA Control Plane (OCP) + Regal + the emerging vendor tooling?**

**The case for *buy/adopt* (strong, and the corpus's own posture for adjacent layers):**
- **Regal is already the answer for lint** — it's CNCF-open-sourced (market §2), it *is* the OPA linter, and B1
  already adopts it (B1-D-B1-02). G8 should **not build a linter**; it builds a *ruleset on Regal* (which the
  SPEC correctly does, §5.1). Custom Regal rules are the right and only build here.
- **OCP already covers regression-testing bundles against historical decision logs** (market §2) — which is the
  substrate for the mandatory-sim guardrail (§6.1) and overlaps E1. The corpus already decided to adopt OCP
  behind B1's `BundleService` (B1-D-B1-01). G8's sim gate should ride OCP/E1, not reinvent replay.
- **Nirmata's AI Platform Engineering Assistant** (market §3) is a *direct competitor* to the
  authoring/approval/audit layer — multi-agent policy authoring, copilot, signed PRs with approver steps,
  rollback safety — **for the Kyverno half**. It demonstrates the market is building exactly G8's value
  (assisted authoring + review + rollback) as a product. For the Kyverno path, "buy/partner" is a live option;
  building a parallel copilot is racing a funded vendor.
- **The §26.1 generator is the one genuinely differentiating build** (Gemara-intent → Rego, control-traceable) —
  but even there, the LLM-assisted-authoring pattern is commoditizing fast (Nirmata's copilot, the general
  coding-assistant wave). The defensible part is the **Gemara-control-id binding and the lint/test/sim
  guardrails around generated output**, not the generation itself.

**The case for *build* (narrow):**
- The **custom Regal ruleset** (control-id/metadata/portability/replay-completeness checks) is platform-specific
  and *must* be built — no vendor enforces *this* product's authoring contract. ~Small, high-value.
- The **template catalog on top of E3** is content unique to this platform's control corpus; build it (it's
  authoring, not engineering — cheap, parallel).
- The **measurement layer (M1–M9)** has no off-the-shelf equivalent (no vendor measures *your* authoring error
  rate against *your* controls). Build it — it's the thing the META-ADVERSARIAL pass says nobody owns.

**Recommendation (the note):** **Buy/adopt the engine (Regal lint, OCP/E1 replay-sim, B1 bundle/sign); build
only three thin platform-specific layers** — (1) the custom Regal ruleset enforcing the control contract,
(2) the Gemara-bound template catalog over E3, (3) the authoring-error-rate measurement. Explicitly **do not
build** a linter, a replay engine, a bundle pipeline, or a generic policy copilot — all four are now free CNCF
substrate (Regal, OCP) or funded-vendor territory (Nirmata). And for the **Kyverno authoring path**, seriously
evaluate **partnering with / adopting Nirmata's assistant** rather than building a second copilot — the corpus
already concedes Kyverno is a separate effector (G-1), so its authoring layer can be a separate buy. The
build-vs-buy answer mirrors the rest of the corpus: **adopt the substrate, build the control-traceability and
human-factors thin layer that is genuinely yours** — which is, precisely, what makes G8 worth owning at all.
