# F4 — AI / Agent Governance Extension — SPEC

**Component:** F4 · **Domain:** F · **Source:** `policy engine reframed for ai.md` (+ positioning memo Wedge-7)
**Status:** Authored (domain-lead fallback) · **Date:** 2026-05-30
**Persona lens:** AI-platform/agent-governance architect.

> **FRAMING — THIS IS A DELTA SPEC.** F4 adds **nothing structural** to the base platform. The reframe doc's thesis: *the spec was built for cross-product policy decisions, and AI agents are the most demanding such domain — it didn't need to be designed for agents to be right for agents.* Every requirement below is marked as a **DELTA on a base component** (C2 audit schema, D1/D2 identity & scope, D3 approvals, E1 simulation, E3 PDP libraries, F1 API, A1 governance hierarchy). Where a base primitive already covers a need, F4 says "unchanged" rather than re-spec it.

---

## 1. Scope

F4 specifies how the platform governs **AI agents** by relabeling resources, enriching the subject, adding a behavioral-evaluation enforcement tier, extending the audit schema, providing an agent PDP catalog, and aligning to agent-governance standards. It is the realization of §20.3 (AI Governance) and Wedge-7.

**In scope (all as deltas):** 6 agent resources as PDP hooks; the agent subject chain; the behavioral-evaluation tier; audit-schema deltas vs C2; the agent PDP catalog mirroring E3/§17D; 3 approval-gate patterns vs D3; standards alignment (MCP/OTel-GenAI/Sigstore/FINOS-AIGF/NIST-AI-RMF/EU-AI-Act). **Out of scope:** changing the base architecture; building the agent runtimes themselves; replacing the base simulation/lifecycle/approval engines (F4 reuses them).

---

## 2. DELTA-A — The 6 agent resources as PDP hooks (extends E3/§17D, §17C.4)

Each agent resource is a §17C.4 PDP with the standard nine fields (event taxonomy, real-time hook, audit source, replay schema, subject mapping, resource mapping, decision outcomes, missing-capability notes). They attach at different points in the inference pipeline.

| # | Agent resource | Real-time hook (PDP) | Replay/audit source | Example decisions | Base analog |
|---|---|---|---|---|---|
| R1 | **Prompt** (user turn + system prompt) | Pre-inference gateway | Request log + system-prompt attestation | Block PII; require role scope; enforce template fidelity | Admission PDP |
| R2 | **Assembled context window** | After retrieval, before inference | Retrieval log + context hash | Block ungrounded retrieval; enforce data classification; deny cross-tenant context | Admission PDP |
| R3 | **Model invocation** | Inference proxy | Request/response log + model attestation | Route by trust level; deny untrusted model for sensitive task; enforce sampling params | Application PDP |
| R4 | **Tool / MCP call** | MCP gateway | Tool-call trace | Allow/deny tool; approval over threshold; enforce argument schema | Admission PDP (highest-leverage hook) |
| R5 | **Output** | Post-inference filter | Output log + evaluator scores | Block leaked secrets; enforce citation; redact PII | Application PDP |
| R6 | **Resource-budget events** | Per-step accounting | Cost/token ledger | Token/cost/latency/step ceilings per session/tenant/role/task | Per-event accumulator |

- **R-F4-RES-1 (MUST):** Each of R1–R6 is registered as a PDP plugin (F2 §5.3) with both a real-time hook descriptor AND a replay schema. Engines lacking replay are marked `replay: best-effort` (see DELTA-D non-determinism note).
- **R-F4-RES-2 (MUST):** The same §17C.3 action taxonomy applies — `allow`, `deny`, `warn`, `mutate` (redact/downgrade-model), `generate` (create approval request), `suspend_pending_approval`, plus the new `require_async_check` (run an evaluator before committing).
- **R-F4-RES-3 (SHOULD):** R4 (MCP gateway) is the **highest-leverage hook**; prioritize it (MCP is the de-facto tool-call standard — DELTA-G).

---

## 3. DELTA-B — The agent subject chain (extends D1 §15 / D2 §17A.4)

The base subject is a person (`sub`, `groups`, `roles`, `tenant`, `environment` + recommended `risk_level`, `workload_identity`, `data_classification`). For agents the subject is a **chain**, not a person. **Trust is a graded property of the (agent, model, tools, context) tuple, not of the originating user alone.**

New subject fields (additive to §15.2/§15.3; mapped by the §15.4 layer):

| Field | Meaning | Attestation |
|---|---|---|
| `originating_user` | Human/upstream agent that initiated the session | base `sub` |
| `agent_identity` | Which agent, which version | Sigstore agent attestation |
| `model_identity` | Which model/version, hosted where, fine-tuning attestation | model artifact signature |
| `tool_catalog` | MCP servers bound to this session, signed by whom | MCP server bundle signatures |
| `capability_token` | Typed, expirable grants for this session | platform-issued, scope-bound |
| `trust_grade` | Policy-relevant trust level for this tuple | derived (DELTA-E) |
| `delegation_chain` | If agent A spawned agent B, the chain | signed delegation records |

- **R-F4-SUBJ-1 (MUST):** The §15.4 mapping layer gains a **delegation-expansion transform** that materializes the chain into the authorization subject (D2). A tool call is authorized against the *effective* subject (intersection of every grant along the chain — a child agent can never exceed its parent).
- **R-F4-SUBJ-2 (MUST):** `capability_token` is checked on **every** R4 tool call (and R3 model invocation); out-of-scope actions trigger `suspend_pending_approval` (DELTA-F pattern 3).
- **R-F4-SUBJ-3 (MUST):** Model, system prompt, and MCP server are **artifacts** (Sigstore-style attestation, DELTA-G); admission of an agent session verifies their signatures the same way B1 verifies signed bundles (§23 integrity).
- **R-F4-SUBJ-4 (SHOULD):** `trust_grade` is recomputed as behavioral evaluators report (DELTA-C → DELTA-E feedback).

---

## 4. DELTA-C — The behavioral-evaluation tier (NEW enforcement class, between Runtime and Detective)

This is the **one genuinely new piece**. Behavioral evaluators **are policies** in the OPA sense (structured input → decision) but their inputs are model outputs, traces, and accumulated state, and their decision logic MAY include a sub-call to a judge model or numeric evaluator. They sit between §7.2 Runtime and Detective classes.

| Behavioral concern | Evaluation pattern | Decision outcomes |
|---|---|---|
| Hallucination / groundedness | Per-output: cited claims vs retrieved context; judge-model citation check | warn / require_human_review / deny output |
| Data / behavior drift | Periodic eval-suite replay vs fixed reference set; output-distribution monitoring | warn / suspend_for_review / re-tighten trust grade |
| Uncontrolled looping/meandering | Per-step: repeated-state detection, plan-vs-action drift, goal-progress judge | deny next step / require_human_review / terminate session |
| Resource consumption | Per-session: token/cost/wall-clock/tool-call rate ceilings | deny / require_approval / throttle |
| Output quality | Per-output: confidence thresholds, citation coverage, format conformance | warn / require_human_review / deny |
| Sensitive-action detection | Pattern match on tool calls / output content | suspend_pending_approval / deny |

- **R-F4-EVAL-1 (MUST):** Evaluators are PDP plugins (F2 §5) running out-of-process (a judge-model evaluator is itself a model call). No new engine; OPA + sidecar evaluator services within the existing decision-point model.
- **R-F4-EVAL-2 (MUST):** Each evaluator emits a §13 event with two added fields — **evaluator identity** and **evaluator confidence/score** (DELTA-D).
- **R-F4-EVAL-3 (MUST):** Evaluators that gate execution (groundedness on output, loop-detection on next step) use `require_async_check` so the result is computed before commit; latency budget per evaluator is declared and enforced.
- **R-F4-EVAL-4 (SHOULD):** Evaluator non-determinism (judge models vary) is acknowledged: evaluator decisions carry confidence, and the platform MUST NOT claim an evaluator decision is exactly reproducible (DELTA-D replay-completeness).

---

## 5. DELTA-D — Audit-schema deltas vs C2 (§13.3)

The §13 schema already preserves enough for replay; agents fill the existing slots and add a few fields. **All additive — C2's schema is the authoritative base; OTel-GenAI is an optional compatibility target (like OCSF in §13.5).**

| §13 field | Agent meaning (delta) |
|---|---|
| `request_object` | The **assembled context**: system prompt, conversation history, retrieved docs (with hashes + source attestations), tool catalog (versions + attestations), model identity, sampling params |
| `external_data_refs` | RAG retrieval IDs/versions, vector-store digests, MCP server identities/versions |
| `before_state` / `after_state` | When the agent mutates: memory writes, vector-store updates, tool calls with persistent side effects |
| `subject` | The DELTA-B chain (originating_user → agent → model → tools → capability_token → delegation) |

**New fields:**
| New field | Purpose |
|---|---|
| `evaluator_results` (optional sub-record) | Per-step behavioral evaluator scores (identity + confidence + outcome) |
| `replay_completeness` note (agent-specific) | Policy decisions about prompts/tool-calls ARE replay-complete; **exact model outputs are best-effort** (non-deterministic) |

- **R-F4-AUD-1 (MUST):** C2 declares these fields part of its extensible schema (F1 DEFECT-8: schema is explicitly open). No new event type needed — `event_type` namespaced as `agent.prompt`, `agent.tool_call`, `agent.output`, `agent.eval`, etc.
- **R-F4-AUD-2 (MUST — the load-bearing caveat):** The platform MUST distinguish **policy-decision replay (authoritative)** from **model-output replay (best-effort)**. Simulations may claim authority only over the former. This bounds what the differential simulation (DELTA-E) can assert.
- **R-F4-AUD-3 (SHOULD):** OTel GenAI semantic conventions (stabilized late 2025) are the wire format for trace ingest; mapped into the §13 schema which remains authoritative.

---

## 6. DELTA-E — Trust gradient = the existing lifecycle (extends E1 §17.4 + A2 §9.2)

The trust gradient ("start restrictive, loosen over time") is **not a new feature** — it is the §9.2 enforcement-mode lifecycle (Deny→Warn→Dry Run→Audit) + the §17.4 differential simulation, pointed at *agent capabilities* instead of K8s constraints.

- **R-F4-TRUST-1 (MUST):** Initial deployment denies all tool calls except an allow-set; outputs require human approval; retrievals logged with full attestation.
- **R-F4-TRUST-2 (MUST):** After an observation window, **differential simulation replays the agent's past traces under a relaxed policy** and reports newly-allowed actions; a reviewer tags each *intended relaxation / potential regression / requires review* (the §17.4 four-quadrant tagging, unchanged).
- **R-F4-TRUST-3 (MUST):** Relaxation is promoted dry-run→warn→enforce one capability at a time on the **same promotion pipeline** (A2) used for any Rego/Kyverno policy.
- **R-F4-TRUST-4 (MUST — the reverse gradient):** If a behavioral evaluator (DELTA-C) reports hallucination/drift over threshold, the engine **re-tightens automatically** (demote — F1 `lifecycle:demote`), optionally requiring an approval to override. This closes the loop DELTA-C → trust_grade → DELTA-B.
- **R-F4-TRUST-5 (SHOULD):** The differential framework also classifies **behavioral** changes: *"the new groundedness threshold would have suspended 4.3% of previously-allowed outputs in the last 30 days"* — but only as a best-effort claim (R-F4-AUD-2).

---

## 7. DELTA-F — The 3 approval-gate patterns (extends D3 §17B)

Three policy patterns on the **same** §17B primitive (`suspend_pending_approval` + `PolicyApprovalRequest` CRD + webhook):

1. **Pre-prompt approval** (matches admission control): a prompt matching a sensitive-action pattern (financial transfer over threshold, prod-data modification, regulated customer comms) gets a scoped, time-boxed approval token before execution.
2. **Mid-session approval** (matches the K8s "admission can't wait" pattern, §17B.4): an evaluator flags mid-execution (high-impact tool, unusual args); the agent **suspends**, an approval CRD is created, a human approves via the existing webhook (Slack/ServiceNow/Jira), the agent **resumes**.
3. **Capability-token approval** (matches OAuth scopes): at session start the human pre-approves a scope ("write JIRA, read Confluence, query non-PII analytics, spend ≤$50"); the runtime carries the token (DELTA-B); every tool call is checked; out-of-scope → suspend_pending_approval.

- **R-F4-APPR-1 (MUST):** All three reuse D3's `PolicyApprovalRequest` CRD + §17B.3 webhook; no new approval engine.
- **R-F4-APPR-2 (MUST):** Pattern 2 requires the agent runtime to support **suspend-and-resume** keyed by the approval CRD status (the agent-runtime PDP, DELTA-A R5/runtime). Runtimes that can't suspend MUST fall back to deny-with-approval-required (§17B.4).

---

## 8. DELTA-H — Agent PDP catalog (mirrors E3/§17D, nine-field structure)

Each agent PDP library follows the §17D nine-field structure. These bolt into the E3 catalog as new product libraries.

| Library | Real-time hook | Replay source | Example policies |
|---|---|---|---|
| **Model API gateway** (proxy in front of Anthropic/OpenAI/Bedrock/local) | Inference proxy | Request/response log + attestations | Prompt redaction; deny untrusted model for PII-tagged session; enforce sampling params |
| **MCP server / tool gateway** | MCP request interception | Tool-call trace | Allow-list tools per role; require approval for writes; enforce argument schemas |
| **RAG / vector retrieval** | Retrieval interception | Retrieval log + doc IDs + ACL eval | Deny cross-tenant retrieval; enforce data classification; require retrieval scope per task |
| **Agent memory store** | Read/write interception | Memory event log | Deny memory writes with PII; enforce TTL; cross-session leakage detection |
| **Agent runtime / planner** | Step pre-hook | Step log (plan + tools considered) | Step budgets; loop detection; sub-agent spawn approval |
| **Output sink** (to user / system / downstream agent) | Post-inference filter | Output log + evaluator scores | Block ungrounded claims; redact PII; require citation; sensitive-action approval |
| **Eval gate** (pre-promotion) | CI/CD-equivalent eval suite | Eval results | Required safety/quality pass rate; required behavioral coverage |
| **Resource accounting** | Per-event accumulator | Cost/token ledger | Budget ceilings per session/tenant/role/task class |

- **R-F4-CAT-1 (MUST):** Each library is an E3-style library (§17D) + a PDP plugin (F2). Same cross-product pattern (§17D.11) — product event → real-time hook? → PDP → allow/deny/warn/mutate/suspend → decision log → replay schema → simulation → reports.

---

## 9. DELTA-I — Standards alignment (extends A1 §6 Gemara/OSCAL hierarchy)

Align to, don't reinvent:

| Standard | Role | Mapped via |
|---|---|---|
| **MCP** | Primary tool-call interception layer; highest-leverage hook (R4) | DELTA-A R4 |
| **OpenTelemetry GenAI semantic conventions** | Trace wire format (optional, like OCSF) | DELTA-D R-F4-AUD-3 |
| **Sigstore-style attestation** | Sign/verify models, system prompts, MCP server bundles as artifacts at admission | DELTA-B R-F4-SUBJ-3 |
| **FINOS AIGF v2.0** (agentic risk catalog) + **NIST AI Agent Standards Initiative** (Feb 2026) | Upstream control catalogs | mapped through Gemara/OSCAL (§6) |
| **NIST AI RMF GenAI profile + EU AI Act Annex III** | Compliance frameworks pre-built control packs target first | like SOC2/NIST-800-53/CIS in the base |

- **R-F4-STD-1 (SHOULD):** FINOS AIGF/NIST-AI control catalogs are ingested as A1 Gemara documents (FINOS work explicitly uses Gemara — positioning memo); they populate the same lineage graph.

---

## 10. DELTA-J — Visualization (extends E2 §16)

The §16 console is unchanged structurally; the lineage graph gains nodes for **prompts, traces, evaluator results, capability tokens**. Two new views:
- **Session timeline:** every step, every policy decision, every evaluator score, with **fork-and-replay from any step** under a proposed policy change (= the differential-simulation UI applied to one trace).
- **Trust-gradient view:** per agent/class — current trust grade, active constraints, evaluator-score trends, and "candidate relaxations" the simulation engine flags as safe to promote.

- **R-F4-VIZ-1 (SHOULD):** Both are E2 console plugins (F2 visualization-extension SPI); no new GUI framework.

---

## 11. What stays UNCHANGED (the thesis, restated)

Governance hierarchy (A1), policy lifecycle (A2), differential simulation engine (E1), audit replay schema (C2, +fields), approval-webhook pattern (D3), per-product PDP catalog model (E3), scoped-roles + storage-authz (D2), Headlamp/Backstage/OpenShift/Rancher plugin distribution (E2/F2). **The work is exactly the 6 deltas** the reframe doc enumerates: (1) add 5–8 agent PDP types to E3, (2) extend the subject with agent/model/tool/capability/delegation fields, (3) add the behavioral-evaluator tier, (4) augment the audit schema, (5) make the trust gradient a first-class lifecycle pattern via existing sim+differential+promotion, (6) add standards alignment.

---

## 12. Open questions — decided defaults

| # | Question | Decided default | Rationale |
|---|---|---|---|
| OQ-F4-1 | Separate product or extension? | **Extension of the same platform (deltas)** — see ALT for the separate-product case | Reframe doc thesis; reuse > rebuild. ALT-AI-as-wedge argues the GTM counter. |
| OQ-F4-2 | Behavioral evaluators inline (blocking) or async? | **Both: `require_async_check` for gating evaluators with a latency budget; async for periodic drift** — see ALT-behavioral-tier | Output groundedness must gate before commit; drift is periodic. |
| OQ-F4-3 | Is exact-output replay authoritative? | **No — best-effort; only policy-decision replay is authoritative** | R-F4-AUD-2; LLM non-determinism. |
| OQ-F4-4 | New engine for evaluators? | **No — PDP plugins (judge models are model calls)** | Reframe doc; no new primitive. |
| OQ-F4-5 | Ship F4 before or after base MVP? | **After (Phase-3)** — base-first | F3; deltas need a proven base. |
| OQ-F4-6 | Trust grade authority | **Derived from evaluators + simulation, recomputed continuously; demote can be automatic with optional approval-to-override** | R-F4-TRUST-4 closed loop. |

---

## 13. Normative requirements summary

R-F4-RES-1..3, R-F4-SUBJ-1..4, R-F4-EVAL-1..4, R-F4-AUD-1..3, R-F4-TRUST-1..5, R-F4-APPR-1..2, R-F4-CAT-1, R-F4-STD-1, R-F4-VIZ-1. Every one is a **delta** on a named base component; F4 adds no base-architecture primitive.
