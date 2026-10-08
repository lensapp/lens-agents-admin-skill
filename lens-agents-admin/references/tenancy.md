# Tenancy — orgs, projects, project members, API tokens

> **Exact parameters aren't here — read the live schema.** For any tool, use `tools/list` (automatic over MCP) or the REST OpenAPI at `<publicUrl>/v1/openapi.json` (`/v1/docs` for the UI). This file covers what those can't: what the tools are for and the non-obvious rules.

The tenancy spine is **org → project**. Projects hold the resources
(policies, credentials, connections, sandboxes). A user or API token reaches a
project through a **direct project role** (`ADMIN`/`MEMBER`) — there are no
teams. On upgrade, each team grant became a direct role (the highest any of its
teams gave), and a user who was only in a team became an org MEMBER.

## Organizations

Tools: `list_orgs`, `get_org`, `create_org`, `update_org`, `delete_org`,
`remove_org_member`.

Call `list_orgs` and use the org it returns. **If it's empty** (common right
after a fresh install/activation), **create the org yourself**: ask the user for
an org name, then `create_org { name, displayName }`, and continue with its id — don't punt
to the web UI. `create_org` works on an **OIDC session** (the coding-agent
onboarding path), because an org is a human's tenant; it's the one creation an
**API-token** principal can't do (an admin agent must have its org already).

**Promote or demote an org admin** (REST only, no MCP tool): `PATCH
/v1/orgs/{orgId}/members/{userId}` with `{ "role": "ADMIN" | "MEMBER" }`. Needs an
org admin; the last admin can't be demoted.

**Emergency halt** (REST only, no MCP tool) — the org-wide stop switch:
- `POST /v1/orgs/{orgId}/halt` with `{ "reason": "…" }` (org admin; reason
  required, max 500 chars): refuses new sandbox-mediated egress and requests on
  sandbox tokens (including connector calls), refuses managed inference for
  **every** principal, cuts open inference streams, and pushes a deny-all network
  policy to every sandbox. Sandboxes keep running; nothing is destroyed.
  Engaging an already-halted org is not an error.
- `GET /v1/orgs/{orgId}/halt` (any org member) — whether a halt is in force, plus
  the history.
- `DELETE /v1/orgs/{orgId}/halt` (org admin) — lift it. Halts stay in the history.

## Projects

Tools: `list_projects`, `get_project`, `get_project_public_key`,
`create_project`, `update_project`, `delete_project`, `rotate_project_keys`.

`create_project`'s `name` is a unique slug within the org; `displayName` is
also required.

## Project members

Tools: `list_project_members { projectId }`, `set_project_member { projectId,
userId | apiTokenId, role }`, `remove_project_member { projectId, userId |
apiTokenId }` — give exactly one of `userId` / `apiTokenId`. REST: `GET
/v1/projects/{projectId}/members`, `PUT`/`DELETE`
`…/members/users/{userId}` and `…/members/api-tokens/{apiTokenId}`.

A project role is how a **user or API token gains project access** — for an API
token this is the **only** way it becomes a project admin (there is no
org-admin token). Org admins (OIDC) implicitly have ADMIN on every project
without a membership.

- Setting or removing a member needs an **org-admin OIDC session**: these two
  tools aren't offered to API tokens, and they are refused from a sandbox
  session — even a user-issued sandbox token that replays an org admin's OIDC
  context. `list_project_members` is also offered to API tokens.
- A user must already be an org member (invite first); a token must belong to
  the project's org. `set_project_member` on an existing member changes its role.
- `remove_org_member` also removes that person's project roles in the org.

## Invitations

`invite_org_member` — add someone to the org.
`list_my_invitations`, `accept_invitation`, `decline_invitation` — these act on
a human's own invitations, so they need that human's OIDC session.

## API tokens

Tools: `list_api_tokens`, `create_api_token`, `revoke_api_token`. A created
token's value is shown **once** — store it immediately. Minting/revoking a token
needs an **org-admin (OIDC) session** (these tools aren't offered to a token).

An API token is a **bearer credential for a non-human principal** (an external
agent, or a managed agent you provision). **A token is never an org admin, and
there is no `orgAdmin` flag** — its power comes entirely from its **project
roles**:

- Give the token the **project ADMIN** role
  (`set_project_member { projectId, apiTokenId, role:"ADMIN" }`): it can administer that project — create/
  update/delete its policies, credentials, sandboxes, and project-scoped
  bindings, over the **global `/mcp`**. This is how you provision a per-project
  admin agent ("Odin").
- **Project MEMBER** (`role:"MEMBER"`):
  read/observe, list/get clusters and AWS connections — but not open a sandbox
  terminal (ADMIN only), create policies/sandboxes, or add clusters/AWS connections.

Org-scoped actions (create org/project, mint/revoke tokens, org-level policies &
bindings, project membership, project lifecycle) always require an **org-admin human
(OIDC)**. See `rbac.md`.
