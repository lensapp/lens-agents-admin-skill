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
- `injections[]` is **required** (it may be empty) — it declares where the
  credential is injected. Each injection binds the secret to one domain (bare
  hostname: no scheme, port, or wildcard) + header, with the header value
  templated: `headerFormat` **must** contain `{value}` (a format without it is
  rejected on write). `{base64:<prefix>{value}}` base64-encodes the prefix plus
  the secret, e.g. `Basic {base64:user:{value}}`. One injection per
  (domain, header) — a duplicate is rejected.
- Optional `bodyField` also writes the bare value into that field of an
  `application/x-www-form-urlencoded` body — only where the field is already
  present, on requests the header injection matches. Needed for SDKs that send
  the token twice (Slack's sends it as `token` in the body too).
- `rules` omitted (or empty) = inject on all paths.
- Reads show `headerFormat` in full only to project **ADMIN**s; everyone else
  gets `[REDACTED]` and `headerFormatRedacted: true`.
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
