# G6 — Observability, SLOs & Day-2 Operations — ADVERSARIAL REVIEW

**Component:** G6 · **Date:** 2026-05-30 · **Reviewer role:** red team, hostile to the G6 SPEC/PLAN. First-contact skeptic. Mandate: break the day-2 story, especially the three the brief named — (1) the epoch-migration claim, (2) the 14-service un-runnability for Jess, (3) the unowned "who watches the watchmen."
**One-line verdict:** The epoch-migration mechanism is **correct and is the strongest part of the component** — but the SPEC *oversells* what it solves; the un-runnability risk is **real, partially mitigated, and not fully dischargeable by G6 alone**; and there are several places where G6 quietly assumes other components (C2, G3, G4, C3) deliver things that are themselves unowned or unbuilt. Prioritized defects below.

---

## Prioritized defect list

### G6-D1 (CRITICAL) — The epoch boundary makes *new* history tamper-evident, but it does **not** make the migration itself trustworthy unless the sealing checkpoint was *already* clean — and nothing guarantees it was.

**Attack.** R-G6-RUN-7 step 1 says "take a final signed checkpoint over the complete v1.0 chain." But what if the v1.0 chain *already* had a silent gap or a broken link before the migration (because the chain-integrity monitor, R-G6-OBS-9, was added by *this very component* and may not have existed during v1.0's life)? Then the epoch seal cryptographically blesses a **broken** chain as the eternal Epoch-0 root. The migration doesn't *create* tampering — but it **launders** a pre-existing integrity defect into a signed, immutable, "verified" artifact. R-G6-RUN-10's pre-flight ("full chain verifies clean") is a SHOULD, not a MUST.

**Severity:** Critical — it inverts the product's core promise (a signed root that attests to corrupt history is worse than no root, because it confers false confidence).
**Fix:** Promote R-G6-RUN-10's "chain verifies clean before sealing" from SHOULD to **MUST**, and make the seal *refuse* if any gap/break exists — forcing the gap to be investigated and explicitly recorded (as a known evidence-gap, IR-2) *before* Epoch 0 is sealed. You may not seal over a break. **Also:** this exposes that the chain-integrity monitor must have existed *for the whole life of Epoch 0*, which it didn't — so the very first epoch seal can only honestly attest "verified clean *as of the seal*," not "was never tampered." The SPEC should state this limitation honestly rather than implying the seal proves lifelong integrity.

### G6-D2 (CRITICAL) — "Jess can run it" is asserted by a ≤2-pages/shift *budget*, but a budget is a target, not a mechanism; nothing in G6 *causes* the 14-service stack to actually generate ≤2 pages/shift.

**Attack.** META Risk #10 is "14 services is un-runnable by a 2–4 person team." G6's answer is R-G6-ONCALL-1: "≤2 actionable pages per shift; more is an alerting bug." That is **defining the problem away**, not solving it. A stack with 14 services, 9 CRDs, conversion webhooks, a hash chain, a signing key, an IdP, and admission webhooks inline in customer deploys has an *intrinsic* incident surface. Declaring that it "must" page ≤2/shift doesn't make the underlying failure rate low; it risks **under-alerting** (suppressing real pages to hit the budget) — which for an *evidence* product means a silently-broken chain that nobody got paged for. The budget can be met by the dangerous failure mode (don't page) as easily as the safe one (be reliable).

**Severity:** Critical — this is the buying-decision risk the whole component exists to address, and the SPEC's answer is partly circular.
**Fix:** The page budget must be backed by (a) genuine reliability engineering (HA, the buffer-don't-drop discipline, graceful degradation) that *lowers the actual incident rate*, and (b) a guardrail that the budget is met by *fixing causes*, not *muting alerts* — e.g. the chain-integrity and ingest-drop alerts are **exempt from the budget** (they always page, even if that blows the budget) precisely so the budget can never be met by silencing them. The honest conclusion, which the SPEC half-makes via DEC-G6-2: for sub-floor teams, **managed is the real answer**, and self-hosted-by-a-2-person-team should carry an explicit "not recommended below N SREs" warning rather than a budget that pretends it's fine.

### G6-D3 (HIGH) — The epoch migration assumes a *clean cutover instant*, but the stack is distributed (hub + up-to-5 spokes, per-source chains) — there is no single global "moment" to seal.

**Attack.** R-G6-RUN-7 describes sealing Epoch 0 and opening Epoch 1 as if there's one chain. But C2 chains are **per-source** (N-C2-301), and the platform is **hub-and-spoke with spokes that buffer locally and ship asynchronously** (F2 failure modes). At the instant the hub seals Epoch 0, a spoke may still be holding buffered v1.0 events that haven't arrived. If they arrive *after* the seal, do they go into the sealed Epoch 0 (violating the seal) or into Epoch 1 (so v1.0-schema events live in the 1.0-rc epoch — a schema/epoch mismatch the verifier in RUN-9 doesn't handle)?

**Severity:** High — the mechanism is correct for one chain but underspecified for the real distributed, buffered topology, which is exactly where it'll be used.
**Fix:** Seal is **per-source-chain**, not global; each source seals its own Epoch 0 only after its buffer is drained and acknowledged, then opens its own Epoch 1. The transition is a **rolling per-source migration**, not a big-bang. The verifier (RUN-9) must tolerate sources being in different epochs simultaneously during the roll. Add R-G6-RUN-7a covering per-source rolling seal + late-arriving-event handling (late v1.0 events from a not-yet-sealed source append to that source's Epoch 0 normally; a source can't be sealed until its upstream buffer is confirmed empty).

### G6-D4 (HIGH) — G6 hands off DR, keys, capacity, and tenant-isolation to G3/G4/G1/G5, but **those components are unbuilt** (empty dirs at authoring time) — G6's runbooks reference contracts that don't exist yet.

**Attack.** §7 dependencies lean hard on G3 (buffer/DR/RPO/RTO), G4 (key custody/rotation crypto), G1 (capacity numbers), G5 (tenant boundary). At authoring time these are empty directories. So the cert/secret/key-rotation runbook (§4.4) "hands off to G4" for the part that actually matters (what happens after a key *compromise*), and the incident runbook (§4.7) "hands off to G3" for recovery — but if G3/G4 don't specify those, G6's runbooks have **dangling pointers at exactly the highest-severity steps**. This is the same "the flag sits in the index but the load-bearing doc still says X" propagation failure META-ADVERSARIAL M-1 called out, one level down.

**Severity:** High — a runbook whose most critical branch is "see G4" is not an executable runbook if G4 is silent.
**Fix:** G6 must specify the **hand-off contract as a MUST-be-satisfied interface**, not a pointer — i.e. "G4 MUST provide: a key-compromise recovery procedure that re-establishes trust without rewriting history; if absent, G6's IR-2 cannot complete." Make the dependency a *named requirement on the other component* so it surfaces in cross-cut reconciliation rather than silently failing. (This review explicitly flags it for the domain-lead/cross-cut wave.)

### G6-D5 (HIGH) — "Telemetry MUST NOT carry customer audit content" (OBS-2) collides with the SRE's actual need to debug a specific event's slow replay.

**Attack.** R-G6-OBS-2 forbids jwt_claims, request_object, before/after, external-data values, and verdicts in telemetry — telemetry may reference events by opaque id only. Good for privacy/integrity. But when Jess is paged for "replay job X is slow," the trace (OBS-7) gives her an `event_id` and nothing else; to actually debug she must cross from the mutable telemetry plane into the C2 evidence plane — which is **scope-controlled and (per G5/D2) tenant-isolated**. So the on-call SRE either can't see the data she needs (debugging blind), or G6 has quietly created a path for an SRE to read customer evidence content "for debugging" — which is the cross-tenant-disclosure surface G5 exists to prevent.

**Severity:** High — an unresolved tension between operability and the integrity/privacy firewall; whichever way it's resolved has a cost the SPEC doesn't acknowledge.
**Fix:** Make the boundary-crossing explicit and *governed*: an SRE looking up event content from a telemetry id is a **scoped, audited access** (it emits its own C2 event — "operator viewed evidence for debugging"), bounded by G5 tenancy, and ideally on *metadata only* (latency, sizes, completeness label) rather than content. State that on-call debugging operates on the **honesty/fidelity labels and timings**, not the payload, by default — and content access is break-glass.

### G6-D6 (MEDIUM) — The admission-hot-path SLO (99.9% / p99<250ms) may be unmeetable *because of the platform's own design*, and G6 doesn't reconcile with the things that make admission slow.

**Attack.** p99 < 250 ms added webhook latency assumes the eval is fast. But the platform's value-add — capturing external-data *values* at decision time (C2-rc N-C2-EDV, the cosign-verifier call), computing jwt_claims completeness, emitting the rich audit event synchronously — adds latency *on the admission path*. A cosign/registry round-trip alone can exceed 250 ms. So the SLO and the evidence-completeness requirement (which the whole product depends on) are in tension, and G6 sets the SLO without acknowledging that C2-rc's own MUSTs may blow it.

**Severity:** Medium — a cross-component contradiction that surfaces as an SLO that can't be met without weakening the evidence model, or an evidence model that can't be honored without blowing the SLO.
**Fix:** Split the SLI: the *decision* (allow/deny) must be < 250 ms; the *evidence enrichment* (value capture, completeness scoring) MAY be **asynchronous/out-of-band** (decide fast, enrich the audit event in the ingest path) — as long as the enrichment is lossless (ties back to the ingest SLO). Flag for B5/C2: external-data value capture should not be a synchronous admission-blocking call where avoidable.

### G6-D7 (MEDIUM) — The "every day-2 op emits a governed C2 event" rule (RUN-0) feeds operational noise into the evidence chain — contradicting OBS-3.

**Attack.** R-G6-OBS-3 says "platform operational metrics MUST NOT be written into the C2 chain." R-G6-RUN-0 says "every enforcement/chain/key-touching op emits a C2 event." These are close to colliding: an upgrade, a scale event, a cert rotation — are those "operational noise" (forbidden in the chain) or "governed actions" (required in the chain)? The line between "the operator scaled a worker pool" (noise) and "the operator invoked break-glass to bypass a control" (evidence) is real but the SPEC doesn't draw it.

**Severity:** Medium — an internal inconsistency that, if resolved the wrong way, either pollutes the chain (inflating G2 retention cost) or fails to audit a genuine governance bypass.
**Fix:** Draw the line explicitly: **only operations that change *enforcement behavior or evidence integrity*** (break-glass, self-exemption, key rotation, epoch transition, a policy/bundle change) are governed C2 events; **routine ops** (scaling, replica restarts, cert renewals that don't change trust) are platform telemetry only. RUN-0 should scope "governed" to the integrity/enforcement-affecting subset, not literally every op.

### G6-D8 (MEDIUM) — Managed-tier "customer-held key, vendor-run compute" (DEC-G6-2) is asserted as squaring the circle, but a vendor running the signing *process* with a customer-held *key* still sees plaintext evidence in memory.

**Attack.** DEC-G6-2 claims the managed tier preserves auditor-independence because the customer holds the key and the evidence store. But to *sign checkpoints*, the signing process (vendor-run compute) must have access to the key material (or an HSM/KMS the vendor's process can call) and to the plaintext events. So the vendor's compute does see the evidence and does invoke the key — "customer-held key" is doing less work than the SPEC implies unless it means a customer-controlled KMS/HSM that the vendor process calls but can't exfiltrate. The integrity claim is weaker than stated.

**Severity:** Medium — overstates how cleanly the managed tier preserves the integrity boundary.
**Fix:** Specify the mechanism precisely: managed tier requires the signing operation to run against a **customer-controlled KMS/HSM** (vendor process can request signatures but cannot extract the key), and evidence to be stored in **customer-owned storage** the vendor writes to but doesn't custody. Acknowledge residual trust (vendor's process touches plaintext in memory) as a real, bounded surface, not zero. Hand the crypto detail to G4.

### G6-D9 (LOW) — 14-service golden-signal table is a snapshot; the plugin model (F2 §5) means the service count is *unbounded*, and plugins have no golden-signal contract.

**Attack.** §2.3 enumerates 14 services. But F2's whole extensibility model lets customers add PDP plugins, evidence collectors, export adapters — each a new out-of-process service. None of them appears in the golden-signal table, and R-G6-OBS-6 ("every service emits its row") has no teeth for third-party plugins. So the observability story is complete for the shipped stack and **silent for everything the customer plugs in** — which is most of the long-tail operational surface.

**Severity:** Low (for POC; rises with adoption).
**Fix:** Extend R-F2-PLG-1's capability descriptor to **require a golden-signal `/metrics` contract** as a load condition — a plugin that doesn't emit the four signals is refused or marked `observability: none`. Make observability a first-class plugin SPI obligation, mirroring how F2 already requires plugins to emit C2 events (R-F2-PLG-3).

### G6-D10 (LOW) — No owner for *G6's own* observability (the monitoring stack) being down.

**Attack.** R-G6-OBS-5 correctly says self-monitoring must run outside the platform's control loop. But then: who watches the *monitoring*? If Prometheus/the OTLP egress is down, no alert fires (the thing that fires alerts is down) and the platform looks healthy while blind. The watchmen-watching-themselves recursion the component is named for isn't fully terminated — it's pushed one level out.

**Severity:** Low — but it's the literal "who watches the watchmen" the component title promises to answer, left one turtle short.
**Fix:** A dead-man's-switch / heartbeat: the monitoring stack emits a continuous heartbeat to an **external** (off-platform, e.g. a SaaS uptime monitor or the customer's existing observability) endpoint; absence of heartbeat = the monitoring is down = page. Terminate the recursion at a deliberately-external, dumb, independent watcher rather than pretending it's turtles-free.

---

## Summary table

| ID | Sev | One-line | Disposition |
|---|---|---|---|
| G6-D1 | CRITICAL | Epoch seal can launder a pre-existing chain break into a signed "verified" root | Promote RUN-10 clean-chain pre-flight to MUST; seal refuses over a gap |
| G6-D2 | CRITICAL | ≤2-pages/shift budget defines un-runnability away; can be met by under-alerting | Exempt integrity alerts from budget; honest "managed for sub-floor teams" |
| G6-D3 | HIGH | Epoch seal assumes one global instant; topology is per-source + buffered spokes | Per-source rolling seal; verifier tolerates mixed epochs |
| G6-D4 | HIGH | Hand-offs to G3/G4/G1/G5 dangle — those components unbuilt | Turn pointers into named MUST-requirements on the other components |
| G6-D5 | HIGH | Telemetry no-content rule vs SRE's need to debug a specific event | Boundary-crossing is scoped+audited; debug on labels, not payload |
| G6-D6 | MEDIUM | Admission SLO (250ms) collides with synchronous evidence-value capture | Split SLI: decide fast, enrich evidence async/lossless |
| G6-D7 | MEDIUM | "Audit every op" (RUN-0) vs "no op noise in chain" (OBS-3) | Scope governed events to enforcement/integrity-affecting only |
| G6-D8 | MEDIUM | Managed-tier "customer-held key" still exposes plaintext to vendor compute | Require customer KMS/HSM; acknowledge residual trust |
| G6-D9 | LOW | Plugins (unbounded) have no golden-signal contract | Make /metrics a plugin SPI load condition |
| G6-D10 | LOW | Nobody watches the monitoring stack itself | External dead-man's-switch heartbeat |

**Bottom line.** The component's central claim — that an append-only tamper-evident log is migrated by a **chain-epoch boundary, never a rewrite** — is **sound and is the right answer** (G6-D1/D3 refine *how* to seal safely, they don't refute it). The weakest parts are the *human* claims: that a 2–4 person team can run 14 services (G6-D2 — only honestly answered by the managed tier), and the dangling hand-offs to sibling components that don't exist yet (G6-D4). **Biggest day-2 risk:** not the migration (which is solved) but the **un-runnability of the stack by its target operator** — the migration is a one-time event you can rehearse; the daily operational burden is forever, and G6 can mitigate but cannot, by itself, make 14 services feel like 2.
