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

Limits form a **4-level stack** — org, user, agent, sandbox — and **the
most-restrictive applicable limit wins**. `actorType: agent` targets an API
token's id; `actorType: sandbox` targets one specific sandbox.

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
