# MCP connectors — give agents upstream tools

> **Exact parameters aren't here — read the live schema.** For any tool, use `tools/list` (automatic over MCP) or the REST OpenAPI at `<publicUrl>/v1/openapi.json` (`/v1/docs` for the UI). This file covers what those can't: what the tools are for and the non-obvious rules.

Register an **upstream MCP server** once; the platform discovers its tools,
aggregates them into the in-sandbox MCP endpoint, and governs every
`tools/call`. Policies decide which agent sees which server and which of its
tools.

## MCP servers

Tools: `list_mcp_servers`, `get_mcp_server`, `create_mcp_server`,
`update_mcp_server`, `delete_mcp_server`, `sync_mcp_server_tools`
((re)discover the upstream tool catalog after it changes).

The reserved server name **`nexus-api`** is the platform's own self-reference
(used so first-party tools can reach project sandboxes) — `create_mcp_server`
refuses to shadow it.

## MCP server credentials

Tools: `list_mcp_server_credentials`, `create_mcp_server_credential`,
`delete_mcp_server_credential`. Auth types: `static` (a fixed bearer value),
`oauth-client-credentials`, and `oauth-authorization-code` (the latter
supports per-actor PKCE grants). A policy's connector ref picks which
credential each actor uses via `credentialId`.

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
