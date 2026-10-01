# Plan: llm-gateway

A small OpenAI-compatible service that sends each LLM request to whichever free provider is available right now, and moves to the next one when a provider is rate-limited, down, or out of quota. Based on [FindIt's llm.js](https://github.com/Me-Bro/FindIt/blob/main/llm.js).

## Decisions

| Topic | Decision |
|---|---|
| Providers (v1) | Groq, Google Gemini, OpenRouter, Cerebras. All speak the OpenAI chat-completions format, so one adapter covers them. |
| Paid fallback | None. Free providers only. |
| Location | `llm-gateway/` in this repo, run as a service in `docker-compose.yml`. |
| Stack | Node 22, plain JavaScript (ESM), `node:http` + `fetch`, no framework, `node:test` for tests. |
| API shape | OpenAI-compatible (`/v1/chat/completions`), so n8n's built-in OpenAI credential and nodes work with it. |

## Architecture

```
n8n ──(OpenAI credential, base URL http://llm-gateway:8080/v1)──> llm-gateway
                                                                   │
                         router: pick candidates for the model alias, skip
                         disabled / out-of-quota, cooling ones go last
                                                                   │
              ┌───────────────┬───────────────┬────────────────────┤
            Groq           Gemini        OpenRouter            Cerebras
```

Request flow:
1. Authenticate the caller with the gateway key (`Authorization: Bearer $GATEWAY_API_KEY`).
2. Resolve the model: an alias (`fast`, `smart`, `vision`, `json`) maps to each provider's own model. A real model name is sent only to providers that list it.
3. Order the candidates: ready providers first (by priority or round-robin), then providers that are cooling down. Disabled and out-of-quota providers are skipped.
4. Try each candidate in turn. On failure, record a cooldown based on the error type (table below) and move to the next.
5. Return the first success, adding the headers `x-llm-provider` and `x-llm-model`. If every candidate fails, return a 503 with each provider's error.

## Failure handling

| What happened | Cooldown | Try next provider? |
|---|---|---|
| 429 rate limited | `Retry-After` header if present, otherwise 60s, doubling on repeat failures (max 15 min) | yes |
| 5xx, network error, timeout | 30s | yes |
| 401 | provider disabled until restart (bad key), shown in `/status` | yes |
| 403 | only that provider+model disabled until restart (OpenRouter uses 403 for model-specific restrictions) | yes |
| 400 / 413 / 422 (e.g. unsupported parameter, prompt too long) | none: it's about this request, not the provider | yes |
| 200 but bad JSON when JSON was requested | none (model slip, as in `llm.js`) | yes |
| A success | resets that provider's backoff | no |

Timeouts are configurable per provider (default 30s for normal requests, 120s for streaming).

## Quotas

- Each provider can set `limits.rpm` (requests per minute) and `limits.rpd` (requests per day). Fill them in from each provider's free-tier page, since the numbers change.
- Usage counters are saved to `/data/usage.json` (a Docker volume) so restarts don't reset them. Daily counters reset at midnight UTC.
- A provider at its limit is skipped (not cooled down) until its window resets.

## Endpoints

| Endpoint | Purpose |
|---|---|
| `POST /v1/chat/completions` | Main endpoint. Supports `stream: true`: the stream passes straight through and the gateway can only switch providers before the first byte arrives. |
| `GET /v1/models` | Lists the aliases plus every configured real model (n8n's model dropdown uses this). |
| `GET /status` | Each provider's state (ready / cooling until X / disabled / quota exhausted), usage today, last error, average latency. |
| `GET /health` | Liveness check for Docker. |

## Configuration

`llm-gateway/providers.json` (committed, no secrets):

```json
{
  "strategy": "priority",
  "providers": [
    {
      "name": "groq",
      "baseUrl": "https://api.groq.com/openai/v1",
      "keyEnv": "GROQ_API_KEY",
      "models": { "fast": "…", "smart": "…", "json": "…", "vision": "…" },
      "limits": { "rpm": null, "rpd": null }
    },
    { "name": "gemini",     "baseUrl": "https://generativelanguage.googleapis.com/v1beta/openai", "keyEnv": "GEMINI_API_KEY",     "models": {} },
    { "name": "cerebras",   "baseUrl": "https://api.cerebras.ai/v1",                              "keyEnv": "CEREBRAS_API_KEY",   "models": {} },
    { "name": "openrouter", "baseUrl": "https://openrouter.ai/api/v1",                            "keyEnv": "OPENROUTER_API_KEY", "models": {} }
  ]
}
```

- A provider whose key isn't set in `.env` is ignored, as in `llm.js`.
- An alias missing for a provider means that provider isn't used for it (e.g. no `vision` model, so it's skipped for image requests).
- `npm run models` lists every provider's available models (via each one's `GET /models`) to make filling in aliases easy.

Secrets go in the root `.env`: `GATEWAY_API_KEY`, `GROQ_API_KEY`, `GEMINI_API_KEY`, `OPENROUTER_API_KEY`, `CEREBRAS_API_KEY`.

## Layout

```
llm-gateway/
  src/
    server.js      HTTP server, routes, auth
    router.js      candidate ordering + failover loop
    provider.js    OpenAI-compatible adapter (request, stream, error classification)
    cooldown.js    per provider+model cooldown with backoff
    quota.js       rpm/rpd counters, persisted to /data/usage.json
    config.js      loads providers.json + env, validates
    log.js         structured JSON logs
  scripts/models.js
  test/            node:test suites against a fake provider server
  providers.json
  Dockerfile
  package.json
```

## Milestones

1. ✅ **Core** (done): config, adapter, router with failover and error classification, cooldowns, non-streaming `/v1/chat/completions`, `/health`, gateway auth, tests for every row of the failure table.
2. **Quotas and visibility:** rpm/rpd limits, saved usage counts, `/status`, `npm run models`.
3. **Streaming and models:** `stream: true` passthrough, `/v1/models`, aliases.
4. **Hook-up:**
   - ✅ Add the `llm-gateway` service to `docker-compose.yml`, reachable only on the Docker network and from localhost (done early; each container only gets its own env vars).
   - Create an n8n **OpenAI** credential: base URL `http://llm-gateway:8080/v1`, key `GATEWAY_API_KEY`.
   - Switch the LinkedIn workflow: replace the Claude node with a gateway call (`model: "smart"`, `response_format: {"type": "json_object"}`) and parse the JSON leniently (not every free model supports strict JSON schemas).
   - Add a failure email ("no draft today: all providers failed") so a quiet day isn't mistaken for a working one.
   - Remove the Anthropic step from the setup guide.

## Risks

- **Free tiers change** (models retired, limits cut). That's why everything lives in config, and `/status` plus `npm run models` make drift visible.
- **Privacy:** some free tiers may use prompts for training (Gemini's free tier, some OpenRouter `:free` models). Don't send private data through the gateway.
- **Inconsistent quality:** post style can vary with whichever provider answers. Put the provider you like best first and use `priority` routing for the LinkedIn workflow.
- **Feature gaps:** JSON mode, tool calling and image input differ per model. Aliases should point only at models that support what the alias promises.
