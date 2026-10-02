# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

**freeflow**: a local n8n instance (Docker Compose) plus `llm-gateway`, a small Node service that exposes one OpenAI-compatible endpoint and fails over between free LLM providers (Groq, Gemini, Cerebras, OpenRouter). Most "code" here is n8n workflow JSON in `workflows/`, with a setup/usage guide per workflow in `docs/`. The only real codebase is `llm-gateway/`.

## Commands

Stack (run from repo root; see README for the full table):

- Start / rebuild: `docker compose up -d --build`
- After changing gateway source: `docker compose up -d --build llm-gateway`
- After changing `llm-gateway/providers.json`: `docker compose restart llm-gateway` (the file is bind-mounted read-only)
- After changing `.env`: `docker compose up -d` (`restart` does **not** re-read `.env`)
- `docker compose down -v` deletes the `n8n_data` volume (all workflows, credentials, history). Don't run it.
- Ports: host `15678` → n8n and `18080` → llm-gateway (unusual on purpose, to avoid clashing with other local projects). Inside the Docker network they remain `5678` / `8080`, and `N8N_PORT=5678` in `.env` is the in-container port. If you change a host port, also update `WEBHOOK_URL`, the docs, and the OAuth redirect URLs (Google, LinkedIn).
- The volume is pinned to `name: n8n-101_n8n_data` in `docker-compose.yml`. Without that, Compose would name it after the folder (`freeflow_n8n_data`) and a folder rename would silently start n8n with an empty database. Keep the `name:` line.

llm-gateway (run inside `llm-gateway/`; Node 22+, zero dependencies, no build step, no linter):

- Tests: `npm test` (`node --test test/*.test.js`)
- Single test: `node --test --test-name-pattern="real model name" test/gateway.test.js`
- Run outside Docker: `npm start` (loads `../.env` via `--env-file-if-exists`; `config.js` defaults to `127.0.0.1:18080`, matching the Docker host port, while the Dockerfile sets `PORT=8080` inside the container)

Workflows (n8n CLI inside the container; `./workflows` is mounted at `/home/node/workflows`):

- Export one: `docker compose exec n8n n8n export:workflow --id=<ID> --output=/home/node/workflows/<file>.json --pretty`
- Import: `docker compose exec n8n n8n import:workflow --input=/home/node/workflows/<file>.json`
- Workflow IDs are fixed and human-readable: `RootEdLinkedIn01` (`rooted-daily-linkedin-post.json`), `SheetLeadEmail01` (`lead-outreach-email.json`).

## llm-gateway architecture

Plain ESM, `node:http` + global `fetch`, no framework. Request path:

1. `server.js`: routes (`GET /health`, `POST /v1/chat/completions`), Bearer auth against `GATEWAY_API_KEY` for every `/v1/*` path, body validation, OpenAI-style error bodies (`{error: {message, type}}`) so n8n's OpenAI nodes surface the message. Streaming is rejected with 400.
2. `router.js`: `plan()` resolves the requested `model` per provider (an alias key in `providers[].models` maps to that provider's model; a real model name only matches providers that list it as a value), applies `priority` or `round-robin` ordering, drops disabled entries, and puts cooling ones last. `complete()` tries candidates in order and maps each `ProviderError.kind` to a cooldown effect. All-fail is 503, or 400 if every attempt was `kind: 'request'`.
3. `provider.js`: the single adapter for all providers. It classifies failures into `rate_limit | server | auth | request | bad_json`, treats HTTP 200 bodies containing `error` as failures (OpenRouter does this), and when `response_format` is `json_object`/`json_schema` it extracts JSON from fenced/chatty output and rewrites `message.content`.
4. `cooldown.js`: in-memory state keyed by `provider:model` (cooldowns, 403 disables) or by provider name (401 disables). Nothing persists, so a restart clears all cooldowns and disables.
5. `config.js`: `buildConfig(raw, env)` validates `providers.json` and keeps only providers whose `keyEnv` is set.

Dependencies are injected (`call`, `fetchImpl`, `now`, `log`) so tests run without the network. `test/helpers.js` starts real fake-provider HTTP servers plus the real gateway server, with a fake clock for cooldown timing. `gateway.test.js` covers the failover table end to end and `units.test.js` covers the pure functions. If you change failover behavior, update the failure table in `llm-gateway/README.md` as well.

`docs/llm-gateway-plan.md` is the design and roadmap. Only milestone 1 and the compose hook-up are done. Quotas (`limits`, `/data/usage.json`), `/status`, `/v1/models`, streaming and `npm run models` are described there but **not implemented**.

## n8n workflows

- The n8n UI (its SQLite DB in the `n8n_data` volume) is the source of truth, and `workflows/*.json` are snapshots. If you edit the JSON directly, it only takes effect after an import, and a later UI export will overwrite your edit. Keep both in sync and tell the user which direction is needed.
- Credentials are never in the JSON. Nodes reference credentials the user creates in the UI (Anthropic, SMTP, LinkedIn OAuth2, Google Sheets/Gmail OAuth2, and an OpenAI credential named `llm-gateway` pointing at the gateway).
- Most logic lives in **Code** nodes (`parameters.jsCode`). Each workflow keeps user-editable config in a `SETTINGS` object at the top of a specific Code node (`Build prompt` for the LinkedIn workflow, `List leads` for the applicant email), and the docs tell users to edit it there.
- LLM calls are plain **HTTP Request** nodes that build `requestBody` in a preceding Code node:
  - The LinkedIn workflow calls the Anthropic API directly (`claude-opus-5-5`, with the `server-side-fallback-2026-07-01` beta header).
  - The applicant email workflow calls `http://llm-gateway:8080/v1/chat/completions` with `model: "json"`. Inside Docker, n8n reaches the gateway by service name. From the host it's `localhost:18080`.
- The LinkedIn workflow stores post history in `$getWorkflowStaticData('global')`, which is only saved for scheduled runs, not manual test runs.
- Each workflow has a matching guide in `docs/` and a link in the README's "Workflows" list. When you add or change a workflow, update both.

## Environment

All settings live in one root `.env` (template: `.env.example`). `docker-compose.yml` passes each container only its own variables, so n8n never receives the provider API keys. Keep it that way when you add variables. Both services bind to `127.0.0.1` only. Timezone defaults to `Asia/Kolkata`.
