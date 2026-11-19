# Install the platform locally (from nothing)

This is the authoritative runbook to stand up the whole Mirantis Agent Platform
on a local single-node **minikube** cluster, then hand off to the onboarding
playbook. **This local trial is the self-serve install** — so when a user just
says "install Mirantis Agent Platform" (no other context), do this, no need to
ask. (A production deployment is a guided engagement; point them to Getting
Started / contact us — see the end of this file.) It's the host-shell phase —
**there is no MCP yet**; the admin `/mcp` only becomes reachable after Step 3
(port-forward) + Step 4 (activation and sign-in setup). Run these as the coding
agent doing the install; pause only for the human's browser steps (the Step 4
Activate screen, Lens ID approval and sign-in, and the Step 5 MCP login).

> **Before you start — get a full-access shell.** Every step below runs real
> **host** commands (Docker/OrbStack, minikube, kubectl, Helm) that need the
> **Docker socket** and **network** (image pulls, the Helm OCI chart,
> the Lens ID sign-in setup). A restricted/read-only or no-network agent sandbox **cannot**
> complete this. **Ask the user once, up front, for full/unsandboxed shell
> permissions** (e.g. Codex: full-access mode) — don't start sandboxed and hit
> the wall mid-install. Two common gotchas: (1) the container runtime may be
> **stopped** — start Docker/OrbStack and wait for its socket before
> `minikube start`; (2) the CLIs may be **off your shell's `PATH`** (e.g.
> OrbStack's `docker` at `/usr/local/bin/docker`) and you may need
> `PATH=/usr/local/bin:$PATH` so minikube's driver check can find `docker` —
> resolve full paths before concluding a tool is missing.

> **Always open sign-in URLs in the user's own system browser**, never an
> embedded/in-agent browser — password managers and the existing Lens ID session
> live in the real browser.

> **Isolation caveat to tell the user:** this local setup is **container-level
> isolation only** (shared host kernel), **not** microVM. Production adds
> per-sandbox microVM isolation (Kata/gVisor runtime class), which minikube
> can't provide.

## Prerequisites

| Tool | Check | Notes |
|------|-------|-------|
| `docker` | `docker info` | minikube's driver (see below) |
| `minikube` | `minikube version` | |
| `kubectl` | `kubectl version --client` | |
| `helm` **v3.14+** | `helm version` | OCI, and `--reset-then-reuse-values` for upgrades |
| `openssl` | `openssl version` | generates the encryption key |
| `curl` | `curl --version` | health check |

Plus: outbound internet, a **Lens ID** account (sign up at `app.k8slens.dev`; it
signs in to the platform through Lens Cloud),
and **one inference-provider credential** — an **OpenRouter** API key (fewest
steps: no cloud account, no IAM, no region), an **AWS Bedrock** API key, an
**Azure (Microsoft Foundry)** resource URL + API key with a Claude deployment,
or an **OpenAI** API key.

> **minikube MUST use the Docker driver.** Sandboxes lock down egress with
> `nftables`, so the node kernel needs `nf_tables`. `--driver=docker` runs the
> node on your host kernel (which has it). minikube's VM drivers (vfkit, qemu,
> hyperkit) ship an ISO kernel **without** `nf_tables`: the platform runs but
> **every sandbox fails to start**. Docker Desktop Kubernetes and `kind` work
> for the same reason. Do not substitute a VM driver.

## Step 1 — Start the cluster (a dedicated, named profile)

> Use a **fresh dedicated profile** `-p lens-agents` — never the default
> `minikube` (this guide's teardown deletes PVCs/profile and could destroy
> workloads you rely on). A coding agent must not reuse/reconfigure/delete any
> other minikube profile. Check `minikube profile list` first.

```bash
minikube start -p lens-agents --driver=docker --cpus=4 --memory=6g
kubectl config use-context lens-agents
kubectl cluster-info
```

## Step 2 — Install the platform (choose an inference provider)

The chart is an OCI artifact on GHCR; it bundles PostgreSQL and defaults its IdP
to Lens ID. Releases also publish it as
`oci://ghcr.io/lensapp/charts/mirantis-agent-platform` — the same chart under
another name, rendering the same resources (still named `lens-agents`), so either
works and a release can move between them with `helm upgrade`. This guide uses
`oci://ghcr.io/lensapp/lens-agents`.

**Managed inference** means the platform holds the provider credential and
proxies every LLM call (metered/gated/audited).

> **Ask the user which credential they hold** before you install, and use that
> block. Do **not** default to Bedrock: with no token and no AWS credentials
> it isn't offered, yet a policy that names it is still accepted — every call
> fails, and the agent looks broken rather than unconfigured.

Three shared flags: `encryption.key` (64 hex, at-rest encryption), `config.publicUrl`,
`sandboxIngress.host=localtest.me` (required for sandbox web UIs — `localtest.me`
is public wildcard DNS resolving `*.localtest.me`→127.0.0.1).

**OpenRouter (fewest steps — one key, ~400 models):**
```bash
helm install lens-agents oci://ghcr.io/lensapp/lens-agents \
  --set encryption.key="$(openssl rand -hex 32)" \
  --set config.publicUrl=http://localhost:3002 \
  --set sandboxIngress.host=localtest.me \
  --set inference.openrouter.token="<openrouter-api-key>" \
  --wait --timeout 10m
```
- The key is all you need. Both wire formats ride it, so a Claude-Code-shaped
  and an OpenAI-SDK-shaped agent both get a managed endpoint, and either can
  drive any model it routes.
- The platform seeds no model: an Anthropic-SDK agent needs `ANTHROPIC_MODEL`
  (`<vendor/model>`) in its policy env.

**AWS Bedrock (needs a key, or an IAM role on EKS):** same three shared flags, plus:
```bash
  --set inference.bedrock.token="<bedrock-api-key>"
```
- No token and no AWS credentials (e.g. an IAM role) = Bedrock isn't offered.

**OpenAI (GPT only):** same three shared flags, plus:
```bash
  --set inference.openai.token="<openai-api-key>"
```
- The key is what makes the provider selectable. OpenAI hosts no Claude, so a
  policy selecting `openai` gets **no** managed Anthropic endpoint.
- Optional `--set inference.openai.baseUrl="https://<host>/<prefix>"` targets any
  OpenAI-compatible endpoint (corporate gateway, self-hosted). Must be
  `https`, and a token must sit beside it or the install fails. For a LiteLLM
  proxy, prefer the dedicated LiteLLM backend below.

**Azure (Claude on Microsoft Foundry):** same three shared flags, plus:
```bash
  --set inference.azure.baseUrl="https://<resource>.services.ai.azure.com" \
  --set inference.azure.token="<azure-api-key>"
```
- `inference.azure.baseUrl` = the Foundry **resource root** on
  `*.services.ai.azure.com` (**not** `*.openai.azure.com`); leave the surface
  suffix off (the proxy appends `/anthropic`). Copy the host from the Claude
  deployment's Target URI.
- `inference.azure.token` = that deployment's Key (sent as `api-key`).
- The platform seeds no model: set `ANTHROPIC_MODEL` (the deployment name) in
  the policy env for Anthropic-SDK agents.

**LiteLLM (your own LiteLLM proxy):** same three shared flags, plus:
```bash
  --set inference.litellm.baseUrl="https://litellm.example.com" \
  --set inference.litellm.token="<litellm-key>"
```
- `baseUrl` = the proxy root (a trailing `/v1` is accepted); must be `https`.
  A `token` (or `forwardTags`) without `baseUrl` fails the install.
- The platform seeds no model, and LiteLLM model names are yours: Anthropic-SDK
  agents need `ANTHROPIC_MODEL` in the policy env to send a request.
- Optional `forwardTags: true` tags each call `project:<org-slug>/<project-slug>`
  so LiteLLM's spend reports break down by project. Leave it off if the proxy sets
  `router_settings.enable_tag_filtering` (it refuses tags no deployment has).
- Spend is metered **only by the cost LiteLLM reports**. For streamed calls set
  `litellm_settings.include_cost_in_streaming_usage: true` and give every model a
  price, or that spend counts against no limit.

**Provider availability (how the platform decides what's offered):** a backend
needs to be served and to have a key. Bedrock, **Bedrock Mantle**, **OpenAI** and
**OpenRouter** are always served; **Azure** and **LiteLLM** only when their base
URL is set. The key is the deployment token here (Bedrock without a token needs
AWS credentials the platform detects at boot; Mantle needs a real Bedrock token) — or the org's own:
**every deployment token is optional**, because an org admin can set the org's
inference key per backend on the **Inference Keys** page, and the org key wins.
`inference.deploymentKeyFallback: false` stops orgs without their own key from
using the deployment's. `GET /v1/inference/providers` reports what an org without
its own key can use (see `inference.md`).

> Keep tokens out of shell history: `--set inference.<provider>.existingSecret=<name>`
> (default keys `NEXUS_BEDROCK_TOKEN` / `NEXUS_AZURE_TOKEN` /
> `NEXUS_OPENAI_TOKEN` / `NEXUS_OPENROUTER_TOKEN` / `NEXUS_LITELLM_TOKEN`, override with
> `existingSecretKey`). On EKS, Bedrock can resolve from an IAM role (IRSA / Pod
> Identity) and skip the token; set `inference.bedrock.ambientCredentials` to
> `"false"` if that role has no Bedrock access, or `"true"` if detection can't
> see the credential source.

Confirm the rollout:
```bash
kubectl rollout status deploy/lens-agents --timeout=5m
kubectl get pods   # lens-agents-* and lens-agents-postgresql-* should be Running
```

## Step 3 — Expose the platform

```bash
kubectl port-forward svc/lens-agents 3002:3002   # keep running
curl -fsS http://localhost:3002/health && echo OK
```
`/health` answers `{"status":"ok","version":"<x.y.z>"}`, so it doubles as a
version check (the web UI sidebar header shows it too, "Platform · v…").
The forward binds `127.0.0.1`; `*.localtest.me`→127.0.0.1 too, so the platform
and every sandbox UI are reachable over this one forward. (If your resolver
blocks loopback DNS, add an `/etc/hosts` entry or a dnsmasq wildcard.)

## Step 4 — Activate and sign in (needs the human + their system browser)

A fresh install asks for an **activation** and a way to **sign in** before anyone
can use it. Activation is local (no Lens service involved); the install then runs
a **30-day evaluation** (see below).

1. Read the activation code the chart generated (`helm install` prints this command too):
   ```bash
   kubectl -n default get secret lens-agents-activation -o jsonpath='{.data.activationCode}' | base64 -d; echo
   ```
   (`<fullname>-activation` — `lens-agents-activation` here — or
   `activation.code.existingSecret`; key `activation.code.key`, default
   `activationCode`. The chart keeps the code across upgrades; only people who
   can read the release's Secrets have it.)
2. Open `http://localhost:3002`. The **Activate** screen asks for a name, company,
   work email and the code. Or from the CLI:
   `nexusctl installation activate --server http://localhost:3002 --code <code> --name "<name>" --company "<company>" --email <email>`.
   Activation creates the **first organization, named after the company** (when
   there is none). **Use the email of the Lens ID account you'll sign in with** —
   only that sign-in becomes the org's admin. If `config.ownerEmails` is set, the
   email must be one of them.
3. Choose sign-in (only when `oidc.clientId` is empty, as here): **Use Lens ID as
   OIDC** → approve the device code with Lens ID in Lens Cloud, which registers
   this install's sign-in client. CLI:
   `nexusctl installation use-lens-id --server http://localhost:3002 --code <code>`.
   The Lens ID that approves owns that registration — prefer a long-lived
   service account over a personal one. (Your own IdP instead: set
   `oidc.issuer`, `oidc.jwksUri` and `oidc.clientId` and upgrade.)
4. Sign in with Lens ID. **The person whose email activated the install becomes
   admin of that first org on first sign-in** (unless the IdP marks the email
   unverified) — required to create sandboxes. Everyone else joins by invitation.

Check it: `nexusctl installation status` shows `Activated:` and `Sign-in:`;
REST `GET /v1/installation` (no auth) reports `activated` and `signInConfigured`.

- **Without the Activate screen** (GitOps, production): set all three of
  `activation.activator.name` / `.company` / `.email` and the install activates
  itself when it starts (the chart refuses a partial set; the platform
  won't start with an invalid email, or one not in `config.ownerEmails` when that
  is set; an already-activated install ignores them). They land in the release's
  ConfigMap, readable by anyone who can read ConfigMaps there. The Lens ID step
  still asks for the code. With a renderer that has no cluster access (Argo CD,
  `helm template`) set `activation.code.existingSecret` (a code of 16+
  characters), or every render makes a new code.

### Evaluation and license key

- The evaluation lasts **30 days** from activation. When it
  ends without a valid key, **only the web UI** is blocked (**Evaluation
  ended**; **License expired** when a key lapses after it): an installation owner
  enters a key there, or continues the evaluation for another 30 days. `nexusctl`,
  MCP clients, the REST API and running sandboxes keep working. An expired or
  revoked key counts as no key.
- Enter the key on the **License** page (the **Evaluation** card in the sidebar)
  or with `nexusctl installation license --owner "<name>" <key>` (the name
  exactly as issued with the key; without arguments it shows the license). It
  needs a **person's** (OIDC) session of an installation owner — any signed-in
  person when `config.ownerEmails` is empty; an API token can't.
- Or by Helm values: `license.owner` plus `license.existingSecret` (key
  `license.existingSecretKey`, default `LICENSE_KEY`) — or `license.key`, which
  puts the key in the release's values. The chart refuses an owner without a key
  or the reverse. The platform stores the key at start on an activated install
  (replacing one entered in the UI) and won't start with a key that isn't valid
  for the owner; an expired or revoked one only logs a warning.

## Step 5 — Connect to the admin MCP and onboard

Now the admin surface is live at the **global** `http://localhost:3002/mcp`.
Connect and sign in (OIDC — full admin, no token to manage):

```bash
# Claude Code (v2.1.186+)
claude mcp add --scope user --transport http lens-agents http://localhost:3002/mcp
claude mcp login lens-agents      # system browser, caches token
# Codex
codex mcp add lens-agents --url http://localhost:3002/mcp
codex mcp login lens-agents
```
- Sign in in the **system browser** (not an embedded/in-agent browser — password
  managers and any existing session live there; Codex's built-in browser usually
  can't complete the sign-in). A just-added server may not hot-load in the
  current session — run the launch in a fresh `claude -p` child / `codex exec`,
  or restart. Keep the Step 3 port-forward alive.
- **MCP client can't do interactive OIDC?** Authenticate with a bearer token
  instead (you still drive everything over MCP): as the admin, create an API
  token in the platform UI (**API tokens**) or via `create_api_token`, then add
  the server with a header —
  `claude mcp add --transport http lens-agents <url> --header "Authorization: Bearer <token>"`.
  Exact params for any tool are in the live schema (`tools/list` / `/v1/openapi.json`).

Then run **playbook #1 "Onboard from zero"** (`playbooks.md`): `list_orgs`
(returns the org activation created, named after the company — use it; only if
it's empty, **ask the user for an org name and `create_org` yourself**; you're
OIDC-signed-in, so don't punt to the UI) → `create_project` →
`create_policy` (with `managedInference.provider` matching the
provider you installed) → `create_policy_binding` → `create_sandbox` (the Prism
agent; set `LLM_PROVIDER` to the same provider, and the now-**required**
`cpu`/`memory`, e.g. `"500m"`/`"2Gi"`) → poll `get_sandbox` for the
chat URL. Hand the user **both** URLs: the platform web UI (`config.publicUrl`)
and the Prism chat URL (`exposedPorts[0].url`). See `agents.md` for the exact
`create_sandbox` payload.

> **Provider must match in three places** (`bedrock` | `azure` |
> `bedrock-mantle` | `openai` | `openrouter`): the Helm `inference.*` flag
> (Step 2), the policy `managedInference.provider`, and the sandbox
> `LLM_PROVIDER` env. A mismatch = the agent won't answer — and an
> **unrecognized** `LLM_PROVIDER` doesn't throw, it falls back to `bedrock`.

## Troubleshooting (host-side)

| Symptom | Cause / fix |
|---|---|
| Sandbox stuck `creating`/`error`, platform fine | Almost always the `nftables` kernel — you're on a VM driver. Recreate the profile with `--driver=docker`. |
| `exposedPorts[0].url` stays `null` | `sandboxIngress.host` wasn't set at install — reinstall with `--set sandboxIngress.host=localtest.me`. |
| Chat URL won't resolve | Resolver blocks loopback DNS — `/etc/hosts` entry or dnsmasq. |
| Prism UI loads but agent doesn't answer | No/wrong inference credential, or provider mismatch across install/policy/sandbox. |
| Pods `Pending` | Cluster too small — `minikube stop/start -p lens-agents --cpus=4 --memory=6g`. |
| Web UI shows only **Activate** | Not activated yet — do Step 4. |
| Web UI shows only **Evaluation ended** / **License expired** | An installation owner enters a license key or continues the evaluation there; CLI, MCP and API keep working. |
| A valid Lens ID token gets `401` on REST / `/mcp` | The client gate: the token was issued to a client that is neither the install's own nor in `oidc.delegatedClients`. The log line reads "Client gate: refused this token" with a `reason`. See Production readiness below. |

Logs: `kubectl logs deploy/lens-agents --tail=100`; `kubectl get pods -n lens-agents-sandbox`.

## Cleanup

```bash
kubectl config current-context   # confirm 'lens-agents' first
helm uninstall lens-agents --wait
kubectl delete pvc -l app.kubernetes.io/instance=lens-agents --ignore-not-found
minikube delete -p lens-agents   # removes only this dedicated profile
```

## Upgrading to 0.36

- Upgrade with `helm upgrade lens-agents oci://ghcr.io/lensapp/lens-agents
  --reset-then-reuse-values …`. **`--reuse-values` fails**: it keeps the old
  chart's values, which lack the `activation` values the new templates need.
- **Remove the `hub.*` values and `oidc.useExternalProvider`** — they are no
  longer read.
- The Lens Hub enrollment is forgotten; users, orgs, data and the Lens ID
  sign-in client stay. The web UI shows the **Activate** screen once (Step 4, or
  the `activation.activator` values); people sign in as before, and `nexusctl`,
  MCP clients and the REST API keep working meanwhile. The 30-day evaluation
  starts at that activation.
- The web UI switches to the **Mirantis Agent Platform** brand.

## Production readiness (tell the user)

This trial is deliberately minimal: container isolation, port-forward access, a
bundled DB, a single install-time inference credential. Production adds microVM
sandbox isolation (Kata/gVisor runtime classes), real ingress + TLS, a managed
database, RBAC, and hardened credential management — a guided-evaluation
engagement, not self-serve.

**Client gate (on by default for Lens ID).** `oidc.clientGate`
(`OIDC_CLIENT_GATE`): `auto` (default) | `always` | `never` — any other value
fails the chart. `auto` turns it on whenever the install signs in through the
Lens ID client Lens Cloud provisioned (no `oidc.clientId`). It then accepts only access tokens (`typ: Bearer`) issued to that client or
to a client in `oidc.delegatedClients` (`OIDC_DELEGATED_CLIENTS`); anything else
gets **401** — no role needed. Until that client exists only delegated clients
get through. `nexusctl` and MCP clients keep working: the platform signs them in
through its own client.
A pre-provisioned `oidc.clientId` must also accept `<publicUrl>/oauth/callback`,
or MCP-client sign-ins (Claude Code, Codex) fail with `Invalid parameter:
redirect_uri`. Behind an ingress, keep its header limit at least
`config.maxHeaderBytes` (default `32768`). `sandboxIngress.corsOrigins` lists
origins a web UI may call private sandbox ports from.

**Event webhooks.** `eventWebhooks.urls` (`EVENT_WEBHOOK_URLS`; each `https`, or
`http` to an in-cluster host — a single-label name, `*.svc`,
`*.svc.cluster.local` — or loopback; no URL twice) plus `eventWebhooks.secret`
(≥32 bytes, kept in the Secret `<release>-event-webhooks`) or `existingSecret`
with `existingSecretKey` (default `EVENT_WEBHOOK_SECRET`), and `timeoutMs`
(default `5000`, max `30000`). `urls` without a secret fails the chart; empty
`urls` turns the feature off. **Every receiver gets every event type** — answer
`2xx` to a type you don't handle. The body is `{id, type, occurredAt, …}` (the
type is only in the body). Types:
- `access.revoked` `{userSub, projectIds}` (at most 1000 ids per event) when a
  **person** loses project access (removed from a project or the org, demoted
  from org admin, project or org deleted — a role change that keeps access, and
  API tokens, send nothing).
- `spending_limit.threshold` `{orgId, limitId, period, threshold, limitCents,
  spendCents, periodStart, periodEnd}` when an **org-level** limit reaches **80**
  or **100** % (see `governance.md`).
- `sandbox.deleted` `{orgId, projectId, sandboxId}` once a sandbox's runtime
  deletion was **requested** and its record removed — not proof the pod has
  stopped (the agent can run briefly after). A person's delete and auto-delete
  both send it; a failed delete sends nothing.

Each delivery is signed
`X-Lens-Agents-Signature: t=<unix>,v1=<hex HMAC-SHA256 of "<t>.<body>">` with
`X-Lens-Agents-Event-Id`; refuse a `t` more than 5 min off your clock. Delivery is
at least once (deduplicate on `id`), retried with backoff from 10 s up to 10 min
for 24 h, redirects not followed; `400`/`410`/`413`/`422` drop the event.

**Other values.** `sandbox.k8s.disk` (`SANDBOX_DISK`) caps a sandbox's volume
size; empty falls back to `sandbox.k8s.agentVolumeSize`
(`SANDBOX_K8S_AGENT_VOLUME_SIZE`, the default size), else `10Gi` — checked at
startup, and a default above the ceiling won't start.
`sandbox.k8s.cpuRequestRatio` / `sandbox.k8s.memoryRequestRatio` (default `"1"`)
set the share of each sandbox limit its pod reserves (see `agents.md`).
`workloadIdentity.enabled` (default `true`) and `workloadIdentity.sessionTtlSeconds` (default `3600`) — see
`tenancy.md`. `oidc.serviceClients` (`OIDC_SERVICE_CLIENTS`, default `[]`) names
clients whose client-credentials token is the **installation service account**;
`oidc.sandboxIngressClients` (client → realm role) and
`oidc.sandboxIngressClientsWithoutRole` (a list) let other clients' tokens open
private sandbox ports — a null role or an empty list item fails the chart (both in
`rbac.md`). `SANDBOX_INGRESS_WEBHOOK_MAX_BODY_BYTES` (env, not a chart value,
default 1 MiB) caps a webhook port's body (see `agents.md`). Install the matching
CLI with `curl -fsSL <publicUrl>/install.sh | sh`, or on Windows (PowerShell)
`irm <publicUrl>/install.ps1 | iex` (into `%LOCALAPPDATA%\Programs\nexusctl`; an
x64 build, which ARM64 runs emulated) — `nexusctl`; both refused over plain
HTTP except on loopback.

**Tools cache** (for sandbox `tools`, see `agents.md`):
- `sandbox.toolsCache.enabled` (default `true`; `SANDBOX_TOOLS_CACHE` `auto`|`off`).
  On first use the platform deploys a `nexus-tools-cache` Deployment, Service,
  PVC and Secret into the sandbox namespace, from the agent-loader image (no extra
  image to mirror). The chart's Role grants deployments, services and
  secret `update` for it.
- `sandbox.toolsCache.disk` (default `20Gi`) — the PVC size (storage class
  `sandbox.k8s.agentVolumeStorageClass`). The PVC is created once, so raising the
  value later means expanding the PVC yourself.
- `sandbox.toolsCache.extraHosts` (default `[]`; `*.example.com` matches
  subdomains) adds download hosts to the built-in allow-list (GitHub, nodejs.org,
  go.dev, static.rust-lang.org, releases.hashicorp.com, get.helm.sh, dl.k8s.io and
  similar).
- **Fail-closed:** while the cache isn't ready, a sandbox with tools doesn't
  start; disabled, each sandbox downloads its tools directly, from any host.
  Sandboxes without `tools` are unaffected. Sandbox pods must reach the cache
  Service (port 3128).
