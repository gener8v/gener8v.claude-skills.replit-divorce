---
name: replit-localize-migration-consolidation
description: Reconcile schema-vs-migrations drift in a Replit/drizzle project that started with `drizzle-kit push` and later switched to file-based migrations. Consolidates the migration history into a single fresh baseline, then hardens the migration runner to handle all three real-world DB states (fresh / push-baselined / migration-history transition). Invoke as part of `replit-localize` Steps 3+4 or directly when `npm run db:migrate` fails on a fresh DB with `relation already exists` errors. Modifies migrations/, the migration runner script, and verifies against three test scenarios.
---

# Migration Debt Repayment + Runner Hardening

Replit/drizzle projects that started life with `drizzle-kit push` almost universally have **schema drift** between `migrations/*.sql` and `shared/schema.ts` — the original `push` snapshot was never captured as a migration file, and subsequent file-based migrations only cover changes after the switch. Symptom: a fresh DB can't be bootstrapped from `npm run db:migrate` alone; production and live-dev DBs are happy because they predate the file-based discipline.

This skill repays that debt.

## Step 1 — Confirm drift exists

Run the diagnostic:

```bash
# Tables declared in schema.ts
grep -oE 'pgTable\("([a-z_]+)"' shared/schema.ts \
  | sed -E 's/pgTable\("([a-z_]+)"/\1/' | sort -u > /tmp/schema.txt

# Tables created across all existing migrations
grep -hE '^CREATE TABLE' migrations/*.sql \
  | sed -E 's/^CREATE TABLE "?([a-z_]+)"?.*/\1/' | sort -u > /tmp/migrations.txt

# Tables in schema but NOT in migrations
comm -23 /tmp/schema.txt /tmp/migrations.txt
```

Non-empty output = drift exists. If the output is empty, the migrations already fully cover the schema — skip this skill.

Also check for duplicates (retroactive baseline migrations that hand-coded missing tables go stale silently):

```bash
grep -hE '^CREATE TABLE' migrations/*.sql | sort | uniq -c | awk '$1>1'
```

Duplicates mean the migration history is contradictory; the consolidation pass is required regardless of drift.

## Step 2 — Repayment recipe

Four steps:

### 2a. Archive existing migrations

```bash
git mv migrations migrations-archive
```

Or use the regular `Edit` tool to move files (git's rename detection ≥50% similarity threshold re-discovers the rename when you commit; works without `git mv`).

The archive directory keeps the history visible in the repo for reference. It's just not on the active drizzle path.

### 2b. Generate a fresh baseline

```bash
mkdir migrations
npx drizzle-kit generate --name=baseline
```

This produces:

- `migrations/0000_baseline.sql` — a single SQL file with all CREATE TABLE statements derived from current `shared/schema.ts`.
- `migrations/meta/_journal.json` — fresh journal naming `0000_baseline` as `idx=0`.
- `migrations/meta/0000_snapshot.json` — schema snapshot.

Inspect `0000_baseline.sql`. Confirm it has every table the schema defines. Spot-check ~5 tables to confirm column definitions match what you expect.

### 2c. Refactor the migration runner

The migration runner (`scripts/db-migrate.js` or equivalent) needs to read the baseline tag and timestamp **dynamically** from `migrations/meta/_journal.json` — do not hardcode them. Pattern:

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

Then the runner must handle three distinct DB states (see Step 3 for testing).

### 2d. Apply locally, verify, then verify the prod scenario

Apply to a fresh local DB:

```bash
docker compose down -v   # destroys local DB volume
docker compose up -d --wait postgres
npm run db:migrate       # should succeed cleanly
```

Then run the three-scenario test from Step 3.

## Step 3 — Three-scenario testing

The runner must handle three DB states. Each MUST be tested before commit:

### Scenario 1: Fresh empty DB

```bash
docker compose down -v && docker compose up -d --wait postgres
npm run db:migrate
# Expected: drizzle's migrate() applies the baseline; tables exist.
psql … -c "SELECT count(*) FROM drizzle.__drizzle_migrations;"
# Expected: 1 row (the baseline).
```

### Scenario 2: Idempotent re-run

```bash
# After Scenario 1 succeeded:
npm run db:migrate   # second run
# Expected: no-op, no duplicate inserts.
psql … -c "SELECT count(*) FROM drizzle.__drizzle_migrations;"
# Expected: still 1 row.
```

### Scenario 3: Push-baselined DB (legacy state)

This is the case where the DB exists from `drizzle-kit push` but has no `drizzle.__drizzle_migrations` entries. Simulate:

```bash
# After Scenario 1: drop the drizzle schema to simulate push-baselined state
psql … -c "DROP SCHEMA drizzle CASCADE;"
npm run db:migrate
# Expected: runner detects "tables exist but no migration history" → registers
# baseline as already-applied without re-executing.
psql … -c "SELECT count(*) FROM drizzle.__drizzle_migrations;"
# Expected: 1 row.
```

### Scenario 4 (one-time only, simulate the prod transition)

Production has `__drizzle_migrations` entries from the OLD migrations (0000…0030 or whatever). The new baseline's hash is none of those. The runner must detect orphan hashes and swap them for the new baseline row:

```bash
# Insert fake stale rows to simulate prod state pre-transition
psql … <<SQL
INSERT INTO drizzle.__drizzle_migrations (hash, created_at) VALUES
  ('stale_hash_1', 1700000000000),
  ('stale_hash_2', 1700100000000),
  ('stale_hash_3', 1700200000000);
SQL

npm run db:migrate
# Expected: runner detects orphan hashes (none match the new baseline) →
# deletes them, inserts a single new baseline row.
psql … -c "SELECT * FROM drizzle.__drizzle_migrations;"
# Expected: 1 row only — the new baseline.

# Idempotency check
npm run db:migrate   # run again
# Expected: still 1 row; no churn.
```

**Skipping any scenario means deploying without proof the runner is safe in the prod scenario you've never seen locally.** The transition scenario is the highest-stakes — it runs once per production DB and only once. If the runner is wrong, the prod DB ends up in an inconsistent state.

## Step 4 — Commit

After all four scenarios pass:

```
feat(migrations): consolidate to single baseline; align DB with schema.ts

- Archive original migrations to migrations-archive/ (preserved for history,
  not on the active drizzle path)
- Regenerate migrations/0000_baseline.sql from shared/schema.ts via
  drizzle-kit generate --name=baseline
- Refactor migration runner to read baseline tag and timestamp dynamically
  from migrations/meta/_journal.json (no more hardcoded INITIAL_MIGRATION_WHEN
  or computeInitialHash())
- Runner handles three DB states: fresh / push-baselined / migration-history
  transition (orphan-hash swap, idempotent)
- All four scenarios verified before commit
```

## What this skill doesn't do

- Doesn't migrate production data (no data movement; only schema definitions).
- Doesn't handle post-baseline migrations — those happen via the normal `drizzle-kit generate` workflow once this consolidation is done.
- Doesn't decide the migration tool (assumes drizzle; the same pattern applies to Prisma's `prisma migrate` with adjusted commands).

## Parent skill

Invoked from `replit-localize` Steps 3+4. The hardened runner is also a prerequisite for `replit-to-gcp-terraform-cloud-sql` — the production deploy's first apply runs this same migration code against a freshly-provisioned Cloud SQL instance.
