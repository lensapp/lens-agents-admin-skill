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

## The only things that need a human OIDC session

Two narrow, human-identity operations are not available to an API token (they
aren't platform administration):

- **Creating a brand-new organization** (`create_org`) — an org is your tenant;
  it's set up during human onboarding.
- **Personal invitations** — `list_my_invitations` / `accept_invitation` /
  `decline_invitation` act on a specific human's own invitations. (You *can*
  still `invite_org_member`.)

If you ever hit one of these, ask the user to do it from their own signed-in
session; everything else, do yourself.

## If you're an agent WITHOUT an admin token

A managed agent given only an ordinary (non-admin) API token is limited to
project-scoped work: read policies/credentials/audit/usage, manage clusters and
AWS connections in its project, and run in-sandbox shell tools — but not create
policies/agents/connectors. If an admin action is refused, you're on a
non-admin token: tell the user to provision this agent with an **admin-scoped
token** (mint via `create_api_token` with `orgAdmin: true` from an admin
session) rather than trying to escalate.
