# DT-44 — Install Headlamp plugin and authenticate via Keycloak OIDC

**Personas:** Marcus (Platform Security Engineer), Jess (SRE / Cluster Operator)
**Spec sections:** §16.2 Framework Requirements (Headlamp plugin model, OIDC, Kubernetes-native RBAC discovery), §17A.4 Keycloak Integration, §17A.5 Storage-Level Access Controls, §16.3 Required Views
**Type:** Low-level
**Pre-condition:** Headlamp is deployed in `cluster-a` (and on engineers' workstations). Keycloak realm `platform` has the platform OIDC client registered with the required claim mappers (§17A.4: `sub`, `preferred_username`, `email`, `groups`, `realm_access.roles`, `resource_access.platform.roles`, `namespaces`, `policy_domains`, `tenants`). Marcus holds Platform Governance Admin; Jess holds Security Reviewer scoped to `cluster=cluster-a`, `namespaces=[prod-east, payments-prod, …]`.
**Trigger:** A new release of the platform's Headlamp plugin (Governance Console) is published; Marcus rolls it out, then Jess logs in for the first time.

## Steps
1. Marcus installs the Headlamp plugin from the platform's OCI plugin registry: `headlamp plugins install oci://registry.local/plugins/governance-console:v1.4`. The plugin manifests register the five §16.3 views (Governance Graph, Rego Explorer, Runtime Enforcement, Audit Correlation, Namespace Authoring) under a sidebar group.
2. On load, the plugin uses Headlamp's Kubernetes-native context discovery (§16.2) to detect the active kubeconfig context `cluster-a`, the API server URL, and the in-cluster service endpoints for the Governance API and Compliance Analytics. Cluster context appears in the plugin header; no manual cluster registration is needed.
3. The plugin redirects Marcus to Keycloak's authorization endpoint with `client_id=governance-console`, scopes `openid profile groups platform-claims`, and PKCE. Marcus authenticates; Keycloak issues an ID token + access token carrying the §17A.4 required claims.
4. The plugin's OIDC handler validates the token, then calls the Governance API `/v1/subject/normalize`, which resolves the token claims into the §17A.4 normalized subject JSON (`subject_id`, `username`, `roles`, `groups`, `namespaces`, `policy_domains`, `tenants`). The subject is cached for the session.
5. Marcus's normalized subject (`roles=[platform-governance-admin]`, broad scope) drives view rendering: all five §16.3 views are visible; the Runtime Enforcement View lists every cluster.
6. Jess logs into Headlamp on her workstation and opens the Governance Console plugin. The same OIDC flow runs; her token yields a normalized subject with `roles=[security-reviewer]`, `namespaces=[prod-east, payments-prod, …]`, `tenants=[platform]`.
7. View scoping is applied by both the plugin (UI hides Namespace Authoring's "New Policy" button — she lacks `policy:edit`) and the storage layer (§17A.5: queries against the Governance API are filtered by her subject; out-of-scope clusters and namespaces return zero rows). Jess opens the Runtime Enforcement View and sees only `cluster-a` data scoped to her namespaces.

## Success criteria (testable)
- `headlamp plugins install …` registers the plugin and the five §16.3 views in the sidebar without restart.
- Plugin auto-discovers cluster context from the active Headlamp kubeconfig; no manual cluster URL entry is required (§16.2 Kubernetes-native RBAC discovery).
- Unauthenticated views redirect to Keycloak; PKCE OIDC flow completes; token contains all §17A.4 required claims; missing-claim tokens are rejected with a clear error.
- `/v1/subject/normalize` produces the §17A.4 normalized subject JSON exactly matching the spec shape.
- Jess's view set is restricted by her subject scope in both UI (hidden actions for missing permissions) and storage (out-of-scope queries return zero rows, verified by a direct API call bypassing the UI per §17A.5).
- Logout clears the plugin session and revokes the cached subject; re-login is required.

## Flowchart

```mermaid
flowchart TD
  INST[Marcus: headlamp plugins install\ngovernance-console:v1.4] --> REG[Plugin registers 5 views §16.3]
  REG --> DISC[Auto-discover kubeconfig context\ncluster-a §16.2]
  DISC --> LOGIN[Open plugin → redirect to Keycloak\nPKCE OIDC §16.2]
  LOGIN --> TOKEN[ID + access token with\n§17A.4 required claims]
  TOKEN --> NORM["/v1/subject/normalize\n→ normalized subject JSON §17A.4"]
  NORM --> MARCUS[Marcus: admin scope\n→ all views, all clusters]
  NORM --> JESS[Jess: security-reviewer\nnamespaces=[prod-east,…]]
  JESS --> SCOPE[UI hides edit actions +\nstorage filters queries §17A.5]
  SCOPE --> RTE[Runtime Enforcement View\nscoped to her namespaces]
```

## Notes
Related: DT-35 (adding a claim), DT-53 (granting the namespace-author role), DT-55 (storage-scope verification), HL-16 (IdP-driven claim evolution). §17A.5: storage-level filtering must be verified independently of the UI to detect GUI-only authorization regressions.
