---
name: replit-localize-compose-postgres
description: Replace Replit's auto-provisioned Postgres with a local Docker Compose service so developers can clone the repo and `docker compose up` without Replit. Invoke as part of `replit-localize` Step 1 or directly when the user asks "set up local Postgres" / "I want to run this off Replit on my laptop". Modifies the codebase — creates `compose.yaml` and adds `db:up`/`db:down` npm scripts.
---

# Local Postgres via Docker Compose

Replace Replit's implicit Postgres with a Compose service. Single file (`compose.yaml`) at the repo root, plus two npm scripts.

## Why Compose, why these specific flags

- **Compose** is universally available on developer machines (Docker Desktop, OrbStack, Colima, Podman). No platform-specific tooling.
- **`postgres:16`** matches what Replit auto-provisions in 2024+ Replit-Agent fullstack projects. Don't mismatch — drift between dev Postgres and prod Postgres surfaces as feature-version bugs.
- **`--wait` flag** (Compose v2.20+) on `docker compose up --wait postgres` blocks until the healthcheck passes. Lets the npm script chain (`db:up && db:migrate && db:seed`) work without ad-hoc sleep loops or wait scripts.
- **`uniform-bucket-level-access`-style isolation** isn't needed locally — but a named volume + healthcheck is.

## `compose.yaml`

Place at repo root:

```yaml
services:
  postgres:
    image: postgres:16
    container_name: <project-short-name>-postgres   # e.g., fls-ask-postgres
    environment:
      POSTGRES_USER: <user>                          # e.g., ask
      POSTGRES_PASSWORD: <password>                  # local-dev only; not a real secret
      POSTGRES_DB: <dbname>                          # e.g., ask_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U <user> -d <dbname>"]
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  postgres_data:
```

Substitute the four `<…>` placeholders. The local-dev password is intentionally not a real secret — it doesn't need to be — but match it consistently across `compose.yaml`, `.env.example`, and any `DATABASE_URL` defaults.

## npm scripts

Add to `package.json`:

```json
{
  "scripts": {
    "db:up":   "docker compose up -d --wait postgres",
    "db:down": "docker compose down"
  }
}
```

Don't put `--wait` inside the YAML — keep it in the script so it's explicit. The flag is what makes `db:up && db:migrate` work without sleeps.

## Connection string

In `.env.example` (created during the `replit-localize-dotenv` step):

```
DATABASE_URL=postgresql://<user>:<password>@localhost:5432/<dbname>
```

On Linux/macOS this connects to the published port 5432. On Windows with WSL2, same — Compose publishes via the host network.

## Verify

After `npm run db:up`:

```bash
docker compose ps postgres   # STATUS column should read "healthy"
psql postgresql://<user>:<password>@localhost:5432/<dbname> -c "SELECT version();"
```

## Common pitfalls

- **Port 5432 already in use** — the developer has Postgres running natively. Either stop the native one (`brew services stop postgresql`) or change the published port (`"5433:5432"` in compose) and update DATABASE_URL.
- **Volume persists stale data** — `npm run db:down` keeps the volume; `docker compose down -v` removes it (use only when you want a fresh DB).
- **Replit projects often had their own Postgres URL convention** — Replit injects `DATABASE_URL` as `postgresql://...@some-replit-host`. The local replacement needs the same env-var name but a different value. Caught by the env-var contract audit (`replit-audit-env-vars`).

## When to commit

After Compose + npm scripts work end-to-end (`npm run db:up` succeeds, `psql` connects). One commit:

```
feat(local-dev): compose-based Postgres for local development

- compose.yaml with healthcheck for postgres:16
- db:up / db:down npm scripts using --wait flag
- .env.example updated with local DATABASE_URL pattern
```

## What this skill doesn't do

- Doesn't write migrations (`replit-localize-migration-consolidation` handles that).
- Doesn't seed (`replit-localize-dotenv` adds the seed script; the seed itself usually lives at `scripts/seed/`).
- Doesn't address production Postgres (`replit-to-gcp-terraform-cloud-sql` handles that).

## Parent skill

Invoked from `replit-localize` Step 1. The output enables every subsequent step that needs a working local DB.
