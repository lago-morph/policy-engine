# DT-71 — GitLab library — require code-owner approval for policy file changes

**Personas:** Marcus (Platform Security Engineer)
**Spec sections:** §17D.5 GitLab Library (Merge request opened/updated; Merge request approved), §17D.1 Library elements, §17D.11 Cross-Product Decision Point Pattern, §13 Standardized Audit Event Schema
**Type:** Low-level
**Pre-condition:** The org's policy monorepo `platform/policies` lives in GitLab. `CODEOWNERS` maps `/policies/*` to the `@security-team` group. The §17D.5 GitLab library is installed: an MR-opened/updated webhook fires the CI "policy job" PDP, and an MR-approval rule named `policy-files-security-review` is registered on the project. Marcus owns the library configuration.
**Trigger:** A developer pushes commit `e1a4…` to MR `!482` that modifies `policies/k8s/require-signed-image.rego` and `policies/k8s/exception_schema.yaml`. GitLab fires `merge_request:opened` to the §17D.5 webhook.

## Steps
1. The §17D.5 "Merge request opened/updated" decision point receives the MR payload. The policy job PDP inspects the MR diff, sees paths under `policies/*`, and classifies the MR as `change_class=policy-file` per Marcus's library configuration.
2. The PDP issues two outcomes per §17D.5 supported actions: (a) a CI status `policy/security-review = pending` (block merge), and (b) an `approval_required` API call that adds the `policy-files-security-review` rule to MR `!482`, requiring one approval from CODEOWNERS group `@security-team`.
3. GitLab's native approval engine enforces the rule: the "Merge" button is disabled with reason "Policy file change requires security review" (the §17D.5 example policy text), independent of pipeline state.
4. The decision is emitted as a §13 audit event: `source=gitlab`, `decision_point=merge_request.opened`, `subject={user:dev42, jwt_sub:…}`, `resource_id=group/platform/policies!482`, `action=require_approval`, `control_id=POL-CHANGE-REVIEW-001`, `policy_version=gitlab-lib:v3`, `correlation_id=mr-482-e1a4`.
5. The developer self-approves; the rule rejects the approval because the approver is the MR author, not a `@security-team` member. The status check stays `pending` and merge remains blocked (§17D.5 "Merge request approved" decision point: `require extra approval`).
6. A `@security-team` member reviews the Rego diff and approves on the MR. GitLab fires `merge_request:approved`; the §17D.5 PDP re-evaluates, sees the approval is from a CODEOWNERS member of the right group, and flips the CI status `policy/security-review = success`. A second §13 audit event is emitted with `action=allow` and the same `correlation_id`.
7. Marcus pulls the §17E real-time enforcement slice for `control_id=POL-CHANGE-REVIEW-001` and confirms two paired events (require_approval → allow) for `!482`. Bypass attempts (force-push to protected branch, direct push) are covered by sibling §17D.5 rows and out of scope here.

## Success criteria (testable)
- Any MR whose diff touches `policies/*` is automatically tagged with the `policy-files-security-review` approval rule by the §17D.5 PDP within one webhook delivery.
- The GitLab merge UI blocks merge with the exact §17D.5 reason "Policy file change requires security review" until a `@security-team` CODEOWNERS approval is recorded.
- MR author self-approval does not satisfy the rule (separation-of-duties).
- Each decision emits a §13 audit event with `source=gitlab`, `decision_point`, `subject`, `resource_id`, `policy_version`, `correlation_id`, and matched `control_id`.
- The two paired events (require_approval → allow) share one `correlation_id` so the §17E report can render the full MR lifecycle on one row.

## Flowchart

```mermaid
flowchart TD
  MR[GitLab MR !482 opened/updated\ntouches policies/*] --> HOOK[§17D.5 webhook\nMerge request opened/updated]
  HOOK --> PDP{Diff paths under\npolicies/*?}
  PDP -- Yes --> RULE[Attach approval rule\npolicy-files-security-review]
  RULE --> BLOCK[CI status pending\nMerge button disabled]
  BLOCK --> AUDIT1[§13 event: require_approval\ncorrelation_id=mr-482-e1a4]
  BLOCK --> SELF[Author self-approval]
  SELF -- rejected, not @security-team --> BLOCK
  BLOCK --> SEC[@security-team CODEOWNER approves]
  SEC --> HOOK2[§17D.5 Merge request approved]
  HOOK2 --> PASS[CI status success → merge enabled]
  PASS --> AUDIT2[§13 event: allow\nsame correlation_id]
  AUDIT2 --> REP[§17E real-time slice\ncontrol POL-CHANGE-REVIEW-001]
```

## Notes
Related: DT-70 (Jenkins gate), HL-02 (image-signing rollout). The protected-branch and protected-variable rows of §17D.5 are sibling controls, not part of this scenario.
