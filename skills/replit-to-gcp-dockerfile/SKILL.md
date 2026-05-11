---
name: replit-to-gcp-dockerfile
description: Build the production Dockerfile + .dockerignore for a Replit-localized Node + Vite + Express project targeting Cloud Run. Includes the three companion code changes the Dockerfile needs to work in production — `DB_SSL_MODE` env override (so Cloud SQL Auth Proxy doesn't force TLS), `/api/health` healthcheck endpoint (NOT `/healthz` — GFE reserves it), and reserved-PORT-env-var avoidance. Invoke as part of `replit-to-gcp` Step 2 or directly when the user asks "containerize this app" / "give me a production Dockerfile." Modifies code.
---

# Production Dockerfile for Replit-Localized Node + Vite + Express

Multi-stage alpine-based Dockerfile sized for Cloud Run (~450 MB uncompressed, ~140 MB compressed). Plus three small code changes the production runtime needs.

## Prerequisites

- `replit-localize-dep-cleanup` has run. Without it, the runtime `node_modules` is ~700 MB and the image will be ~1 GB. The dep cleanup is non-negotiable for an efficient image.
- `replit-localize-migration-consolidation` has run. The migration runner needs to handle fresh-DB state correctly because it runs at container start.

## The Dockerfile

Place at repo root as `Dockerfile`:

```Dockerfile
# syntax=docker/dockerfile:1.7
#
# Production container. Targeted at Cloud Run; runnable locally for parity testing.
# Build: docker build -t <app>:local .
# Run:   docker run --rm -p 8080:8080 --env-file .env.local <app>:local

# ---- Builder ------------------------------------------------------------
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --no-audit --no-fund
COPY . .
RUN npm run build

# ---- Runner -------------------------------------------------------------
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production \
    PORT=8080

# bash needed for scripts/start-prod.sh's `set -o pipefail` (busybox ash supports
# pipefail on recent versions but pinning bash is cheaper than relying on that).
RUN apk add --no-cache bash

# Production deps only.
COPY package.json package-lock.json ./
RUN npm ci --omit=dev --no-audit --no-fund && npm cache clean --force

# Build outputs + runtime files.
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/migrations ./migrations
COPY scripts/db-migrate.js scripts/post-deploy.js scripts/start-prod.sh ./scripts/

# Non-root execution. node:1000 already exists in the official image.
RUN chown -R node:node /app && chmod +x ./scripts/start-prod.sh
USER node

EXPOSE 8080
CMD ["bash", "scripts/start-prod.sh"]
```

### Why alpine, not bookworm-slim

`node:20-alpine` is ~50 MB base; `node:20-bookworm-slim` is ~150 MB. For a Replit-origin Node + Express + pg + pdfkit + archiver + @aws-sdk stack, all the runtime deps are pure JS — musl-libc compatibility isn't a concern.

If you encounter a native module that doesn't have musl prebuilds, add `apk add --no-cache python3 make g++` to the builder stage (these are stripped from the runner). But for typical Replit-Agent fullstack projects, pure-alpine works.

### Why `apk add bash`

`scripts/start-prod.sh` uses `set -o pipefail`, which is bash-specific. Alpine's busybox ash supports `pipefail` on recent versions, but pinning bash is cheaper than relying on that and survives future busybox changes. The cost is ~2 MB.

Alternative: rewrite `start-prod.sh` as a Node entrypoint to drop the bash dependency. More invasive; not worth the savings.

### Why only `scripts/db-migrate.js scripts/post-deploy.js scripts/start-prod.sh`

NOT the whole `scripts/` directory. The seed scripts (`scripts/seed/*.ts`) need tsx (a devDep) and aren't called at runtime anyway. Copying only the three runtime JS scripts keeps the image lean.

## `.dockerignore`

Place at repo root as `.dockerignore`:

```
# VCS / editor / OS
.git
.gitignore
.github
.vscode
.idea
.DS_Store

# Environment files (never bake secrets into images)
.env
.env.local
.env.*.local
.env.example

# Local dev artifacts
node_modules
dist
*.log
*.tar.gz
.claude
.claude-user

# Replit residue (defensive — most should be deleted by now)
.replit
replit.nix
replit.md

# Local-only Postgres compose
compose.yaml

# Historical migrations (not on the active path; archived for reference)
migrations-archive

# Replit Agent design specs (build-time artifacts, not runtime)
attached_assets

# Docs not needed at runtime
docs
README.md

# Don't ship the Dockerfile/dockerignore into the image
Dockerfile
.dockerignore

# Python venvs / Mac quarantine
__pycache__
*.pyc
```

The big wins: `node_modules` (rebuilt fresh in the builder stage), `attached_assets/` (Replit working dir), `migrations-archive/` (legacy migrations), and `docs/` (not runtime).

## Companion code change 1 — `DB_SSL_MODE` env override

The Replit-localized codebase likely has this anti-pattern in `server/db.ts` and `scripts/db-migrate.js`:

```ts
ssl: process.env.NODE_ENV === "production" ? { rejectUnauthorized: false } : false
```

Two reasons this breaks in production:

1. **Cloud SQL via the Auth Proxy uses a Unix socket** at `/cloudsql/<conn>`. No pg-level TLS — the proxy itself is the authenticated TLS layer. Forcing SSL fails.
2. **`esbuild` bakes `NODE_ENV="production"` into the bundle** at build time. So the env-var-based toggle inside the bundle can't be changed at runtime.

Refactor to a `DB_SSL_MODE` env var with three modes:

```ts
function resolveSsl() {
  const mode =
    process.env.DB_SSL_MODE ??
    (process.env.NODE_ENV === "production" ? "no-verify" : "disable");
  switch (mode) {
    case "disable":   return false;
    case "no-verify": return { rejectUnauthorized: false };
    case "verify":    return { rejectUnauthorized: true };
    default: throw new Error(`Invalid DB_SSL_MODE: ${mode}`);
  }
}

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: resolveSsl(),
});
```

Apply at every pool construction site — typically `server/db.ts` AND `scripts/db-migrate.js`. The migration runner can't import from `server/` (different module context), so inline the helper in both places.

Cloud Run uses `DB_SSL_MODE=disable` (socket connection). Public-IP'd managed Postgres setups use `DB_SSL_MODE=no-verify`. Strict-TLS setups use `verify`.

## Companion code change 2 — `/api/health` (NOT `/healthz`)

Add a healthcheck endpoint in `server/index.ts`, **registered before any middleware**:

```ts
import express from "express";
const app = express();

// Healthcheck. Registered before any middleware so it can't be affected by
// body parsers, auth, rate limiters, or request loggers. No DB dependency —
// answers "is the Express server process alive and listening" only.
//
// Path is /api/health, not /healthz: Google Front End (which fronts Cloud Run)
// appears to reserve /healthz for its own internal probing and returns a 404
// at the edge before requests reach the container. Confirmed experimentally.
// /api/health is unambiguous, on the API surface, and passes through cleanly.
app.get("/api/health", (_req, res) => {
  res.status(200).type("text/plain").send("ok");
});

// … rest of middleware setup
```

**Critical:** the path is `/api/health`, NOT `/healthz`. The Kubernetes convention `/healthz` is intercepted by GFE in front of Cloud Run and returns a 404 to external requests. You can confirm this by hitting `/healthz` vs any other 404 path — `/healthz` returns 1568 bytes of Google-branded error HTML; other 404s pass through to the Express catch-all. There's no documented workaround; just use a different path.

## Companion code change 3 — DON'T set `PORT` explicitly

Cloud Run **reserves** the `PORT` env var and injects it itself based on `container_port` in the service config. Adding `env { name = "PORT" }` to the Cloud Run service definition is rejected:

```
The following reserved env names were provided: PORT.
These values are automatically set by the system.
```

`PORT` is set by Cloud Run; the application reads it as usual. The Dockerfile has `ENV PORT=8080` for local-docker-run parity (when you `docker run` locally, Cloud Run isn't there to inject PORT), but the production Cloud Run service config does NOT have a PORT env block. Documented in `replit-to-gcp-terraform-cloud-run`.

Other reserved names to be aware of: `K_SERVICE`, `K_REVISION`, `K_CONFIGURATION`, `CLOUD_RUN_TIMEOUT_SECONDS`.

## Local verification

```bash
docker build -t <app>:local .

# Start local Postgres (from replit-localize-compose-postgres)
npm run db:up

# Run the production container against local Postgres
docker run --rm -d -p 8080:8080 --name <app>-test \
  -e DATABASE_URL="postgresql://<user>:<pass>@host.docker.internal:5432/<dbname>" \
  -e SESSION_SECRET=local-test \
  -e DB_SSL_MODE=disable \
  <app>:local

sleep 6
docker logs <app>-test | tail -10
# Expect:
#   [start-prod] Running database migrations...
#   [db-migrate] ✅ Migrations up to date.
#   [start-prod] Starting server...
#   [express] serving on port 8080

curl -sS -w 'HTTP %{http_code}\n' http://localhost:8080/api/health
# Expect: ok\nHTTP 200

curl -sS -w 'HTTP %{http_code}\n' http://localhost:8080/
# Expect: <SPA HTML>\nHTTP 200

curl -sS -w 'HTTP %{http_code}\n' http://localhost:8080/api/auth/me
# Expect: {"success":false,"error":"Authentication required"}\nHTTP 401

docker rm -f <app>-test
```

If `/api/health` doesn't return 200, your route isn't registered or middleware is intercepting it.

## Image size expectations

| Stage | Size |
| --- | --- |
| Builder image (intermediate, discarded) | ~600 MB |
| Runner image (final, what Cloud Run pulls) | ~450 MB uncompressed |
| Compressed layer total | ~140 MB |

If the runner image is significantly bigger (>600 MB), `replit-localize-dep-cleanup` wasn't run (or didn't move enough deps to `devDependencies`).

## When to commit

After the local-docker-run verification passes:

```
feat(deploy): production Dockerfile + DB_SSL_MODE + /api/health

- Multi-stage alpine Dockerfile, ~450 MB runner image
- .dockerignore excludes attached_assets/, migrations-archive/, .env*, etc.
- DB_SSL_MODE env override in server/db.ts and scripts/db-migrate.js
  (replaces NODE_ENV-based toggle; supports Cloud SQL Auth Proxy socket
  connections which don't use pg-level TLS)
- /api/health endpoint registered before any middleware (NOT /healthz —
  GFE reserves it at the edge)
- Container verified end-to-end against local Postgres
```

## What this skill doesn't do

- Doesn't provision GCP infrastructure. That's the Terraform sub-skills.
- Doesn't deploy. That's `replit-to-gcp-gha-deploy`.
- Doesn't configure the Cloud Run startup_probe. That's in `replit-to-gcp-terraform-cloud-run` — but `/api/health` is the path it'll point to.

## Parent skill

Invoked from `replit-to-gcp` Step 2. The image this produces is what every subsequent Terraform slice references (Artifact Registry stores it; Cloud Run pulls it; the deploy workflow pushes it).
