---
name: replit-audit-lifecycle-scripts
description: Audit a Replit-origin project's lifecycle scripts — `start-prod.sh`, `db-migrate.js`, `post-deploy.js`, and any `*-gate*.sh` — for Replit-specific assumptions, hardcoded constants, and orphaned build paths. Invoke as part of `replit-audit` or when the user asks "how does this app boot?" / "what does start-prod do?". Read-only.
---

# Lifecycle Script Audit

Reads every script in `scripts/` and `script/` and documents the boot/build/deploy sequence, flagging Replit-specific assumptions for portability work.

## What to read

In a typical Replit-Agent fullstack project:

```
scripts/
├── start-prod.sh         ← production entrypoint
├── db-migrate.js         ← migration runner
├── post-deploy.js        ← post-boot hook
├── *-gate*.sh            ← guards / checks
├── seed/                 ← seed scripts
└── …
script/                   ← yes, also a different directory
└── build.ts              ← canonical build
```

Read each script in full. Don't skim — the lifecycle is where Replit-coupling hides.

## What to look for in each script

### `start-prod.sh` (or equivalent production entrypoint)

Document the boot sequence:

```markdown
1. Migrate database (node scripts/db-migrate.js)
2. Start server in background (node ./dist/index.cjs &)
3. Capture PID
4. Run post-deploy hook (node scripts/post-deploy.js)
5. wait $SERVER_PID
```

Flag:

- **Misleading comments** — "Called by Replit's postDeploy hook" comments are often false; the script is actually invoked from `start-prod.sh`. Don't trust comments; trace the call chain.
- **Hardcoded values that should be env-derived** — port numbers, paths, retry counts.
- **`set -o pipefail` or other bash-isms** — fine on Linux, will need `bash` in the runtime container if the base is alpine (no `bash` by default).
- **Reseed-on-every-boot patterns** — running seeds in `start-prod.sh` is a Replit pattern that breaks on Cloud Run autoscale where instances spin up frequently.

### `db-migrate.js` (or equivalent migration runner)

Two patterns to identify:

**Pattern A: synthetic-baseline hack.** Replit users almost always start with `drizzle-kit push` before introducing migration files. So when the file-based migrations are first introduced, migration `0000_*.sql` already represents schema that exists in the live DB. The runner has to "skip" that initial migration by inserting a synthetic row into `drizzle.__drizzle_migrations` before calling `migrate()`. Look for:

```js
const INITIAL_MIGRATION_WHEN = 1234567890;   // hardcoded timestamp
const INITIAL_MIGRATION_HASH = "abc...";      // hardcoded
INSERT INTO drizzle.__drizzle_migrations VALUES (?, ?)
```

This is fragile (the hash and timestamp drift when the baseline migration is regenerated). The fix is to read the baseline tag and timestamp dynamically from `migrations/meta/_journal.json` — done in `replit-localize-migration-consolidation`. Flag the file for that work.

**Pattern B: naive SSL config.** Replit's auto-Postgres works with `ssl: false`. Other Postgres providers don't:

```js
new Pool({
  connectionString: DATABASE_URL,
  ssl: process.env.NODE_ENV === "production" ? { rejectUnauthorized: false } : false
})
```

This is broken for Cloud SQL Auth Proxy (Unix socket — no pg-level TLS needed at all) AND for managed Postgres requiring proper cert chain validation. The fix is `DB_SSL_MODE` env var refactor — done in `replit-to-gcp-dockerfile` (alongside the production container work).

### `post-deploy.js` (or equivalent post-boot hook)

Usually does some combination of:

- Health-checks the local server.
- Calls an admin/seed endpoint to populate first-time data.
- Logs warnings/errors but doesn't exit non-zero (designed not to block server startup).

Flag:

- **Hardcoded URLs** (`http://localhost:5000`) — fine on Replit but may need parametrization off-Replit.
- **Seed endpoints that load files from `attached_assets/`** — Replit's working directory. Any file the production runtime needs from `attached_assets/` needs relocation (done in `replit-localize`).
- **Env-var-gated short-circuits** — `if (!TOKEN) process.exit(0)` patterns. Document the env var; it controls whether the hook fires in production.

### `*-gate*.sh` scripts

Any script with "gate" in the name almost always greps against `.replit` to enforce that it's running inside Replit:

```bash
grep -q "agent.stack" .replit || { echo "Not in Replit"; exit 1; }
```

These **break the build outside Replit**. Flag for removal or refactor in `replit-localize`.

### `script/build.ts` (or build entry)

Specific to Replit-Agent fullstack projects: a single TS file that runs both Vite (client) and esbuild (server) builds. Look for:

- **`allowlist` arrays** of packages to bundle vs externalize. Often includes packages that aren't even installed (Replit Agent left them as defaults). Cross-check against `package.json` — packages in the allowlist but not in deps are "ghosts" to clean up in `replit-localize-dep-cleanup`.
- **Hardcoded build paths** referencing `attached_assets/` or other Replit directories.

## What to write to the audit document

```markdown
## 4. Lifecycle scripts

### Boot sequence
1. <step>
2. <step>
…

### Findings

| Script | Finding | Severity | Fix-it skill |
| --- | --- | --- | --- |
| `scripts/start-prod.sh` | `set -o pipefail` — needs `bash` in alpine image | Low | `replit-to-gcp-dockerfile` |
| `scripts/db-migrate.js` | Synthetic-baseline hack with hardcoded INITIAL_MIGRATION_WHEN | Medium | `replit-localize-migration-consolidation` |
| `scripts/db-migrate.js` | `ssl: NODE_ENV === "production" ? {rejectUnauthorized: false} : false` — breaks Cloud SQL Auth Proxy | Medium | `replit-to-gcp-dockerfile` (DB_SSL_MODE refactor) |
| `scripts/post-deploy.js` | Loads `attached_assets/docs/seed.sql` at runtime | Medium | `replit-localize-asset-relocation` |
| `scripts/pre-build-gate.sh` | Greps `.replit` to confirm Replit environment | High (breaks build off-Replit) | Remove in `replit-localize` |
```

## Parent skill

Invoked from `replit-audit`. Findings feed `replit-localize` (Phase 2) and `replit-to-gcp` (Phase 3).
