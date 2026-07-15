# lens-agents-admin skill

An [Agent Skill](https://agentskills.io) that turns any coding agent (Claude
Code, Codex, OpenCode, …, or a Prism agent) into a **Lens Agents platform
admin** — grounded in the platform's MCP admin surface so it can install, set
up, and govern the platform (projects, policies, connections, agents, spending,
audit) without you knowing the internals.

The skill lives in [`lens-agents-admin/`](lens-agents-admin/): the agent-facing
runbook is [`SKILL.md`](lens-agents-admin/SKILL.md), with deep detail in
[`references/`](lens-agents-admin/references/). Humans don't need to read those.

## Quick start — paste this into your coding agent

Works in **any** Agent Skills client (Claude Code, Codex, OpenCode, … — CLI or
desktop app). Just paste this prompt; the agent installs the skill itself and
follows it:

> **Install the Lens Agents admin skill from https://github.com/lensapp/lens-agents-admin-skill and follow it to set up Lens Agents.**

The agent installs this skill (so it's reusable — your ongoing Lens Agents
admin) and follows `SKILL.md` to stand up the platform and launch a first agent
(Prism), handing you both URLs at the end.

### If you are that coding agent

1. **Install this skill so it's reusable** — run in a terminal:
   `npx skills add lensapp/lens-agents-admin-skill -g -a <your-agent> --copy`
   (`-a claude-code` | `codex` | `cursor` | …; `-g` = user-level, discovered by
   both the CLI and the desktop app). **Can't install it?** Just read
   `lens-agents-admin/SKILL.md` + `lens-agents-admin/references/*.md` from this
   repo and follow them directly.
2. **Then read `lens-agents-admin/SKILL.md` and follow it.** For an install
   request it walks minikube → Helm → activate → onboard, ending with a summary
   and both URLs. Run the install yourself (or read the files directly) — don't
   just paste the `npx` command back into the user's chat.

Then connect the agent to your platform's MCP and sign in (OIDC gives it full
admin — no API token to manage):

```bash
# Claude Code
claude mcp add --transport http lens-agents <publicUrl>/mcp   # e.g. http://localhost:3002/mcp
claude mcp login lens-agents

# Codex
codex mcp add lens-agents --url <publicUrl>/mcp
codex mcp login lens-agents
```

Complete the sign-in in your **system browser** (not an embedded one). Now just
talk to the agent: *"set up my Lens Agents platform"*, *"give me a Kubernetes SRE
agent"*, *"integrate our reservations system over MCP"*.

### Staying up to date

Installed skills are local copies — they don't auto-update. To refresh to the
latest version, re-run the update:

```bash
npx skills update lens-agents-admin      # or: npx skills add lensapp/lens-agents-admin-skill -a <agent> --copy
```

If the agent's guidance ever looks out of step with the platform, that's the fix.

> The agent administers the platform **as you** (your OIDC session). The web
> admin UI at `<publicUrl>` remains — use it to validate and observe what the
> agent did.

## Seeding a Prism agent with this skill (no code, no image rebuild)

A managed Prism agent can carry this skill too, so it can explain the platform
and do its project-scoped work. Drop the bundle into the sandbox's skills dir —
the catalog hot-reloads on the next turn (no restart):

- **Via the platform (recommended):** `mcp__lens-agents__shell_write_file` the
  bundle into `/data/skills/lens-agents-admin/` in the running sandbox, or seed
  the `/data` volume before launch.
- **Or** point `PRISM_SKILLS_DIR` at a pre-seeded mount.

Give it an admin persona (no code) with the agent's own tools: `rename_self`
("Lens Admin") + `update_soul`, or set `AGENT_NAME` at `create_sandbox` time.

> **Scope, honestly.** A seeded Prism can *explain* the platform and do its
> project-scoped work today, but a sandbox's default connection is the
> **project-scoped** MCP endpoint with a sandbox identity — which exposes **no**
> admin tools. So a sandboxed Prism is **not** a full platform admin out of the
> box, even with the skill. Full in-sandbox administration is a deliberate
> platform feature (the privileged **admin agent**): it needs (1) the platform's
> API-token org-admin change, (2) prism-agent MCP-token plumbing, and (3) a
> provisioning path that points that one agent at the **global** `/mcp` with an
> admin-scoped token. Until that ships, **use a coding agent (OIDC) as your
> admin** (top of this README) — it has the full surface today.

## Philosophy

This skill does **not** re-document tool parameters — an agent already gets
authoritative schemas from `tools/list` (over MCP) and the REST OpenAPI at
`<publicUrl>/v1/openapi.json`. The skill carries what those specs **can't**: the
platform mental model, cross-cutting rules, gotchas, correct sequencing, the
install path, and how to configure the managed agents it launches. So it ages
well (no parameter drift).

## Layout

The skill lives in the [`lens-agents-admin/`](lens-agents-admin/) directory
(named to match the skill's `name`, per the [agentskills.io spec](https://agentskills.io/specification)
— this is what makes `npx skills add` copy the whole bundle, references
included, rather than just `SKILL.md`):

```
lens-agents-admin/
  SKILL.md            # runbook + router: mental model, connect, "read the live schema", reference map, rules
  references/
  concepts.md         # the WHY — governance model, Mode 1/2, autonomy-vs-policy, honest limitations
  architecture.md     # internals: two MCP endpoints, sandbox-as-principal, the two proxies, network policy, TTLs
  install-local.md    # stand up the whole platform on minikube (Bedrock/Azure), then onboard
  rbac.md             # who administers (OIDC / admin-scoped token) — read before acting
  tenancy.md          # orgs, teams, projects, membership, API tokens
  policies.md         # policy + binding semantics (ceiling/clip/drift, people-vs-sandbox axes)
  inference.md        # managed inference: backends, metering boundary, PII fail modes, env traps
  credentials.md      # credential injection model (decoy/boundary)
  connections.md      # Kubernetes clusters, AWS connections
  mcp-connectors.md   # upstream MCP servers, their credentials, HTTP connectors
  agents.md           # launch AND configure a managed (Prism) agent
  governance.md       # spending limits, usage/cost, audit trail
  gotchas.md          # fail-open/closed cheat-sheet, thresholds, caps, reserved names
  playbooks.md        # ordered recipes for the common jobs
```

Facts trace to the platform docs + source; if the platform changes, re-sync this
skill (params stay current automatically via the live specs).
