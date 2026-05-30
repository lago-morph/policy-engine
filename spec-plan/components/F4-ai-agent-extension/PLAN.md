# F4 — AI / Agent Governance Extension — PLAN

**Component:** F4 · **Source:** reframed-for-ai.md (+ Wedge-7) · **Status:** Authored (domain-lead fallback)

F4 is a **delta layer**: every workstream extends a base component. It ships in Phase-3 (after base MVP, per F3) but design can begin in parallel with Phase-2.

---

## 1. Dependency DAG (F4 deltas onto the base)

```
[C2 §13 schema (extensible)] ──> [DELTA-D audit fields: evaluator_results, trace ctx, agent subject]
[D1 §15.4 mapping] ───────────> [DELTA-B subject chain + delegation-expansion transform]
[D2 §17A scope] ──────────────> [DELTA-B effective-subject intersection (child ≤ parent)]
[F2 plugin SPI] ──────────────> [DELTA-A 6 agent PDPs] + [DELTA-C evaluator plugins] + [DELTA-H catalog]
[E1 §17.4 differential sim] ──> [DELTA-E trust gradient via existing sim+promotion]
[A2 §9.2 lifecycle] ──────────> [DELTA-E promote/demote per capability]
[D3 §17B approval CRD] ───────> [DELTA-F 3 approval patterns]
[A1 §6 Gemara] ───────────────> [DELTA-I FINOS-AIGF/NIST-AI catalogs as Gemara docs]
[E2 §16 console] ─────────────> [DELTA-J session timeline + trust-gradient views]
[F1 API] ─────────────────────> agent sub-resources (sessions, traces, evaluator results, capability tokens)
```

Critical path within F4: **DELTA-D (audit fields) + DELTA-B (subject chain) → DELTA-A (agent PDPs, esp. R4 MCP gateway) → DELTA-C (behavioral tier) → DELTA-E (trust gradient).**

## 2. Parallel workstreams

- **WS-A — Subject chain (DELTA-B):** extend §15.4 mapping with agent/model/tool/capability/delegation fields + delegation-expansion + Sigstore artifact verification. Foundation for all agent authz.
- **WS-B — Audit deltas (DELTA-D):** C2 schema fields + OTel-GenAI ingest mapping + the policy-vs-output replay-completeness distinction. Foundation for replay/sim.
- **WS-C — Agent PDPs (DELTA-A/H):** the 6 resources + 8 catalog libraries as plugins; **prioritize R4 MCP gateway** (highest leverage). Parallel once F2 SPI + DELTA-D exist.
- **WS-D — Behavioral tier (DELTA-C):** evaluator plugin framework, judge-model integration, `require_async_check` with latency budgets, evaluator identity+confidence. The genuinely new build.
- **WS-E — Trust gradient (DELTA-E):** wire behavioral evaluators → trust_grade → demote (reverse gradient) onto E1 differential + A2 promotion. Depends on WS-B/C/D.
- **WS-F — Approval patterns (DELTA-F):** 3 patterns on D3's CRD; agent-runtime suspend/resume.
- **WS-G — Standards + viz (DELTA-I/J):** FINOS/NIST catalogs as Gemara docs; session-timeline + trust-gradient console views.

## 3. Critical path

`C2 extensible schema confirmed (F1 DEFECT-8) → DELTA-D agent audit fields + DELTA-B subject chain → DELTA-A R4 MCP gateway PDP → DELTA-C behavioral evaluators → DELTA-E trust gradient closed loop`. The MCP gateway + behavioral tier are the two longest poles.

## 4. Milestones

- **F4-M1:** Subject chain (DELTA-B) maps an agent session into the D2 effective subject; delegation intersection enforced; artifact signatures verified at admission.
- **F4-M2:** Audit schema emits agent events (DELTA-D) with evaluator sub-record; OTel-GenAI ingest maps in; replay-completeness distinction enforced.
- **F4-M3:** MCP gateway PDP (R4) allow/deny/approval on tool calls against capability token; the headline agent demo.
- **F4-M4:** Behavioral tier (DELTA-C) — groundedness + loop-detection + budget evaluators emitting scored events; `require_async_check` gating works within latency budget.
- **F4-M5:** Trust gradient (DELTA-E) — differential replay of agent traces under a relaxed policy; reviewer tagging; one-capability-at-a-time promotion; automatic re-tighten on evaluator threshold.
- **F4-M6:** 3 approval patterns (DELTA-F); session-timeline + trust-gradient console views (DELTA-J); FINOS/NIST catalogs (DELTA-I).

## 5. What can be built concurrently / what blocks what

- Concurrent: WS-A and WS-B (subject vs audit) are independent; WS-C catalog libraries parallelize among themselves; WS-G standards/viz parallel to everything.
- Blocks: WS-B (audit schema delta) blocks WS-E (trust gradient needs replayable agent traces). WS-A (subject chain) blocks WS-C agent authz and WS-F capability-token approval. F2 plugin SPI blocks all PDP/evaluator plugins. **Base MVP (F3) blocks F4 ship** (base-first).
- Cross-domain: F4 is a consumer of C2/D1/D2/D3/E1/E3/A1/A2/E2/F1 — it touches nearly every domain but **adds no base primitive**, so it's additive load, not refactor load (the thesis).

## 6. Test strategy

- **Delta-conformance tests:** prove each F4 delta is additive — base golden fixtures (§13 events, §15.4 mappings, §17.4 diffs) still pass with agent fields present; no base regression.
- **Subject-chain authz tests:** child agent cannot exceed parent grant (effective-subject intersection); expired capability token → suspend; unsigned model/MCP server → admission deny.
- **Behavioral evaluator tests:** groundedness/loop/budget evaluators return correct outcomes; `require_async_check` respects latency budget; evaluator non-determinism surfaced via confidence (not asserted as reproducible).
- **Trust-gradient tests:** differential replay of recorded traces under relaxed policy yields correct newly-allowed set; automatic re-tighten fires on threshold breach; demote audited.
- **Replay-completeness tests:** policy-decision replay is exact; exact-output replay is flagged best-effort and NOT exported as authoritative evidence (ties F1 DEFECT-6).
- **Approval-pattern tests:** pre-prompt token scoping; mid-session suspend/resume via CRD status; capability-token enforcement on every tool call.
- **Standards tests:** OTel-GenAI trace maps losslessly into §13; FINOS-AIGF control ingests as Gemara doc and appears in lineage.

## 7. Risks

- Behavioral evaluators are model calls → cost, latency, and non-determinism on the enforcement path (DELTA-C); strict latency budgets + async-where-possible.
- The replay-completeness caveat (exact outputs best-effort) can be oversold by sales as "we replay agents exactly" → enforce R-F4-AUD-2 as a product guardrail.
- MCP/OTel-GenAI/FINOS-AIGF are fast-moving 2025–2026 standards → version-pin and treat as optional compatibility targets, schema authoritative (like OCSF).
- Scope creep: F4 is tempting to ship before the base is proven → hold base-first (F3 OQ-3-2).
