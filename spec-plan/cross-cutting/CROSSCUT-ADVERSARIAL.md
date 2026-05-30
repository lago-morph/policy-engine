# Cross-Domain Adversarial Reconciliation — CROSSCUT-ADVERSARIAL

**Date:** 2026-05-30 · **Author role:** Cross-domain adversarial reconciler (final hostile pass).
**Inputs read in full:** all 6 `domains/*/DOMAIN-ADVERSARIAL.md`; the 23 `components/*/ADVERSARIAL-REVIEW.md`
(CRITICAL/HIGH findings); the frozen `C2-audit-schema/SPEC.md` (v1.0); `B4/SPEC.md` (action taxonomy);
`D1/SPEC.md`, `C3/SPEC.md`, `B5/SPEC.md`, `E3/SPEC.md` (seam verification); `DECISIONS.md`.

**Purpose.** Consolidate, de-duplicate and **rank** every contradiction that crosses a domain boundary;
assign each a resolution, an owning component, and a correctness-vs-hardening verdict; surface NEW
contradictions the domain leads missed at the component seams; and name the **build-blocking subset** that
must land before the foundation contracts (C2 schema v1.0, B4 action taxonomy, D2 scope predicate, the
`replay_completeness` semantics) freeze.

**The single most dangerous structural fact this pass surfaced:** *C2 schema v1.0 is already declared
FROZEN, but it has already baked in two of the very modeling errors that Domains B and E say must be fixed
"before C2 bakes them in."* The freeze happened in parallel with the domain reviews, so the freeze is
premature. See **XD-1** and **XD-3**. The first cross-cut action is to **un-freeze C2 to v1.0-rc**, land the
correctness fixes below, then re-freeze. Everything downstream (E1, C3, C4, C5, F4, C1) depends on it.

---

## 1. Consolidated, de-duplicated, ranked cross-domain register

Severity: **C** = Correctness (must land before the relevant contract freezes) · **H** = High ·
**M** = Med. "Build-blocking" column = blocks a foundation-contract freeze (C2 schema / B4 taxonomy /
D2 predicate / replay-completeness semantics). Fix-class: **Correctness** (do before build) vs
**Hardening** (can defer with a documented risk).

| ID | Title | Domains | Sev | Build-block | Fix-class |
|---|---|---|---|---|---|
| **XD-1** | External-data **value** capture is unowned; C2 says it retains values but makes capture optional, B-domain engines never emit it ⇒ flagship image-signing replays non-`complete` | B,C,E,F | **C** | **YES (C2)** | Correctness |
| **XD-2** | `replay_completeness` computed in ≥5 places (C2 store, E1 per-bundle, F1 jobs, F4, E3 prior); E1 MUST recompute, C2 flag is only a lower bound — not stated in frozen C2 | C,E,F | **C** | **YES (C2/semantics)** | Correctness |
| **XD-3** | Action model: B4 linear 13-action precedence conflates exclusive disposition with co-occurring obligations; **C2's frozen `decision` enum already baked the conflation in** | B,C,E | **C** | **YES (B4+C2)** | Correctness |
| **XD-4** | Separation-of-Duties silently broken: D2 roles additive/union (author+approver co-holdable) defeats D3/D4 SoD and E1/E3 tagging SoD | D,E | **C** | no | Correctness |
| **XD-5** | Storage-layer authz mandated (§17A.1/§23) but storage deferred (§26.1/§22.2); analytics/reporting aggregate reads bypass the D2 scope interceptor | D,F,C | **C** | **YES (D2 contract)** | Correctness |
| **XD-6** | "Declared vs verified": coverage/completeness/SoD-emission self-asserted but rendered certified; A1 badges, A2 gates, C5 rollups, E1/E2/E3 differentiators all inherit | A,C,E | **C/H** | partial (C2 label) | Correctness |
| **XD-7** | Composed fail-closed defaults ⇒ correlated fleet-wide mass-deny outage; no "infrastructure-degraded" mode / circuit breaker; spans B fail-closed + D fail-closed self-DoS | B,D | **H** | no | Hardening (urgent) |
| **XD-8** | `correlation_id` contract unowned & overloaded across D1/D2/D3/B5/C2; **B5 retry mints a NEW id, fragmenting the governance transaction** the approval flow depends on | B,D,C | **H** | **YES (C2 §6)** | Correctness |
| **XD-9** | CRD ownership collision: B4 §17C.6 surface claimed by F2 (+3 new CRDs), F1 (REST projection), E1/E3 (controllers/instances) | B,F,E | **H** | no | Correctness |
| **XD-10** | Approval flow not durable, not retry-traceable, not secure (approve-then-swap, forged callback, approver-downgrade, status-trust expiry) — spans B + D | B,D | **C/H** | no | Correctness |
| **XD-11** | Action **taxonomy** drift: E3 verbs (`detect/alert/require review/notify/attach evidence/require MFA/fail build/clear hold`) absent from B4's CLOSED 13-enum and from C2's `decision`/`action_performed` enums — **three closed enums, three vocabularies** | B,C,E | **H** | **YES (B4+C2)** | Correctness |
| **XD-12** | NEW: `jwt_claims_completeness` (full/partial/reconstructed) is a producer/scorer orphan — D1 emits `jwt_claims` but never sets the completeness sub-state C2 §5.1 needs for `complete` and C3 DT-31 detects on | D,C | **H** | **YES (C2 contract)** | Correctness |
| **XD-13** | Stale policy-dependency catalog ⇒ confidently-wrong `complete`/equivalence; the C-domain's deepest defect, also feeds B (read-set) and E (replay) | C,B,E | **C** | **YES (C2 §4.2)** | Correctness |
| **XD-14** | NEW: §19 absence-of-evidence detection needs **policy-state-as-of-creation-time**; B5 exports current state, C4 must reconstruct historical expectation; C4's bundle-at-time inference is a single unverified fact | B,C | **H** | no | Correctness |
| **XD-15** | Live-vs-replay completeness-label divergence; "≥95% tie-out" (DT-78) hides non-determinism; conservative-label-wins not enforced cross-domain | C,E | **H** | no | Correctness |
| **XD-16** | Behavioral (AI-judge) evaluators are non-deterministic/latency-heavy on the enforcement path ⇒ break the platform's "deterministic replay" guarantee; must be a labeled best-effort tier, never co-mingled in evidence | F,C,E | **C** | partial (semantics) | Correctness |
| **XD-17** | PII/identity-data handling fragmented across D1(emit)/C2(store)/D2(export)/D4(retain); F4 agent context (prompts) is the most sensitive payload and lands in the deferred store; vs E1/E3 replay-fidelity need | D,C,F,E | **H** | no | Hardening |
| **XD-18** | Three signed-evidence-package formats (C1, C5, C2); auditor cannot verify uniformly; C2 must own the integrity primitive | C | **M** | no | Correctness |
| **XD-19** | Signing-key compromise / correlation-id collision in federated store unmodeled — the integrity foundations everyone trusts | C,D | **H** | no | Hardening |
| **XD-20** | K8s-RBAC (§16.2) vs governance-claim authz (§17A.4) have no defined precedence where E2 graph crosses D2 scope | D,E | **M** | no | Correctness |
| **XD-21** | F3 MVP cut line violates its own "no MVP item relies on a deferred item" — D2/storage deferred but in MVP; E1 + D2 mislabeled "thin slice" | F | **H** | no | Planning |
| **XD-22** | Webhook endpoints (D2 config role + D3 emit) are an unguarded PII egress / SSRF channel | D | **M** | no | Hardening |

---

## 2. Per-defect detail: conflict, impact, resolution, owner, class

### XD-1 — External-data **value** capture is unowned (the flagship-example killer) · CRITICAL · build-blocking

- **Domains:** B (B5-AR-5, B1-AR-7), C (X5, DC-3), E (X4, E1-D2, E3-G4/D3), F (C-5).
- **Conflict.** Everyone's headline example is "require signed image" (§18.1/HL-06). Replaying it faithfully
  requires the **value** of `image-signature-status` **at decision time**. The producing side (B-domain
  PDPs, B1 `decision.evidence`) captures `nd_builtin_cache` (B1-R26) but **not external-data values**
  (B5-AR-5). The schema side, C2, **claims** to retain raw external-data responses (C2 §8.3 N-C2-402,
  D-C2-05) and provides a `value_ref` field (field 27) — **but `value_ref` is OPTIONAL** and external-data
  *value capture is nobody's MUST*. C2 §5.2 even lists "value unavailable but re-resolvable" as a *normal*
  `best_effort` path, which silently legitimizes not capturing the value. E1/E3 both independently report
  the flagship examples therefore "replay as `partial`."
- **Impact.** The platform's single most-marketed differentiator (authoritative replay of the signing
  decision) is, as specified, **best_effort, not authoritative, for exactly the example used to sell it.**
  This is the contradiction at the seam between "C2 says it has the value" and "no producer emits it."
- **Resolution.** Make external-data **value** capture a first-class MUST, not an option:
  (a) B1/B5 decision evidence MUST capture the external-data **value** used (extend B1 `decision.evidence`
  §5; the verifier's answer at t3), not just version; (b) C2 field 27 `value_ref` becomes **conditionally
  required** whenever the policy-dependency catalog (§4.2) flags the provider as volatile/non-re-resolvable
  (signature status, CVE feed); (c) own a content-addressed external-data snapshot store — **assign to C2**
  (it already owns CAS via §8.4); (d) C2 §5.2's "re-resolvable ⇒ best_effort" path is narrowed to providers
  the catalog marks *stable*; volatile providers with no captured value ⇒ `insufficient`, never `best_effort`.
- **Owner:** C2 (store + required-ness) with B1/B5 (capture at source). **Class: Correctness — un-freeze C2.**

### XD-2 — `replay_completeness` computed in ≥5 places; E1 must RECOMPUTE; C2 flag is a lower bound · CRITICAL · build-blocking

- **Domains:** C (X4, DC-4), E (C1: "computed in three places with three meanings"), F (C-5).
- **Conflict.** C2 stores a per-event `replay_completeness` flag (field 28, **R**). But E1 reads a **new
  bundle** at replay time — a newer policy may consult a field the old event never captured — so the *same
  event* can be `complete` for the bundle it was decided under and `insufficient` for the bundle being
  simulated. E1's per-bundle introspection is authoritative *for a given run*; C2's stored flag is only a
  **lower bound**. F1 jobs, F4 audit deltas, and E3's static `replay_completeness_notes` add three more
  computation sites. The frozen C2 SPEC §5 says "the classifier runs in the normalizer and again in E1/C4
  and they MUST agree on the same inputs" — but **does not say the inputs are bundle-relative**, so it
  asserts an agreement that is false by construction the moment the bundle changes.
- **Impact.** E1 over-reports authority ("47 newly blocked, 0 newly allowed" stated as fact over a subset
  that's actually `insufficient` for the new bundle). The console misleads the exact compliance users
  (Priya, Daniel) who rely on it. A2 promotion gates (A2-DEF-01) consume this and promote on blind data.
- **Resolution.** Codify in the shared `DATA-MODEL.md` and amend C2 §5: **completeness is policy-relative;
  the stored C2 flag is a per-event lower bound computed against the deciding bundle; E1 MUST recompute per
  simulated bundle and that recomputation is authoritative for the run; conservative-label-wins on
  divergence (`insufficient > best_effort > complete`); divergence on a `complete`-stored event is a
  finding, not tolerance.** One vocabulary (`complete | best_effort | insufficient`) across C2/E1/F1/F4.
- **Owner:** C2 (semantics + DATA-MODEL) + E1 (recompute) + F1/F4 (adopt). **Class: Correctness.**

### XD-3 — Action model: linear precedence conflates disposition + obligations; **C2 already froze it** · CRITICAL · build-blocking

- **Domains:** B (X-2; B4-AR-7, B2-AR-7, B1-AR-9), C (the consumer), E (E1 normalizes outcomes).
- **Conflict.** B4-R7 defines a **single linear precedence** over 13 heterogeneous actions
  (`deny > require_approval > quarantine > mutate/generate/annotate > …`). Domain B's lead correctly says
  this conflates the **exclusive admission disposition** (exactly one of allow / deny /
  suspend-pending-approval) with **co-occurring obligations** (mutate, annotate, notify, require_scan — all
  can happen together) and **must be split into `disposition` + `obligations[]` BEFORE C2 and E1 bake in the
  linear-precedence assumption.** **But C2 is already FROZEN, and it already baked it in:** C2 field 9
  `decision` is a **flat enum** `allow | deny | warn | mutate | suspend_pending_approval | require_approval |
  unknown` — `mutate` sits as a *sibling* of `deny`, so a deny-that-also-mutates has no representation; and
  `action_performed` (field 15, **optional**) is a second, non-reconciled effect enum. B4-R6 half-acknowledges
  the split ("action=require_approval, disposition=deny-pending-approval") but C2 has no `disposition` field
  and no `obligations[]` array to record it.
- **Impact.** A v1alpha/v1.0 **data-model error that is already frozen** and gets dramatically more expensive
  every day E1, C3, C4, C5 and F4 build against it. B1's derived `allowed` boolean (B1-AR-9) mis-collapses
  `require_approval`. Reports cannot faithfully show "denied AND mutated AND notified."
- **Resolution.** **Un-freeze C2 to land this.** Replace field 9 with **`disposition`** (exclusive:
  `allow | deny | suspend_pending_approval | unknown`) + **`obligations[]`** (set, drawn from the B4 effect
  actions: `mutate, generate, annotate, notify, require_scan, quarantine, warn, …`). B4 splits its 13-action
  enum into one exclusive disposition axis + an obligations set and **drops the single linear precedence**
  (precedence only meaningful within the disposition axis). B1 drops the `allowed` boolean (or makes it the
  3-state disposition). E1 normalizes to disposition+obligations, not allow/deny-class.
- **Owner:** B4 (model) + C2 (schema fields) jointly. **Class: Correctness — un-freeze C2 AND B4 taxonomy.**

### XD-4 — Additive roles defeat Separation-of-Duties · CRITICAL

- **Domains:** D (X1; D2-A5, D3-A11, D4-A8), E (C3: tagging SoD).
- **Conflict.** D2's SPEC describes roles as **additive with union semantics**. D3-R6 / D4 SEC-11 / E1's
  tag-write flow all depend on **requester ≠ approver** and **author ≠ enforce-promoter ≠ tagger**. One
  subject holding `namespace-policy-author` + `namespace-policy-approver` for the same scope can author *and*
  approve — the exact "blind promotion" HL-17 claims to prevent.
- **Impact.** The entire approval/promotion story (D3, A2, E1 simulation-gated promotion) is built on an SoD
  invariant that the role model silently breaks. Self-grant + self-approve voids it.
- **Resolution.** SEC-11 mutual-exclusion is **normative**: SoD pairs (author/approver, requester/approver,
  author/promoter, author/tagger) cannot be co-held for the same scope; enforced at `role:grant` time **and**
  re-checked at action time. A **single tag-write service** (E1-owned) enforces SoD; E2 is a client, not a
  second source of truth (resolves E-domain C3).
- **Owner:** D2 (SPEC §2 amend) + D3/D4 (enforce) + E1 (tag service). **Class: Correctness.**

### XD-5 — Storage-authz mandated but storage deferred; aggregate reads bypass the interceptor · CRITICAL · build-blocking (D2 contract)

- **Domains:** F (the domain-dominating finding: F1-D1, F2-D1, F3-D2, F4-D2), D (D2-A1/A2, Y4), C (analytics/reporting reads).
- **Conflict.** §17A.1/§23 demand storage-enforced authz and tamper-evident evidence; §26.1 defers storage
  "out of scope for the POC" and §22.2 permits "ordinary storage." F1 has no substrate to push its scope
  predicate into; F2's "BYO ordinary storage" can't do row/field-level scope. **Worse (D2-A2 + Y4):** §14
  analytics and §17E reporting aggregate **across scopes by nature** (per-subject cross-tenant counters) and
  **likely bypass the D2 interceptor entirely** — voiding the §17A.1 invariant for the *most data-rich*
  queries. This is a Domain D ↔ Domain C ↔ Domain F seam nobody fully owns.
- **Impact.** The platform's core security invariant is enforced by a **lint** (D2-A1), not a control; one
  bypass voids it; the richest queries (the ones an attacker wants) are the ones most likely to escape.
- **Resolution.** Storage is **partially in MVP**: D2 ships a **scope-predicate library (row AND field
  level)** that F1, F2 controllers, the C2 query API (§8.2 N-C2-401 already references this), and **the
  C3/C5 analytics/reporting read paths** all link — never reimplemented (resolves F C-2 triple-impl drift).
  F2 defines a **minimum storage contract** (scope columns, append-only/versioned audit, content hashing);
  "ordinary storage" ⇒ "ordinary storage *that supports the minimum contract*." For relational, promote
  **RLS-under-interceptor** from ALT to mandatory.
- **Owner:** D2 (predicate library + contract) + F2 (minimum storage contract) + C3/C5 (route reads through
  it). **Class: Correctness — blocks the D2 foundation contract.**

### XD-6 — "Declared vs verified" honesty deficit, platform-wide · CRITICAL→HIGH

- **Domains:** A (A1-DEF-01/06, A2-DEF-01/10), C (DC-12, X4), E (whole-domain theme).
- **Conflict.** Coverage `full`, EvidenceRequirements, `complete` labels, SoD, and "≥95% tie-out" are all
  **self-assertions rendered as certifications.** A `full` coverage link to a dry-run-only control is a lie
  (A1-DEF-06); A2 gates run on audit data whose completeness they never check (A2-DEF-01); C5 rollups can
  aggregate away the evidence-quality denominator (DC-12); E1/E2/E3 present confidence they haven't earned.
- **Impact.** The platform's entire value is *not lying about evidence fidelity* (C domain net assessment).
  This is the first objection an external auditor (Daniel) raises.
- **Resolution.** Everywhere, distinguish **assertion** from **evidence**: A1 coverage badges =
  "management assertion" until backed by an operating-effectiveness signal; A2 promotion gates MUST consume
  and surface `replay_completeness`, with explicit acknowledgement to promote over low-completeness data;
  C5 every rollup carries a **denominator evidence-quality %**; E1/E2/E3 make coverage/completeness/
  enforceability **headline, sortable facts, never footnotes**. The C2 honesty label (field 28) is the
  substrate but is insufficient alone — consumers must surface it.
- **Owner:** A1/A2 (gates+badges), C5 (rollups), E1/E2/E3 (UX), on C2's label. **Class: Correctness.**

### XD-7 — Composed fail-closed ⇒ correlated mass-deny outage; no circuit breaker · HIGH

- **Domains:** B (X-4; B1-AR-6, B2-AR-5, B5-AR-2/7, B3-AR-5), D (X3; D1/D2/D3 fail-closed self-DoS).
- **Conflict.** Every component correctly fails closed in isolation — but composed, a degraded **shared
  dependency** (bundle server, cosign verifier, Keycloak/D1, audit sink) causes **fleet-wide mass-deny**:
  Keycloak down = no authn, verifier slow = `failurePolicy: Fail` mass-deny, approval webhook down = no
  deploys. B5 makes "time out to failurePolicy" a flow invariant, *guaranteeing* the mass-deny, and the
  composed defaults can prevent recovery of the very dependency that failed.
- **Impact.** The security-correct defaults, composed, are a single correlated availability failure mode
  with no circuit breaker — a self-inflicted fleet outage.
- **Resolution.** A platform-wide **"infrastructure-degraded" mode** distinct from "policy says no": when a
  *shared* dependency degrades, drop to warn/advisory with loud alerting + a §19 catch-up scan, rather than
  mass-deny. Pair with **audited, rate-limited, time-boxed break-glass** + RTO at each fail-closed boundary
  (D4). The system-namespace carve-out (B2-R4) is necessary but insufficient. This is a **flow-level (B5)**
  decision, not five independent component choices.
- **Owner:** B5 (flow) + B1/B2/B3 + D4 (break-glass). **Class: Hardening (but urgent — availability P0).**

### XD-8 — `correlation_id` unowned/overloaded; B5 retry mints a NEW id, fragmenting the governance transaction · HIGH · build-blocking (C2 §6)

- **Domains:** D (Y1), B (X-3; B5-AR-4), C (C2 §6 owner).
- **Conflict.** `correlation_id` threads D1 (`jwt_claims` audit), D2 (`authz_denied`/`boundary_crossing`),
  D3 (approval chain == CR name), B5 (minted server-side per request, B5-R1), and C2 (§6 join key). No
  component **owns** the contract — and they conflict: **B5-R1 mints one id per request, but the approval
  retry (B5-R7 / B2-R19) is a *later, separate* admission that mints a NEW `correlation_id`** (B5-AR-4). So
  the original denied request (id X) and the approved retry (id Y) **do not share a join key** — the
  governance transaction fragments across C2's chain and C4's reconstruction. C2 §6 defines anchors but has
  **no notion of a transaction id that survives retry**, and `parent_correlation_id` (field 4) is documented
  only for synthetic replay events, not for approval retry.
- **Impact.** The headline §17B.4 deny→approve→deploy flow cannot be reconstructed end-to-end; C4 sees the
  approved deploy as an unpaired (bypass-looking) event; the audit trail of "who approved what that then
  deployed" is broken.
- **Resolution.** Define `correlation_id` **once in C2 §6** (per-request join key, minted server-side per
  B5-R1) **plus a stable `governance_transaction_id`** that survives retry (B4 mints it at approval-request
  time, B5 propagates it on the retry, C2 carries it — reuse `parent_correlation_id` or add a frozen
  optional `governance_transaction_id`). C2 cluster-scopes the id to defeat federated collision (DC-11).
- **Owner:** C2 (contract + field) + B4/B5 (mint/propagate). **Class: Correctness — touches frozen C2.**

### XD-9 — CRD ownership collision (§17C.6) · HIGH

- **Domains:** B (B4 owns §17C.6), F (C-1; F2 + 3 new CRDs, F1 REST projection), E (C4; E1/E3 controllers).
- **Conflict.** The §17C.6 CRD surface is claimed by **B4** (schema), **F2** (controllers/operator + 3 new
  CRDs), **F1** (REST sub-resource projection), and **E1/E3** (`PolicySimulationRun`, `PolicyActionLibrary`
  controllers/instances). Three-to-four owners, one surface ⇒ drifting schemas.
- **Resolution (single rule, ratify in B4 SPEC):** **B4 owns all governance CRD *schemas*; F2 owns
  controllers + operator; F1 owns the REST projection; E-components own their controllers/instances.**
- **Owner:** B4 (schema authority) ratifies; F2/F1/E1/E3 reference. **Class: Correctness (ownership).**

### XD-10 — Approval flow not durable, not retry-traceable, not secure · CRITICAL→HIGH

- **Domains:** B (X-3; B2-AR-2/3, B4-AR-1/2, B5-AR-4), D (D3-A1/A5/A8, D4).
- **Conflict.** The deny→CRD hand-off is best-effort/lossy (admission webhooks are stateless). On top:
  **approve-then-swap** — admission re-eval matches resource ref, not spec digest (D3-A1, contradicts
  D3-R3); **forged callback** asserts arbitrary `approved_by` via shared HMAC (D3-A5); `requiredApproval` is
  author-writable ⇒ approver-downgrade (B4-AR-1); single-use consumption is forgeable/TOCTOU (B4-AR-2);
  expiry relies on a reconciler so `status=approved` is honored past TTL (D3-A8).
- **Impact.** The headline §17B.4 capability is, as specified, **neither durable, nor traceable across retry
  (see XD-8), nor secure against approver-downgrade/forgery/expiry-bypass.** Three B components + D3 each own
  a piece and each assumes another fixes it.
- **Resolution.** Build as **one vertical slice (B2+B4+B5+D3) with a single owner**: concrete durable
  deny→CRD mechanism; **approval bound to the manifest digest** (re-eval compares digest, not ref);
  **approver-bound callback proof** (OIDC/approver-signed, not shared HMAC); controller-derived **immutable**
  `requiredApproval`; **serialized CAS** single-use consumption; PDP compares `now` vs `expires_at` **every
  eval** (status advisory); denied CR cooldown (D3-A4). Threads the `governance_transaction_id` from XD-8.
- **Owner:** B2+B4+B5+D3 single slice. **Class: Correctness (security).**

### XD-11 — NEW: action **taxonomy** drift — three closed enums, three vocabularies · HIGH · build-blocking

- **Domains:** B (B4 closed 13-enum), C (C2 `decision`/`action_performed` enums), E (E3 verbs; E-domain C2).
- **Conflict.** B4-R5 declares the 13-action enum **CLOSED** (`allow, deny, warn, mutate, generate,
  cleanup, quarantine, suspend, require_approval, require_scan, notify, annotate, exception`). C2 froze a
  **different** `decision` enum (field 9: adds `suspend_pending_approval`, `unknown`; omits `generate,
  cleanup, quarantine, require_scan, notify, exception`) plus a **third** closed `action_performed` enum
  (field 15: `block, mutate, annotate, route_for_approval, log_only`). And **E3 introduces a fourth set of
  verbs** the others don't recognize — `detect`, `alert`, `require review`, `notify`, `attach evidence`,
  `require MFA`, `fail build`, `clear hold` (E3 SPEC §17D tables; E3-D2). Each enum is *closed*, so a value
  legal in one is illegal in the others. E1's outcome normalization and E2's by-action graph rendering break
  on any verb they don't recognize.
- **Impact.** Classification, rendering, reporting and replay all silently mis-bucket or reject events that
  carry a "foreign" verb. A closed enum that disagrees with another closed enum is a guaranteed runtime
  contradiction, hardening as C2/E1/C3/C5 build against it.
- **Resolution.** **One canonical action taxonomy** owned by B4, expanded to absorb E3's detect-only/CI verbs
  (`detect→notify-class`, `alert→notify`, `fail build→deny-class`, `attach evidence→annotate/obligation`,
  `require review→require_approval`, `require MFA→obligation`); E3 **maps** product verbs onto it rather than
  inventing. C2's `decision`/`action_performed` are **regenerated from** the canonical taxonomy after the
  XD-3 disposition/obligations split (they become the disposition axis + obligation set), so there is **one
  vocabulary, derived once.** Done as part of un-freezing C2.
- **Owner:** B4 (taxonomy) + C2 (regenerate enums) + E3 (map). **Class: Correctness — un-freeze C2 + B4.**

### XD-12 — NEW: `jwt_claims_completeness` is a producer/scorer orphan · HIGH · build-blocking (C2 contract)

- **Domains:** D (D1 producer), C (C2 scorer, C3 consumer).
- **Conflict.** C2 field 19 `jwt_claims_completeness` (`full | partial | reconstructed`) is **conditionally
  required**, and C2 §5.1 makes `jwt_claims_completeness=full` a **precondition for `replay_completeness=
  complete`**, while C3 D-JWT-DRIFT (DT-31) detects on claim presence/omission. **But D1's SPEC — the
  producer of the `jwt_claims` block (D1 §165, §332) — never specifies emitting `jwt_claims_completeness`.**
  D1 says it emits `jwt_claims` and that `replay_completeness=complete` is "preserved," but the
  full/partial/reconstructed determination is left to no one: D1 doesn't set it, and the C2 normalizer
  cannot know whether D1 captured *all* policy-consulted claims without the policy-dependency catalog (XD-13)
  *and* a contract for who asserts fullness. The `reconstructed` value (claims rebuilt from a Keycloak
  token-issuance event joined by `sub`+time, C2 §6.2) requires the Keycloak adapter to *know* it
  reconstructed — also unspecified in D1.
- **Impact.** Either every event defaults `jwt_claims_completeness` absent ⇒ C2 cannot reach `complete`
  (mass under-reporting), or the normalizer guesses `full` ⇒ **false `complete`** (the worst outcome: a
  `complete` event that silently dropped a claim the policy used). C3's DT-31 JWT-drift detection fires on
  the wrong baseline.
- **Resolution.** Make it an explicit **D1↔C2 contract** in DATA-MODEL.md: the D1 normalization output MUST
  carry the **set of claims captured**; the C2 normalizer computes `jwt_claims_completeness` by comparing
  that set against the **policy-dependency catalog's required-claim list** (§4.2); the Keycloak-reconstruction
  path explicitly stamps `reconstructed`. Absent the catalog ⇒ `partial`, never `full` (honesty tenet,
  mirrors D-C2-06).
- **Owner:** C2 (scorer + contract) + D1 (emit captured-claim set). **Class: Correctness.**

### XD-13 — Stale policy-dependency catalog ⇒ confidently-wrong `complete` · CRITICAL · build-blocking (C2 §4.2)

- **Domains:** C (DC-1, the domain's #1 risk; C2-A1, C3-A2 equivalence hashing), B (catalog from B1 Rego
  metadata), E (replay relies on it).
- **Conflict.** The entire honesty edifice (`complete`, equivalence, coverage, `jwt_claims_completeness`,
  XD-12) rests on the policy-dependency catalog (C2 §4.2, sourced from B1 Rego metadata) being **current**.
  C2 trusts it blindly; a single stale catalog produces *confidently-wrong* `complete`. C3's
  input-equivalence hashing (C3-A2) derives "relevant claims/external-data" from the **same** catalog.
- **Impact.** The worst possible outcome — a `complete` label (trusted, unscrutinized) that is wrong because
  the catalog didn't list an input the policy actually consulted.
- **Resolution.** Pin the catalog version **per `policy_version`** and bind it into the event (or
  `content_hash`); at runtime, compare the bundle's **observed** external-data reads / claim reads against
  the catalog — on mismatch, **downgrade to `best_effort` with reason `catalog_stale`** (add to §5.5 frozen
  reason vocabulary). C3 equivalence derives its relevance set from the same pinned catalog.
- **Owner:** C2 (§4.2 pin + reason code) + B1 (Rego-metadata catalog production). **Class: Correctness.**

### XD-14 — §19 absence-of-evidence needs policy-state-as-of-creation-time · HIGH

- **Domains:** B (X-6; B2-AR-8, B5-AR-6, B5-R11), C (C4 reconstruction, C4-A5 bundle-at-time inference).
- **Conflict.** "No Gatekeeper event ⇒ bypass" false-positives on out-of-scope/dryrun/dropped events. B5's
  "expected-decision-set" export (B5-R11) must be **time-travel** (policy state *as of each resource's
  creation*), not current state, or it false-positives on every recent promotion/exception change (B5-AR-6).
  C4's whole reconstruction rests on "the bundle deployed to this scope at T" — a **single inferred fact**
  (C4-A5); if the deployment record is incomplete, C4 replays the wrong policy with possibly-`high` confidence.
- **Impact.** The bypass-detection differentiator (HL-06 "did it ever get bypassed?") either floods false
  positives or produces wrong verdicts with high confidence — a credibility bomb in front of the auditor.
- **Resolution.** B5 exports **policy-state-as-of-creation-time**; C4 reconstructs historical expectation
  from it via the three-view reconciliation (decision/audit/inventory); C4 **cross-checks** its inferred
  `policy_version` against any **real** C2 decision events in the same scope/window (which carry the actual
  `policy_version`) — mismatch caps confidence and flags the inference. Document the ephemeral-bypass residual
  blind spot honestly.
- **Owner:** B5 (export) + C4 (reconstruct + cross-check). **Class: Correctness.**

### XD-15 — Live-vs-replay label divergence; "≥95% tie-out" hides non-determinism · HIGH

- **Domains:** C (X4, DC-4), E (E1 re-score).
- **Conflict.** The completeness/disposition label is computed live (C2), at reconstruction (C4), and at
  replay (E1); they can disagree. DT-78's "≥95% tie-out" frames divergence as an **accepted tolerance** —
  but divergence on a `complete` event is a **defect** (non-deterministic policy, time-dependent external
  data, digest mismatch), not noise.
- **Resolution.** **Conservative-label-wins**; **any** live-vs-replay divergence recorded as a finding with
  root-cause, never hidden; target divergence on `complete` events ~0%; capture the catalog version that
  produced each label. (Couples to XD-2.)
- **Owner:** C2/C4/E1. **Class: Correctness.**

### XD-16 — Behavioral (AI-judge) evaluators break deterministic-replay · CRITICAL

- **Domains:** F (F4-D1/D2), C (audit schema must label the tier), E (replay-completeness vocabulary).
- **Conflict.** F4 places non-deterministic, latency-heavy, expensive model-call evaluators **inline** on
  the real-time enforcement path via `require_async_check`, while calling them "just PDP plugins." The
  platform's defining guarantees (deterministic replay, sub-ms admission, reproducible decisions) **do not
  hold** for this tier.
- **Resolution.** Behavioral evaluators are a **distinct, clearly-labeled best-effort tier**, never
  co-mingled with the deterministic tier in reports/evidence; adopt the F4-ALT two-tier async architecture;
  the "deterministic replay" claim is explicitly scoped in product + marketing to exclude best-effort
  agent-output replay; C2 events from this tier carry a frozen marker capping them at `best_effort`
  (analogous to N-C2-SYNTH).
- **Owner:** F4 + C2 (tier marker). **Class: Correctness (guarantee scoping).**

### XD-17 — Identity-data/PII handling fragmented; agent context in deferred store; vs replay fidelity · HIGH

- **Domains:** D (Y2, D4-A7), C (C2 store), F (F4-D2 agent prompts), E (X2 redaction vs replay fidelity).
- **Conflict.** D1 emits `email`/`sub` into `jwt_claims`; D2 redacts on export but at-rest holds raw; D4
  wants hash-at-rest but notes immutability vs GDPR right-to-erasure conflict; F4 puts entire assembled agent
  context (prompts, retrieved docs, tool args) into `request_object` — the most sensitive payload — into the
  **same deferred store** (XD-5), flowing to every plugin/export-adapter/`/audit/events` reader. Meanwhile
  E1/E3 need full input preserved for replay fidelity, and in regulated namespaces (payments-prod) — where
  differential sim is most valuable — redaction forces `partial`/`insufficient` replays.
- **Resolution.** A **single identity-data handling policy** spanning D1(emit)/C2(store, §7.5
  redaction-aware hashing)/D2(export)/D4(retain+erasure). For replay: **tokenized/structural-only** inputs
  that preserve policy-relevant shape without raw PII (DECISIONS.md entry). F4 agent context requires
  **field-level scope + classification-aware redaction (mandatory, not optional)** and a raw-prompt retention
  policy that doesn't itself violate EU AI Act/GDPR.
- **Owner:** D4 (policy) + C2 (store/redaction) + F4 (agent context) + D2 (export). **Class: Hardening
  (but legally load-bearing).**

### XD-18 — Three signed evidence-package formats · MED

- **Domains:** C (X2; C1, C5, C2).
- **Conflict.** C1, C5 and C2 each have a signing path ⇒ three incompatible formats; an auditor can't verify
  uniformly. **Resolution:** **C2 owns the integrity primitive** (Merkle + ed25519 + manifest, §7.6
  N-C2-304); C1/C5 own content assembly and call C2's primitive; no component implements its own signing.
- **Owner:** C2 primitive; C1/C5 assemble. **Class: Correctness (ownership).**

### XD-19 — Signing-key compromise / federated correlation-id collision unmodeled · HIGH

- **Domains:** C (DC-10/DC-11; C2-A5, C3-A9, C5-A6), D (key trust).
- **Conflict.** All tamper-evidence rests on the signing key and on `correlation_id` uniqueness; both have
  unmodeled failure modes an adversary targets precisely because they're foundational. **Resolution:**
  external timestamp/transparency-log anchor in addition to platform-signed checkpoints; out-of-band key
  distribution named in exports; D-CHAIN verifies the external anchor; **cluster-scope** the `correlation_id`
  and dedup on the scoped id (HL-20). (Couples to XD-8.)
- **Owner:** C2 + C3/C5. **Class: Hardening.**

### XD-20 — K8s-RBAC vs governance-claim authz precedence · MED

- **Domains:** D (D2 §16.2/§17A.4), E (E2 graph crosses both).
- **Conflict.** Kubernetes-native RBAC discovery (§16.2) and governance `namespaces` claims (§17A.4) are two
  authorization systems with **no defined precedence** where E2's graph spans them. **Resolution:**
  governance claim authoritative for *governance data*; K8s RBAC for *K8s objects*; reconcile the
  cluster-list-vs-data-scope UX. Owned jointly E2 + D2.
- **Owner:** E2 + D2. **Class: Correctness.**

### XD-21 — MVP cut line violates its own dependency rule; "thin slice" mislabel · HIGH

- **Domains:** F (F3-D1/D2).
- **Conflict.** F3's rule "no MVP item relies on a deferred item" is violated: D2/storage is deferred yet in
  MVP (XD-5); D2 (scope predicate over deferred storage) and E1 (differential simulation, the novel core)
  are labeled "thin slice" but are the two hardest builds. **Resolution:** re-label D2 and E1 **MVP-core,
  high-effort**; size honestly; resolve the storage-minimum-contract (XD-5) so D2 no longer depends on a
  deferred item. **Owner:** F3. **Class: Planning honesty.**

### XD-22 — Webhook endpoints are an unguarded PII-egress / SSRF channel · MED

- **Domains:** D (Y5; D2-A8 config role, D3-A6 emit).
- **Conflict.** A Workflow Integrator can point approval webhooks (carrying subject/resource PII) at an
  attacker URL ⇒ exfil + SSRF, **without holding any read permission.** **Resolution:** endpoint
  allow-listing + internal-address blocking as a joint D2/D3 control; make it a D4 SEC requirement.
- **Owner:** D4 (SEC req) + D2/D3 (enforce). **Class: Hardening.**

---

## 3. NEW contradictions the domain leads missed (the seams)

These were not in any single domain's register; they live precisely at the cross-domain interfaces the
brief named, and I verified each against the actual SPEC text:

1. **XD-3 / XD-11 — C2 already froze the action-model error the domains said to fix "before C2 bakes it
   in."** Domain B (X-2) and Domain E (C2) both said "fix the action model before C2/E1 bake in the
   linear-precedence assumption." Nobody noticed that **C2 v1.0 is already declared FROZEN with the
   conflated flat `decision` enum** (`mutate` as a sibling of `deny`, no `disposition`/`obligations[]`
   split) **and three mutually-incompatible closed action enums** (B4's 13, C2's `decision`, C2's
   `action_performed`, plus E3's verbs). The premise "before C2 bakes it in" is already false — the freeze
   was premature. **This is the single highest-leverage finding of this pass.**

2. **XD-12 — `jwt_claims_completeness` is a producer/scorer orphan (D1→C2→C3).** The brief flagged the
   D1 jwt_claims → C2 jwt_claims_completeness → C3 drift seam; verified: **D1's SPEC never emits the
   completeness sub-state** C2 §5.1 needs for `complete` and C3 DT-31 detects on. No contract says who sets
   full/partial/reconstructed. The normalizer can't compute it without both the catalog (XD-13) *and* D1's
   captured-claim set. Risk: silent false-`complete` events that dropped a policy-consulted claim.

3. **XD-8 — B5 retry mints a NEW correlation_id (B5-R1) → C2 chain/C4 reconstruction fragment.** The brief
   flagged the B5 correlation_id-minting-on-retry → C2 chain → C4 reconstruction seam; verified:
   B5-R1 mints one id per request and the approval **retry** is a separate admission, so the denied request
   and approved deploy don't share a join key. C2 §6 has anchors but **no transaction-id-that-survives-retry**;
   `parent_correlation_id` is documented only for synthetic replay, not approval retry. C4 sees the approved
   deploy as unpaired (bypass-looking).

4. **XD-1 sharpened — C2 *claims* to retain external-data values but makes capture OPTIONAL, and §5.2
   legitimizes not capturing them.** The known theme was "values aren't captured." The sharper, missed point:
   C2's own SPEC (field 27 `value_ref` = optional; §5.2 "re-resolvable ⇒ best_effort" as a *normal* path) is
   internally complicit — it asserts retention in §8.3 while making the field optional and providing a
   `best_effort` escape hatch. The contract contradicts itself, which is *why* no producer emits the value.

---

## 4. Build-blocking subset — must land before the foundation contracts freeze

These MUST be resolved before the foundation contracts (C2 schema, B4 taxonomy, D2 scope predicate, the
shared `replay_completeness` semantics in `DATA-MODEL.md`) are (re-)frozen. Everything in Domains C/E/F/A
builds against them, and each gets dramatically more expensive once consumers bake it in.

**Un-freeze C2 to v1.0-rc, land these, then re-freeze:**

| Order | ID | What lands in the contract |
|---|---|---|
| 1 | **XD-3** | Split `decision` → `disposition` (exclusive) + `obligations[]` (set); drop B1 `allowed` boolean; drop B4 single linear precedence |
| 2 | **XD-11** | One canonical action taxonomy (B4-owned); C2 enums regenerated from it; E3 maps onto it |
| 3 | **XD-1** | External-data **value** capture = MUST (B1/B5 emit; C2 `value_ref` conditionally-required for volatile providers; C2 owns the CAS snapshot store); narrow §5.2's re-resolvable escape |
| 4 | **XD-2** | `replay_completeness` is policy-relative; C2 stored flag = lower bound; E1 recomputes per-bundle (authoritative for the run); conservative-label-wins; one vocabulary across C2/E1/F1/F4 |
| 5 | **XD-13** | Pin policy-dependency catalog per `policy_version`, bind into event; runtime read-set mismatch ⇒ `best_effort:catalog_stale` (add reason code) |
| 6 | **XD-12** | D1 emits captured-claim set; C2 computes `jwt_claims_completeness` vs pinned catalog; Keycloak path stamps `reconstructed`; absent catalog ⇒ `partial` never `full` |
| 7 | **XD-8** | Define `correlation_id` once in C2 §6 + add stable `governance_transaction_id` surviving approval retry (B4 mints, B5 propagates); cluster-scope ids |
| 8 | **XD-5** | D2 ships row+field-level scope-predicate library; analytics/reporting (C3/C5) read paths route through it; F2 minimum storage contract |
| 9 | **XD-9** | Ratify CRD ownership rule in B4 SPEC (B4 schema / F2 controllers / F1 REST / E controllers) |

XD-4 (SoD), XD-6 (declared-vs-verified), XD-10 (approval-flow security), XD-16 (behavioral-tier scoping) are
**correctness** but live in component SPECs rather than the frozen foundation contracts — they must land
before *build of those components*, but do not gate the C2/B4/D2 freeze. XD-7, XD-17, XD-19, XD-22 are
hardening (urgent but deferrable with a documented risk acceptance).

---

## 5. Top-10 ranked cross-domain defects (the report-back)

| # | ID | Title | Severity |
|---|---|---|---|
| 1 | XD-3 | Action model conflates disposition + obligations — **and C2 already froze it** | **CRITICAL** |
| 2 | XD-1 | External-data **value** capture unowned/optional ⇒ flagship image-signing replays non-`complete` | **CRITICAL** |
| 3 | XD-2 | `replay_completeness` computed in ≥5 places; E1 must recompute per-bundle; C2 flag is only a lower bound | **CRITICAL** |
| 4 | XD-5 | Storage-authz mandated but deferred; analytics/reporting aggregate reads bypass the D2 scope interceptor | **CRITICAL** |
| 5 | XD-13 | Stale policy-dependency catalog ⇒ confidently-wrong `complete`/equivalence (the C-domain's deepest defect) | **CRITICAL** |
| 6 | XD-4 | Additive D2 roles defeat author≠approver SoD (D3/D4 + E1/E3 tagging) | **CRITICAL** |
| 7 | XD-10 | Approval flow not durable, not retry-traceable, not secure (approve-then-swap / forged callback / expiry-bypass) | **CRITICAL→HIGH** |
| 8 | XD-11 | Action-taxonomy drift — three closed enums (B4 13 / C2 decision / C2 action_performed) + E3 verbs, all disagreeing | **HIGH** |
| 9 | XD-8 | `correlation_id` unowned + B5 retry mints a NEW id ⇒ governance transaction fragments across C2/C4 | **HIGH** |
| 10 | XD-12 | NEW: `jwt_claims_completeness` orphaned — D1 never emits it, C2 needs it for `complete`, C3 detects on it | **HIGH** |

**Bottom line for the cross-cut wave:** the architecture survives, but **C2 v1.0 was frozen too early.** Five
of the top six defects (XD-1, XD-2, XD-3, XD-11, XD-12, XD-13) are corrections to the *frozen* contract.
Re-open C2 to v1.0-rc, land the §4 build-blocking subset in plan order, then re-freeze before E1/C3/C4/C5/F4/C1
consume it. Resolve the action-model (XD-3) first — it is the contract change that hardens fastest and
poisons the most downstream consumers.
