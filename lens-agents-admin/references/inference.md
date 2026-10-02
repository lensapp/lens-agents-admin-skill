# Managed inference — providers, metering, and the traps

Managed agents reach their LLM through the platform's **managed inference
endpoint**, which meters spend, enforces budgets, masks PII, and audits every
call. A policy's `managedInference.provider` selects the backend; the provider
**credential** is either the **deployment key** set at platform install time
(Helm `inference.*` / `NEXUS_*` env) or the org's **own inference key**, set by
an org admin (see "Org inference keys" below). The org key wins.

## Six backends (and availability)
A backend is **served** by the deployment, and **usable** by an org when it also
has a key: the org's own, or the deployment's (if the deployment allows the
fallback).
- **`bedrock`** — AWS Bedrock. **Always served**; the deployment key falls back
  to the AWS default credential chain if no token is set.
- **`azure`** — Claude on **Microsoft Foundry** (Anthropic Messages API,
  api-key auth). Served **only when** the Foundry base URL is set; needs the
  deployment token or an org key.
  One Foundry resource+key serves two surfaces (`/anthropic` for Claude,
  `/openai` for GPT).
- **`bedrock-mantle`** — an AWS OpenAI/Anthropic-**compatible** endpoint that
  serves Claude **and** GPT off **one Bedrock key**. Always served; needs a real
  Bedrock token (deployment or org `bedrock` key — the AWS credential chain
  doesn't count). Derives its host from the region.
- **`openai`** — OpenAI, bearer-token auth. Always served; needs the deployment
  token or an org key. **GPT only** — OpenAI hosts no Claude, so a sandbox here gets
  `OPENAI_BASE_URL` and **no managed Anthropic endpoint**. POSTs are allowlisted
  to the three paths whose spend can be metered (`/v1/chat/completions`,
  `/v1/responses`, `/v1/embeddings`) — any other POST **404s**; `GET /v1/models`
  is served too, so model pickers work. Realtime voice rides a separate
  WebSocket route: served **unmasked** (audited `piiMaskingSkipped` **when the
  policy asked for masking**) and capped at **8 concurrent sessions per project,
  per replica** (503 past that). `inference.openai.baseUrl` retargets the backend at
  any OpenAI-compatible endpoint — must be `https`.
- **`openrouter`** — a reseller in front of ~60 vendors (~400 models),
  bearer-token auth. Always served; needs the deployment token or an org key.
  Two surfaces off
  that one key — and **unlike the sibling two-surface backends, each surface
  reaches every vendor it resells**, not one model family: an Anthropic-SDK
  agent can drive Gemini, an OpenAI-SDK agent can drive Claude.
  - **Metering:** spend is billed on **the cost OpenRouter reports on each
    response**, not a rate-card token price — so a model missing from the price
    table still meters. **BYOK** meters on upstream charge + fee where the
    response carries its upstream-cost figure; where that figure is missing it
    falls back to table pricing, and the gap returns.
  - **Egress:** keep `openrouter.ai` **denied**. One key reaching 60 vendors is
    the cheapest way for an agent to spend money nothing meters.
- **`litellm`** — your own LiteLLM proxy, authenticated with a LiteLLM key sent as
  a bearer token. Served **only when** its base URL is set; needs the deployment
  token or an org key.
  Serves both wire formats (Anthropic `/v1/messages`, OpenAI chat/responses/
  embeddings, plus `GET /v1/models`), so Claude Code and OpenAI-SDK agents alike
  get a managed endpoint.
  - **Metering:** spend is metered **only by the cost LiteLLM reports** — never a
    rate card. Streamed calls report it only if the LiteLLM config sets
    `litellm_settings.include_cost_in_streaming_usage: true`; without that, and for
    any model LiteLLM has no price for, spend **counts against no spending limit**.

`GET /v1/inference/providers` reports the backends an org **without** its own key
can use (the deployment-key set; empty when the fallback is off).
`list_inference_providers { orgId }` (REST `GET /v1/orgs/{orgId}/inference/providers`)
reports, per served backend, whose key this org uses: `keyOrigin` = `org`,
`deployment`, or `null` — `null` means a policy selecting that backend fails.

## Org inference keys
An org admin can give the org its **own key** per key source, so its spend runs
on its own provider account. Tools: `list_inference_keys`, `set_inference_key
{ orgId, source, token }`, `remove_inference_key { orgId, source }` — **OIDC
org admin only** (not API tokens, not sandbox identities); `list_inference_providers` is
also offered to API tokens. REST: `GET/PUT/DELETE
/v1/orgs/{orgId}/inference/keys[/{source}]`; UI: the **Inference Keys** page.
- **Sources:** `bedrock` (native Bedrock + Mantle), `azure` (both Foundry
  surfaces + realtime), `openai` (+ realtime), `openrouter`, `litellm`. Only the
  key is the org's — the endpoint stays the deployment's (all orgs share its
  Foundry resource / LiteLLM proxy).
- Token ≥ **16** characters; stored encrypted, never returned — reads show the
  **last 4** characters only. Each set/remove writes an audit record.
- A change takes effect within **30 s** — no sandbox restart.
- Without an org key the org uses the deployment key, unless the install sets
  `inference.deploymentKeyFallback: false` (`NEXUS_INFERENCE_DEPLOYMENT_KEY_FALLBACK`);
  then that backend is unusable for it. A call with no key gets **404** ("not
  configured for this organization"); a failed key lookup gets **503**.

## Selection is per-policy and enforced at the backend
Set `managedInference: { enabled: true, provider: <backend> }` on the policy
(absent ⇒ inference OFF; opt-in). To enable several backends, add
`providers: [...]` (unique, and it must include `provider`, which stays the
primary). The proxy checks the **backend** (not just the
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
- **Managed-inference gate: fail-CLOSED** — no policy / no grant → 403; a
  policy-resolver error → 502.
- **Budget check: fail-OPEN** — a budget-service outage doesn't take down the
  proxy. On breach → **HTTP 429 + `Retry-After` + RFC 7807 `budget-exceeded`**;
  the container keeps running (paused, not killed).
- **PII request masking: fail-CLOSED by default** (`failOpen: false` blocks the
  request if masking fails — compliance). `failOpen: true` proceeds unmasked
  (operator accepts the risk) — this covers masking that ran and failed. If
  masking can't run at all (no anonymizer, or the PII service is unavailable),
  the request gets **503** regardless of `failOpen`. **PII response un-masking: always fail-open**
  (upstream already replied).
- Embedding requests are **exempt from masking** (masking would corrupt vectors)
  — metered, not masked.

## Defaults & platform config
**The platform sets no model.** For an Anthropic-SDK agent set `ANTHROPIC_MODEL`
in the policy env (or let the agent pick its own; the web UI's agent templates
pre-fill one). Managed model config: temp **0.3**, max
**16k** output tokens, **100**-step limit, **30k**-char tool-output truncation,
prompt caching auto. Install-time env (every token optional):
`NEXUS_BEDROCK_TOKEN`, `NEXUS_AZURE_BASE_URL` + `NEXUS_AZURE_TOKEN`,
`NEXUS_OPENAI_TOKEN` + optional `NEXUS_OPENAI_BASE_URL`,
`NEXUS_OPENROUTER_TOKEN`, `NEXUS_LITELLM_BASE_URL` + `NEXUS_LITELLM_TOKEN` +
optional `NEXUS_LITELLM_FORWARD_TAGS`, and `NEXUS_INFERENCE_DEPLOYMENT_KEY_FALLBACK`
(default `true`). An Azure or LiteLLM token (or LiteLLM forward tags) without its
base URL is refused at boot. See `install-local.md` for the Helm form.

> **Provider must match in three places** for a managed agent to answer: the
> platform install (`inference.*`, or the org's own key), the policy (`managedInference.provider`), and
> the sandbox's `LLM_PROVIDER` env (see `agents.md`).
