# Policies & bindings — the heart of access control

> **Exact parameters aren't here — read the live schema.** For any tool, use `tools/list` (automatic over MCP) or the REST OpenAPI at `<publicUrl>/v1/openapi.json` (`/v1/docs` for the UI). This file covers what those can't: what the tools are for and the non-obvious rules.

A **policy** declares what an agent may do: network egress, credential
injection, managed-inference routing, PII masking, and which integrations/MCP
connectors it can use. A **policy binding** attaches one or more policies to
**subjects** (agents/users/sandboxes). Nothing takes effect until a binding
exists.

## Policies

Tools: `list_policies`, `list_org_policies`, `get_policy`, `create_policy`,
`update_policy`, `delete_policy`. `create_org_policy` makes an org-scoped,
reusable **ceiling** (restriction-only — a project policy cannot broaden it).

Non-obvious semantics, by field:

- **Network** — verdicts are evaluated **first-match-wins**;
  `networkDefaultVerdict: deny` plus explicit allowed domains is the
  least-privilege pattern (default-deny). Keep the LLM provider's own hosts
  denied so inference stays routed through managed inference
  (metered/governed) rather than going direct.
- **`credentials[]`** references a credential by name (see `credentials.md`).
  It only takes effect if the same domain is also allowed in the network
  config — the credential ref and the `allowedDomains` entry are both
  required, one without the other does nothing.
- **`integrations[]`** (kubernetes / aws-connection, see `connections.md`) is
  **not allowed on org-scoped policies** — project policies only.
- **`managedInference`** — omitting the object leaves inference disabled. It
  only *selects* the managed backend (e.g. `bedrock`); the underlying
  provider credential itself is a platform install-time value, never set
  here.
- **`piiMasking`** — `types` picks from the platform's PII enum;
  `failOpen: false` means fail-**closed** (block rather than risk leaking
  unmasked PII) — read the flag name carefully, it inverts easily.
- **`connectors[]`** binds upstream MCP servers (see `mcp-connectors.md`).
  **`allowedTools: []` means deny all tools in that ref, not allow all** —
  list names to whitelist exactly those. The field is called `connectors`
  over MCP; REST also accepts a legacy `mcpServers` alias for the same thing.
- `update_policy` accepts the same shape, all fields optional. Passing
  `managedInference: null` or `piiMasking: null` is how you *disable* a
  previously-set block — omitting the field just leaves it unchanged.

## Policy bindings

Tools: `list_policy_bindings`, `get_policy_binding`, `create_policy_binding`,
`update_policy_binding`, `delete_policy_binding`. `list_policy_binding_drift`
is advisory: it shows where a project binding got **clipped** by the org
ceiling (the effective grant ended up narrower than the binding asked for).

A binding's `position` sets merge precedence — lower position wins when
multiple bindings apply to the same subject.

**Subject kinds** (discriminated on `kind`): `everyone`, `user`, `api_token`
(note: `api_token`, not `agent_token`), `all_sandboxes`.

**Axis rule:** you may **not** mix people subjects (`everyone`/`user`/
`api_token`) with the sandbox axis (`all_sandboxes`) in one binding — split
them into separate bindings. For a managed agent (a sandbox), bind to
`all_sandboxes` (or a sandbox-scoped policy). `create_org_policy_binding`
forms a ceiling; project bindings union under it but cannot broaden it.

## Org ceilings — bounding a project (and its project-admin agents)

Ceilings clip **per axis**: an org binding on `all_sandboxes` caps every
**sandbox** in the org; an org binding on `everyone`/`api_token` caps **people/
tokens**. A project binding is only clipped by the ceiling on *its own* axis — so
to bound what a project's sandboxes may do, you need an **`all_sandboxes` org
ceiling**.

Clipping is restriction-only and happens **at resolve time**, not at save:

- Egress domains not covered by the ceiling's allowlist are **dropped**; managed
  inference is **AND-gated** (the sandbox keeps it only if the ceiling also grants
  it); if the ceiling lists connectors, connectors it doesn't list are **dropped**.
  A credential aimed at a forbidden domain is neutralized because its domain is clipped.
- **Per-axis carve-out:** a ceiling only caps an axis it actually populates. A
  ceiling that lists *no* connectors leaves the connector axis **uncapped** (so a
  domains-only ceiling doesn't silently strip every sandbox's connectors); likewise
  a default-deny ceiling with *no* domain entries doesn't restrict domains. To cap
  the connector axis, give the ceiling its own `connectors:[…]` allow-list.
- A project binding that asks for more than the ceiling allows is **accepted but
  clipped** — inspect with `list_policy_binding_drift` (per-binding) and
  `get_sandbox_network_clip` (per-sandbox: blocked domains, `managedInferenceBlocked`,
  `blockedConnectors`).

**Why this matters for admin agents ("Odin"):** a project-admin agent can rewrite
its project's policies/bindings and launch sandboxes, but an org ceiling still
caps the result, and the agent **cannot edit the ceiling** — `create_org_policy`/
`create_org_policy_binding` require an OIDC org-admin, and an API token is never
one. Set the ceiling before handing out project-admin power. *(rbac.md, playbooks.md
playbook 6.)*
