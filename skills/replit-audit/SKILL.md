---
name: replit-audit
description: Discover what's Replit-specific in a codebase — Replit-managed services, hidden coupling, security debt baked in by Replit Agent, and orphaned infrastructure paths. Use when the user asks to "audit this Replit project," "what's tying us to Replit," "find Replit-specific dependencies," or before any Replit→cloud migration. Produces a portability inventory the user can act on. Read-only — does not modify files.
---

# Replit Project Audit

You are running a discovery pass on a Replit-origin codebase to surface everything that ties it to the Replit platform. Output is a written inventory the user can act on. **Do not modify files in this skill** — that's `replit-localize`'s job.

## What this skill produces

A document (recommend `docs/REPLIT_AUDIT.md` or `docs/INFRASTRUCTURE.md`) covering:

1. **Project profile** — stack, deployment pattern, and the Replit fingerprint (see below).
2. **Env-var contract** — every variable the runtime expects, where it comes from on Replit, and whether it needs a portable replacement.
3. **Replit-coupling inventory** — every file, dependency, hook, and code path that references Replit specifically. Each item gets a severity rating and recommended replacement.
4. **Security debt** — credentials in code, plaintext secrets in `.replit`, dev-credential leakage into client bundles.
5. **Operational caveats** — undocumented behavior that's not visible in static analysis (migration runner quirks, seed-file destructiveness, etc.).

## The fingerprint — what you're auditing

Replit-Agent-built fullstack projects share a distinctive shape. The canonical fingerprint:

- `[agent].stack = "MOCKUP_JS"` and `mockupState = "FULLSTACK"` in `.replit`
- Node + Vite + Postgres + an `autoscale` deployment target
- Three `@replit/vite-plugin-*` dev-deps: `cartographer`, `dev-banner`, `runtime-error-modal`
- A `replit.md` at the root with Replit-Agent-flavored project description

If three or more of these are present, you're looking at a Replit-Agent-fullstack project and the patterns in this skill apply almost verbatim.

## Audit recipe — six passes

### Pass 1: Configuration files

Read each of these in full:

- **`.replit`** — the primary platform configuration. Note:
  - `[deployment]` block: identifies the canonical build/run path. Anything outside this is orphaned.
  - `[userenv.shared]` and `[userenv.development]` blocks: **inspect for plaintext secrets.** Replit's UI encourages users to write them here. Universal source of secret leaks. Flag every secret-like value as a `VLN` item.
  - `[postMerge]` and other hook blocks: Replit IDE features. Often contradict documented "never push" rules in `replit.md`.
  - `[[ports]]` mappings: usually `localPort = 5000 → externalPort = 80`, which collides with macOS AirPlay on local dev.
- **`replit.md`** — historical project description. Treat as partially-stale background, not authoritative. Cross-check against actual code state.
- **`replit.nix`** if present — system packages. Most can be inferred from Dockerfile needs.

### Pass 2: Replit-specific dependencies

Run the four-fold env-var-and-package grep:

```bash
# Replit Vite plugins (all three usually co-occur)
grep -rE "@replit/vite-plugin-(cartographer|dev-banner|runtime-error-modal)" .

# Project-local plugins that read Replit env vars
grep -rE "REPLIT_(INTERNAL_APP_DOMAIN|DEV_DOMAIN|DB_URL|HOME|SLUG|OWNER)" .

# Any other @replit/* package
grep -rE "@replit/" package.json package-lock.json

# Replit Agent residue
grep -rE "@anthropic-ai/claude-code" package.json
```

Each of these three Vite plugins has a different mitigation pattern:

- `cartographer` and `dev-banner` — usually `REPL_ID`-gated; safe to leave (they no-op off-Replit). Document the gate.
- `runtime-error-modal` — often **NOT** gated. Audit the import site and recommend gating.

`@anthropic-ai/claude-code` in runtime `dependencies` is a strong Replit-Agent provenance signal. Should not be in a production container. Verify it's actually unused (grep imports) before recommending removal.

### Pass 3: Environment variable contract

Deterministic four-grep recipe:

```bash
grep -rhE "process\.env\.[A-Z_]+"          server/ scripts/ shared/ 2>/dev/null | sort -u
grep -rhE "import\.meta\.env\.[A-Z_]+"     client/ shared/ 2>/dev/null         | sort -u
grep -rE  "VITE_"                          .                                    | head -20
grep -riE "replit"                         .                                    | head -50
```

Compile the union into a single env-var contract document. For each variable note:

- Source on Replit (auto-provided, user-set, hardcoded fallback)
- Source on target platform (Secret Manager? plain env? OIDC?)
- Code paths that read it
- Whether the fallback (`process.env.X || "default"`) hides a missing-secret bug

**Catch:** look for non-DB uses of `DATABASE_URL` — Replit's "always present" assumption gets reused as a quick HMAC seed or dev-fallback key. Those break the moment `DATABASE_URL` rotates.

### Pass 4: Lifecycle scripts and hooks

Read every file in `scripts/` and `script/`. Look specifically for:

- **`scripts/start-prod.sh`** or `start-prod.*` — the production entrypoint. Document the boot sequence (typically: migrate → server → optional post-deploy). The script's comments often misattribute it to Replit's hook system when it's actually invoked from elsewhere.
- **`scripts/db-migrate.js`** or equivalent — the migration runner. Replit projects often have a synthetic-baseline pattern (`scripts/db-migrate.js` inserts a fake row in `__drizzle_migrations` to make `migrate()` skip the initial migration). This is a Replit-ism — Replit users almost always start with `drizzle-kit push` before introducing migration files, so the first "migration" already exists in the DB and would fail with "relation already exists" on a normal migrate.
- **`scripts/post-deploy.js`** or equivalent — post-boot hooks. Often gated by an env var. Verify the env var actually exists in production env.
- **Any `*-gate*.sh` script** — these often grep against `.replit` to make sure they're running in Replit. They break the build outside Replit.

### Pass 5: Orphaned build paths

`.replit`'s `[deployment]` block names ONE build/run path. Cross-check against any other build scripts in the repo (`script/build-prod.sh`, alternate Dockerfiles, etc.). Anything not referenced by `.replit` is orphaned and probably stale. Flag for removal.

### Pass 6: Security debt

Two specific audits, each producing one or more `VLN-N` items if findings surface:

**a) Plaintext secrets in `.replit` (VLN-001 pattern):**

Look in every `[env...]`, `[userenv...]`, `[secrets]` block. Cross-reference each suspected secret against active code paths. For any `*_ENCRYPTION_KEY`:

- Grep for the encrypt/decrypt helper callers AND check the column's NOT NULL/NULL status.
- Active routes + non-nullable column = rotation requires a re-encryption migration, NOT a simple env-var swap. Document this constraint explicitly.

**b) Bundled credentials in client code (VLN-002 pattern):**

```bash
grep -rE "(password|SeedPass|secret|api[_-]?key)[\"\\s]*[:=]" client/ | grep -iv "^\\s*//"
```

Replit's "demo mode" picker pattern routinely bakes seed credentials into the client bundle as a convenience. The fix template is API-driven dev-user listing (NODE_ENV-gated `GET /api/dev/seed-users` that returns only email + display info, no credentials). Document this fix even if not implementing in this skill.

## Investigate-before-changing — the meta-discipline

When something *looks* like cruft or unused dependency, **grep for actual usage before mutating it.** Two cases from the canonical migration where the discipline mattered:

- `@anthropic-ai/claude-code` *looked* unused → grep confirmed zero imports → safe to recommend removal.
- `AMS_SYNC_ENCRYPTION_KEY` *looked* like a placeholder string → grep showed 4 active callers in `server/routes.ts` plus a `text NOT NULL` schema column → rotation requires a re-encryption migration, not an env-var swap.

Apply this everywhere. The audit's job is to *recommend* changes, not to make them; recommendations must be grounded in actual code state, not appearance.

## Output format

Write findings to `docs/REPLIT_AUDIT.md` (or `docs/INFRASTRUCTURE.md` if the user prefers — the canonical case study uses the latter name). Structure:

```markdown
# Replit Audit — <Project Name>

**Date:** <YYYY-MM-DD>
**Fingerprint:** <which Replit-Agent template signals are present>

## 1. Stack and deployment profile
…

## 2. Env-var contract
| Variable | Source on Replit | Suggested portable source | Active callers |
| --- | --- | --- | --- |
…

## 3. Replit-coupling inventory
| Item | Type | Severity | Recommendation |
| --- | --- | --- | --- |
…

## 4. Security debt (VLN-N items)
…

## 5. Operational caveats
…

## 6. Recommendations summary
…
```

Severity scale for the coupling inventory:

| Severity | Meaning |
| --- | --- |
| **🔴 Hard-blocking** | Breaks the build / runtime off Replit (e.g., `*-gate.sh` scripts that grep `.replit`). |
| **🟡 Behavioral** | Silently produces wrong output off Replit (e.g., `runtime-error-modal` plugin that's ungated; `DATABASE_URL`-fallback HMAC seed). |
| **🟢 Cosmetic** | Doesn't affect function but signals Replit origin (e.g., `@replit` Twitter meta tag, `replit.md` reference comments). |

## What this skill explicitly doesn't do

- It doesn't modify any files. The output is a written inventory; the user (or `replit-localize`) does the modifications.
- It doesn't decide *whether* to migrate. That's a strategic decision; the audit informs it.
- It doesn't audit business logic, data model, or product correctness. Strictly Replit-coupling.

## When you're done

Return a short summary to the user covering:

1. The fingerprint (one line — "this is a Replit-Agent fullstack project with the canonical shape").
2. Count of items by severity.
3. Top 3–5 highest-severity findings.
4. Pointer to the full audit document.

Then surface whether the user wants to proceed to `replit-localize` (Phase 2) immediately or address specific findings first.
