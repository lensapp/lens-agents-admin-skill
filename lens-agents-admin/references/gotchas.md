# Gotchas & thresholds — the traps and the numbers

## Fail-open vs fail-closed cheat-sheet
Know which way each guard fails before you rely on it:

| Guard | Direction | Meaning |
|-------|-----------|---------|
| Managed-inference gate | **fail-CLOSED** | no policy/grant/resolver error → 403 |
| Budget check | **fail-OPEN** | budget-service outage won't block traffic |
| PII request masking | **fail-CLOSED** by default | blocks request if masking fails (`failOpen:true` to override) |
| PII response un-mask | **fail-OPEN** | upstream already replied |
| Per-actor MCP tool resolution | **fail-CLOSED** | empty allow-map = deny-all (must not collapse to "no filter") |
| Policy **resolve() error** | **fail-CLOSED** | deny-all fallback |
| **Empty/absent** policy | egress-deny but **inference stays OFF** | not the same as a resolve error |

## Default thresholds (embed these — don't send the user to look them up)
- Sandbox **idle shutdown: 30 min** (platform-managed; not an isolation control; self-hosted agent sandboxes have none).
- **kubectl JWT: 15 min**; **EKS token / AWS STS: 900 s**; generic JWT default 60 s.
- **Heartbeat: 8 h default, 1 min minimum**; auto-stop after **5 consecutive failures**.
- Managed LLM: **100**-step limit, **16k** max output tokens, temp **0.3**, **30k**-char tool-output truncation.
- **Daily session reset 4 AM** local; conversation compaction keeps the **recent 20** messages.
- Memory retention: importance **<3 AND >30 days** deleted; ≥3 kept indefinitely; summary never auto-deleted.
- Audit API: **max 200** entries/page (cursor pagination). Public exposed-port concurrency default **32**.
- Free tier: **100 agent-hours/month** (Standard).
- Crypto: ephemeral per-sandbox CA **ECDSA P-256** (in-memory only); credential storage **AES-256-GCM**; tokens stored as **SHA-256** (one-way, shown once).

## Naming & shape constraints
- **Slug regex** `^[a-z0-9][a-z0-9-]*[a-z0-9]$` (lowercase alphanumeric + hyphens, no leading/trailing hyphen) for connection/resource/server names.
- Sandbox: **at most one exposed port and one persistent volume**. Exposed-port `auth`: `public` (openable without a session) vs `private` (needs OIDC).
- Sandbox images **must have `/bin/sh` and a writable CA bundle**; **`FROM scratch` and non-debug distroless are unsupported**.
- Reserved connector name **`nexus-api`** (the platform's own system MCP row) — don't create/rename/delete it; `create_mcp_server` refuses to shadow it.
- Sandbox workspace root is **`/home/sandbox`**.

## Behavior traps
- **A sandbox on the project `/mcp` sees no admin tools** — by design (see `architecture.md`). Administer via the global `/mcp`.
- **`allowedTools: []` on a connector ref = deny all**, not allow all.
- **`piiMasking.failOpen: false` = fail-closed** (blocks) — the flag name inverts easily.
- **`OPENAI_BASE_URL` must end in `/v1`** or managed GPT calls 404 (see `inference.md`).
- **Direct provider egress is unmetered** — keep provider hosts denied (see `inference.md`).
- **Token revocation doesn't kill live sessions** — stop the sandbox for immediate cutoff; otherwise the 30-min idle bounds the window.
- **The platform `shell_*` tools run as the *caller*, in a fresh sandbox — not inside a target agent's container.** So you can't use them to seed a skill into another agent's `/data` or drive its runtime. To seed/drive a managed agent, go through *its own* chat UI / WS (e.g. ask it to install the skill from its repo link), or pre-seed the `/data` volume at create time.
- **A Slack bot-token credential only allows the methods you whitelist — and the adapter's set ≠ the agent's set.** The Bolt adapter calls a fixed handful (auth.test, chat.postMessage/update, conversations.info/replies, reactions.add, users.info, files.*). If the *agent itself* calls other Slack methods — e.g. it enumerates channels/DMs via `conversations.list` / `users.conversations` — those must be in the credential's injection rules too, or the proxy 403s them. Easy to miss precisely because the adapter never calls them, so a "working" Slack connection still fails the agent's own API use (see `playbooks.md` playbook 7).
- **Managed inference is opt-in** — an empty policy egress-denies but leaves inference OFF; the agent won't answer until a policy enables it.
- Provider must match across **install + policy + sandbox `LLM_PROVIDER`** or the agent won't answer.

## Doc inconsistencies to treat carefully
- Provider list: treat the models/inference docs as authoritative — **Bedrock, Azure (Foundry), Bedrock Mantle**. Older pages say "Anthropic and AWS Bedrock only."
- Spending scopes: the current model is **four** (org/team/agent/sandbox); some pages still say three.
- SSO: docs describe **OIDC**; an FAQ mentions SAML — the platform documents OIDC.
