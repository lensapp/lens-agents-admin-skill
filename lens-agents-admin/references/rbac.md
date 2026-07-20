# Access — who can administer, and how far

Administration has **two tiers**. Know which one you're on — it decides what you
can do and what will be refused.

## Org admin — a human's OIDC session (full control)

A **human's coding agent signed in via OIDC** (`mcp login`) whose account is an
org admin has **full control of the org**: create/manage projects, policies,
bindings, credentials, connections (Kubernetes/AWS), MCP connectors,
agents/sandboxes, spending, teams, and API tokens, plus read audit/usage — and
the org-scoped operations that nothing else can do. This is the onboarding path;
**just do the work**.

## Project admin — an API token on a team with project-ADMIN role

An **API token is never an org admin**, but it *can* administer a **project** it
holds ADMIN role on. Add the token to a team and set that team's project role to
**ADMIN** (`set_team_project_access`); then, over the **global `/mcp`**, it can
create/update/delete that project's **policies, credentials, sandboxes, and
project-scoped policy bindings** (shipped in NEXUS-96). This is how you provision
a **per-project admin agent** ("Odin") that runs without a human session.

What a project-admin token **cannot** do — these are org-scoped and **OIDC-only**
(the tools aren't even offered to a token): create orgs/projects, mint/revoke API
tokens, manage teams, write org-level policies/bindings, project lifecycle
(update/delete project, rotate keys), or reach any other project. If one of these
is refused, hand it to a human org-admin session.

## Sandbox principals are MEMBER — unless elevated via a self-ref connector

A managed agent on its **default sandbox identity** is capped at project
**MEMBER** regardless of team role — read/observe, manage clusters/AWS, run
in-sandbox shell tools, but **not** create policies/credentials/sandboxes.

To make a sandbox a **project admin** ("Odin"), give it a **self-reference MCP
connector** (`url: <publicUrl>/mcp`) with a **project-admin API token** attached
as its policy-binding credential. The platform dispatches that connector's
first-party tools as the **token's** api-token principal (not the sandbox), so
Odin gets native admin tools while the token stays server-side — never in its
env. Its authority is exactly the token's scope, so **scope the token
least-privilege**; `revoke_api_token` is the kill switch. Full runbook:
`playbooks.md` playbook 6 (and `agents.md`).

## Org creation: do it yourself on OIDC

**Creating a brand-new organization** (`create_org`) needs an **OIDC session** —
which you have on the coding-agent onboarding path. So if `list_orgs` is empty,
**ask the user for an org name and call `create_org` yourself**; do **not** send
them to the web UI to make it. Org creation is only out of reach for an
**API-token** principal, which must already have its org.

## The only things that always need the *human's* own session

- **Personal invitations** — `list_my_invitations` / `accept_invitation` /
  `decline_invitation` act on a specific human's own invitations, so they need
  that person's session no matter how you're authed. (You *can* still
  `invite_org_member`.)

If you hit one of these, ask the user to do it from their own signed-in session;
everything else — org creation included, on OIDC — do yourself.
