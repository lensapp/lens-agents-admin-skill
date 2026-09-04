# Governance — spending, usage, audit

> **Exact parameters aren't here — read the live schema.** For any tool, use `tools/list` (automatic over MCP) or the REST OpenAPI at `<publicUrl>/v1/openapi.json` (`/v1/docs` for the UI). This file covers what those can't: what the tools are for and the non-obvious rules.

## Spending limits

Cap managed-inference spend per scope. When a limit is hit, the next LLM
request is rejected (HTTP 429, RFC 7807 `budget-exceeded` problem detail);
heartbeats skip gracefully instead of erroring.

Tools: `list_spending_limits`, `get_spending_limit_status`,
`set_spending_limit`, `remove_spending_limit`.

**Visibility (NEXUS-100):** the three *read* tools — `get_usage_cost_summary`,
`get_usage_cost_timeseries`, `get_spending_limit_status` — are visible to `oidc`,
`api-token`, **and `sandbox`** principals; a sandbox call is **self-scoped** —
clamped to its own sandboxId within its own project regardless of any supplied
`actorType`/`actorId`/`projectId`/`orgId` (a spoofed org can't win; a foreign org
is denied), and `get_spending_limit_status` returns org-level limits plus that
sandbox's own row only. The **mutations** — `list_spending_limits`,
`set_spending_limit`, `remove_spending_limit` — stay **OIDC-only** (not visible to
`sandbox` or `api-token`). So a managed agent (e.g. Prism) can watch its own budget
without being made a project admin.

A sandbox's status read includes its **project ceiling** as well as its own row
and the org's, because that ceiling is part of the budget it actually has. That
does mean a sandbox can see project-aggregate spend, not only its own.

Being OIDC-typed is **not** enough to mutate a limit. A shell sandbox token
replays its creator's whole OIDC context, and an API token can hold team ADMIN
on a project — either would let the capped party lift its own cap. Mutations
therefore require a session that is not sandbox-mediated; reads are unaffected.

Limits form a **5-level stack** — org, project, user, agent, sandbox — and
**every applicable limit gates the request, so the most-restrictive one wins**.
`actorType: agent` targets an API token's id; `actorType: sandbox` targets one
specific sandbox; `actorType: project` targets a project and catches spend by
any principal inside it.

**Who may set which scope** — this is the delegation model, not a detail:

| Scope | Who may set/remove it |
|-------------------|----------------------------------------------------------|
| `org`, `project` | **Org admins only.** These are ceilings; a project's own admins must not be able to raise the bound they were given. |
| `sandbox` | An **admin of that sandbox's project** — the same role that creates and edits the sandbox — or an org admin. |
| `agent`, `user` | **Org admins only.** |

So the intended shape is: an org admin sets the org and per-project ceilings,
then leaves per-sandbox budgeting to the project admins who actually run the
agents. Don't tell a developer to ask for org-admin rights to cap their own
sandbox — they already have the rights, provided they are a project ADMIN and
not merely a MEMBER.

**A per-sandbox cap is a budget, not a containment boundary.** `create_sandbox`
is visible to `oidc` and `api-token` principals and needs only project ADMIN, so
an agent with that reach can start a fresh sandbox — which has no cap, since caps
are opt-in — or destroy and recreate its own to reset the counter. The **project
and org ceilings** are what actually bound it: they count spend by any principal
in scope, so a respawned sandbox lands under the same ceiling. When someone asks
you to bound an agent's spend, set the project ceiling; per-sandbox caps beneath
it are for attribution and early warning.

**A ceiling bounds managed inference, not all spend.** BYO-key LLM traffic
through the forward proxy is metered into the same limits but is not gated by
them, so a limit stops spend only on the paths that route through the managed
inference endpoint. Keep provider hosts denied in policy if a cap has to hold.

## Usage & cost

`get_usage_cost_summary` (totals with breakdowns by model/provider/actor/
project) and `get_usage_cost_timeseries` (cost over time).

## Audit trail

Every governed action (REST, MCP tool call, proxy decision) is recorded.

Tools: `query_audit_trail`, `get_audit_stats` (adds a `breakdowns`
dimension), `get_audit_timeseries` (requires an `interval`, optional
`groupBy`).

Use the audit trail to *validate* what you (or another agent) did — pair
every change with "check the admin UI's audit view / run
`query_audit_trail`."
