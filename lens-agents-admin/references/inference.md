# Managed inference — providers, metering, and the traps

Managed agents reach their LLM through the platform's **managed inference
endpoint**, which meters spend, enforces budgets, masks PII, and audits every
call. A policy's `managedInference.provider` selects the backend; the provider
**credential** is set at **platform install time** (Helm `inference.*` /
`NEXUS_*` env), not over MCP.

## Three backends (and availability)
- **`bedrock`** — AWS Bedrock. **Always available** (falls back to the AWS
  default credential chain if no token is set).
- **`azure`** — Claude on **Microsoft Foundry** (Anthropic Messages API,
  api-key auth). Available **only when** the Foundry base URL + token are set.
  One Foundry resource+key serves two surfaces (`/anthropic` for Claude,
  `/openai` for GPT).
- **`bedrock-mantle`** — an AWS OpenAI/Anthropic-**compatible** endpoint that
  serves Claude **and** GPT off **one Bedrock key**. Available **whenever a
  Bedrock token is set**. Derives its host from the region.

`GET /v1/inference/providers` reports the live set; the UI gates its picker on
it. A policy can only select a provider the deployment actually has.

## Selection is per-policy and enforced at the backend
Set `managedInference: { enabled: true, provider: <backend> }` on the policy
(absent ⇒ inference OFF; opt-in). The proxy checks the **backend** (not just the
URL) — a sandbox can't reach a backend its policy didn't select (data-residency
defense).

## The metering boundary rule (most important)
Metering/budget/PII happen **because the request goes through the managed
endpoint** — there's no TLS interception on direct provider traffic. If a policy
grants direct network egress to a provider host (`bedrock-runtime.*`,
`bedrock-mantle.*.api.aws`, the Foundry host), that traffic is **neither metered
nor gated**. **Keep provider hosts DENIED** (the default) so agents can only
reach models via the managed endpoint — the deny rule is the only guardrail.

## Fail modes (know which way each fails)
- **Managed-inference gate: fail-CLOSED** — no policy / no grant / resolver error
  → 403.
- **Budget check: fail-OPEN** — a budget-service outage doesn't take down the
  proxy. On breach → **HTTP 429 + `Retry-After` + RFC 7807 `budgetExceeded`**;
  the container keeps running (paused, not killed).
- **PII request masking: fail-CLOSED by default** (`failOpen: false` blocks the
  request if masking fails — compliance). `failOpen: true` proceeds unmasked
  (operator accepts the risk). **PII response un-masking: always fail-open**
  (upstream already replied).
- Embedding requests are **exempt from masking** (masking would corrupt vectors)
  — metered, not masked.

## Defaults & platform config
Default models: Bedrock `us.anthropic.claude-opus-4-8`; Azure `claude-opus-4-8`;
classification = a Claude **Haiku**. Managed model config: temp **0.3**, max
**16k** output tokens, **100**-step limit, **30k**-char tool-output truncation,
prompt caching auto. Install-time env: `NEXUS_BEDROCK_TOKEN`,
`NEXUS_AZURE_BASE_URL` + `NEXUS_AZURE_TOKEN` (both-or-neither),
`NEXUS_AZURE_ANTHROPIC_MODEL`. See `install-local.md` for the Helm form.

> **Provider must match in three places** for a managed agent to answer: the
> platform install (`inference.*`), the policy (`managedInference.provider`), and
> the sandbox's `LLM_PROVIDER` env (see `agents.md`).
