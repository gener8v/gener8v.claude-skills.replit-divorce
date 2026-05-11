---
name: replit-localize
description: Make a Replit-origin project runnable on a developer's machine without Replit's implicit infrastructure (auto-Postgres, auto-secrets, auto-deployment). Use after `replit-audit` is complete or when the user says "make this work locally," "decouple from Replit," "set up local Postgres," or any variant. Modifies the codebase to add Compose-based services, dotenv strategy, fix portability bugs, and consolidate migration debt. Ready-state criterion: `npm install && npm run setup && npm run dev` works on a fresh clone.
---

# Replit Project Localization

You are converting a Replit-origin codebase into one that runs anywhere a Node + Docker developer can clone it. The ready-state criterion is objective:

> A fresh `git clone` on a clean machine, followed by `npm install && npm run setup && npm run dev`, produces a working application connected to a working database with seeded data.

If that isn't true at the end, the skill isn't done.

This skill **modifies the codebase**. Commit each cohesive change with a meaningful message and push as you go. Don't batch unrelated work.

## Prerequisites

- A `replit-audit` pass should have run, producing an inventory of Replit-coupling. If it hasn't, do a light audit first (read `.replit`, grep for `@replit/`, list env vars) before starting changes.
- The user should be on a Mac or Linux dev machine with Docker, Node 20+, and git.

## Work plan

The localization work falls into seven cohesive steps. Each gets its own commit. Order matters because later steps reference earlier outputs.

1. **Local Postgres via Compose** — replace Replit's auto-Postgres.
2. **Dotenv strategy** — replace Replit's auto-secrets with `.env.local` + `--env-file` flag.
3. **Migration debt repayment** — consolidate `drizzle-kit push` history into a clean baseline.
4. **Migration runner hardening** — ensure the runner survives the consolidation cleanly.
5. **Boot verification** — fix the cross-platform bugs that surface once you actually try to run the app locally (port collisions, Linux-only APIs).
6. **Security cleanup** — VLN-001 (`.replit` plaintext secrets) and VLN-002 (bundled credentials) per the audit.
7. **Dependency cleanup** — remove deps the audit flagged as unused and reclassify client-only deps that shouldn't be in runtime `dependencies`.

## Step 1 — Local Postgres via Compose

Create `compose.yaml` at repo root:

```yaml
services:
  postgres:
    image: postgres:16
    container_name: <project-name>-postgres
    environment:
      POSTGRES_USER: <user>
      POSTGRES_PASSWORD: <password>
      POSTGRES_DB: <dbname>
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

Add npm scripts wrapping `docker compose`:

```json
{
  "scripts": {
    "db:up":   "docker compose up -d --wait postgres",
    "db:down": "docker compose down"
  }
}
```

The `--wait` flag (Compose v2.20+) makes `db:up` block until the healthcheck passes — no need for sleep loops or ad-hoc wait scripts. Subsequent `db:migrate` and `db:seed` chain cleanly off this.

Commit: `feat(local-dev): compose-based Postgres for local development`.

## Step 2 — Dotenv strategy

Use Node ≥20.6's `--env-file=` flag for dev scripts. Avoid adding `dotenv` or `dotenv-cli` as a runtime/dev dependency.

In `package.json`:

```json
{
  "scripts": {
    "dev":       "tsx --env-file=.env.local server/index.ts",
    "db:migrate":"node --env-file=.env.local scripts/db-migrate.js",
    "db:seed":   "tsx --env-file=.env.local scripts/seed/<your-seed>.ts"
  }
}
```

For tools whose config is itself a JS/TS file (e.g., `drizzle.config.ts`), inline a 6-line dotenv-style loader so the config can read `.env.local` at load time without a wrapper:

```ts
// drizzle.config.ts
import { existsSync, readFileSync } from "node:fs";
import { resolve } from "node:path";

const envPath = resolve(__dirname, ".env.local");
if (existsSync(envPath)) {
  for (const line of readFileSync(envPath, "utf8").split("\n")) {
    const m = line.match(/^([A-Z_][A-Z0-9_]*)=(.*)$/);
    if (m && !process.env[m[1]]) process.env[m[1]] = m[2].replace(/^["']|["']$/g, "");
  }
}
```

No-op when `.env.local` is absent (production). Cleaner than wrapping every `drizzle-kit` invocation. Same pattern works for any tool with a JS/TS config.

Create `.env.example` documenting every variable from the audit's env-var contract. **Do not commit `.env.local`** — add to `.gitignore` if not already there.

Commit: `feat(local-dev): dotenv strategy via --env-file flag`.

## Step 3 — Migration debt repayment

This step is only needed if the audit flagged migration drift (almost universal for Replit/drizzle projects with a `drizzle-kit push` history). The symptom: `migrations/*.sql` doesn't fully reproduce `shared/schema.ts` on a fresh DB.

**Diagnostic to confirm drift:**

```bash
grep -oE 'pgTable\("([a-z_]+)"' shared/schema.ts \
  | sed -E 's/pgTable\("([a-z_]+)"/\1/' | sort -u > /tmp/schema.txt
grep -hE '^CREATE TABLE' migrations/*.sql \
  | sed -E 's/^CREATE TABLE "?([a-z_]+)"?.*/\1/' | sort -u > /tmp/migrations.txt
comm -23 /tmp/schema.txt /tmp/migrations.txt   # tables in schema but not in migrations
```

If the output is non-empty, drift exists. Also check for duplicate CREATEs:

```bash
grep -hE '^CREATE TABLE' migrations/*.sql | sort | uniq -c | awk '$1>1'
```

Duplicates surface retroactive baseline migrations that handcoded missing tables — they go stale when the initial migration is later regenerated.

**Repayment recipe:**

1. Move existing `migrations/` to `migrations-archive/` (preserves history out of drizzle's view, in the repo for reference).
2. From an empty `migrations/` directory, run `npx drizzle-kit generate --name=baseline`. Produces a single consolidated baseline SQL file from `shared/schema.ts`.
3. Inspect the baseline. Confirm it has every table the schema defines.
4. Update the migration runner (Step 4 below) before applying. The runner needs to handle the prod transition.

Use the regular `Write` tool to create files at the new path — git's rename detection (≥50% similarity threshold) re-discovers the rename automatically when you commit. Same trick avoids `git mv` for `replit.md → docs/PROJECT_CONTEXT.md` and similar moves.

Commit: `feat(migrations): consolidate to single baseline; align DB with schema.ts`.

## Step 4 — Migration runner hardening

The migration runner (`scripts/db-migrate.js` or equivalent) needs to handle three states cleanly:

1. **Fresh empty DB** — `migrate()` applies the baseline normally.
2. **Push-baselined DB** (legacy state from before file-based migrations) — tables exist, no `__drizzle_migrations` entries. Runner should register the current baseline as already-applied without re-executing it.
3. **Migration-history transition** (the first deploy of the consolidated baseline against a prod DB that had the old 0000–N rows) — old hashes exist in `__drizzle_migrations`, none matching the new baseline. Runner should swap: delete the legacy rows, insert a single new baseline row. Idempotent on re-run.

Read the baseline tag and timestamp **dynamically** from `migrations/meta/_journal.json` — do not hardcode them. Pattern (Node):

```js
import { readFileSync } from "node:fs";
import { resolve, dirname } from "node:path";
import { fileURLToPath } from "node:url";
import { createHash } from "node:crypto";

const __dirname = dirname(fileURLToPath(import.meta.url));
const MIGRATIONS_FOLDER = resolve(__dirname, "../migrations");

function readJournal() {
  return JSON.parse(readFileSync(resolve(MIGRATIONS_FOLDER, "meta", "_journal.json"), "utf8"));
}

function hashSqlFile(tag) {
  const sql = readFileSync(resolve(MIGRATIONS_FOLDER, `${tag}.sql`), "utf8");
  return createHash("sha256").update(sql).digest("hex");
}
```

**Test all three scenarios before committing.** Test (1) by dropping the local DB. Test (2) by running once and re-running. Test (3) by manually inserting a fake stale row into `drizzle.__drizzle_migrations` and re-running — the runner should detect the orphan and swap. Skipping these means deploying without proof that the runner is safe in the prod scenario you've never seen locally.

Commit: `feat(migrations): runner handles fresh / push-baselined / transition states`.

## Step 5 — Boot verification

Once Compose is up and migrations apply, actually run `npm run dev`. You will hit cross-platform issues that don't show up on Replit's Linux runtime:

- **`reusePort: true` in `httpServer.listen`** — Linux autoscale optimization. Crashes with `ENOTSUP` on macOS. Strip the option (the application doesn't need it off Replit).
- **Port 5000 default** — macOS reserves it for AirPlay Receiver. Replit's external port mapping (`localPort = 5000 → externalPort = 80`) made 5000 conventional. Pick a non-5000 default (e.g., 5050) for local dev. Document in `.env.example`.
- **`@replit` Twitter meta tag in `client/index.html`** — cosmetic but worth removing for a clean divorce.

Verify by curl: `GET /` returns 200, `GET /api/auth/me` returns the expected error JSON, `GET /api/dev/seed-users` (if applicable) returns expected data in dev mode.

Commit: `fix(local-dev): remove Replit-only socket option and stale meta tag`.

## Step 6 — Security cleanup

Address the VLN items the audit produced.

**VLN-001 — plaintext secrets in `.replit`:** rotate every value. For any `*_ENCRYPTION_KEY`, confirm whether existing data is encrypted with the current value — if so, rotation requires a re-encryption migration, not a swap. Document the constraint; defer the migration to a later phase if the data layer migration isn't in this engagement's scope.

**VLN-002 — bundled credentials in client code:** apply the fix template:

1. Add a `GET /api/dev/seed-users` route gated by `if (process.env.NODE_ENV !== "production")`. Returns array of `{ email, role, displayName }` from the live `users` table — no credentials.
2. Update the dev-login picker UI to `useEffect`-fetch this endpoint at mount instead of importing a bundled credentials map.
3. The click handler fills **only the email field** in the login form. Operators type the password manually (which they know from the seed scripts or out-of-band).
4. Remove the bundled credentials file entirely.
5. Confirm: `grep -rE "<old-bundled-password>" dist/` returns zero hits.

Document the seed-password contract (which password to type) in `docs/PROJECT_CONTEXT.md` as a local-dev convenience, NOT in any bundle.

Commit: `fix(security): drive dev login picker from API, not bundle (VLN-002)`.

## Step 7 — Dependency cleanup

Two passes:

**a) Truly unused deps:**

For each package in `dependencies` (and any "allowlist" in a build script), grep:

```bash
grep -rE "(from|require)\\s*\\(?['\"]<pkg>['\"]" server/ scripts/ shared/ client/ 2>/dev/null
```

Zero hits = unused. Also check for dynamic imports — `await import("<pkg>")` — with a broader pattern:

```bash
grep -rE "['\"]<pkg>['\"/]" server/ scripts/ shared/ 2>/dev/null
```

**b) Client-only deps in runtime `dependencies`:**

Replit-Agent SPA fullstack templates universally put React, all `@radix-ui/*`, animation libs, etc. in `dependencies` rather than `devDependencies`. Vite bundles these into `dist/public/` at build time — they're not needed at runtime in the production container.

Diagnostic — every package imported by non-client code:

```bash
grep -rhoE "from ['\"]([@a-z0-9][^'\"./][^'\"]*)['\"]" server scripts shared script \
  | sed -E "s/from ['\"]//; s/['\"]\$//" \
  | sed -E 's|^(@[^/]+/[^/]+).*|\1|; s|^([^@/][^/]*).*|\1|' \
  | sort -u
```

Anything in `dependencies` but NOT in that list → move to `devDependencies`. Vite still picks them up at build time and bundles them; the runtime container's `npm ci --omit=dev` no longer installs them.

Also: `@types/*` packages are TypeScript-only and never imported at runtime — they always belong in `devDependencies` regardless of where the underlying JS package lives.

Run `npm install` after each change to update the lockfile. Run `npm run build` to confirm bundle sizes are unchanged (proof that the moves were correct).

Commit: `chore(deps): drop unused, move client-only deps to devDependencies`.

## Final verification

The ready-state criterion:

```bash
git clone <repo> /tmp/fresh-clone   # genuinely fresh clone
cd /tmp/fresh-clone
cp .env.example .env.local          # edit if needed
npm install
npm run setup
npm run dev
# … open http://localhost:<PORT> and confirm app loads + login works
```

If any step fails, address before declaring the skill done.

## Output to surface

When complete, summarize for the user:

1. The seven commits, in order, with one-line descriptions.
2. The audit's VLN items: which are now resolved, which are deferred (with reason).
3. The new `npm run setup` chain — explicit list of what each step does.
4. Any remaining Replit-coupling that was deferred (e.g., R2 storage staying as Replit's R2-via-AWS-SDK, which is portable but Replit-flavored).

Then ask whether the user wants to proceed to `replit-to-gcp` immediately or test the localized setup further before deploying.
