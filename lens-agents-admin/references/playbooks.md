# Playbooks — ordered recipes

Run these as an admin (an OIDC session, or an admin-scoped API token). Confirm
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

1. Connect the cluster: `create_cluster { projectId, name: "prod-eks", displayName, relayUrl }`. *(connections.md)*
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

## After any change

Tell the user what to verify in the **admin UI** (the new policy/binding, the
sandbox `state`, the audit entry) — the UI is their validation/observation
surface; you drive changes over MCP.
