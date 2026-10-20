# Agents — launch and configure managed agents

Two halves: **launch** a sandbox (over MCP) and **configure** the agent
running inside it (the agent's own tools / its env). Prism is one such agent
image.

> **Exact parameters aren't here — read the live schema** (`tools/list` /
> `<publicUrl>/v1/openapi.json`). This file covers the semantics.

## Launch — `create_sandbox` and lifecycle
Tools: `list_sandboxes`, `get_sandbox`, `create_sandbox`, `update_sandbox`,
`start_sandbox`, `stop_sandbox`, `restart_sandbox`, `delete_sandbox`, `list_sandbox_revisions`,
`rollback_sandbox`, `update_sandbox_image`, `get_sandbox_effective_policy`.

Semantics that bite:
- **Policies attach to the sandbox itself** (`policyIds: [...]` — the field was
  renamed from `policies`, which is now rejected) — this is how the agent gets
  egress, credentials, integrations, and managed inference. These are live
  references to shared policies, not a `policy_bindings` row; `update_sandbox
  { policyIds }` replaces them without a restart. A sandbox can also carry its
  own embedded `policy` and `credentials`, created in the same call.
- **Caps: up to four exposed ports, one persistent volume.** `exposedPorts[].auth`:
  `public` = openable without a platform session (handy for a trial chat UI);
  `private` (the default) = requires a platform sign-in with access to the project.
  On a `public` port the caller's own `Authorization` header reaches the app (e.g.
  a bot's JWT); on `private` ports it is stripped.
- **Volume size:** `volumes[].size` (a whole-number quantity, e.g. `"20Gi"`) sets
  the PVC request — **at create only** (`volumes` isn't updatable). Above the
  platform's `SANDBOX_DISK` ceiling → 400 `disk X exceeds the maximum allowed (Y)`;
  omitted = the default size (`SANDBOX_K8S_AGENT_VOLUME_SIZE`, `10Gi`). An
  unknown key in a volume entry → 400. Local Docker ignores it; an existing PVC
  is never resized (a smaller one only logs a warning). CLI: `--volume PATH[:SIZE]`.
- **`cpu` and `memory` are required** (K8s quantities, e.g. `cpu: "500m"`,
  `memory: "2Gi"` / `"512Mi"`) and must **not exceed** the platform's
  `SANDBOX_CPU` / `SANDBOX_MEMORY` ceilings (Helm chart default `500m` / `2Gi`,
  set by `sandbox.k8s.cpu` / `sandbox.k8s.memory`). `update_sandbox`
  can change them — a different value creates a new revision and restarts the
  sandbox (the same value doesn't).
- Image must have `/bin/sh` + a writable CA bundle; `FROM scratch` / non-debug
  distroless are **unsupported** (see `gotchas.md`).
- After create, **poll `get_sandbox`** until `state` is `started` and
  `exposedPorts[0].url` is populated — that URL is the agent's chat UI.
  `completed` means the main command exited 0; a non-zero exit is `error` with
  `exitCode` set (exec still works in both until the sandbox stops). A finished
  main command is **not run again** — not on a new runtime, not by auto-start;
  only an explicit start runs it again.
  `started` means the container runs, not that the agent answers yet: REST
  `GET /v1/projects/{projectId}/sandboxes/{sandboxId}/port-readiness` reports
  `answering`.
- **`restart_sandbox`** always re-provisions the active revision, even on a
  running sandbox; from stopped it behaves like start. Refused while another
  transition is in flight (a concurrent restart gets 409; also refused while
  settling or with a stop pending) — wait and retry. It returns the pre-restart
  state: record `health.startedAt`, then poll `get_sandbox` until `state` is
  `started` and `startedAt` differs; `state: error` is terminal.
- **Lifecycle timers** — all off unless set; metadata-only (no restart); `null`
  clears:
  - `autoStopMinutes` (1–43200, i.e. up to 30 days): stop after that long with no
    activity — an open exec/terminal session or a request to a **private** port.
    Egress and public-port traffic don't count.
  - `autoDeleteMinutes` (0–43200): delete the sandbox **and its volume** that long
    after it was asked to stop (`0` = as soon as it has stopped). A sandbox in
    `error` is never deleted.
  - `autoStart`: an exec, terminal or private-port request to an auto-stopped
    sandbox starts it again and waits for it (60 s by default). A sandbox a person
    stopped stays stopped.
  - Reads show `lastActivityAt`, `stoppedAt`, `autoStopAt`, `autoDeleteAt`. Timers
    are swept once a minute; each stop/delete writes a `sandbox.auto_stop` /
    `sandbox.auto_delete` audit event.
  - CLI: `nexusctl sandbox create|update --auto-stop 30m --auto-delete 1d
    --auto-start` (`update` also takes `--clear-auto-stop`, `--clear-auto-delete`,
    `--clear-auto-start`; durations are `m`/`h`/`d`, or `0`); `nexusctl sandbox
    describe` shows when each timer is due.
- **Spec additions** (same fields on REST and MCP `create_sandbox`/`update_sandbox`):
  - `user` (`USER` or `USER:GROUP`, a name from the image or a numeric id;
    overrides the image's USER — an unknown name stops the start) and `workdir`
    (absolute; created if the image lacks it; default `WORKSPACE_PATH` env, else
    the image's WORKDIR). A change is a new revision + restart; `null` goes back
    to the default (the image's USER; for `workdir`, the default above). CLI
    `--user`/`--workdir`, `--clear-user`/`--clear-workdir`.
  - `files: [{path, content}]` — written (0644, owned by the agent user) on
    **every start**, before the agent runs; the agent's edits are lost at the
    next start. ≤256 files, 128 KiB each, 1 MiB total; absolute paths without
    `.`/`..`, each part ≤255 bytes, not under `/.lens` or `/.nexus-discovery`;
    secret-looking names (`.env*`, `.npmrc`, `id_rsa`, `*.pem`, …) are refused.
  - `scripts: [{when:"pre-start", run, user?, description?}]` — run in order with
    `sh -e` after the files and the network policy, before the agent, on every
    start; one that fails stops the sandbox. ≤32, 128 KiB each, 512 KiB total.
    **Not secret** — returned on reads (secrets go in as credentials). On
    Kubernetes an HTTP health check's `initialDelaySeconds` must outlast them.
  - `tools: ["jq@1.7.1", "node@22.11.0"]` — installed with **mise** before the
    agent starts, on every start, by an `install-tools` step that runs as root in
    the image before the agent's container; their bin directories go **first on
    `PATH`** for the agent, exec sessions and pre-start scripts. `name@version`
    (a mise registry name, no backend prefix; the version may be `latest`), ≤32,
    each name once; only the syntax is checked, so an unknown tool fails at start.
    Prebuilt releases only — compiling is off and npm/cargo/pip-style backends are
    disabled. Read-only to the agent: `npm install -g` into a declared node fails,
    so install into the agent's home instead. A tool that can't be installed (3
    tries) leaves the sandbox in `error` with `Tool install failed: …`; the logs
    show `[tools n/N]` lines. A change is a new revision + restart, `[]` removes
    them, a rollback restores them. Downloads **don't** go through the sandbox
    policy (it isn't in force yet) but through the platform's **tools cache**
    (`install-local.md`, Other values): while it's on but not ready the sandbox
    doesn't start, and a tool from a host off its allow-list fails to install
    (fail-closed).
  - With an image whose supervisor predates files/scripts the sandbox doesn't
    start (its supervisor is refused, "supervisor is too old"; the create/update
    itself succeeds). `list_sandboxes` omits `files`/`scripts` (it does list
    `tools`); `get_sandbox` has them. nexusctl sets files, scripts, `tools` and
    `imageUpdate` only from a spec file (`-f`).
- **Image updates:** `imageUpdate: {auto}` follows the image's tag (it must have
  one and not be pinned to a digest). `imageUpdateStatus` (`current`,
  `available`, `checkedAt`, `checkError`, `heldDigest`) reports a newer digest;
  `auto: true` also rolls it out as a new revision once the sandbox is idle (10
  min) or at the latest an hour after it was found, and a rollout that errors
  within 10 min goes back and holds that digest. Never an older digest; toggling
  it doesn't restart. Take one by hand with `update_sandbox_image { projectId,
  sandboxId, digest }` (REST `POST …/sandboxes/{sandboxId}/update-image`) —
  `digest` must equal `imageUpdateStatus.available` or it's 409; project admin.
  Revisions show `resolvedImage`; automatic ones are `createdBy: system`.
  Tuning is env only (`IMAGE_UPDATE_*`), not Helm.
- **Icon (optional):** REST only, no MCP tool —
  `GET`/`PUT`/`DELETE /v1/projects/{projectId}/sandboxes/{sandboxId}/icon`. `PUT`
  takes the **raw image** as the body (PNG, JPEG, WebP, GIF or AVIF — not SVG;
  ≤10 MiB, ≤50 megapixels); the platform centre-crops it to a square, scales it
  to at most 1024 px and stores WebP (first frame only). Project ADMIN sets or
  removes it; anyone with project access reads it. Metadata only: no revision, no
  restart. Reads carry `iconHash`; `?v=<iconHash>` makes the answer cacheable. The
  web UI edits it in the sandbox's settings; the Lens Agents client shows it as
  the agent's icon.
- **Logs:** REST `GET /v1/projects/{projectId}/sandboxes/{sandboxId}/logs`
  (`tailLines`, default 500, max 10000; `follow`) is a server-sent-events stream —
  `log` frames `{text}`, ending with `end {reason: eof|max-duration}` or `error`;
  keepalive every 15 s; a follow ends after 60 min. Project members only, never a
  sandbox token; **409** when there's no current runtime, and only the current
  runtime's logs exist. **No MCP tool** — CLI `nexusctl sandbox logs <sandbox>
  --project <p> [-f] [--tail N]` (`N` 1–10000), or the sandbox page's **Logs** tab.
- **Without a timer, a sandbox runs until stopped.** `stop_sandbox` for an
  immediate cutoff (e.g. after revoking access — revocation alone doesn't kill a
  running process).

## Prism launch env (set in `create_sandbox` `env`)
The container is one agent = one `AGENT_ID` = one `/data` SQLite DB = one LLM
provider. Image `ghcr.io/lensapp/prism-agent:latest`, command `exec ./start.sh`,
`/data` volume, chat port `3003`, and `imageUpdate: {auto: true}` so it picks up
new `:latest` releases while idle (the web UI template does this; it replaced
`EXIT_ON_UPDATE`, and existing Prism sandboxes with `EXIT_ON_UPDATE=1` were
switched over on upgrade).

- **Identity (first boot only — ignored on restarts of a reused `/data`):**
  `AGENT_ID`, `AGENT_NAME` (default `Prism`), `TEAM_NAME`.
- **Provider:** `LLM_PROVIDER` = `bedrock` (default), `azure`,
  `bedrock-mantle`, `openai`, or `openrouter`. **A typo does not throw — it
  silently falls back to `bedrock`**, so a misspelt value looks like a provider
  mismatch, not a config error. Must match the policy's
  `managedInference.provider` and the platform install.

That's all you set. Everything else the agent needs at runtime — managed
inference, MCP tools, credentials (incl. Slack tokens), and the platform
connection URLs — comes from its **policy** and is injected by the platform; don't
put it in `env`. (Slack tokens → `playbooks.md` playbook 7; project-admin "Odin" →
`playbooks.md` playbook 6.)

**Microsoft Teams (web UI Prism template):** the template can also connect the
agent to Teams. It asks for a single-tenant Teams app's ID and tenant ID (set
as `TEAMS_APP_ID` / `TEAMS_APP_TENANT_ID`) and its client secret, stored as an
OAuth client-credentials credential (`TEAMS_APP_PASSWORD`, token from
`login.microsoftonline.com`, see `credentials.md`), and exposes a public port
`teams` on `3979` for the bot endpoint.

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
- **Ask the agent to install it itself** — the reliable runtime path. No platform
  tool writes into another agent's container, so you can't write *its* `/data`
  over MCP. Instead talk to the agent over its chat and ask it to install the skill from its repo link, e.g.
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
No platform tool reaches inside the launched agent's container, so drive the
agent through its chat.
