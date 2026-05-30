# F2 — Deployment & Extensibility — ADVERSARIAL REVIEW

**Reviewer persona:** Red-team SRE + supply-chain security + plugin-ecosystem skeptic.

---

## 1. Headline finding

The deployment model is competent but **inherits the platform's deepest contradiction: §26.1 declares storage "out of scope for the POC," yet F2 requires every CRD and plugin to carry §17A.5 scope metadata and the platform requires storage-level authz (§17A.1).** F2 hand-waves "BYO ordinary storage" while the entire authorization and audit-integrity story depends on storage that can enforce scope and tamper-evidence (§23). Shipping on a store that can't do row/field-level scope reproduces F1 DEFECT-1 at the infrastructure layer. **DEFECT-1 (critical).**

## 2. Defect list (prioritized)

**DEFECT-1 (critical) — "BYO ordinary storage" undercuts authz + evidence integrity.** §22.2 says ordinary relational/document/object storage is acceptable; §23 says evidence must be tamper-evident/versioned and authz must be storage-enforced. A naive Postgres/S3 install satisfies §22.2 but not §23 unless F2 mandates scope columns + append-only/versioned audit + integrity hashes. F2 must specify the **minimum storage contract** (scope predicate support, append-only audit, content hashing), not just "BYO."

**DEFECT-2 (high) — failurePolicy default is a foot-gun stated as a feature.** "Fail-closed for high-assurance, fail-open otherwise, configurable" sounds responsible but means a degraded webhook silently fails OPEN for most namespaces — exactly when an attacker would DoS the webhook to slip a deployment through. And §14.2's bypass detection is *retrospective*, so the gap is real-time. Need: explicit per-namespace declaration, a default that errs closed for any namespace mapped to a control with `enforcement: required`, and a webhook-health SLO that pages.

**DEFECT-3 (high) — Out-of-process plugins are isolated from crashes but NOT from data exfiltration.** R-F2-PLG-2 isolates faults, but a malicious evidence-collector or export-adapter plugin sees audit events (which F4 says contain prompts/context/PII). Signing (R-F2-PLG-4) proves provenance, not trustworthiness. Need: per-plugin scope grants (a plugin only receives events within its declared scope), egress controls on export adapters, and data-classification-aware redaction before a plugin sees an event.

**DEFECT-4 (high) — CRD ownership collision with B4.** §17C.6 already defines `PolicyApprovalRequest`, `PolicySimulationRun`, `PolicyActionLibrary`, `PolicyEvidenceSchema`, `PolicyException`, `PolicyRemediationAction`. F2 re-lists them AND adds `GovernanceBundle`/`PolicyEnginePlugin`/`ExportAdapter`. Two domains (B4 and F2) both claim the CRD surface. Without a single owner, you get duplicate/divergent CRDs. Must be reconciled at cross-cut; nominate one component as CRD-schema owner.

**DEFECT-5 (medium) — Pull-based spoke caching makes "current policy" ambiguous.** Spokes enforce from the last cached signed bundle when the hub link drops (good for availability) but this means a promoted/demoted policy (incl. F4 behavioral re-tighten) does NOT take effect until the spoke re-pulls. A trust re-tightening that the platform believes is active may not be. Need bundle TTL + staleness SLO + "policy not yet effective on spoke X" surfaced to the console.

**DEFECT-6 (medium) — Capacity math ignores simulation burst and replay materialization cost.** §3.1 sizes audit at 30–75 GB, fine. But §17A.5 requires replay datasets to be *materialized as scoped subsets before use*; 50 concurrent analysts each materializing a 30-day namespace replay multiplies storage + I/O well beyond the steady-state number. Worker pool of 2 (R-F2-SCALE-4) will queue badly. Size the materialization tier and cache scoped datasets.

**DEFECT-7 (medium) — Operator is a single blast radius.** One operator reconciling all CRDs + install + upgrade is a single point of compromise; a bug in upgrade reconcile can take down enforcement across all spokes. POC-acceptable, but flag the split-controllers path as a real requirement before GA, and ensure the operator cannot itself disable enforcement without an audit + approval.

**DEFECT-8 (low) — Wasm dismissed too quickly.** §26.1 says Wasm non-required, and F2 defaults out-of-process. Fine, but the positioning memo's "policy-impact check on PRs" and browser/local simulation use cases (§26.1) are exactly where Wasm earns value; don't architect it out — keep the in-process SPI path open.

**DEFECT-9 (low) — No mention of secrets/credential management for spoke enrollment.** "Register cluster credentials in `GovernancePlatform`" stores spoke kubeconfigs/tokens centrally — a juicy target. Needs a secret-ref model (External Secrets/SPIFFE), not inline creds.

## 3. Inconsistencies vs other components

- **vs F1:** both depend on a storage scope-predicate that §26.1 defers — same root contradiction.
- **vs B4:** CRD ownership overlap (DEFECT-4).
- **vs C2/C4:** real-time fail-open relies on C4 retrospective detection to catch bypass — acceptable only if C4 is in MVP, which F3 must confirm.
- **vs F4:** agent PDPs as plugins is elegant, but agent events carry the most sensitive payloads — DEFECT-3 (plugin data exfil) is most acute for F4.

## 4. "Won't survive production because…"

…the platform's safety guarantees (scope-enforced authz, tamper-evident evidence, real-time enforcement) are quietly delegated to "BYO ordinary storage" and a "configurable failurePolicy," so the first production incident is a fail-open webhook or an unscoped query on a store that never enforced scope — and the audit trail that would prove it sits in the same under-specified store.

## 5. Top fixes to merge into SPEC

1. Define a **minimum storage contract** (scope predicate, append-only/versioned audit, content hashing) — close DEFECT-1.
2. failurePolicy: default-closed for `enforcement: required` scopes + webhook-health SLO that pages (DEFECT-2).
3. Per-plugin scope grants + egress control + pre-plugin redaction (DEFECT-3).
4. Single CRD-schema owner across B4/F2 (DEFECT-4).
5. Bundle staleness SLO + "policy-not-yet-effective" surfacing (DEFECT-5); size the replay-materialization tier (DEFECT-6).
