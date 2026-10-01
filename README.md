# n8n-101

Local [n8n](https://n8n.io) instance for automating stuff, run with Docker Compose.

## First-time setup

1. Copy `.env.example` to `.env` and set `N8N_ENCRYPTION_KEY` to a random value (`openssl rand -hex 32`). Back this key up.
2. `docker compose up -d`
3. Open http://localhost:5678 and create the owner account.

## Everyday commands

| Task | Command |
|---|---|
| Start | `docker compose up -d` |
| Stop | `docker compose down` |
| Logs | `docker compose logs -f n8n` |
| Upgrade | `docker compose pull; docker compose up -d` |

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
