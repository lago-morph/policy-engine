# DT-43 — Use Namespace Authoring View as a Namespace Policy Author

**Personas:** Sam (Application Developer, Namespace Policy Author for `payments-*`)
**Spec sections:** §16.3 Namespace Authoring View, §17A.2 Namespace Policy Author / Namespace Policy Approver, §17A.3 Permission Primitives, §17A.5 Storage-Level Access Controls, §17.2 Namespace Simulation, §7 Policy Lifecycle
**Type:** Mid-level
**Pre-condition:** Sam is authenticated via Keycloak with normalized subject (§17A.4) `roles=[namespace-policy-author]`, `namespaces=[payments-prod, payments-dev]`, `policy_domains=[runtime-security]`, `tenants=[payments]`. A peer, Ravi, holds `namespace-policy-approver` for the same namespaces. The platform enforces scope in both the GUI and the storage layer (§17A.5).
**Trigger:** Sam's team needs a per-pod request-rate ceiling for `payments-prod/api` to protect a downstream legacy SDK; he opens the Namespace Authoring View to draft a namespace-scoped policy.

## Steps
1. Sam opens the Namespace Authoring View (§16.3) from the Headlamp plugin. The view lists only policies and simulation datasets whose authorization metadata (`namespaces`, `tenant`, `policy_domains`) intersect his subject scope — i.e. objects under `payments-*` (§17A.5). Other tenants' policies are absent from the query result, not merely hidden by GUI filtering.
2. Sam clicks "New Policy" and selects the Kyverno `validate` template (§17D Kubernetes library). The form pre-fills `tenant=payments`, `namespaces=[payments-prod]`, `policy_domains=[runtime-security]`, `created_by=sam`, `visibility=namespace-scoped` per §17A.5 required metadata; the namespace selector is constrained to his authorized set.
3. Sam authors `payments-api-rate-limit-v1` requiring `metadata.annotations["payments/rps-ceiling"]` ≤ 500. He saves a draft; the storage layer rejects writes outside the namespace set — verified by an internal `policy:edit` permission check (§17A.3).
4. Sam clicks "Simulate" → Namespace Simulation (§17.2). The simulation dataset is auto-materialized from the last 30 days of `payments-prod` admission events (§17A.5: "Audit replay datasets must be materialized as scoped datasets before use"). Events outside his namespaces are excluded at materialization time, not at presentation time.
5. The Namespace Authoring View renders namespace-scoped violation visibility: 4 historical pods would have been newly blocked. Sam tags 3 as "intended enforcement" and 1 as "potential false positive" (§17.4 differential semantics), then iterates the policy to add an exemption annotation, re-simulates, and reaches 0 false positives.
6. Sam clicks "Request Promotion → warn" (§7 lifecycle). The platform creates a promotion request; because Sam holds `policy:edit` and `policy:test` but not `policy:promote-dry-run` for `payments-prod`, the request is routed to Ravi as Namespace Policy Approver.
7. Ravi opens the same view in his approver scope, sees the simulation summary, signed policy diff, and Sam's tags. He clicks Approve. The policy promotes to `warn` mode in `payments-prod` only; namespace-scoped approval state in the view updates to `warn @ payments-prod`. A later promote-to-enforce will repeat the gate.

## Success criteria (testable)
- Namespace Authoring View lists only objects whose §17A.5 metadata intersects Sam's subject scope; cross-namespace queries return zero rows at the storage layer (not the GUI).
- New-policy form pre-fills and locks the required scope metadata (`tenant`, `namespaces`, `policy_domains`, `visibility=namespace-scoped`).
- Namespace Simulation materializes a scoped dataset from the past audit window and excludes out-of-scope events at materialization.
- Sam cannot promote past `warn` without an in-namespace Approver; the workflow honors §17A.3 permission primitives (`policy:promote-dry-run`, `policy:promote-enforce`).
- Approval by Ravi (same namespace) promotes the policy to `warn` in `payments-prod` only; `payments-dev` and other namespaces are unaffected.
- An attempt by Sam to read or simulate against `treasury-prod` policies via the API returns no rows / 403 from the storage layer, independent of the GUI.

## Flowchart

```mermaid
flowchart TD
  AUTH[Sam logs in via Keycloak\nsubject: payments-* author §17A.4] --> VIEW[Namespace Authoring View §16.3\nshows only payments-* objects §17A.5]
  VIEW --> NEW[New policy: Kyverno validate\nscope metadata pre-filled & locked]
  NEW --> DRAFT[Draft payments-api-rate-limit-v1\nstorage enforces namespace bound §17A.3]
  DRAFT --> SIM[Namespace Simulation §17.2\nscoped dataset materialized §17A.5]
  SIM --> TAG[3 intended enforcement\n1 false positive → iterate §17.4]
  TAG --> REQ[Request promotion → warn §7\nrouted to Approver Ravi]
  REQ --> APPROVE[Ravi (in-namespace Approver) approves\n→ warn @ payments-prod only]
```

## Notes
Related: DT-50 (namespace-scoped simulation deep-dive), DT-53 (granting the role), DT-55 (storage-scope verification), HL-08 (NS authoring end-to-end). §17A.5: "GUI-only authorization is insufficient" — tests must hit the storage API directly to confirm scope.
