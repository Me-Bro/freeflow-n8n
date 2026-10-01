# llm-gateway

OpenAI-compatible endpoint that sends each request to an available free provider (Groq, Gemini, Cerebras, OpenRouter) and moves to the next one on rate limits, outages or bad keys. Design and roadmap: [docs/llm-gateway-plan.md](../docs/llm-gateway-plan.md).

## Run

1. In the repo root `.env`, set `GATEWAY_API_KEY` and at least one provider key (see `.env.example`).
2. Start it with the rest of the stack, from the repo root:

   ```sh
   docker compose up -d --build llm-gateway
   ```

   It listens on `localhost:8080` for this computer and on `http://llm-gateway:8080` inside the Docker network (that's the URL n8n uses). `providers.json` is mounted into the container, so model changes only need `docker compose restart llm-gateway`.

   Without Docker (Node 22+, no dependencies): `npm start` / `npm test` in this folder.

3. Call it like OpenAI:

   ```sh
   curl http://127.0.0.1:8080/v1/chat/completions \
     -H "Authorization: Bearer $GATEWAY_API_KEY" -H "Content-Type: application/json" \
     -d '{"model":"smart","messages":[{"role":"user","content":"Say hi"}]}'
   ```

   The `x-llm-provider` and `x-llm-model` response headers show who answered.

## Models

`model` is either an alias (`fast`, `smart`, `json`, `vision`) or a real model name. Each provider maps the aliases in `providers.json`. A provider without a mapping for an alias is skipped for it, and a real model name goes only to providers that list it. The model names in `providers.json` are starting points: check them against each provider's model list once you have keys.

## Failover rules

| Provider response | Effect |
|---|---|
| 429 | cool down for `Retry-After`, otherwise 60s, doubling to 15 min |
| 5xx / timeout / network error | cool down 30s |
| 401 | provider disabled until restart (bad key) |
| 403 | that provider+model disabled until restart (some providers use 403 for model-specific restrictions) |
| 400 / 413 / 422, or bad JSON when `response_format` asks for JSON | try the next provider, no cooldown |

Cooling providers are still tried, just last. If every provider fails, the gateway returns 503 with each attempt, or 400 if every provider rejected the request itself.

## Environment

| Variable | Default | |
|---|---|---|
| `GATEWAY_API_KEY` | required | key callers send as `Authorization: Bearer …` |
| `GROQ_API_KEY`, `GEMINI_API_KEY`, `CEREBRAS_API_KEY`, `OPENROUTER_API_KEY` | | providers without a key are skipped |
| `HOST` / `PORT` | `127.0.0.1` / `8080` | |
| `PROVIDERS_FILE` | `./providers.json` | |
| `LOG_LEVEL` | `info` | `debug`, `info`, `warn`, `error` |
