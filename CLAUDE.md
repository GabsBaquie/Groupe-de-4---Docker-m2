# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Docker Compose stack for self-hosting [Rallly](https://rallly.co) (scheduling/polling app), backed by PostgreSQL, with Mailpit as a local SMTP sink for lab/dev email testing. No application code lives in this repo — it is pure infrastructure configuration.

## Commands

- Start the stack: `docker compose up -d`
- Stop the stack: `docker compose down`
- View logs: `docker compose logs -f [service]`
- Recreate after config changes: `docker compose up -d --force-recreate`

There is no build, lint, or test tooling in this repo.

## Architecture

Three services defined in `docker-compose.yml`:

- **rallly** — the app, port `3000`. Reads config from env vars (`SECRET_PASSWORD`, `DATABASE_URL`, `SUPPORT_EMAIL`), waits on `db` via healthcheck, and sends outbound mail through `mailpit` (not real SMTP).
- **db** — Postgres 16, persisted via the `db-data` named volume. `rallly` depends on its healthcheck (`pg_isready`), not just container start.
- **mailpit** — fake SMTP server; captures all mail the stack sends. Web UI on port `8025` to read captured emails instead of a real inbox.

Config is injected entirely through environment variables sourced from `.env` (gitignored; see `.env.example` for required keys: `SECRET_PASSWORD`, `POSTGRES_PASSWORD`, `SUPPORT_EMAIL`).
