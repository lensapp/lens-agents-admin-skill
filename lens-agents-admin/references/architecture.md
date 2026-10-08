# Architecture — the internals you need to operate it

## Two MCP endpoints (and why it matters)
- **Global `/mcp`** — the **first-party admin tools** (projects, policies,
  sandboxes, connectors, credentials, spending, audit) **plus** the
  org-scoped upstream connector aggregator. **This is where you administer.**
- **Project-scoped `/projects/:projectId/mcp`** — **only** that project's
  upstream MCP connectors. **No first-party/admin tools at all.** Sandboxes are wired to
  this endpoint on purpose: a managed agent can reach its project's tool surface
  **without being able to drive its own sandbox over MCP** (prevents
  self-administration loops). A sandbox token that tries another
  `projectId` in the path is rejected 403.

Consequence: **an agent connected to the project endpoint will not see admin
tools** — that's not a bug.
Full administration requires the global `/mcp`.

## Four auth types (what each principal can do)
`oidc | api-token | cluster-jwt | sandbox` (composite auth order on `/mcp`:
sandbox-token → api-token → OIDC).
- **oidc** — humans; sees **all** tools; org-admin if the DB says so.
- **api-token** — external callers / a per-project admin agent; Bearer (`lns_`
  prefix); **never org-admin**; sees the api-token-visible tool subset and can
  administer a **project** it holds **ADMIN** role on (a direct project role) — org-scoped
  ops stay OIDC-only. See `rbac.md`.
- **cluster-jwt** — the per-cluster kubectl JWT a sandbox's kubeconfig resolves
  to (`lnsc_` prefix, 15 min). The relay itself authenticates its tunnel with an
  `lnst_` token.
- **sandbox** — a managed agent's identity; scoped to one project as **MEMBER**,
  **never** org-admin. On the global `/mcp` the only first-party tools visible to
  it are **three self-scoped spend/usage reads** — `get_usage_cost_summary`,
  `get_usage_cost_timeseries`, `get_spending_limit_status` — which return only its
  own budget's data; every spending-limit *mutation* stays OIDC-only.

**Sandbox-as-principal (the current model):** policies attach **directly to a
sandbox** (`create_sandbox` takes `policyIds: [...]` — live references to shared
policies that `update_sandbox` can replace — plus an optional embedded `policy` and
`credentials`). "agent_token" was renamed **`api_token`**. Two independent
policy axes: **people** (user/api_token, capped by the `everyone` ceiling) and
**sandboxes** (capped only by the `all_sandboxes` ceiling — `everyone` does
**not** cap sandboxes). PII masking is the inverted case: the ceiling
doesn't cap it — a project's `piiMasking` **replaces** the org's, and the org's
applies only where the project sets none.

## Credential injection & egress
Egress is enforced at the sandbox network boundary: a policy-aware proxy
terminates TLS there to **inject credentials** and **re-sign AWS SigV4**, and
applies the **domain allowlist** (first-match-wins, default-deny). The agent's
own process only ever holds decoys — the real secret is added at the boundary.

Two policy-resolution edge cases worth knowing:
- **Empty/absent policy ≠ deny-everything**: it egress-denies but still allows
  the platform's own host and leaves **managed inference OFF** (opt-in).
- A genuine policy-resolution **error** fails **closed** (deny-all).

The platform's forward proxy (which carries a sandbox's `upstream`-transport
traffic) checks every CONNECT against the sandbox's network policy: a target the
policy doesn't route `upstream` gets **403** and an audit failure (reason
`policy`, or `policy-unavailable` when no policy resolved).
`FORWARD_PROXY_POLICY_CHECK=audit` (env, not a chart value) lets those through,
logged and marked on the audit entry; the default is `enforce`.

## How sandboxes reach clusters
Outbound traffic routes through a **cluster relay tunnel** for `clusterId`-tagged
routes; the relay is an **outbound-only** daemon inside the target network (no
inbound ports, no VPN). Or a **direct relay** (public HTTPS). Either way the same
policy + audit apply — connectivity mode is a transport choice, not a trust
choice.

## Short-lived credentials (know the TTLs)
- **kubectl** cluster JWT: **15 minutes**, auto-rotated. It impersonates
  `sandbox:<id>` in group `<org>/<project>` for a sandbox; `agent:<tokenName>` (API
  token) or `oidc:<email>` (human) in groups `<org>/<project>` and
  `<org>/<project>:<role>` (`admin`|`member`) of the cluster's project.
- **EKS** token: SigV4-presigned, **capped at 900s** by AWS.
- **AWS** STS AssumeRole: **900s (15 min)**, session-tagged (`lens:user-id`,
  `lens:user-identity`, `lens:project-id`, `lens:project-name`, `lens:org-id`,
  `lens:org-name`, `lens:connection-name`) → flows to CloudTrail; real creds injected, **never written to sandbox disk**.

## Sandbox runtime shape
A supervisor (PID 1) + static `nft` are side-loaded into `/.lens/`; the user
image doesn't need the nftables package but **must have `/bin/sh`** and a
**writable CA bundle** (`/etc/ssl/certs/ca-certificates.crt`) so the boundary CA
can be appended. **`FROM scratch` and non-debug distroless images are
unsupported** (no shell). K8s provisioner can set a `RuntimeClass` (`kata-clh`,
`gvisor`) for microVM isolation. **Caps: at most four exposed ports and one
persistent volume per sandbox.**
