# Access — you administer as an org admin

When this skill is active you operate as a **platform administrator with full
control of your organization**. You get that admin capability one of two ways,
and they behave identically for administration:

- A **human's coding agent** signed in via OIDC (`mcp login`), or
- An **admin-scoped API token** — e.g. the token the platform admin agent
  (Prism/"Odin") is provisioned with.

Either way you can create and manage projects, policies, policy bindings,
credentials, connections (Kubernetes/AWS), MCP connectors, agents/sandboxes,
spending limits, teams, and read audit/usage. **Just do the work** — you don't
need to reason about auth types for normal administration.

## Org creation: do it yourself on OIDC

**Creating a brand-new organization** (`create_org`) needs an **OIDC session** —
which you have on the coding-agent onboarding path. So if `list_orgs` is empty,
**ask the user for an org name and call `create_org` yourself**; do **not** send
them to the web UI to make it. Org creation is only out of reach for an
**API-token** principal (e.g. a provisioned admin agent), which must already
have its org.

## The only things that always need the *human's* own session

- **Personal invitations** — `list_my_invitations` / `accept_invitation` /
  `decline_invitation` act on a specific human's own invitations, so they need
  that person's session no matter how you're authed. (You *can* still
  `invite_org_member`.)

If you hit one of these, ask the user to do it from their own signed-in session;
everything else — org included, on OIDC — do yourself.

## If you're an agent WITHOUT an admin token

A managed agent given only an ordinary (non-admin) API token is limited to
project-scoped work: read policies/credentials/audit/usage, manage clusters and
AWS connections in its project, and run in-sandbox shell tools — but not create
policies/agents/connectors. If an admin action is refused, you're on a
non-admin token: tell the user to provision this agent with an **admin-scoped
token** (mint via `create_api_token` with `orgAdmin: true` from an admin
session) rather than trying to escalate.
