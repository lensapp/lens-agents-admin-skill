# Credentials — secrets the agent never sees

> **Exact parameters aren't here — read the live schema.** For any tool, use `tools/list` (automatic over MCP) or the REST OpenAPI at `<publicUrl>/v1/openapi.json` (`/v1/docs` for the UI). This file covers what those can't: what the tools are for and the non-obvious rules.

A **credential** is a secret stored (encrypted at rest) in a project. It is
**injected into outbound requests by the proxy**, per-domain, so the agent
never reads the raw value — it only sees the request succeed or fail. A
policy references it by name (`credentials[]` in `policies.md`).

Tools: `list_credentials`, `get_credential`, `create_credential`,
`update_credential`, `delete_credential`. The stored `value` is never
returned by any read call.

Notes:
- `injections[]` is **required** — a credential must declare where it's
  injected. Each injection binds the secret to one domain + header, with the
  header value templated (a `{value}` placeholder substituted with the
  secret at request time).
- Scope injection tightly with per-injection method/path rules when possible.
- A credential only reaches an agent when **both** sides agree: the agent's
  policy has a matching `credentials[]` ref *and* the same domain is allowed
  in that policy's network config (`allowedDomains`) — either alone does
  nothing.
- Kubernetes and AWS get first-class handling instead of raw credentials:
  short-lived, per-request Kubernetes JWTs and AWS STS re-signing
  (AssumeRole), so the agent never holds a standing secret for either. Prefer
  the connection integrations (`connections.md`) over a raw credential for
  those two systems.
