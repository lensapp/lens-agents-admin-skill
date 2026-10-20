# Tenancy — orgs, projects, project members, API tokens, workload identity

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
after a fresh install/enrollment), **create the org yourself**: ask the user for
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

`delete_project` and `delete_org` are refused with **409** while the project or
org still has sandboxes ("…has N sandboxes. Delete them first.", or "…still being
deleted. Try again shortly") — delete the sandboxes first. The web UI's org and
project **Settings** pages rename and delete. The Lens Agents client API can
also delete a workspace (org) or project — the same 409 while agents remain, and
a person's own Personal workspace is refused (403).

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

`invite_org_member { orgId, email, role, projectAccess? }` — add someone to the
org. `projectAccess` is `[{ projectId (uuid), role: "ADMIN"|"MEMBER" }]` (at most
100, each project once) and is for **MEMBER** invitations only — `role: "ADMIN"`
with projects is refused, since an org admin already reaches every project. On
accept the person gets those project roles; a project deleted meanwhile is
skipped, an existing project role is raised but never lowered, and an existing
org member is upgraded to ADMIN but never downgraded.

`update_org_invitation { orgId, invitationId, role?, projectAccess? }` (at least
one; REST `PATCH /v1/orgs/{orgId}/invitations/{invitationId}`) changes a
**pending** invitation. `projectAccess` replaces the whole set; changing `role` to
ADMIN drops the project roles. An accepted or declined invitation gives **409**, a
revoked or unknown one 404; invitations don't expire. Reads list `projectAccess`
with project names.

All invitation tools are **OIDC-only**; invite and update need an org admin and
are refused from a sandbox session. nexusctl `invitation create` takes only
`--email`/`--role` — use MCP, REST or the web UI's invite dialog to give project
roles.

`list_my_invitations`, `accept_invitation`, `decline_invitation` — these act on
a human's own invitations, so they need that human's OIDC session. Answering one
that's already accepted or declined gives 400.

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

## Workload identity (federated API tokens)

A workload — a CI job, a Kubernetes pod — exchanges **its own OIDC token** for a
short-lived API-token session, so there's no stored secret. **No MCP tools**: REST,
`nexusctl` and the web UI only (the org's **Trusted Issuers** page, the project's
**Federated Tokens** page). Recipe: `playbooks.md` playbook 8.

- **Trusted issuers** (org-scoped). Any org member reads them; only a **human org
  admin** adds, changes or removes one (REST `/v1/orgs/{orgId}/trusted-issuers`;
  changing one is REST only). The issuer and JWKS URLs must be `https`. Keys come
  from pinned `jwks`, a `jwksUri`, or OIDC discovery — read once on save, and an
  issuer whose keys can't be read is refused (pin them instead); private or
  symmetric keys are refused. Every issuer needs an `audience` (nexusctl and the UI
  default it to `defaultAudience`, `<publicUrl>/orgs/<orgId>`) and at least one
  `requiredClaims` entry. The issuer URL can't change after creation.
- **Presets** (`nexusctl workload-identity issuer add --preset …`, or the UI's
  issuer type): `github` (`https://token.actions.githubusercontent.com`;
  `repository_id`, `repository_owner_id`), `gitlab` (`https://gitlab.com`;
  `project_id`, `namespace_id`), `buildkite` (`https://agent.buildkite.com`;
  `organization_id`, `pipeline_id`), `google-cloud` (`https://accounts.google.com`;
  `sub`), `kubernetes` (no URL — pass `--issuer`; `sub`).
- **Federated tokens** (project-scoped) are API tokens with no secret and a
  project role, `ADMIN` or `MEMBER`; their names share the org's API-token
  namespace, and `list_api_tokens` doesn't show them. Project members list them;
  a **human project admin** creates or revokes them (revoked, never deleted).
- **Rules** say which workloads may act as a token: an issuer plus claims that
  must **equal** their values exactly — no globs, non-string claims never match;
  the first matching rule wins. A rule must pin every claim the issuer requires,
  and adding a required claim to an issuer is 409 while some rule lacks it. Pin
  ids that can't be re-registered (`repository_id`, not the name) and the branch
  or environment.
- **Exchange:** `POST /v1/workload-identity/token` (no auth) with
  `{subject_token, subject_token_type:"urn:ietf:params:oauth:token-type:jwt", api_token_id}`
  → `{access_token, token_type:"Bearer", expires_in}`; the `access_token` is an
  `lns_` token with the federated token's role. The OIDC token needs `exp` and a
  non-empty `sub` (30 s clock skew). The session lasts
  `workloadIdentity.sessionTtlSeconds` (default 1 h) whatever the OIDC token's own
  expiry, and ends when the token is revoked or its rule or issuer is deleted.
  CLI: `nexusctl workload-identity token --federated-token <id> (--token-file <f> | --audience <aud>)`
  (in GitHub Actions, `--audience` makes it fetch the runner's token itself).
- **Refusals** are always the same 401. A token no rule matches writes an
  `api-token.workload-exchange` failure audit row (at most one per token every 5
  min); audit shows a workload as `<token name> (<sub>)`. Find out why with
  **explain** — any project member, creates nothing: REST
  `POST /v1/projects/{projectId}/workload-identity/explain`,
  `nexusctl workload-identity explain --project <p> --token-file <f|-> [--federated-token <t>]`,
  or **Explain a token** in the UI.
- A minted session, like any API token, can't manage issuers, tokens or rules —
  those need a human OIDC session. `workloadIdentity.enabled: false` turns the
  feature off.
- CLI: `nexusctl workload-identity issuer list|add|remove`,
  `nexusctl federated-token list|create|delete`,
  `nexusctl federated-token rule list|add|remove`.
