---
name: replit-localize-dotenv
description: Replace Replit's auto-injected environment variables with a `.env.local`-based pattern that uses Node's built-in `--env-file` flag (no `dotenv` runtime dep). Invoke as part of `replit-localize` Step 2 or directly when the user asks "how do I set env vars locally without Replit?" / "do I need dotenv?". Modifies `package.json`, creates `.env.example` and `.gitignore` entry, and adds a self-loader to `drizzle.config.ts` if drizzle is in use.
---

# Dotenv Strategy via `--env-file`

Use Node ≥20.6's built-in `--env-file=` flag for development scripts. Avoid the `dotenv` or `dotenv-cli` package entirely.

## Why no `dotenv` package

Three reasons:

1. **No runtime dep.** Production runs without `dotenv` because the platform injects env vars natively (Cloud Run reads from Secret Manager; Railway from its own panel; Vercel etc.). Adding `dotenv` to dependencies just for dev is dead weight in the production container.
2. **Different start paths for dev vs prod, visibly.** When the dev script reads `--env-file=.env.local` and the prod script doesn't, the difference is right in `package.json` — no hidden conditional behavior.
3. **Node has this built in** since v20.6.0. Avoid the package if Node provides the feature.

## `package.json` scripts

```json
{
  "scripts": {
    "dev":        "tsx --env-file=.env.local server/index.ts",
    "db:migrate": "node --env-file=.env.local scripts/db-migrate.js",
    "db:seed":    "tsx --env-file=.env.local scripts/seed/<your-seed>.ts",
    "setup":      "npm run db:up && npm run db:migrate && npm run db:seed",
    "start":      "NODE_ENV=production node dist/index.cjs"
  }
}
```

The `start` script (production) intentionally has NO `--env-file=` — the platform injects env vars. If you see `--env-file` in a production script, that's a sign the developer didn't trust the platform's env injection and worked around it. Investigate before keeping.

## `.env.example`

Create a committed example at repo root documenting every env var from `replit-audit-env-vars`'s output. Use commented placeholder values, not real secrets:

```
# Database — see compose.yaml for local credentials
DATABASE_URL=postgresql://ask:ask@localhost:5432/ask_dev

# Session signing — any sufficiently random string is fine for local dev
SESSION_SECRET=local-dev-session-secret-not-for-production

# R2 / S3-compatible object storage
R2_ENDPOINT=https://<account_id>.r2.cloudflarestorage.com
R2_BUCKET=<bucket-name>
R2_ACCESS_KEY_ID=<from-Cloudflare-dashboard>
R2_SECRET_ACCESS_KEY=<from-Cloudflare-dashboard>

# Anthropic API
ANTHROPIC_API_KEY=<sk-ant-…>

# … one entry per env var the app reads
```

The developer runs `cp .env.example .env.local`, fills in real values, and is set.

## `.gitignore`

Confirm `.env.local`, `.env*.local` are in `.gitignore`. `.env.example` IS committed; the `.local` variants are NOT.

## Tools whose config is a JS/TS file — inline a 6-line dotenv loader

`drizzle-kit`, Vite, and similar tools have config files (`drizzle.config.ts`, `vite.config.ts`) that load BEFORE any npm script wrapping happens. They can't see `.env.local` through the `--env-file=` flag. Two options:

1. Wrap every invocation with a dotenv-cli command — adds a devDep, more noise.
2. **Inline a tiny loader at the top of the config file.** Cleaner.

Pattern for `drizzle.config.ts`:

```ts
// drizzle.config.ts — load .env.local at config-load time
import { existsSync, readFileSync } from "node:fs";
import { resolve } from "node:path";

const envPath = resolve(__dirname, ".env.local");
if (existsSync(envPath)) {
  for (const line of readFileSync(envPath, "utf8").split("\n")) {
    const m = line.match(/^([A-Z_][A-Z0-9_]*)=(.*)$/);
    if (m && !process.env[m[1]]) process.env[m[1]] = m[2].replace(/^["']|["']$/g, "");
  }
}

// … rest of drizzle config
```

Six lines. No-op when `.env.local` is absent (production). The `!process.env[m[1]]` check means the platform's already-set env vars win over `.env.local` — so the same config file works in dev and prod without changes.

Same pattern works for any tool with a JS/TS config (Vite, ESLint, etc.) that needs to read env vars at config-load time.

## Verify

```bash
# Dev script picks up .env.local
npm run dev
# Server logs should show DATABASE_URL connection, no "DATABASE_URL is undefined" errors.

# Drizzle picks up .env.local
npx drizzle-kit generate
# Should succeed without "DATABASE_URL is required" error.

# Production script does NOT pick up .env.local
DATABASE_URL="" npm run start
# Should fail with the expected "DATABASE_URL is required" error — confirms prod doesn't read .env.local.
```

## When to commit

After `npm run dev` runs cleanly against the local Compose Postgres. One commit:

```
feat(local-dev): dotenv strategy via --env-file flag

- npm scripts use Node's built-in --env-file= flag for dev
- .env.example documents every required env var
- drizzle.config.ts self-loads .env.local (no dotenv-cli devDep needed)
- .env.local is gitignored; .env.example is committed
```

## What this skill doesn't do

- Doesn't decide which env vars are needed. That's the `replit-audit-env-vars` output.
- Doesn't handle secret management for production. `replit-to-gcp-terraform-secrets` handles that.

## Parent skill

Invoked from `replit-localize` Step 2. Output makes every subsequent step (migrations, seeds, server boot) work locally.
