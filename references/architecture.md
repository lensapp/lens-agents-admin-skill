# Architecture — the internals you need to operate it

## Two MCP endpoints (and why it matters)
- **Global `/mcp`** — the **first-party admin tools** (projects, policies,
  sandboxes, connectors, credentials, spending, audit, shell) **plus** the
  org-scoped upstream connector aggregator. **This is where you administer.**
- **Project-scoped `/projects/:projectId/mcp`** — **only** that project's
  upstream MCP connectors. **No first-party/admin tools.** Sandboxes are wired to
  this endpoint on purpose: a managed agent can reach its project's tool surface
  **without being able to drive its own sandbox over MCP** (prevents recursive
  `shell_exec`/self-administration loops). A sandbox token that tries another
  `projectId` in the path is rejected 403.

Consequence: **an agent connected to the project endpoint will not see admin
tools** — that's not a bug. Full administration requires the global `/mcp`.

## Four auth types (what each principal can do)
`oidc | api-token | cluster-jwt | sandbox` (composite auth order on `/mcp`:
sandbox-token → api-token → OIDC).
- **oidc** — humans; sees **all** tools; org-admin if the DB says so.
- **api-token** — external callers / an admin agent; Bearer (`lns_` prefix); sees
  a subset; carries an **org-admin scope flag** (an admin-scoped token
  administers exactly like an OIDC admin — see `rbac.md`).
- **cluster-jwt** — the cluster relay identity (`lnsc_` prefix).
- **sandbox** — a managed agent's identity; scoped to one project as **MEMBER**,
  **never** org-admin. No first-party admin tool is even visible to it.

**Sandbox-as-principal (the current model):** policies attach **directly to a
sandbox at create time** (`create_sandbox` takes `policies: [...]`, a frozen
inline attachment). "agent_token" was renamed **`api_token`**. Two independent
policy axes: **people** (user/api_token, capped by the `everyone` ceiling) and
**sandboxes** (capped only by the `all_sandboxes` ceiling — `everyone` does
**not** cap sandboxes). PII masking is the inverted case: it composes
**additively** (a stricter lower layer tightens masking, isn't capped).

## Credential injection & egress
Egress is enforced at the sandbox network boundary: a policy-aware proxy
terminates TLS there to **inject credentials** and **re-sign AWS SigV4**, and
applies the **domain allowlist** (first-match-wins, default-deny). The agent's
own process only ever holds decoys — the real secret is added at the boundary.

Two policy-resolution edge cases worth knowing:
- **Empty/absent policy ≠ deny-everything**: it egress-denies but still allows
  the platform's own host and leaves **managed inference OFF** (opt-in).
- A genuine policy-resolution **error** fails **closed** (deny-all).

## How sandboxes reach clusters
Outbound traffic routes through a **cluster relay tunnel** for `clusterId`-tagged
routes; the relay is an **outbound-only** daemon inside the target network (no
inbound ports, no VPN). Or a **direct relay** (public HTTPS). Either way the same
policy + audit apply — connectivity mode is a transport choice, not a trust
choice.

## Short-lived credentials (know the TTLs)
- **kubectl** cluster JWT: **15 minutes**, auto-rotated, impersonates
  `agent:<name>` with `<org>/<team>` groups.
- **EKS** token: SigV4-presigned, **capped at 900s** by AWS.
- **AWS** STS AssumeRole: **900s (15 min)**, session-tagged (agent/org/project) →
  flows to CloudTrail; real creds injected, **never written to sandbox disk**.

## Sandbox runtime shape
A supervisor (PID 1) + static `nft` are side-loaded into `/.lens/`; the user
image doesn't need the nftables package but **must have `/bin/sh`** and a
**writable CA bundle** (`/etc/ssl/certs/ca-certificates.crt`) so the boundary CA
can be appended. **`FROM scratch` and non-debug distroless images are
unsupported** (no shell). K8s provisioner can set a `RuntimeClass` (`kata-clh`,
`gvisor`) for microVM isolation. **Caps: at most one exposed port and one
persistent volume per sandbox.**
