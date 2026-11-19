# Credentials — secrets the agent never sees

> **Exact parameters aren't here — read the live schema.** For any tool, use `tools/list` (automatic over MCP) or the REST OpenAPI at `<publicUrl>/v1/openapi.json` (`/v1/docs` for the UI). This file covers what those can't: what the tools are for and the non-obvious rules.

A **credential** is a secret stored (encrypted at rest) in a project. It is
**injected into outbound requests by the proxy**, per-domain, so the agent
never reads the raw value — it only sees the request succeed or fail. A
policy references it by name (`credentials[]` in `policies.md`).

Tools: `list_credentials`, `get_credential`, `create_credential`,
`update_credential`, `delete_credential`. The stored `value` is never
returned by any read call. Credentials a sandbox owns (embedded in
`create_sandbox` / `update_sandbox`) don't appear in `list_credentials` and are
deleted with the sandbox.

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
- **OAuth client-credentials** (REST, web UI, or embedded in
  `create_sandbox`/`update_sandbox` — not MCP `create_credential`, which still
  takes a `value`): send
  `oauthClientCredentials { tokenUrl (https), clientId, clientSecret, scope? }`
  instead of `value`. The platform mints and refreshes the access token and
  injects it; the agent's own token request gets a placeholder answer. The first
  token is minted at create, and a client whose tokens last **≤ 20 min** is
  refused. The policy must allow the **token endpoint host** as well as the API
  host. Reads report `authType`.
- **GitHub App** (REST `POST /v1/projects/{projectId}/credentials`, the web UI's
  **Secret: GitHub App**, or embedded in `create_sandbox`/`update_sandbox` and
  nexusctl spec files — **not** MCP `create_credential`, which still takes a
  `value`): send `jwtAssertion { preset:"github", clientId, privateKey,
  installationId, repositories | repositoryIds, permissions }` instead of `value`.
  `privateKey` is the App's RSA key in PEM; give exactly one of `repositories`
  (names without the owner) or `repositoryIds`; `permissions` maps a GitHub
  permission to `read`/`write`/`admin` and can't be empty. The platform signs a
  short-lived JWT and exchanges it for an **installation token** limited to those
  repositories and permissions, and renews it before it expires. The first token
  is minted at create, so a wrong key, installation, repository or permission is
  refused right away with GitHub's reason. GitHub.com only. With `injections`
  empty it injects `api.github.com` (`Authorization: token {value}`) and
  `github.com` (`Basic {base64:x-access-token:{value}}`, for git over HTTPS); the
  policy must still reference the credential and allow those domains, but no
  token-endpoint host. Reads report `authType: "jwt-assertion"`. **No in-place
  rotation:** an update can rename it or replace its injections, but a `value`
  is refused — to change the key, installation, repositories or permissions,
  delete and recreate it.
- **Embedded (sandbox-owned) credentials:** `create_sandbox`
  `credentials { <name>: {...} }` takes exactly one of `value`,
  `oauthClientCredentials` or `jwtAssertion` per entry, plus `injections`;
  `update_sandbox` takes at most one per entry — an existing static one can
  rotate its `value` and/or replace its injections, an existing minted one only
  its injections. A minted source (`oauthClientCredentials`/`jwtAssertion`) on
  update can only **add a new name**:
  to switch an existing embedded credential to one, remove it in one update and
  add it in the next. Two updates adding the same name at once: the loser gets
  **409 conflict** ("Read the sandbox and try again"). The web UI's YAML mode
  refuses minted sources; use the form or a nexusctl spec file.
- Kubernetes and AWS get first-class handling instead of raw credentials:
  short-lived, per-request Kubernetes JWTs and AWS STS re-signing
  (AssumeRole), so the agent never holds a standing secret for either. Prefer
  the connection integrations (`connections.md`) over a raw credential for
  those two systems.
