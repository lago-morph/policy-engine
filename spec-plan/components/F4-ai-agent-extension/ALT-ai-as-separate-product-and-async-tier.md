# F4 — ALT Architecture — AI Governance as a Separate Wedge Product + Behavioral Tier as Async Evaluator

**Component:** F4 · **Type:** Alternative architecture (high-value component) · **Status:** Authored (domain-lead fallback)
**Persona lens:** Contrarian founder/architect optimizing for time-to-market and clean enforcement guarantees.

This ALT presents two coupled deviations from the primary F4 SPEC:
- **ALT-1 (product axis):** ship AI/agent governance as a **separate product / wedge**, not an extension of the same platform.
- **ALT-2 (architecture axis):** implement the behavioral tier as an **async out-of-band evaluator service**, not an inline `require_async_check` enforcement hook.

Either can be adopted independently; together they form a distinct "agent-governance-first" architecture.

---

## ALT-1 — AI governance as a separate wedge product

### The bet
The primary SPEC's thesis is *reuse*: agents are "the most demanding cross-product policy domain," so extend the base platform. ALT-1 inverts: the base platform is a **6–18 month build** (F3 sequencing), while the **agent-governance market is forming right now** (NIST AI Agent Standards Initiative Feb 2026, FINOS AIGF v2.0 late 2025, MCP as de-facto standard — positioning memo Wedge-7). Waiting for the base MVP to ship before bolting on F4 (F3's base-first sequencing) **cedes the only time-sensitive market** to Cerbos/Permit.io/Aserto/Oso, who are already repositioning toward agent authorization.

ALT-1: build a **standalone agent-governance product** that ships the agent-relevant slice **first**, sharing libraries with the base platform but not waiting for it.

### What the separate product is
A focused "agent control plane": MCP gateway PDP (R4) + capability tokens (DELTA-B) + behavioral evaluators (DELTA-C, async) + agent audit/replay (DELTA-D) + the 3 approval patterns (DELTA-F). It targets the FINOS-AIGF / NIST-AI-RMF / EU-AI-Act control packs directly. It does **not** require Gemara, Gatekeeper, Conftest, or the full §17D catalog to exist first.

### What it shares vs duplicates
- **Shares (as libraries):** the §13 audit schema (consumed as a spec, per the positioning memo's "make the schema the product" pivot), the §15.4 JWT-mapping contract, the §17.4 differential-simulation algorithm, the §17B approval CRD pattern.
- **Duplicates / stubs initially:** a minimal control model (FINOS-AIGF as the only catalog, not full Gemara hierarchy), a minimal console (trust-gradient + session-timeline only).

### Trade-offs vs primary (extension) SPEC

| Axis | Primary (extension) | ALT-1 (separate product) |
|---|---|---|
| Time to first agent-governance revenue | Slow — gated by base MVP (Phase-3) | Fast — ships the agent slice first |
| Engineering reuse | Maximal (no new primitive) | Partial — some duplication/stubbing |
| Market timing (forming category) | Risks missing the window | Captures the window (Wedge-7 tailwind) |
| Coherence / single lineage graph | One graph, full traceability | Two products to reconcile later |
| Defensibility long-term | Connective-tissue spine (Stack C) | Point product, easier to displace |
| Risk if agent market stalls | Low (base platform stands alone) | High (bet the company on Wedge-7 the memo says "watch 6 months") |
| Buyer | Platform/compliance teams | AI-platform / agent teams (different buyer, faster cycle) |

### When ALT-1 wins
When the agent-governance market window is the dominant strategic concern and revenue-this-year matters more than long-term defensibility — i.e. the positioning memo's Stack C / Wedge-7 lead, accepting the "optional long bet" risk. The memo itself hedges ("watch it for six months"); ALT-1 is the un-hedged version.

### When the primary wins
When the strategic goal is the defensible "connective-tissue spine" (Stack A) and the base platform's other wedges (OPA successor, simulation, GRC evidence) are the real revenue — agents are then a high-value *extension* that strengthens the spine without a separate go-to-market. **The reframe doc's engineering argument (no refactor needed) favors the primary; the positioning memo's market-timing argument favors ALT-1. They are different axes — the right answer may be "build base-first (primary), but package and MARKET an agent-first slice (ALT-1 framing)."**

---

## ALT-2 — Behavioral tier as async out-of-band evaluator (not inline `require_async_check`)

### The bet
The primary SPEC places gating behavioral evaluators inline via `require_async_check` — the action waits for the evaluator before commit. ADVERSARIAL DEFECT-1/6 show this breaks the platform's deterministic-replay guarantee and adds fatal latency (a judge-model call per output). ALT-2: **never block the hot path on a non-deterministic model-call evaluator.** Instead:

- **Inline tier (deterministic, fast):** cheap, deterministic pre-filters run inline (regex/secret-scan/PII-detector/schema-validation/budget-counter). These keep the deterministic-replay guarantee and sub-second latency. They handle the clear-cut blocks.
- **Async evaluator tier (best-effort, out-of-band):** judge-model groundedness, drift, loop-detection, quality scoring run **after** the action is provisionally committed (or in a shadow lane), emitting scored events. Their outcomes feed **trust-grade adjustment, suspend-the-session, retro-flagging, and human-review queues** — not the inline allow/deny of the specific action.

### How enforcement still works without inline blocking
- For **reversible** actions (output to a user that can be recalled/redacted, a draft, a non-committed tool result), provisional-commit + async-evaluate + recall-on-fail is acceptable.
- For **irreversible** high-impact actions (financial transfer, prod data write), the **deterministic inline tier + the approval gate (DELTA-F pattern 2)** handle them — these never depend on a model-call evaluator; they depend on pattern-match + human approval.
- The async tier's job is **trend and trust**, not per-action veto: persistent groundedness failures re-tighten the trust grade (DELTA-E reverse gradient), which deterministically restricts future actions.

### Trade-offs vs primary (inline) SPEC

| Axis | Primary (inline `require_async_check`) | ALT-2 (async evaluator) |
|---|---|---|
| Deterministic-replay guarantee | Broken for gated outputs | Preserved (inline tier is deterministic; async is explicitly best-effort) |
| Latency on hot path | High (judge-model per output) | Low (only cheap deterministic checks inline) |
| Can it block a specific bad output before it's seen? | Yes (if you accept the latency) | Only via deterministic pre-filters; judge-model catches it after, via recall/trust |
| Cost on enforcement path | High (model call per gated action) | Lower (async can sample, batch, run cheaper) |
| Fit with reframe doc | Matches its "require_async_check" wording | Departs from it, but matches its own caveat that outputs are non-deterministic |
| Risk | UX death + guarantee erosion | A bad output may be briefly visible before recall |

### When ALT-2 wins
Almost always for interactive agents and for preserving the platform's core guarantee. The primary's inline framing is defensible only for non-interactive, low-throughput, irreversible-action agents where latency is acceptable and exact blocking is required — and even then the deterministic inline tier + approval gate cover most of it. **Recommendation: adopt ALT-2's two-tier split as the default; keep inline `require_async_check` as an opt-in for specific irreversible-and-must-block cases.**

---

## Combined recommendation

- **Architecture:** adopt **ALT-2** (two-tier: deterministic inline + best-effort async) — it resolves ADVERSARIAL DEFECT-1/5/6 and preserves the platform's defining guarantee. Fold this back into the primary SPEC's DELTA-C as the default.
- **Product/GTM:** keep the primary's **base-first engineering** (no refactor; F3 sequencing) but borrow **ALT-1's packaging** — market an agent-governance slice early even though it's built on shared platform libraries. The two memos disagree on the axis, not the build: engineering reuses, marketing wedges.
