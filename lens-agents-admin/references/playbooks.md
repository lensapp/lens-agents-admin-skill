# Playbooks — ordered recipes

Run these as an admin — an OIDC org-admin session, or (for project-scoped steps)
an API token on a team with project-ADMIN role. Confirm
each heavy step with the user first. Substitute `<...>` from earlier outputs.
For exact tool params, read `tools/list` / the OpenAPI spec — these show the
*sequence*, not full payloads.

## 0. No platform yet? Install it first

If nothing is running, follow `install-local.md` (minikube → Helm → activate →
connect to the global `/mcp`), then continue with playbook 1 below.

## 1. Onboard the platform from zero → a running agent

1. `list_orgs` → pick the org id. **If it's empty** (common on a fresh install): you're on an OIDC session here, so **ask the user for an org name and `create_org { name }` yourself**, then use its id — don't send them to the UI. (Only an API-token principal must defer org creation to a human.) *(tenancy.md)*
2. `create_project { orgId, name: "demo", displayName: "Demo" }` → note `projectId`. *(tenancy.md)*
3. `create_policy { projectId, name: "starter", managedInference: { enabled: true, provider: "bedrock" } }` → note `policyId`. *(policies.md — match the provider the platform was installed with: bedrock | azure | bedrock-mantle)*
4. `create_policy_binding { projectId, name: "starter-sandboxes", policyIds: ["<policyId>"], subjects: [{ kind: "all_sandboxes" }] }`. *(policies.md)*
5. `create_sandbox { projectId, name: "prism-demo", image: "ghcr.io/lensapp/prism-agent:latest", command: "exec ./start.sh", cpu: "1", memory: "2Gi", env: {...}, volumes: [{mountPath:"/data"}], exposedPorts: [{name:"chat",port:3003,auth:"public"}], policies: ["<policyId>"] }`. *(agents.md — `cpu`/`memory` required, ≤ the SANDBOX_CPU/SANDBOX_MEMORY ceiling)*
6. Poll `get_sandbox` until `state=running` and `exposedPorts[0].url` is set.
7. **Wrap up — always end an install/onboard with a plain-language summary.** Tell the user:
   - **What you installed:** the Lens Agents platform + a running **Prism** agent (its first managed agent).
   - **How to reach each, and what each is for:** the **Lens Agents web UI** (the `config.publicUrl`, e.g. `http://localhost:3002`) to *govern the platform* — projects, policies, agents, audit — vs. the **Prism chat UI** (`exposedPorts[0].url`) to *talk to the agent*. Name this platform-vs-agent distinction explicitly; it's the #1 point of confusion.
   - **What's next:** e.g. connect a Kubernetes cluster or other integration, tighten policies, or spin up more agents — and that they can just **ask you (their admin agent)** to do any of it, or use the web UI to watch what you did.

## 2. "I want a Kubernetes SRE agent"

1. Connect the cluster — **three steps**: `create_cluster { projectId, name: "prod-eks", displayName, relayUrl: "tunnel://" }` (capture the one-time `tunnelToken`) → **deploy the relay into that cluster** via the `nexus-kube-relay` Helm chart → **apply cluster RBAC** binding a Role to the impersonated identity (group `<org>/<team>` or user `agent:<tokenName>`), or the agent authenticates but can do nothing. Full runbook incl. the local-trial in-cluster tunnel URL (`ws://localhost:...` fails from a pod) and the RBAC manifest: *(connections.md)*
2. Write a policy that grants the cluster + needed egress + inference:
   `create_policy { projectId, name: "sre", networkDefaultVerdict: "deny", allowedDomains: [...], integrations: [{type:"kubernetes",name:"prod-eks"}], managedInference: {enabled:true, provider:"..."} }`. *(policies.md)*
3. `create_policy_binding { ..., policyIds:["<policyId>"], subjects:[{kind:"all_sandboxes"}] }`.
4. `create_sandbox { ..., cpu, memory, policies:["<policyId>"] }`; wait for `running`. *(agents.md — `cpu`/`memory` required)*
5. Give it its first goal over **its own chat UI / WS** (e.g. "watch prod-eks for failing pods and report") — not the platform `shell_*` tools, which run in a fresh sandbox as the caller, not inside the agent's container. *(agents.md)*

## 3. "Integrate with <system> over MCP"

1. `create_mcp_server { projectId, name, displayName, transport: "streamable-http", url }`. *(mcp-connectors.md)*
2. If it needs auth: `create_mcp_server_credential { projectId, serverId, name, authType, ... }`.
3. `sync_mcp_server_tools { serverId }` → inspect the discovered tools.
4. Expose to the agent via its policy `connectors: [{ connectorId, allowedTools:[...], credentialId? }]`, then (re)bind. *(policies.md)*
5. The agent now sees the tools at `/projects/:projectId/mcp`.

*(No native MCP server yet? Web-research how to run one, then start from step 1.
For a plain REST+OpenAPI upstream use `create_http_connector` instead.)*

## 4. Give an agent a scoped secret (e.g. GitHub)

1. `create_credential { projectId, name: "github-pat", value, injections: [{ domain:"api.github.com", headerName:"Authorization", headerFormat:"token {value}" }] }`. *(credentials.md)*
2. Reference it + allow the domain in the policy: `credentials:[{credentialName:"github-pat", envVarKey:"GITHUB_TOKEN"}]`, `allowedDomains:[{pattern:"api.github.com", verdict:"allow", transport:"upstream"}]`. *(policies.md)*
3. Bind; the proxy injects the token — the agent never sees it.

## 5. Set budgets and watch spend

1. `set_spending_limit { actorType:"sandbox", actorId:"<sandboxId>", period:"month", limitCents:5000 }`. *(governance.md)*
2. `get_usage_cost_summary` / `get_usage_cost_timeseries` to watch spend.
3. `query_audit_trail { source:"llm-proxy", result:"failure" }` to see budget rejections.

## 6. Provision an "Odin" admin agent (a Prism that administers its projects)

Give a managed Prism **project-admin** power by wiring it a project-admin API
token through the project's built-in **`nexus-api` connector**: the platform
dispatches that connector's first-party tools as the token's principal, so Odin
sees `create_policy`, `create_sandbox`, etc. as **native tools** — the token stays
server-side (encrypted), **never in Odin's env**. Needs an admin who can mint
tokens + grant team access (OIDC org-admin). Ask **which project(s)** Odin manages.

> **Set an org ceiling first (highly recommended).** A project-admin agent can
> write its project's policies/bindings and launch sandboxes, so without a ceiling
> it could grant *its project's sandboxes* broad egress, managed inference, or
> connectors. Cap that with an **org `all_sandboxes` ceiling**:
> `create_org_policy { orgId, name:"org-ceiling", networkDefaultVerdict:"deny", allowedDomains:[…], managedInference:{enabled:true, provider:"…"} }`
> → `create_org_policy_binding { orgId, policyIds:["<ceilingId>"], subjects:[{kind:"all_sandboxes"}] }`.
> The ceiling is **restriction-only** — every project binding is clipped against
> it at runtime (uncovered egress domains dropped, managed inference AND-gated; a
> credential to a forbidden domain is neutralized because its domain is clipped).
> **Per-axis carve-out:** the ceiling only caps an axis it actually populates — the
> example above (domains + inference, no connectors) caps egress and inference but
> leaves the **connector** axis uncapped; add a `connectors:[…]` allow-list to the
> ceiling if you also need to bound which connectors project sandboxes may use.
> (Likewise a default-deny ceiling with *no* domain entries wouldn't restrict
> domains — so keep `allowedDomains` non-empty as above.) Odin **cannot raise or
> remove the ceiling**: org policy/binding writes require an OIDC org-admin, and an
> API token is never one. Use `list_policy_binding_drift` and
> `get_sandbox_network_clip` to see what the ceiling clipped. *(policies.md — note
> the two axes: `all_sandboxes` caps sandboxes, `everyone`/`api_token` caps people.)*

1. For each managed project, create a **dedicated** least-privilege team with
   **project-ADMIN** role — do **not** elevate an existing shared team (it may carry
   many users + throwaway agent tokens you'd be making project admins):
   `create_team { orgId, name:"<scope>-admins" }` → `set_team_project_access { teamId, projectId, role:"ADMIN" }`. *(tenancy.md)*
2. `create_api_token { orgId, name:"odin-<scope>" }` → capture the raw token **once**.
   **Scope it least-privilege** — admin on only the projects Odin should manage
   (Odin's authority *is* this token's scope). *(tenancy.md)*
3. `add_team_member { teamId, memberType:"AGENT", apiTokenId:"<id>" }` for each
   managed project's team → the token is now project-admin there.
4. Pick a **home project** for Odin's sandbox. **Don't create a connector** — use
   the project's reserved **`nexus-api`** system connector (it already points at the
   platform's own `/mcp`): `list_mcp_servers { projectId:<home> }` → grab the
   `nexus-api` server id. *(mcp-connectors.md — `create_mcp_server` refuses to shadow it.)*
5. `create_mcp_server_credential { projectId:<home>, serverId:"<nexus-api id>", name:"odin-<scope>-token", authType:"static", staticValue:"<token>" }` —
   stores the token encrypted, server-side.
6. `create_policy { projectId:<home>, name:"odin-admin", managedInference:{enabled:true, provider:"..."}, connectors:[{ connectorId:"<nexus-api id>", allowedTools:[<admin tools>], credentialId:"<cred id>" }] }`.
   The connector grant's **`credentialId`** is what makes those first-party tools
   run as the token's principal. *(policies.md)*
7. `create_sandbox { projectId:<home>, name:"odin", image:"ghcr.io/lensapp/prism-agent:latest", command:"exec ./start.sh", cpu:"1", memory:"2Gi", env:{ LLM_PROVIDER:"..." }, volumes:[{mountPath:"/data"}], exposedPorts:[{name:"web",port:3003,auth:"private"}], policies:["<homePolicyId>"] }`; poll `get_sandbox` for the chat URL. Attaching the policy **inline** here scopes admin to *this* sandbox — don't bind it `all_sandboxes` or you elevate every sandbox in the project. Prefer `auth:"private"` (requires a platform session to reach the chat) for a project-admin agent; `"public"` only for a throwaway trial. *(agents.md — `cpu`/`memory` required)*
8. Seed the admin skill so Odin knows it's an admin — the `shell_*` tools run in a
   *fresh* sandbox as **you**, not inside Odin's container, so you can't write its
   `/data` from here. Instead **ask Odin over its chat to install the skill itself**,
   giving it the repo link `https://github.com/lensapp/lens-agents-admin-skill`.
   First add the egress its install path needs to Odin's policy `allowedDomains`
   (public repo — just domains, no credential, so this is *not* the playbook-4
   secret recipe): a `git clone`/tarball fetch needs `github.com` + `codeload.github.com`
   (and `raw.githubusercontent.com`); `npx skills add …` additionally needs
   `registry.npmjs.org`. Then give it an admin persona via `rename_self`/`update_soul`
   or `AGENT_NAME`. *(agents.md, policies.md)*
9. **Wrap up:** hand the user Odin's chat URL + the platform web UI; state its
   scope (**exactly its token's projects** — project-admin, never org-admin), that
   an org ceiling bounds what it can grant, and the kill switch (`revoke_api_token`
   disables it immediately; follow with `stop_sandbox` to cut a live session).

## 7. Connect an agent to Slack (two tokens, Socket Mode)

Prism talks to Slack over **Socket Mode** with **two** tokens: a **bot token**
(`xoxb-`, authenticates every Web API call) and an **app-level token** (`xapp-`,
scope `connections:write`, opens the socket). You inject each as a platform
credential (header rewrite on the wire) *and* map it to the env var Prism reads, so
the real token never sits in the sandbox env. Ask the user for both tokens (or set
placeholders and let them fill real values via the UI — the bot token is validated
at boot, so it must be real before the restart in step 5).

1. **In Slack** (`api.slack.com/apps`): create an app → **enable Socket Mode** →
   create an **app-level token** with `connections:write` (this is the `xapp-`) →
   add **bot scopes** `app_mentions:read`, `chat:write`, `channels:history`,
   `groups:history`, `im:history`, `im:read`, `channels:read`, `groups:read`,
   `users:read`, `reactions:write`, `files:read`, `files:write` → subscribe to bot
   events `message.im`, `message.channels`, `message.groups`, `app_mention`,
   `member_joined_channel`, `member_left_channel` → install to the workspace → copy
   the **Bot User OAuth token** (`xoxb-`) and the app-level token (`xapp-`).
   (Slack scope requirements shift; if a call is denied for a missing scope, grant it.)
2. Bot-token credential — inject `Authorization: Bearer {value}` on the Slack hosts,
   whitelisting the Web API methods (POST `/api/<method>`). Two groups:
   - **Adapter core** (the Bolt runtime calls these): `auth.test`, `chat.postMessage`,
     `chat.update`, `conversations.info`, `conversations.replies`, `reactions.add`,
     `users.info`, `files.getUploadURLExternal`, `files.completeUploadExternal` — plus
     the `files.slack.com` rules (`GET /files-pri/**` download, `POST /upload/**`).
   - **Agent-initiated** (add only what the agent itself will call — the adapter does
     **not** call these): `conversations.list`, `conversations.history`,
     `conversations.members`, `conversations.open`, `users.conversations`, `users.list`,
     `chat.postEphemeral`, `chat.delete`, `reactions.remove`, `team.info`, `views.open`,
     `views.update`. (`bots.info` only if you set the optional `SLACK_EXPECTED_APP_ID`.)

   `create_credential { projectId, name:"<agent>-slack-xoxb", value:"<xoxb-…>", injections:[ { domain:"slack.com", headerName:"Authorization", headerFormat:"Bearer {value}", rules:[ POST /api/auth.test, /api/chat.postMessage, /api/chat.update, /api/conversations.info, /api/conversations.replies, /api/reactions.add, /api/users.info, /api/files.getUploadURLExternal, /api/files.completeUploadExternal, …plus any agent-initiated methods above ] }, { domain:"files.slack.com", headerName:"Authorization", headerFormat:"Bearer {value}", rules:[ GET /files-pri/**, POST /upload/** ] } ] }`.
   Least-privilege: grant only the groups the agent needs. If the agent enumerates
   channels/DMs it needs `conversations.list`/`users.conversations` — easy to miss,
   since the adapter never calls them (see `gotchas.md`). *(credentials.md)*
3. App-token credential — one method:
   `create_credential { projectId, name:"<agent>-slack-xapp", value:"<xapp-…>", injections:[ { domain:"slack.com", headerName:"Authorization", headerFormat:"Bearer {value}", rules:[ POST /api/apps.connections.open ] } ] }`. *(credentials.md)*
4. Policy — map both creds to the env vars Prism reads and open Slack egress:
   `create_policy { projectId, name:"<agent>-slack", credentials:[ {credentialName:"<agent>-slack-xoxb", envVarKey:"SLACK_BOT_TOKEN"}, {credentialName:"<agent>-slack-xapp", envVarKey:"SLACK_APP_TOKEN"} ], allowedDomains:[ {pattern:"slack.com", verdict:"allow", transport:"direct"}, {pattern:"*.slack.com", verdict:"allow", transport:"direct"} ] }`.
   `slack.com` + `*.slack.com` covers the Web API, Socket Mode (`wss-*.slack.com`),
   and `files.slack.com` — nothing else is needed. `managedInference` stays off here
   (the agent's main policy provides it). *(policies.md)*
5. Attach the policy **inline** to the agent's sandbox alongside its existing ones:
   `update_sandbox { …, policies:[<existing…>, "<slackPolicyId>"] }`. Attaching is
   metadata-only (no restart), and Prism reads `SLACK_*` **at boot** — so
   `stop_sandbox` then `start_sandbox` to inject the tokens and connect. Make sure the
   **real** bot token is set first: `auth.test` runs at config load and *throws* on a
   bad token (the Socket Mode connect itself is fail-open). *(agents.md)*
6. Verify with `query_audit_trail { source:"sandbox-proxy", actorId:"<sandboxId>" }` —
   `POST slack.com/api/auth.test` **success** (bot token good), `POST
   slack.com/api/apps.connections.open` **success** (app token good), and `GET
   wss-primary.slack.com/link` **success** (socket open) mean it's connected. *(governance.md)*
7. **Link a conversation** — a web-UI flow the *user* runs (the agent's Channels
   page issues a 6-char code). **Personal mode** — DM the bot the code. **Team
   mode** — `/invite` the bot to a channel and post the code there (one channel is
   the ★ main channel for heartbeat alerts). The two modes are mutually exclusive.

> Don't set `SLACK_SKIP_BOT_ID_CHECK` — it's an e2e-test flag that makes the agent
> process other bots' messages. If an **org ceiling** is set (playbook 6), make sure
> `slack.com`/`*.slack.com` are in its allowlist, or Slack egress gets clipped.

## After any change

Tell the user what to verify in the **admin UI** (the new policy/binding, the
sandbox `state`, the audit entry) — the UI is their validation/observation
surface; you drive changes over MCP.
