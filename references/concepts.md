# Concepts — why the platform works the way it does

The rules a schema won't tell you. Internalize these; they explain every design
decision and keep you from fighting the platform.

## Governance before, not after
Policy is evaluated **per request, before the action**, and **everything is
default-deny**. You don't audit an agent's mistakes after the fact — the platform
refuses the request at the boundary. Write policy first, then let agents connect.

## One governance plane, many connection surfaces
A policy written once applies to a Claude Code session, a LangChain agent, a
managed SRE agent, and a CI-run CLI agent — **without re-scoping**. Governance is
built first; the three agent types are just ways to connect to it.

## Two identity models (drives all audit attribution)
- **User identity** — OIDC/SSO; a human; actions attributed to their email.
- **Agent identity** — an opaque bearer token, scoped to one org+team+project;
  a first-class principal. **Agents authenticate as themselves, never
  impersonating the human.**
Every action is attributed to one of these, plus the invocation mode
(chat/heartbeat/cron/subagent).

## Mode 1 vs Mode 2 (what governance actually covers)
- **Mode 1 — agent *outside* the sandbox** (desktop tools, external agents by
  default). "Trust the agent, govern its tools." The platform governs tool
  execution/system access/credentials/audit, but **not the agent's own reasoning
  process**.
- **Mode 2 — agent *inside* the sandbox** (managed agents, local CLI, hosted
  external agents). "Don't trust the agent, govern everything." The whole process
  is sandboxed.
This axis determines coverage. A managed agent (Prism) is always Mode 2.

## Credentials never reach the agent
The agent is given **decoy credentials**; the proxy swaps in the real secret at
the network boundary. **Even a fully compromised agent cannot extract a
credential** — it only ever held a decoy. K8s uses short-lived impersonation
JWTs; AWS uses STS re-signing. This is the platform's central security claim.
Corollary: never try to read/exfiltrate a real credential; it isn't there.

## Default-deny kernel networking
Egress is locked at the kernel (nftables/iptables), TCP redirected to a
policy-aware proxy, DNS to a stub resolver that returns NXDOMAIN for denied
names. **There is no knob to disable it.** First-match-wins; default action deny.

## Autonomy vs policy (critical — don't confuse them)
- **Autonomy level** (1 Observer → 5 Autonomous, default 3 Helper) = what the
  agent *decides* to do. It's a **prompt-layer** behavioral hint and **can be
  bypassed by prompt injection**.
- **Policy + sandbox + credential scope** = what the agent is *able* to do. This
  is the **hard boundary**. "How safe is level 5?" is answered by the policy, not
  the autonomy setting.

## Team is the policy unit; org is the ceiling
Policies apply at **team** level (all agents in a team share them). Projects grant
infrastructure scope. **Org policy is a restriction-only ceiling** — lower layers
can only *narrow* it, never expand. Excess is clipped at resolve time and surfaced
as drift (not rejected on save).

## The metering boundary
Usage metering, spend enforcement, and PII masking happen **because traffic flows
through the managed inference endpoint** — not via TLS interception. If a policy
grants direct network access to a provider host, that traffic is **neither
metered nor gated**. So **keep provider hosts denied** (the default) — agents
reach models only through the managed endpoint. (See `inference.md`.)

## Audit is availability-over-consistency
Audit writes are **async and non-blocking** — an agent is never blocked by an
audit failure, and entries may lag during a DB outage. Operationally: export to
SIEM, monitor for gaps, plan retention (the platform enforces none).

## Managed agents are defined as files
A managed agent's behavior is **seven workspace files** (Soul, User Profile,
Memory Summary, Heartbeat, Vision, Agent Guide, Bootstrap), all loaded every
invocation. Most are agent-editable at runtime; the **Agent Guide is user-only**
(the hard behavioral boundary). See `agents.md`.

## Deployment model sets data residency + telemetry defaults
SaaS / self-hosted / marketplace — same platform code, no features removed.
**Self-hosted/marketplace send nothing to Lens by default** (telemetry off);
SaaS defaults on with opt-out. Bedrock keeps data in the AWS account; Azure in
the Azure tenant.

## Be honest about the documented limitations
A competent admin knows the platform's own stated caveats:
- Default runtime is **containers (shared host kernel)** → for syscall-level
  isolation, deploy the sandbox in a **microVM** (Kata/gVisor). Local minikube is
  container-only.
- **No seccomp** by default.
- **Token revocation doesn't kill active sessions** — the 30-min idle timeout
  bounds the window; **stop the sandbox** for immediate termination.
- **Tenant isolation is application-level** (org-scoped FKs + app authz), not DB
  row-level.
- **Autonomy enforcement is prompt-layer** (see above) — bypassable; policy is
  the real boundary.
Don't overclaim these to the user; state them plainly when relevant.
