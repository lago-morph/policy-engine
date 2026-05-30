# Domain F — Platform & Cross-cutting — DOMAIN ADVERSARIAL (reconciliation)

**Date:** 2026-05-30 · Reconciles the per-component adversarial findings (F1, F2, F3, F4) and surfaces contradictions *between* F components.

---

## 1. The one finding that dominates the domain

**Every F component independently hits the same root contradiction: the spec demands storage-enforced authorization and tamper-evident evidence (§17A.1, §23) while deferring the storage layer entirely (§26.1 "storage implementation is out of scope for the POC") and permitting "ordinary storage" (§22.2).**

- F1 DEFECT-1: the API has no enforcement substrate to push its scope predicate into.
- F2 DEFECT-1: "BYO ordinary storage" undercuts scope-authz + evidence integrity.
- F3 DEFECT-2: the MVP cut line violates its own rule "no MVP item relies on a deferred item" (D2 needs storage; storage is deferred).
- F4 DEFECT-2: agent `request_object` (prompts/context/PII) is the most sensitive payload yet and lands in this same under-specified store.

**Domain resolution (must be ratified at cross-cut with Domain D):** Storage is *partially in MVP*. D2 ships a scope-predicate library (row AND field level); F2 defines a minimum storage contract (scope columns, append-only/versioned audit, content hashing). "Ordinary storage acceptable" is reinterpreted as "ordinary storage *that supports the minimum contract*," not "any storage." This is the single highest-priority cross-domain escalation from Domain F.

---

## 2. Contradictions BETWEEN F components

**C-1 (CRD ownership): F2 ↔ B4 (and within F, F2 ↔ F1).** F2 enumerates the §17C.6 CRDs *plus* three new ones; B4 (§17C) also owns CRDs; F1 exposes them as API sub-resources. Three owners, one surface. **Resolution:** one CRD-schema owner (recommend B4 owns schema, F2 owns controllers+operator, F1 owns the REST projection). Flagged in F2 DEFECT-4 and F3 DEFECT (CRD collision).

**C-2 (authz double-implementation): F1 ↔ F2 ↔ D2.** F1 enforces scope at the API; D2 at storage; F2 puts scope on every CRD/controller. Three enforcement points risk three drifting predicates. **Resolution:** a single shared scope-predicate library (D2-owned) linked by F1, F2 controllers, and the storage layer — never reimplemented.

**C-3 (failurePolicy vs retrospective detection): F2 ↔ F3/C-domain.** F2 defaults some namespaces fail-open and relies on C3/C4 retrospective bypass detection (§14.2) to catch the gap. But F3 puts only *2 detections* in MVP and defers C4. So the real-time gap (fail-open) is backstopped by a retrospective capability that is partly deferred. **Resolution:** if any MVP namespace is fail-open, its bypass detection (C3 Gatekeeper-bypass example) MUST be in MVP; otherwise default that namespace fail-closed.

**C-4 (async/extensible schema dependency): F1 ↔ F4.** F4's audit deltas (DELTA-D) require C2's schema to be explicitly extensible (F1 DEFECT-8) AND to support field-level scope (F1 DEFECT-2). F1 raised both as defects; F4 *depends* on both being resolved. **Resolution:** C2 must declare the audit schema open/extensible with field-level scope before F4 design — sequencing dependency, not just a defect.

**C-5 (replay-completeness consistency): F1 ↔ F4 ↔ E1.** F1's `partial` job results (DEFECT-6), F4's "exact-output replay is best-effort" (R-F4-AUD-2), and E1's simulation completeness model must use ONE consistent vocabulary, or the console shows contradictory authority claims. **Resolution:** a single `replay_completeness` semantics (complete/partial/best-effort) shared across F1 jobs, E1 sims, and C2 events; partial/best-effort results MUST be watermarked and non-exportable as authoritative evidence.

**C-6 (base-first vs market-first): F3 ↔ F4.** F3 sequences base-first; F4's ALT-1 argues the agent market window. **Resolution (not a contradiction once disentangled):** engineering = base-first (reuse); GTM = may wedge agent-first (packaging). Two axes; state both. Already reconciled in F4 ALT and F3 DEFECT-5.

---

## 3. Severity-ranked consolidated defect list (domain view)

| Rank | Defect | Components | Severity | Resolution owner |
|---|---|---|---|---|
| 1 | Storage-authz/evidence vs deferred storage | F1, F2, F3, F4 | Critical | Cross-cut + Domain D |
| 2 | Behavioral evaluators break deterministic-replay guarantee + latency | F4 | Critical | F4 (adopt ALT-2 two-tier) |
| 3 | Agent context = privacy/scope explosion in deferred store | F4 (+F1,F2) | Critical | F4 + C2 field-level scope + retention |
| 4 | Trust-gradient auto-relaxation rewards a patient adversary | F4 | High | F4 (mandatory human approval for privilege increase) |
| 5 | failurePolicy fail-open vs partly-deferred bypass detection | F2, F3 | High | F2 + F3 (fail-closed unless C3 detection in MVP) |
| 6 | CRD ownership collision | F2, F1, (B4) | High | Cross-cut (single CRD owner) |
| 7 | Authz double/triple-implementation drift | F1, F2, D2 | High | Shared predicate library |
| 8 | Runtime-trusted vs choke-point-enforced controls conflated | F4 | High | F4 (label each control) |
| 9 | MVP "thin slices" hide the two hardest builds (D2, E1) | F3 | High | F3 (re-label MVP-core high-effort) |
| 10 | Replay-completeness vocabulary inconsistency | F1, F4, E1 | Medium | Shared semantics + watermarking |
| 11 | Capacity math ignores concurrency/replay-materialization burst | F1, F2, F3 | Medium | F2 (size materialization tier) |
| 12 | Contract-freeze-in-1-week optimism | F3 | Medium | F3 (v0 + budgeted re-freeze) |
| 13 | Plugin data-exfiltration despite fault isolation | F2 (+F4) | Medium | F2 (per-plugin scope grants + egress control) |
| 14 | Single operator blast radius; spoke kubeconfig storage | F2 | Medium/Low | F2 (split controllers later; secret-ref) |
| 15 | Standards pinned to immature MCP/OTel-GenAI/FINOS/NIST | F4 | Medium | F4 (optional compatibility targets + non-MCP fallback) |

---

## 4. What Domain F escalates to the cross-cut wave

1. **Storage minimum contract** (rank 1) — needs Domain D + a platform-wide decision; blocks F1/F2/F3/F4.
2. **Single CRD-schema owner** across B4/F2/F1 (rank 6).
3. **Shared scope-predicate library** and **shared replay-completeness semantics** (ranks 7, 10) — platform-wide contracts.
4. **Behavioral-tier guarantee boundary** (ranks 2, 3) — the platform's "deterministic replay" claim must be explicitly scoped to exclude best-effort agent-output replay, in both product and marketing.
5. **Base-first engineering / wedge-first GTM** stated as two axes (rank — C-6) so the cross-cut MASTER-PLAN-ALT can carry the wedge-first variant.

These belong in `cross-cutting/CROSSCUT-ADVERSARIAL.md` and `DATA-MODEL.md`; Domain F has surfaced them but cannot resolve them alone.
