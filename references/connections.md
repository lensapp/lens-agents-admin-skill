# Connections — Kubernetes clusters & AWS

> **Exact parameters aren't here — read the live schema.** For any tool, use `tools/list` (automatic over MCP) or the REST OpenAPI at `<publicUrl>/v1/openapi.json` (`/v1/docs` for the UI). This file covers what those can't: what the tools are for and the non-obvious rules.

Connections are project resources that agents reach through the platform proxy
(governed, credential-injected, audited). Attach them to an agent via a
policy's `integrations[]` (see `policies.md`).

## Kubernetes clusters

Agents reach a cluster through the platform's **Kubernetes relay** — a tunnel, not a
direct connection to the cluster API. The platform injects a short-lived
(~60s) impersonated JWT per request; the agent never holds a kubeconfig or
any long-lived cluster credential.

Tools: `list_clusters`, `get_cluster`, `create_cluster`, `update_cluster`,
`delete_cluster`, `rotate_cluster_tunnel_token`.

Grant access by adding a kubernetes integration (naming the cluster) to the
agent's policy.

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
