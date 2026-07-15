# Connections — Kubernetes clusters & AWS

> **Exact parameters aren't here — read the live schema.** For any tool, use `tools/list` (automatic over MCP) or the REST OpenAPI at `<publicUrl>/v1/openapi.json` (`/v1/docs` for the UI). This file covers what those can't: what the tools are for and the non-obvious rules.

Connections are project resources that agents reach through the platform proxy
(governed, credential-injected, audited). Attach them to an agent via a
policy's `integrations[]` (see `policies.md`).

## Kubernetes clusters

Agents reach a cluster through the platform's **Kubernetes relay** — a tunnel,
not a direct connection to the cluster API. The platform injects a short-lived
(~60s) impersonated JWT per request; the agent never holds a kubeconfig or any
long-lived cluster credential. **Connecting a cluster is two steps: (1) register
it on the platform, (2) deploy the relay _into the target cluster_.** Registering
alone does nothing until the relay is running — this is the step that isn't in
any API response, so it's spelled out below.

Tools: `list_clusters`, `get_cluster`, `create_cluster`, `update_cluster`,
`delete_cluster`, `rotate_cluster_tunnel_token`.

### Connect a cluster (tunnel mode — the default)

1. **Register the cluster:** `create_cluster { projectId, name, displayName,
   relayUrl: "tunnel://" }`. The `tunnel://` sentinel selects tunnel mode (the
   relay dials **outbound** — no inbound ports, no public endpoint). The response
   returns the **`tunnelToken`** (prefix `lnst_`) **once — capture it immediately**,
   plus the cluster `id`. (Lost it? `rotate_cluster_tunnel_token` mints a new one.)
2. **Deploy the relay into the target cluster** with Helm. The chart is on GHCR;
   its literal artifact name is `nexus-kube-relay` (a platform-internal component —
   type the name verbatim, it must match):

   ```bash
   helm install nexus-kube-relay oci://ghcr.io/lensapp/charts/nexus-kube-relay \
     --namespace nexus-relay --create-namespace \
     --set config.mode=tunnel \
     --set config.tunnelUrl="wss://<platform-host>/v1/tunnel" \
     --set config.tunnelToken="<tunnelToken from step 1>" \
     --set config.jwksUrl="<platformUrl>/v1/projects/<projectId>/.well-known/jwks.json"
   ```
   - `config.tunnelUrl` = the platform URL with `http(s)→ws(s)` and path
     `/v1/tunnel`. **It must be routable from _inside_ the target cluster** — see
     the local-trial note below, which is the #1 trip-up.
   - `config.jwksUrl` lets the relay verify the platform's JWTs (or pass the PEM
     from `get_project_public_key` as `config.publicKey` instead).
   - Chart `values.yaml` keys: `config.mode` (`tunnel`|`server`),
     `config.tunnelUrl`, `config.tunnelToken`, `config.jwksUrl` / `config.publicKey`.
3. **Verify:** the relay pod in `nexus-relay` reaches `Running` and its logs show
   the tunnel connected. Then any agent that has this cluster in its policy can
   kubectl through it.

### Local trial — relay and platform share one minikube

On the local trial the platform runs **inside the same minikube**, and you reach
it from the host only via `kubectl port-forward … 3002`. So the "obvious"
`ws://localhost:3002/v1/tunnel` **fails from inside a pod** — a pod's `localhost`
is itself, not your host. Point the relay at the platform's **in-cluster service
DNS** instead (no port-forward needed for the relay; the forward is only for your
browser/MCP):

```bash
  --set config.tunnelUrl="ws://lens-agents.default.svc.cluster.local:3002/v1/tunnel" \
  --set config.jwksUrl="http://lens-agents.default.svc.cluster.local:3002/v1/projects/<projectId>/.well-known/jwks.json"
```
Confirm the service name/namespace/port first with `kubectl get svc` (the local
install's release is `lens-agents` on `3002`). The relay reaches the cluster's own
API at the in-cluster default `https://kubernetes.default.svc` automatically.

### Direct mode (public relay) — the alternative

Skip the tunnel and expose a relay at a public HTTPS URL:
`create_cluster { …, relayUrl: "https://relay.example.com" }` (optional
`caCertificate` PEM for self-signed TLS). No tunnel token; the platform connects
out to that URL. Prefer tunnel mode unless you specifically need a public relay.

Grant an agent access by adding a kubernetes integration (naming the cluster) to
its policy (see `policies.md`).

## AWS connections

Agents call AWS through the proxy, which performs **STS AssumeRole** with
session tags on every request — no long-lived AWS keys ever reach the agent
itself, only the platform holds them.

Tools: `list_aws_connections`, `get_aws_connection`, `create_aws_connection`,
`update_aws_connection`, `delete_aws_connection`.

Grant access via an aws-connection integration (naming the connection) in the
agent's policy.

> GitHub: there is **no** GitHub-connection tool on the current MCP surface.
> Give an agent GitHub access with a **credential** (`credentials.md`, a PAT
> injected as `Authorization: token {value}` on `api.github.com`) plus the
> matching `allowedDomains` entry in its policy.
