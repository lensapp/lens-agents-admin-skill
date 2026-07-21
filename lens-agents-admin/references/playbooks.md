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
5. `create_sandbox { projectId, name: "prism-demo", image: "ghcr.io/lensapp/prism-agent:latest", command: "exec ./start.sh", env: {...}, volumes: [{mountPath:"/data"}], exposedPorts: [{name:"chat",port:3003,auth:"public"}], policies: ["<policyId>"] }`. *(agents.md)*
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
4. `create_sandbox { ..., policies:["<policyId>"] }`; wait for `running`. *(agents.md)*
5. Give it its first goal over chat / `shell_claude_code` (e.g. "watch prod-eks for failing pods and report"). *(agents.md)*

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
token through a **self-reference MCP connector**: the platform dispatches that
connector's first-party tools as the token's principal, so Odin sees
`create_policy`, `create_sandbox`, etc. as **native tools** — the token stays
server-side (encrypted), **never in Odin's env**. Needs an admin who can mint
tokens + grant team access (OIDC org-admin). Ask **which project(s)** Odin manages.

1. For each managed project, ensure a team with **project-ADMIN** role:
   `create_team { orgId, name }` → `set_team_project_access { teamId, projectId, role:"ADMIN" }`. *(tenancy.md)*
2. `create_api_token { orgId, name:"odin-<scope>" }` → capture the raw token **once**.
   **Scope it least-privilege** — admin on only the projects Odin should manage
   (Odin's authority *is* this token's scope). *(tenancy.md)*
3. `add_team_member { teamId, memberType:"AGENT", apiTokenId:"<id>" }` for each
   managed project's team → the token is now project-admin there.
4. Pick a **home project** for Odin's sandbox; register the self-reference connector:
   `create_mcp_server { projectId:<home>, name:"odin-admin", url:"<publicUrl>/mcp", transport:"streamable-http" }`. *(mcp-connectors.md — this points at the platform's own /mcp.)*
5. `create_mcp_server_credential { serverId, authType:"static", value:"<token>" }` —
   stores the token encrypted, server-side.
6. `create_policy { projectId:<home>, managedInference:{enabled:true, provider:"..."}, connectors:[{ connectorId:"<odin-admin>", allowedTools:[<admin tools>], credentialId:"<cred>" }] }`
   → `create_policy_binding`. The binding's `credentialId` is what makes the
   connector run its calls as the token's principal. *(policies.md)*
7. `create_sandbox { projectId:<home>, name:"odin", image:"ghcr.io/lensapp/prism-agent:latest", command:"exec ./start.sh", env:{ LLM_PROVIDER:"...", NEXUS_API_URL:"..." }, volumes:[{mountPath:"/data"}], exposedPorts:[{name:"chat",port:3003,auth:"public"}], policies:["<homePolicyId>"] }`; poll `get_sandbox` for the chat URL. *(agents.md)*
8. Seed the skill so Odin knows it's an admin: drop this bundle into
   `/data/skills/lens-agents-admin/` — `shell_exec` `npx skills add https://github.com/lensapp/lens-agents-admin-skill/tree/main/lens-agents-admin -g -a claude-code --copy` (or a curl+untar); **not** `shell_write_file` (workspace-bounded). Give it an admin persona via `rename_self`/`update_soul` or `AGENT_NAME`. *(agents.md)*
9. **Wrap up:** hand the user Odin's chat URL + the platform web UI; state its
   scope (**exactly its token's projects** — project-admin, never org-admin) and
   the kill switch (`revoke_api_token` disables it immediately).

## After any change

Tell the user what to verify in the **admin UI** (the new policy/binding, the
sandbox `state`, the audit entry) — the UI is their validation/observation
surface; you drive changes over MCP.
