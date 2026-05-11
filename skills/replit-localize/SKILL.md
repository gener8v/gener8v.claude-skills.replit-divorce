---
name: replit-localize
description: Phase 2 of the Replit-divorce workflow. Coordinates the six sub-skills that make a Replit-origin project runnable on a developer's machine without Replit's implicit infrastructure. Use when the user asks "make this work locally," "decouple from Replit," "set up local dev." Modifies the codebase across multiple commits. Ready-state criterion is objective: `npm install && npm run setup && npm run dev` works on a fresh clone.
---

# Replit Localization — Phase 2 Coordinator

You are running Phase 2 of the Replit-divorce workflow. The ready-state criterion is objective:

> A fresh `git clone` on a clean machine, followed by `npm install && npm run setup && npm run dev`, produces a working application connected to a working database with seeded data.

If that isn't true at the end, the phase isn't done.

This phase **modifies the codebase across multiple commits.** Each sub-skill produces its own cohesive commit. Push as you go — don't batch unrelated work.

## Prerequisites

- `replit-audit` (Phase 1) has produced an inventory of Replit-coupling. The sub-skills here reference its findings (especially `replit-audit-env-vars` for the env-var contract and `replit-audit-security` for VLN-N items).
- Developer machine has Docker, Node 20+, and git.

## Sub-skill sequence

Order matters — later steps reference outputs of earlier steps. Run in this sequence:

| # | Sub-skill | What it produces |
| --- | --- | --- |
| 1 | [`replit-localize-compose-postgres`](../replit-localize-compose-postgres/SKILL.md) | `compose.yaml` + `db:up`/`db:down` npm scripts. Replaces Replit's auto-Postgres. |
| 2 | [`replit-localize-dotenv`](../replit-localize-dotenv/SKILL.md) | `--env-file=.env.local` pattern in npm scripts; `.env.example`; inline dotenv loader in `drizzle.config.ts`. No `dotenv` package dep. |
| 3 | [`replit-localize-migration-consolidation`](../replit-localize-migration-consolidation/SKILL.md) | Archives old migrations, regenerates a clean baseline from `shared/schema.ts`, hardens the migration runner. Tests three DB states + one transition scenario. |
| 4 | [`replit-localize-boot-verification`](../replit-localize-boot-verification/SKILL.md) | Fixes `reusePort: true` (Linux-only), port 5000 (macOS AirPlay collision), `@replit` Twitter meta tag. Small mechanical edits. |
| 5 | [`replit-localize-security-cleanup`](../replit-localize-security-cleanup/SKILL.md) | Resolves VLN-001 (`.replit` plaintext secrets) and VLN-002 (bundled client credentials). |
| 6 | [`replit-localize-dep-cleanup`](../replit-localize-dep-cleanup/SKILL.md) | Removes truly-unused deps; moves client-only React/UI deps to `devDependencies`; moves all `@types/*` to `devDependencies`. |

After Step 1, `npm run db:up` works. After Step 2, `npm run dev` can read env vars. After Step 3, `npm run db:migrate` works against a fresh local DB. After Step 4, `npm run dev` actually serves traffic on macOS. After Step 5, no credentials leak. After Step 6, the production install is ~90 MB instead of ~700 MB.

## Cross-skill consistency

Each sub-skill produces its own commit, but they reference each other:

- Sub-skill 2's `.env.example` is the contract — every variable from `replit-audit-env-vars` should appear with a placeholder value.
- Sub-skill 3's migration runner is invoked by sub-skill 1's `db:up && db:migrate` chain.
- Sub-skill 5's VLN-002 fix removes the bundled credentials file that sub-skill 6's dep cleanup would otherwise leave as a confusing orphan.
- Sub-skill 6 must run **last** — moving deps invalidates `node_modules`, and the verification in step 4 (boot test) needs a stable dep tree.

## Commit cadence

After each sub-skill completes, commit and push. Don't batch. Sample commit-message style:

```
feat(local-dev): compose-based Postgres for local development
feat(local-dev): dotenv strategy via --env-file flag
feat(migrations): consolidate to single baseline; align DB with schema.ts
feat(migrations): runner handles fresh / push-baselined / transition states
fix(local-dev): remove Replit-only socket option and stale meta tag
fix(security): drive dev login picker from API, not bundle (VLN-002)
chore(deps): drop unused, move client-only deps to devDependencies
```

The migration consolidation sub-skill often produces two commits (baseline regen + runner hardening); that's expected.

## Working-tree path renames

If the local repo directory is named after a Replit clone-source (often a deploy-test name) and `origin` later points to the real repo, rename the local directory to match `origin`. Costs a `mv` + memory update; saves the "which repo am I committing to?" tax forever.

When moving files within the repo — `replit.md → docs/PROJECT_CONTEXT.md`, `migrations/ → migrations-archive/`, etc. — use the regular `Write`/`Edit` tools to put files at the new path. Git's rename detection (≥50% similarity threshold by default) re-discovers the rename automatically when you commit. No `git mv` needed; sometimes this is even cleaner because content edits can happen in the same commit as the rename.

## Final verification — the ready-state criterion

After all six sub-skills are committed:

```bash
git clone <repo-url> /tmp/fresh-clone   # genuinely fresh; no stale node_modules
cd /tmp/fresh-clone
cp .env.example .env.local              # operator fills in any real values
npm install
npm run setup                            # chains db:up && db:migrate && db:seed
npm run dev
# ↪ open http://localhost:<PORT> and confirm:
#    GET / returns SPA shell
#    Login flow works (with VITE_DEMO_MODE=true the dev picker is visible)
#    /api/auth/me returns expected error JSON
```

If any step fails, fix before declaring the phase done. The criterion is "fresh clone works in three commands, on a clean machine" — partial success doesn't count.

## Output to surface when complete

Return to the user (or the orchestrator) a summary:

1. The seven-ish commits (sub-skill 3 usually produces two), in order.
2. The audit's VLN items: which are now resolved (link to the resolution commit), which are deferred (with reason — usually encryption-key rotation needing a re-encryption migration).
3. The `npm run setup` chain — explicit list of what each step does.
4. Any remaining Replit-coupling that was deferred (e.g., R2 storage staying as Replit's R2-via-AWS-SDK — portable but Replit-flavored).

Then surface whether the user wants to proceed to `replit-to-gcp` (Phase 3) immediately or test the localized setup further before deploying.

## Parent skill

Invoked from `replit-divorce` (the top-level orchestrator). On completion, return control to the orchestrator for the decision on whether to proceed to `replit-to-gcp`.
