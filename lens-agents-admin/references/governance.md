# Governance — spending, usage, audit

> **Exact parameters aren't here — read the live schema.** For any tool, use `tools/list` (automatic over MCP) or the REST OpenAPI at `<publicUrl>/v1/openapi.json` (`/v1/docs` for the UI). This file covers what those can't: what the tools are for and the non-obvious rules.

## Spending limits

Cap managed-inference spend per scope. When a limit is hit, the next LLM
request is rejected (HTTP 429, RFC 7807 `budget-exceeded` problem detail);
heartbeats skip gracefully instead of erroring.

Tools: `list_spending_limits`, `get_spending_limit_status`,
`set_spending_limit`, `remove_spending_limit`.
CLI: `nexusctl spending-limit set --org <org> --actor-type … --period day|week|month
--limit-cents N`; per sandbox, `nexusctl sandbox set-limit <sandbox> --period …
--limit-cents N` and `nexusctl sandbox spend <sandbox>`. **Since 0.34 those two
take no `--org`**: the org comes from `--project` (or `<org>/<project>/<sandbox>`,
or the context, see `tenancy.md`).

**Visibility:** the three *read* tools — `get_usage_cost_summary`,
`get_usage_cost_timeseries`, `get_spending_limit_status` — are visible to `oidc`,
`api-token`, **and `sandbox`** principals; a sandbox call is **self-scoped** —
clamped to its own sandboxId within its own project regardless of any supplied
`actorType`/`actorId`/`projectId`/`orgId` (a spoofed org can't win; a foreign org
is denied), and `get_spending_limit_status` returns only the limits that gate that
sandbox (org, its project, its own row). `list_spending_limits` (a read) and the
**mutations** `set_spending_limit`, `remove_spending_limit` are **OIDC-only** (not
visible to `sandbox` or `api-token`). So a managed agent (e.g. Prism) can watch its own budget
without being made a project admin.

A sandbox's status read includes its **project ceiling** as well as its own row
and the org's, because that ceiling is part of the budget it actually has. That
does mean a sandbox can see project-aggregate spend, not only its own.

Being OIDC-typed is **not** enough to mutate a limit. A user-issued sandbox
token (`POST /v1/projects/{projectId}/sandbox-tokens`) replays its creator's whole OIDC context, and an API token can be a project
ADMIN — either would let the capped party lift its own cap. Mutations
therefore require a session that is not sandbox-mediated; status reads are
unaffected (`list_spending_limits` is refused from inside a sandbox).

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
project) and `get_usage_cost_timeseries` (cost over time; with `groupBy`, each
bucket also carries `groupTokens`, the total tokens per group — the org
Overview's average cost-per-token chart).

Where a backend reports no cost of its own, spend is priced from the platform's
model price table. A model id is priced at the rate most of the providers listing
it agree on, not at one reseller's rate. Since 0.34 this also covers bare ids (no
`vendor/` prefix), e.g. a dated Claude id served through an OpenAI-compatible
gateway, which had been priced at about a seventh of list with cache tokens at
nothing. Rates that differ only by float noise count as one, and a listing whose
every rate is 0 is skipped, so a paid model is no longer metered as free (ids
named free, such as `:free`, keep their 0). Cost figures and limit counts can
therefore differ from earlier releases, mostly upwards. LiteLLM spend is
unaffected: it uses only the cost LiteLLM reports.

## Audit trail

Every governed action (REST, MCP tool call, proxy decision) is recorded.

Tools: `query_audit_trail`, `get_audit_stats` (adds a `breakdowns`
dimension), `get_audit_timeseries` (requires an `interval`, optional
`groupBy`).

Use the audit trail to *validate* what you (or another agent) did — pair
every change with "check the admin UI's audit view / run
`query_audit_trail`."
