# Install the platform locally (from nothing)

This is the authoritative runbook to stand up the whole Lens Agents platform on
a local single-node **minikube** cluster, then hand off to the onboarding
playbook. **This local trial is the self-serve install** — so when a user just
says "install Lens Agents" (no other context), do this, no need to ask. (A
production deployment is a guided engagement; point them to Getting Started /
contact us — see the end of this file.) It's the host-shell phase — **there is
no MCP yet**; the admin `/mcp` only becomes reachable after Step 3 (port-forward)
+ Step 4 (activation). Run these as the coding agent doing the install; pause
only for the human's browser sign-ins (Steps 4 and the Step 5 MCP login).

> **Before you start — get a full-access shell.** Every step below runs real
> **host** commands (Docker/OrbStack, minikube, kubectl, Helm) that need the
> **Docker socket** and **network** (image pulls, the Helm OCI chart,
> activation). A restricted/read-only or no-network agent sandbox **cannot**
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
| `helm` **v3.8+** | `helm version` | OCI support required |
| `openssl` | `openssl version` | generates the encryption key |
| `curl` | `curl --version` | health check |

Plus: outbound internet, a **Lens ID** account (sign up at `app.k8slens.dev`),
and **one inference-provider credential** — an **AWS Bedrock** API key, **or**
an **Azure (Microsoft Foundry)** resource URL + API key with a Claude deployment.
(Other providers via Team Lens.)

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
to Lens ID. **Managed inference** means the platform holds the provider
credential and proxies every LLM call (metered/gated/audited). Three shared
flags: `encryption.key` (64 hex, at-rest encryption), `config.publicUrl`,
`sandboxIngress.host=localtest.me` (required for sandbox web UIs — `localtest.me`
is public wildcard DNS resolving `*.localtest.me`→127.0.0.1).

**AWS Bedrock (default):**
```bash
helm install lens-agents oci://ghcr.io/lensapp/lens-agents \
  --set encryption.key="$(openssl rand -hex 32)" \
  --set config.publicUrl=http://localhost:3002 \
  --set sandboxIngress.host=localtest.me \
  --set inference.bedrock.token="<bedrock-api-key>" \
  --wait --timeout 10m
```

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
- Optional `--set inference.azure.anthropic.model="<deployment>"` (default `claude-opus-4-8`).

**Provider availability (how the platform decides what's offered):** Bedrock is
**always** available (falls back to the AWS default credential chain if no
token); **Azure** appears only when its base URL+token are set; **Bedrock
Mantle** (OpenAI/Anthropic-compatible off one Bedrock key) appears only when a
Bedrock token is set. `GET /v1/inference/providers` reports the live set.

> Keep tokens out of shell history: `--set inference.<provider>.existingSecret=<name>`
> (default keys `NEXUS_BEDROCK_TOKEN` / `NEXUS_AZURE_TOKEN`, override with
> `existingSecretKey`). On EKS, Bedrock can resolve from an IAM role and skip the
> token.

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
The forward binds `127.0.0.1`; `*.localtest.me`→127.0.0.1 too, so the platform
and every sandbox UI are reachable over this one forward. (If your resolver
blocks loopback DNS, add an `/etc/hosts` entry or a dnsmasq wildcard.)

## Step 4 — Activate and sign in (needs the human + their system browser)

1. Open `http://localhost:3002`.
2. Follow the activation device-flow prompt (registers this install with Lens Cloud).
3. Sign in with Lens ID. **The activating account becomes the organization admin** — required to create sandboxes.

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

Then run **playbook #1 "Onboard from zero"** (`playbooks.md`): `list_orgs` →
`create_project` → `create_policy` (with `managedInference.provider` matching the
provider you installed) → `create_policy_binding` → `create_sandbox` (the Prism
agent; set `LLM_PROVIDER` to the same provider) → poll `get_sandbox` for the
chat URL. Hand the user **both** URLs: the platform web UI (`config.publicUrl`)
and the Prism chat URL (`exposedPorts[0].url`). See `agents.md` for the exact
`create_sandbox` payload.

> **Provider must match in three places** (Bedrock or Azure): the Helm
> `inference.*` flag (Step 2), the policy `managedInference.provider`, and the
> sandbox `LLM_PROVIDER` env. A mismatch = the agent won't answer.

## Troubleshooting (host-side)

| Symptom | Cause / fix |
|---|---|
| Sandbox stuck `creating`/`error`, platform fine | Almost always the `nftables` kernel — you're on a VM driver. Recreate the profile with `--driver=docker`. |
| `exposedPorts[0].url` stays `null` | `sandboxIngress.host` wasn't set at install — reinstall with `--set sandboxIngress.host=localtest.me`. |
| Chat URL won't resolve | Resolver blocks loopback DNS — `/etc/hosts` entry or dnsmasq. |
| Prism UI loads but agent doesn't answer | No/wrong inference credential, or provider mismatch across install/policy/sandbox. |
| Pods `Pending` | Cluster too small — `minikube stop/start -p lens-agents --cpus=4 --memory=6g`. |

Logs: `kubectl logs deploy/lens-agents --tail=100`; `kubectl get pods -n lens-agents-sandbox`.

## Cleanup

```bash
kubectl config current-context   # confirm 'lens-agents' first
helm uninstall lens-agents --wait
kubectl delete pvc -l app.kubernetes.io/instance=lens-agents --ignore-not-found
minikube delete -p lens-agents   # removes only this dedicated profile
```

## Production readiness (tell the user)

This trial is deliberately minimal: container isolation, port-forward access, a
bundled DB, a single install-time inference credential. Production adds microVM
sandbox isolation (Kata/gVisor runtime classes), real ingress + TLS, a managed
database, RBAC, and hardened credential management — a guided-evaluation
engagement, not self-serve.
