# MCP connectors — give agents upstream tools

> **Exact parameters aren't here — read the live schema.** For any tool, use `tools/list` (automatic over MCP) or the REST OpenAPI at `<publicUrl>/v1/openapi.json` (`/v1/docs` for the UI). This file covers what those can't: what the tools are for and the non-obvious rules.

Register an **upstream MCP server** once; the platform discovers its tools,
aggregates them into the in-sandbox MCP endpoint, and governs every
`tools/call`. Policies decide which agent sees which server and which of its
tools.

## MCP servers

Tools: `list_mcp_servers`, `get_mcp_server`, `create_mcp_server`,
`update_mcp_server`, `delete_mcp_server`, `sync_mcp_server_tools`
((re)discover the upstream tool catalog after it changes). `transport` is `sse`
or `streamable-http`; `deferDiscovery: true` on create skips the first probe when
you'll attach a credential and sync afterwards; `update_mcp_server` can also set
`headers` and `enabled`.

**Visibility:** only `list_mcp_servers`, `get_mcp_server`, and
`list_oauth_applications` are offered to API tokens. Everything else here —
server create/update/delete/sync, credential list/create/delete,
`create_http_connector`, `list_connector_entries`,
`set_connector_entry_visibility` — is **OIDC-only**.

The reserved server name **`nexus-api`** is the platform's own self-reference to
`<publicUrl>/mcp` (used so first-party tools can reach project sandboxes) —
`create_mcp_server` refuses to shadow it, but it's already there in every project
(`list_mcp_servers` shows it). You **don't create a self-reference connector** — to
give a managed agent project-admin power (the **"Odin"** pattern), attach a
project-admin API token as a **credential on `nexus-api`**
(`create_mcp_server_credential { serverId:<nexus-api>, authType:"static", … }` —
OIDC-only, so a human session does this step) and
reference that credential from a policy connector grant (`connectors[].credentialId`).
The platform then dispatches `nexus-api`'s first-party admin tools as the **token's**
principal, so the agent sees `create_policy`, `create_sandbox`, etc. as native tools
while the token stays server-side. See `playbooks.md` playbook 6 + `rbac.md`.

## MCP server credentials

Tools: `list_mcp_server_credentials`, `create_mcp_server_credential`,
`delete_mcp_server_credential`. Auth types: `static` (a fixed bearer value),
`oauth-client-credentials`, and `oauth-authorization-code` (the latter
supports per-actor PKCE grants). A policy's connector ref picks which
credential each actor uses via `credentialId`.

### Reusable OAuth applications (deployment catalog)

An operator can pre-register OAuth Login applications so nobody pastes client
secrets into a credential. The catalog is a JSON Secret mounted by the chart —
`oauthApplications.existingSecret` / `existingSecretKey` (default
`applications.json`):

```json
{ "version": 1,
  "applications": [{ "id": "…", "displayName": "…",
    "authorizationServerUrl": "https://…", "tokenUrl": "https://…",
    "clientId": "…", "clientSecret": "…",
    "upstreamUrls": ["https://mcp.example.com/mcp"],
    "defaultScopes": "…", "prompt": "consent", "accessType": "offline" }] }
```

`clientSecret`, `defaultScopes`, `prompt`, and `accessType` are optional; `upstreamUrls` needs
at least one entry. Endpoints must be HTTPS (HTTP only for loopback). `prompt` is
space-separated `none` / `login` / `consent` / `select_account`, with `none` only
on its own. `accessType` (`offline` / `online`) is sent as `access_type` — Google
issues a refresh token only with `offline`, so pair it with `prompt: "consent"`.
**Restart all replicas after changing it**; registration secrets are
never copied to the database.

- `list_oauth_applications { projectId }` (project admin) lists the choices —
  never secrets.
- `create_mcp_server_credential { …, authType:"oauth-authorization-code",
  oauthApplicationId }` uses one instead of inline client fields. The server URL
  must match one of the application's `upstreamUrls`; omit `scope` to use its
  `defaultScopes`, or send an empty scope to request none.
- Credential views report `oauthApplicationAvailable`,
  `oauthApplicationDisplayName`, and `oauthApplicationError`, so a credential whose
  application no longer resolves (removed or changed in the catalog) shows up as
  broken rather than silently failing.

### Extra redirect URIs and the callback outcome

An OAuth Login credential can allow up to **10** `additionalRedirectUris` (for a
client that completes the login on its own redirect) — **REST / web UI only**,
not on `create_mcp_server_credential`; project admin; an update replaces the
list. Each must also be listed in the install's
`config.upstreamOAuthAdditionalRedirectUris` (`UPSTREAM_OAUTH_ADDITIONAL_REDIRECT_URIS`):
`https` (except localhost), and never the platform's own
`<publicUrl>/v1/mcp-servers/oauth/callback`. A sign-in picks one with
`redirectUri` on the OAuth start request. A service on such a URI that forwards
the query to the platform's callback with `Accept: application/json` gets
`200 {status:"connected", serverId}` or `400 {status:"failed", reason}` instead of
the browser page; the state is single-use, so don't retry a forward.

## Expose to agents

Add a connector ref to the agent's **policy** (`policies.md`), naming the
server and optionally an `allowedTools` whitelist plus a `credentialId`. As
with all connector refs, an empty `allowedTools: []` **denies every tool** in
that ref rather than allowing all. Once the policy is bound, the agent
reaches the tools via its project's `/projects/:projectId/mcp` endpoint.

## HTTP connectors (OpenAPI)

For plain REST-with-OpenAPI upstreams (not MCP-native), use
`create_http_connector` — it ingests an OpenAPI 3.x doc and can relay
requests either directly or through a cluster tunnel. Manage which
discovered operations are exposed with `list_connector_entries` /
`set_connector_entry_visibility`.
