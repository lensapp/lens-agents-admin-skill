# Tenancy — orgs, teams, projects, membership, API tokens

> **Exact parameters aren't here — read the live schema.** For any tool, use `tools/list` (automatic over MCP) or the REST OpenAPI at `<publicUrl>/v1/openapi.json` (`/v1/docs` for the UI). This file covers what those can't: what the tools are for and the non-obvious rules.

The tenancy spine is **org → team → project**. Projects hold the resources
(policies, credentials, connections, sandboxes). Teams are granted access to
projects; users and API tokens act within the projects their team reaches.

## Organizations

Tools: `list_orgs`, `get_org`, `create_org`, `update_org`, `delete_org`,
`remove_org_member`.

Call `list_orgs` and use the org it returns. **If it's empty** (common right
after a fresh install/activation), **create the org yourself**: ask the user for
an org name, then `create_org { name }`, and continue with its id — don't punt
to the web UI. `create_org` works on an **OIDC session** (the coding-agent
onboarding path), because an org is a human's tenant; it's the one creation an
**API-token** principal can't do (an admin agent must have its org already).

## Projects

Tools: `list_projects`, `get_project`, `get_project_public_key`,
`create_project`, `update_project`, `delete_project`, `rotate_project_keys`.

`create_project`'s `name` is a unique slug within the org.

## Teams

Tools: `list_teams`, `get_team`, `create_team`, `update_team`, `delete_team`,
`add_team_member`, `remove_team_member`, `set_team_project_access`,
`remove_team_project_access`.

Teams are how a **user or API token gains project access and a project role**
(`ADMIN`/`MEMBER`). (An admin-scoped token is already org-admin, so it has
project ADMIN everywhere without team membership.)

## Invitations

`invite_org_member` — add someone to the org.
`list_my_invitations`, `accept_invitation`, `decline_invitation` — these act on
a human's own invitations, so they need that human's OIDC session.

## API tokens

Tools: `list_api_tokens`, `create_api_token`, `revoke_api_token`. A created
token's value is shown **once** — store it immediately.

An API token is a **bearer credential for a non-human principal** (an external
agent, or a managed agent you provision). Set **`orgAdmin: true`** to mint an
**admin-scoped token** — this is how you provision a platform admin agent
(Prism/"Odin") that can administer the platform. A default token
(`orgAdmin` omitted/false) is project-scoped: read/observe + cluster/AWS/shell,
plus whatever its team grants. Only an admin session can mint tokens.
