# Reframing the platform around AI agents

The clean insight from reading the original spec through an agent lens: **the spec is already 70% agent-shaped, because it's identity-aware, has runtime decision points, captures replay-capable audit events, runs differential simulations, supports suspend-pending-approval, and treats policy promotion as a graduated lifecycle.** Almost every layer translates without changing the architecture; what changes is what the "resource" is, what the "subject" is, and what "evaluation" means.

So I'll go layer by layer, mark what translates directly, what shifts in meaning, and where genuinely new primitives are needed.

---

## What is "the resource" being governed?

In the original spec, the resource is a Kubernetes object, a Terraform plan, a CI artifact, a Keycloak login, or a SonarQube finding. For agents, the resource set expands to roughly six things, each of which is a natural policy decision point:

| Resource | Real-time hook | Examples of policy decisions |
|---|---|---|
| The prompt (user turn + system prompt) | Pre-inference gateway | Block PII, require role scope, enforce template fidelity |
| The assembled context window | After retrieval, before inference | Block ungrounded retrievals, enforce data classification, deny cross-tenant context |
| The model invocation | Inference proxy | Route by trust level, deny untrusted models for sensitive tasks, enforce parameter limits |
| The tool/MCP call | MCP gateway | Allow/deny tool, require approval over thresholds, enforce argument constraints |
| The output | Post-inference filter | Block leaked secrets, enforce citation requirements, redact PII |
| Resource-budget events | Per-step accounting | Token/cost/latency/step ceilings per session, tenant, role, or task class |

Each of these is structurally identical to a §17C.4 PDP — *event taxonomy, real-time hook, audit source, replay schema, subject mapping, supported actions* — just at a different attachment point.

---

## What is "the subject"?

This is where the original Keycloak/JWT layer gets richer rather than replaced. The spec's §15 claim model already wants `sub`, `groups`, `roles`, `tenant`, `environment`, plus recommended `risk_level`, `workload_identity`, `data_classification`. For agents, the subject is a *chain*, not a person:

- **Originating user** — the human (or upstream agent) who initiated the session
- **Agent identity** — what agent is this, what version, signed by whom (Sigstore attestation extends naturally)
- **Model identity** — which model, which version, hosted where, with what fine-tuning attestation
- **Tool catalog** — which MCP servers are bound to this session, signed by whom
- **Capability token** — what this agent is authorized to do in this session (typed grants, expirable)
- **Trust grade** — the policy-relevant trust level for this combination (the gradient your prompt mentions)
- **Delegation chain** — if agent A spawned agent B, the chain is part of the subject

The JWT-to-policy mapping layer (§15.4) does this work already; for agents it grows a few standard fields and a "delegation expansion" transform. The big conceptual shift is that *trust is a graded property of the (agent, model, tools, context) tuple, not of the originating user alone.*

---

## The trust gradient is the lifecycle the spec already describes

Your hypothesis — start restrictive with an untrusted agent and loosen over time — is exactly the §9.2 enforcement-mode lifecycle (Deny → Warn → Dry Run → Audit) plus the §17.4 differential simulation framework, applied to *agent capabilities* instead of *Kubernetes constraints*. The mapping is direct:

- Initial deployment: every tool call denied except a small allowed set; every output requires human approval; every retrieval logged with full attestation.
- After a defined observation window, simulation replays the agent's past traces under a relaxed policy and reports newly-allowed actions. A reviewer tags each as *intended relaxation*, *potential regression*, or *requires review* — the same four-quadrant tagging from §17.4.
- The relaxed policy is promoted from dry-run to warn to enforce, one capability at a time, on the same promotion pipeline used for any other Rego or Kyverno policy.
- Behavioral evaluators (below) feed back into trust grade. If hallucination rate or drift exceeds threshold, the policy engine reverses the gradient and re-tightens automatically, optionally requiring an approval to override.

This is significant: the trust gradient is not a new feature, it is the existing policy-promotion lifecycle pointed at a different resource. The differential simulation framework you already specified is the killer feature here, because it lets you answer *"if I relax this guardrail, what does that mean against the actual production traffic we've seen?"* before you ship the relaxation.

---

## What is "enforcement"? Adding a behavioral tier

The original §7.2 enforcement classes (Runtime, Build-Time, Detective, Manual, Advisory) cover most of what agents need. The genuinely new piece is a behavioral-evaluation tier that sits between Runtime and Detective. These evaluators *are policies* in the OPA sense — they take structured input and return a decision — but their inputs are model outputs, traces, and accumulated state. Think of them as PDPs whose decision logic includes a sub-call to a judge model or a numeric evaluator.

| Behavioral concern | Evaluation pattern | Decision outcomes |
|---|---|---|
| Hallucination / groundedness | Per-output: cited claims vs. retrieved context; judge-model citation check | warn / require_human_review / deny output |
| Data / behavior drift | Periodic eval-suite replay against fixed reference set; output-distribution monitoring | warn / suspend_for_review / re-tighten trust grade |
| Uncontrolled looping or meandering | Per-step: repeated state detection, plan-vs-action cosine drift, goal-progress judge | deny next step / require_human_review / terminate session |
| Resource consumption | Per-session: token / cost / wall-clock / tool-call rate ceilings | deny / require_approval / throttle |
| Output quality | Per-output: confidence thresholds, citation coverage, format conformance | warn / require_human_review / deny |
| Sensitive-action detection | Pattern match on tool calls or output content | suspend_pending_approval / deny |

Each evaluator emits an event that fits the existing §13 audit schema with two added fields: an evaluator identity and an evaluator confidence/score. The §17.4 differential simulation framework now also classifies behavioral changes — *"the new groundedness threshold would have suspended 4.3% of previously-allowed outputs in the last 30 days."*

The architectural point: behavioral evaluators don't need a separate engine. They're policies that happen to consume model outputs and traces. OPA + sidecar evaluator services (which can themselves be model calls) handle this within the existing decision-point model.

---

## The audit schema additions

§13.3 already preserves enough fields for replay; for agents, the `request_object` and `external_data_refs` slots carry the full agent state. Concretely:

- `request_object` becomes the assembled context: system prompt, conversation history, retrieved documents (with hashes and source attestations), tool catalog (with versions and attestations), model identity, sampling parameters.
- `external_data_refs` gets RAG retrieval IDs and versions, vector-store digests, and MCP server identities and versions.
- `before_state` / `after_state` apply when the agent is mutating something: memory writes, vector store updates, tool calls with persistent side effects.
- A new optional sub-record for `evaluator_results` carries the per-step behavioral evaluator scores.
- The `replay_completeness` field gains an agent-specific note: even with full inputs, LLM outputs are non-deterministic. Replay-completeness for *policy decisions about prompts and tool calls* is achievable; replay-completeness for *exact model outputs* is best-effort. This distinction matters for what kinds of simulations the platform can claim as authoritative.

The OpenTelemetry GenAI semantic conventions, which stabilized in late 2025, are the natural wire format for these traces. Treat them the same way OCSF is treated in §13.5 — an optional compatibility target, with the platform's replay schema authoritative.

---

## The per-component PDP catalog, rewritten for agents

The §17D libraries (Kubernetes, Keycloak, Jenkins, GitLab, Trivy, Grafana, ES/Kibana) get analogs for agents. Each follows the same nine-field structure:

| Component | Real-time hook | Replay source | Example policies |
|---|---|---|---|
| Model API gateway (proxy in front of Anthropic, OpenAI, Bedrock, local model) | Inference proxy | Request/response log + attestations | Prompt redaction; deny untrusted model for PII-tagged session; enforce sampling parameters |
| MCP server / tool gateway | MCP request interception | Tool-call trace | Allow-list tools per role; require approval for write operations; enforce argument schemas |
| RAG / vector retrieval | Retrieval interception | Retrieval log with doc IDs and ACL evaluation | Deny cross-tenant retrieval; enforce data classification; require retrieval scope per task |
| Agent memory store | Read/write interception | Memory event log | Deny memory writes containing PII; enforce TTL; cross-session leakage detection |
| Agent runtime / planner | Step pre-hook | Step log with plan and tools-considered | Step budgets; loop detection; sub-agent spawn approval |
| Output sink (back to user, to system, to downstream agent) | Post-inference filter | Output log with evaluator scores | Block ungrounded claims; redact PII; require citation; sensitive-action approval |
| Eval gate (pre-promotion) | CI/CD-equivalent eval suite | Eval results | Required pass rate on safety/quality eval; required behavioral coverage |
| Resource accounting | Per-event accumulator | Cost/token ledger | Budget ceilings per session, tenant, role, task class |

The same §17C.3 action taxonomy applies: `allow`, `deny`, `warn`, `mutate` (e.g., redact, downgrade model), `generate` (e.g., create approval request), `suspend_pending_approval`, `require_async_check` (run an evaluator before committing). The `suspend_pending_approval` outcome is particularly valuable here because many agent decisions reasonably block on a human review queue, and §17B's webhook integration is the right pattern.

---

## What approval gates apply to (your prompt example)

You raised prompts and context windows as approval scope. The pattern that works:

- **Pre-prompt approval** when a user submits a request matching a sensitive-action pattern (financial transfer over a threshold, production data modification, customer communication on regulated topics). The agent receives a token saying "this prompt is approved for this scope until time T."
- **Mid-session approval** when an evaluator detects a flag during execution (e.g., agent is about to invoke a high-impact tool with unusual arguments). The agent suspends, an approval CRD is created, the human approves (possibly in Slack/ServiceNow/Jira via the existing webhook integration), and the agent resumes.
- **Capability-token approval** at session start: the human pre-approves a scope ("you may write to JIRA, read from Confluence, query the analytics warehouse for non-PII data, spend up to $50") and the agent runtime carries that token as part of its subject identity. Every tool call is checked against the token; out-of-scope actions trigger suspend-pending-approval.

These are three policy patterns on the same underlying primitive. The first matches admission control, the second matches the K8s admission-can't-wait pattern (use a CRD-based approval object), the third matches OAuth scope tokens.

---

## Visualization implications

The §16 governance console doesn't fundamentally change; the lineage graph now includes nodes for *prompts*, *traces*, *evaluator results*, and *capability tokens*, alongside controls and policies. The most useful new view is a *session timeline* — every step the agent took, every policy decision along the way, every evaluator score, with the ability to fork-and-replay from any step under a proposed policy change. This is essentially the differential simulation UI applied to a single trace.

The trust-gradient view is also new and valuable: per agent (or per agent class), a dashboard showing current trust grade, currently-active constraints, evaluator score trends, and "candidate relaxations" the simulation engine has identified as safe to promote based on accumulated evidence.

---

## High-level recommendations on standards and protocols

A few external pieces the platform should align to rather than reinvent, since they're maturing fast:

- **MCP** as the primary tool-call interception layer. It is rapidly becoming the de facto standard and provides a clean policy-attachment point. The platform's MCP gateway PDP is the highest-leverage real-time hook.
- **OpenTelemetry GenAI semantic conventions** as the trace wire format, treated the same way OCSF is treated for security events — useful but not the authoritative replay schema.
- **Sigstore-style attestation** for models, system prompts, and MCP server bundles. The supply-chain primitives already in the spec extend cleanly here. A model is an artifact; a system prompt is an artifact; an MCP server is an artifact; all can be signed and verified at admission time.
- **FINOS AIGF v2.0** (agentic risk catalog) and the **NIST AI Agent Standards Initiative** (announced Feb 2026) as the upstream control catalogs. Map them through the same Gemara/OSCAL hierarchy already in §6.
- **NIST AI RMF generative AI profile and EU AI Act Annex III obligations** as the compliance frameworks that pre-built control packs should target first, similar to how the original spec targets SOC 2 / NIST 800-53 / CIS.

---

## What stays unchanged

The point of the reframing exercise: very little of the original architecture has to change. The same governance hierarchy, the same policy lifecycle, the same differential simulation engine, the same audit replay schema (with field additions), the same approval-webhook pattern, the same per-product PDP catalog model, the same scoped-roles and storage-authorization design, the same Headlamp / Backstage / OpenShift / Rancher plugin distribution — all of it applies. The work is in:

1. Adding 5–6 new PDP types to the §17D catalog (model gateway, MCP gateway, RAG, memory, runtime, output sink, eval gate, resource accounting).
2. Extending the JWT/subject model with agent, model, tool, capability-token, and delegation-chain fields.
3. Adding a behavioral-evaluator tier between Runtime and Detective enforcement classes, with the explicit understanding that some evaluators are themselves model calls.
4. Augmenting the audit schema with trace context, evaluator results, and an agent-specific replay-completeness note.
5. Treating the trust gradient as a first-class lifecycle pattern with UI affordances, but implementing it via the existing simulation + differential + promotion mechanics.
6. Adding standards alignment to MCP, OTel GenAI, and the FINOS/NIST agent governance catalogs.

If you imagine the original spec's table of contents and just relabel a few sections — "§17D.2 Kubernetes Library" becomes "§17D.2 MCP Gateway Library," "§13 Audit Schema" gains a sub-section on traces — the document still reads naturally. The architecture was built for cross-product policy decisions, and AI agents are simply the most demanding cross-product policy domain that has ever existed. That's the strategic alignment: the spec didn't need to be designed for agents to be the right design for agents.