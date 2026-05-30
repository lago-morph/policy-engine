# Orchestration Plan — Detailed Spec + Plan for Every Piece

**Status:** ACTIVE · **Date:** 2026-05-30 · **Branch:** `claude/spec-plan-review-parallel-cDAcL`

This document is the meta-plan: *how* we produce an incredibly detailed spec and
implementation plan for every piece of the policy-engine design corpus, exploiting
parallelism to the maximum, with cooperative **and** adversarial scrutiny at every
step, and alternative-architecture trees on high-value pieces.

It is written first and committed before any agent is dispatched, because the
sandbox filesystem is ephemeral — only committed+pushed work survives.

---

## 1. The agent hierarchy (locked)

```
Primary orchestrator (this session)
  ├─ decomposes the corpus into work units
  ├─ dispatches domain-lead parents (parallel, background)
  ├─ commits after every wave (ephemeral sandbox)
  └─ writes the master index + ships the PR

  Domain-lead parents  (6, one per domain)  →  make indices + summaries
    ├─ read the relevant spec sections + companion docs
    ├─ spawn author subagents (parallel) per component
    ├─ spawn adversarial reviewer subagents (parallel) per component
    ├─ spawn alt-architecture subagents on high-value components
    └─ write DOMAIN-INDEX.md + DOMAIN-SUMMARY.md + reconciliation

    Author subagents (cooperative persona)   →  write SPEC.md + PLAN.md
    Adversarial subagents (red-team persona)  →  write ADVERSARIAL-REVIEW.md
    Alt-architecture subagents                →  write ALT-<name>.md
```

**Rationale.** The user specified: subagents create docs, parent agents make indices
and summaries, primary agent is orchestrator. Three levels maximize parallelism and
match that hierarchy exactly. Domain leads push the fan-out down so the primary only
directly coordinates 6 parents + a cross-cut wave.

**Robustness rule for every domain lead:** every component MUST end with at least
`SPEC.md` + `PLAN.md` on disk. If a spawned subagent fails or stalls, the lead authors
the doc itself rather than leaving a gap. Subagents write files incrementally so
partial progress survives. The primary commits after each wave.

---

## 2. Work-unit decomposition — 23 components in 6 domains

| Domain | Component | Source §/doc |
|---|---|---|
| **A · Governance Core** | A1 Governance Model & Gemara hierarchy | §6 |
| | A2 Policy Lifecycle (author→simulate→promote) | §7 |
| **B · Policy Engines & Enforcement** | B1 OPA/Rego integration & signed bundles | §8 |
| | B2 Gatekeeper integration | §9 |
| | B3 Conftest integration | §10 |
| | B4 Engine selection, action taxonomy & CRDs | §17C |
| | B5 Real-time enforcement flow | §18 |
| **C · Evidence, Audit & Analytics** | C1 Privateer integration | §11 |
| | C2 Audit schema framework & standardized event schema | §12–13 |
| | C3 Compliance analytics engine | §14 |
| | C4 Retrospective audit detection | §19 |
| | C5 Reporting | §17E |
| **D · Identity, Authz & Security** | D1 Keycloak/JWT integration & mapping layer | §15 |
| | D2 Scoped roles, permissions & storage authorization | §17A |
| | D3 Approval-gated decisions | §17B |
| | D4 Security requirements | §23 |
| **E · Simulation & Console** | E1 Policy simulation & dry-run framework | §17 |
| | E2 Governance console / Headlamp GUI | §16 |
| | E3 Per-product PDP libraries | §17D |
| **F · Platform & Cross-cutting** | F1 API requirements | §21 |
| | F2 Deployment & extensibility | §24–25 |
| | F3 POC scale, MVP scope & sequencing | §22, §26, §27 |
| | F4 AI / agent governance extension | reframed-for-ai.md |

**High-value components getting an ALT (alternative-architecture) tree:**
A1, B4, C2, D2, E1, F4. (Plus a cross-cut alternative overall-sequencing plan in F3/cross-cutting.)

---

## 3. Per-component deliverables (the doc contract)

Each `spec-plan/components/<id>-<slug>/` directory ends with:

- **`SPEC.md`** — exhaustive engineering spec: scope, data model/entities, interfaces &
  APIs, normative requirements (numbered, MUST/SHOULD), schemas, state machines,
  failure modes, security/authz notes, dependencies on other components, open questions
  with a *decided* default + rationale.
- **`PLAN.md`** — implementation plan that itself exploits parallelism: a dependency DAG,
  parallelizable workstreams, critical path, milestones, test strategy, and an
  explicit "what can be built concurrently / what blocks what" section.
- **`ADVERSARIAL-REVIEW.md`** — red-team pass: attack the spec's assumptions, find gaps,
  inconsistencies vs. other components, unhandled failure/abuse cases, scope creep,
  and "this will not survive contact with production because…" findings. Ends with a
  prioritized defect list.
- **`ALT-*.md`** *(high-value only)* — a genuinely different architecture for the same
  requirement, with trade-off analysis vs. the primary spec.

## 4. Per-domain deliverables (the index contract)

Each `spec-plan/domains/<domain>/`:

- **`DOMAIN-INDEX.md`** — table of components, one-line purpose, file links, status,
  cross-references into spec §-numbers and into scenarios (HL/DT) that exercise them.
- **`DOMAIN-SUMMARY.md`** — the domain in ~1–2 pages: shared data model, internal
  dependencies, the 3–5 hardest decisions, and the consolidated open-questions list.
- **`DOMAIN-ADVERSARIAL.md`** — domain-level reconciliation of the per-component
  adversarial findings; contradictions *between* components in the domain.

## 5. Cross-cutting wave (after domains complete)

`spec-plan/cross-cutting/`:

- **`MASTER-PLAN.md`** — whole-platform implementation plan as a dependency DAG across
  all 23 components; identifies independent workstreams buildable in parallel, the
  critical path, and a phased (MVP→GA) sequence. Maximizes parallelism.
- **`MASTER-PLAN-ALT.md`** — an alternative overall sequencing (e.g. wedge-first per the
  positioning memo vs. platform-first).
- **`CROSSCUT-ADVERSARIAL.md`** — contradictions *across* domains (audit schema vs.
  analytics vs. AI extension vs. storage authz, etc.).
- **`DATA-MODEL.md`** — unified entity/relationship model consolidated from all components.
- **`TRACEABILITY.md`** — matrix: component ↔ spec § ↔ personas ↔ scenarios (HL/DT).

## 6. Top level

- **`00-MASTER-INDEX.md`** — index of everything produced; the entry point.
- **`DECISIONS.md`** — running log of every decision made unattended, with rationale.

---

## 7. Execution waves

- **Wave 0 (done):** inventory + decomposition + this plan. Commit.
- **Wave 1:** dispatch 6 domain-lead parents in parallel (background). Each fans out
  author + adversarial (+ alt) subagents. Commit on completion of each.
- **Wave 2:** cross-cutting agents (master plan, alt plan, cross-cut adversarial, data
  model, traceability). Commit.
- **Wave 3:** primary writes master index, runs a consistency pass, updates top-level
  INDEX.md, pushes, opens PR.
- **Wave 4 (added):** meta-review — C2 `v1.0-rc` reconciled schema + build-blocking
  checklist, independent meta-adversarial second opinion, thesis devil's-advocate.
- **Wave 5 (added) — Domain G, Operational/NFR architecture:** the meta-adversarial review
  found no component owned the non-functional/operational layer. Added **Domain G** with 8
  components (G1 Scale/Performance, G2 Cost/Retention Economics, G3 Availability/DR/Resilience,
  G4 Key Management, G5 Multi-Tenancy Isolation, G6 Observability/Day-2 Ops, G7 Data
  Lifecycle/Retention/Privacy, G8 Rego-Authoring/Human-Factors), same doc contract
  (SPEC/PLAN/ADVERSARIAL + ALT on G1/G3/G4/G5), dispatched as 8 parallel author agents,
  followed by a domain index + NFR cross-cut adversarial + an NFR devil's-advocate
  (over/under-engineered for the POC?).

## 8. Decisions made unattended (see DECISIONS.md for the live log)

- D-001: `.claude/skills/**` and `retrospective/**` are tooling/meta, **excluded** from
  the components to be spec'd (they are how we work, not the product). Documented, not lost.
- D-002: 100 scenario files are **inputs/traceability targets**, not separately re-spec'd;
  they are linked from `TRACEABILITY.md` and each component's SPEC.
- D-003: Three-level agent hierarchy chosen over two-level for parallelism + to match the
  requested orchestrator/parent/subagent structure.
- D-004: Intermediate docs are **kept and indexed**, never consolidated away.
