# Agents — launch and configure managed agents

Two halves: **launch** a sandbox (over MCP) and **configure** the agent
running inside it (the agent's own tools / its env). Prism is one such agent
image.

> **Exact parameters aren't here — read the live schema** (`tools/list` /
> `<publicUrl>/v1/openapi.json`). This file covers the semantics.

## Launch — `create_sandbox` and lifecycle
Tools: `list_sandboxes`, `get_sandbox`, `create_sandbox`, `update_sandbox`,
`start_sandbox`, `stop_sandbox`, `delete_sandbox`, `list_sandbox_revisions`,
`rollback_sandbox`, `get_sandbox_network_clip`.

Semantics that bite:
- **Policies attach inline at create time** (`policies: [...]` on the sandbox) —
  this is how the agent gets egress, credentials, integrations, and managed
  inference. It's a frozen attachment, not a `policy_bindings` row.
- **Caps: one exposed port, one persistent volume.** `exposedPorts[].auth`:
  `public` = openable without a platform session (handy for a trial chat UI);
  `private` = requires OIDC.
- **`cpu` and `memory` are required** (K8s quantities, e.g. `cpu: "1"` / `"500m"`,
  `memory: "2Gi"` / `"512Mi"`) and must **not exceed** the platform's
  `SANDBOX_CPU` / `SANDBOX_MEMORY` ceilings (default `1` / `2Gi`). `update_sandbox`
  can change them — that creates a new revision and restarts the sandbox.
- Image must have `/bin/sh` + a writable CA bundle; `FROM scratch` / non-debug
  distroless are **unsupported** (see `gotchas.md`).
- After create, **poll `get_sandbox`** until `state` is `running` and
  `exposedPorts[0].url` is populated — that URL is the agent's chat UI.
- **Idle shutdown after 30 min** (platform-managed). `stop_sandbox` for an
  immediate cutoff (e.g. after revoking access — revocation alone doesn't kill a
  live session).

## Prism launch env (set in `create_sandbox` `env`)
The container is one agent = one `AGENT_ID` = one `/data` SQLite DB = one LLM
provider. Image `ghcr.io/lensapp/prism-agent:latest`, command `exec ./start.sh`,
`/data` volume, chat port `3003`.

- **Identity (first boot only — ignored on restarts of a reused `/data`):**
  `AGENT_ID`, `AGENT_NAME` (default `Prism`), `TEAM_NAME`.
- **Provider:** `LLM_PROVIDER` = `bedrock` (default) or `azure` — **a typo
  throws at boot.** Must match the policy's `managedInference.provider` and the
  platform install.

That's all you set. Everything else the agent needs at runtime — managed
inference, MCP tools, credentials (incl. Slack tokens), and the platform
connection URLs — comes from its **policy** and is injected by the platform; don't
put it in `env`. (Slack tokens → `playbooks.md` playbook 7; project-admin "Odin" →
`playbooks.md` playbook 6.)

## Configure the running agent (the agent's own tools)
A managed agent's behavior is **seven workspace files**, all loaded every
invocation. Most are set/edited at **runtime** by the agent itself (or by asking
it) via its config tools — not at `create_sandbox` time:

| File | What it is | Runtime setter |
|------|-----------|----------------|
| Soul | personality/values/boundaries | `update_soul` |
| User Profile | learned user/team context | `update_user_profile` |
| Memory Summary | always-in-context curated knowledge | `update_memory` |
| Vision | goals to advance | `update_vision` |
| Heartbeat | monitoring instructions | `update_heartbeat_config` (restarts the timer) |
| **Agent Guide** | operating manual + safety rules | **user-only — no setter** (hard boundary) |
| Bootstrap | first-run onboarding | `clear_bootstrap` |

Plus `rename_self` (name) and `update_settings` (`heartbeatIntervalMs` ≥60000,
`timezone`). To make an agent present a specific persona, set `AGENT_NAME` at
create time and/or have it call `rename_self` + `update_soul`.

Capabilities to know when configuring an agent:
- **Heartbeat** — always-on monitoring; 8 h default, 3-stage delivery gate
  (empty → 24 h exact-dup suppression → Haiku DELIVER/SUPPRESS, fail-open).
  Slack-triggered heartbeats fire from a trigger channel.
- **Scheduling** — one-shot / interval / daily / cron via `schedule_task`; a
  scheduled run is **restricted** (memory + infra tools only; can't edit
  workspace files, change settings, schedule, or delegate).
- **Memory** — two layers (searchable DB + the always-in-context summary);
  `memory_write` / `memory_search` / `memory_delete`.
- **Autonomy** — levels 1–5 (default 3). Remember: autonomy is prompt-layer
  (what it *decides*); policy/sandbox are the hard limit (what it *can*). See
  `concepts.md`.
- **Subagents** — delegate via the `Task` tool; subagents are fully isolated
  (own token/workspace/memory/audit), **flat hierarchy** (no sub-subagents),
  same-team.
- **Channels** — WebSocket web chat + Slack (personal-DM or team-channel per
  agent). For a managed agent you connect Slack **per agent** by injecting its
  own `SLACK_BOT_TOKEN`/`SLACK_APP_TOKEN` via a Slack policy (Socket Mode, two
  tokens) — full runbook in `playbooks.md` playbook 7.

## Seeding a skill (e.g. this one) into a spawned agent — no code, no rebuild
Skills load from `<DATA>/skills` (=`/data/skills`) and hot-reload on directory
mtime. Get `lens-agents-admin/SKILL.md` (+ `references/`) there one of two ways:
- **Pre-seed** the `/data` volume at create time (you own the volume then).
- **Ask the agent to install it itself** — the reliable runtime path. The platform
  `shell_*` tools (`shell_exec`/`shell_write_file`/`shell_claude_code`) run in a
  **fresh sandbox as the caller, not inside the target agent's container**, so you
  can't write *its* `/data` with them. Instead talk to the agent over its chat and
  ask it to install the skill from its repo link, e.g.
  `https://github.com/lensapp/lens-agents-admin-skill` (via `npx skills add <source> --copy`
  or a `git clone`/curl into `$PRISM_DATA_DIR/skills`). Add the egress its install
  path needs to the agent's policy `allowedDomains` first (public repo — domains
  only, no credential): `git clone`/tarball needs `github.com` + `codeload.github.com`
  (and `raw.githubusercontent.com`); `npx skills add` additionally needs
  `registry.npmjs.org`. Picked up on the next turn, no restart.

(A `SKILL.md` that is itself a symlink is skipped.)

## Give it a first goal
After launch, hand the agent its first task over **its own chat UI / WebSocket**
(e.g. "you are the SRE agent for `prod-eks`; watch for failing pods and report").
Note the platform `shell_*` tools won't do this — they run in a fresh sandbox as
the caller, not inside the launched agent's container — so drive the agent through
its chat, not through `shell_claude_code`.
