# n8n-101

Local [n8n](https://n8n.io) instance for automating stuff, run with Docker Compose, plus [llm-gateway](llm-gateway/README.md): one OpenAI-compatible endpoint that fails over between free LLM providers.

| Service | URL from this computer | URL from n8n |
|---|---|---|
| n8n | http://localhost:5678 | |
| llm-gateway | http://localhost:8080/v1 | http://llm-gateway:8080/v1 |

## First-time setup

1. Copy `.env.example` to `.env` and set `N8N_ENCRYPTION_KEY` and `GATEWAY_API_KEY` to random values (`openssl rand -hex 32`). Back up the encryption key.
2. Add the free LLM provider keys you have to `.env` (`GROQ_API_KEY`, etc.).
3. `docker compose up -d --build`
4. Open http://localhost:5678 and create the owner account.

Both services read the one `.env`, but each container only gets its own variables, so n8n never sees the LLM provider keys.

## Everyday commands

| Task | Command |
|---|---|
| Start | `docker compose up -d` |
| Stop | `docker compose down` |
| Logs | `docker compose logs -f n8n` / `docker compose logs -f llm-gateway` |
| Upgrade n8n | `docker compose pull; docker compose up -d` |
| After changing gateway code | `docker compose up -d --build llm-gateway` |
| After changing keys in `.env` | `docker compose up -d` (`restart` does not pick up `.env` changes) |
| After changing `llm-gateway/providers.json` | `docker compose restart llm-gateway` |

Data lives in the `n8n_data` Docker volume, so it survives restarts and upgrades. `docker compose down -v` **deletes it**.

## Versioning workflows in git

The UI is the source of truth; export to `workflows/` to commit snapshots:

```sh
# Export all workflows (one file each)
docker compose exec n8n n8n export:workflow --all --separate --output=/home/node/workflows/

# Import them back (e.g. on a fresh instance)
docker compose exec n8n n8n import:workflow --separate --input=/home/node/workflows/
```

Credentials are not exported this way. Re-create them in the UI.

## Workflows

- [RootEd: daily LinkedIn post](docs/rooted-daily-linkedin-post.md): Claude drafts a post about RootEd every morning, emails it for approval, and posts it to LinkedIn.

## Folders

- `workflows/`: exported workflow JSON (committed)
- `files/`: mounted at `/files` in the container, for workflows that read/write files (gitignored)
