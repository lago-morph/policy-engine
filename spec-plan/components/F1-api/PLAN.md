# F1 — API Requirements — PLAN

**Component:** F1 · **Source:** §21 · **Status:** Authored (domain-lead fallback)

---

## 1. Dependency DAG

```
[D1 JWT/OIDC] ─┐
[D2 storage authz predicate] ─┼─> [F1 authz middleware] ─┐
[§15.4 claim mapping] ────────┘                          │
                                                          v
[A1 controls/lineage] ─┐                          [F1 API gateway + envelope/pagination/errors]
[A2 lifecycle] ────────┤                                  │
[B1/B3/B4 policies] ───┼──> resource handlers ────────────┤
[C2 audit] ────────────┤                                  │
[C3 violations] ───────┤                                  ├─> [OpenAPI contract + SDKs]
[E1 simulate] ─────────┤                                  │
[C5 reports] ──────────┘                                  └─> [F1 async job subsystem]
[F2 plugins] ──> dynamic sub-resource registration
```

Critical path: **D1/D2 + §15.4 mapping → F1 authz middleware → resource handlers**. Everything else parallelizes behind the middleware + envelope contract.

## 2. Parallel workstreams

- **WS-A — Contract & framework (no backend deps):** envelope, pagination, error model (RFC 9457), versioning, OpenAPI skeleton, `/healthz|readyz|version`, `/whoami`. Buildable day 1 against stubs. **Unblocks everything.**
- **WS-B — AuthN/AuthZ middleware:** depends on D1 token validation + D2 scope predicate + §15.4 mapping. Highest-priority real dependency.
- **WS-C — Read resources (controls, policies, rego-packages, evidence, violations, audit):** parallel once WS-A+B land; each behind its component's read API.
- **WS-D — Async job subsystem (simulate, conftest, reports, export):** shared job lifecycle + queue; parallel to WS-C.
- **WS-E — Mutating/workflow (promote, approvals, exceptions, plugins):** depends on A2/D3/F2; later.
- **WS-F — Observability (audit emit, OTel, rate limit):** cross-cutting; lands incrementally.

## 3. Critical path

`D1+D2+mapping ready → WS-B middleware → first read endpoint (GET /controls) end-to-end with scope enforcement → audit emit (WS-F) → simulate async (WS-D)`. The MVP-8 (§21.1) are demo-complete once WS-A,B,C and the simulate/conftest job path (WS-D) are in.

## 4. Milestones

- **M1 (contract):** OpenAPI v1 skeleton + envelope + errors + health/version/whoami green against stubs.
- **M2 (authz):** middleware enforcing R-F1-AUTHZ-1..5 with D2; one read endpoint scope-correct; cross-tenant returns 404.
- **M3 (MVP-8):** all §21.1 endpoints live; simulate/conftest async; audit emit on mutations.
- **M4 (workflow):** promote/approvals/exceptions; lineage read; reports.
- **M5 (extensibility):** plugin registry + dynamic sub-resources (joins F2).

## 5. What can be built concurrently / what blocks what

- Concurrent: WS-A with WS-C handler stubs; all read handlers with each other; job subsystem with read handlers.
- Blocks: WS-B blocks any real (non-stub) data return because scope enforcement is mandatory (can't ship read endpoints that leak). WS-E blocked by A2/D3/F2.
- Cross-domain: F1 is a **consumer** of A–E read/write APIs and **the** integration point the console (E2) calls — coordinate envelope/pagination with E2 early.

## 6. Test strategy

- **Contract tests:** OpenAPI schema validation in CI; every response validated against schema.
- **AuthZ matrix tests (highest value):** for each endpoint × role × scope, assert allowed/denied AND that out-of-scope objects are absent from list results (not merely hidden in UI). Negative tests for scope-widening via filter/cursor.
- **Scope-enumeration tests:** 404 vs 403 discipline; cursor cannot be replayed cross-scope.
- **Async job tests:** queued→running→succeeded/partial/failed/cancelled; idempotency-key replay; `partial` + `replay_completeness` propagation.
- **Failure injection:** backing component down → 503; missing required claim → 403 + JWT-drift audit event (ties to C3 §14.2).
- **Pagination fuzz:** bounded windows on audit; default replay window applied when absent.
- **Load (POC-sized):** 100–1000 evals/sec is engine-side, not API-side; API load test targets 5–50 concurrent console users + job submission bursts.

## 7. Risks

- API becomes a second, drifting authz implementation vs D2 → mitigate by sharing the predicate library, not reimplementing.
- Audit `/events` unbounded queries → enforce bounded windows (R-F1-FILTER-1).
- F4 agent resources balloon the surface → keep them as additive sub-collections under the same envelope; no v2 needed.
