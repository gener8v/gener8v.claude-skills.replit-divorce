---
name: replit-divorce
description: Orchestrate a full migration of a Replit-origin project to portable hosting (GCP Cloud Run, Railway, or any container platform). Use when the user asks to "move this off Replit," "migrate to GCP / Railway," "decouple from Replit," "make this deployable somewhere else," or any variant. Triages current state, sequences `replit-audit` → `replit-localize` → `replit-to-gcp` as needed, surfaces operator decisions explicitly.
---

# Replit-Divorce Orchestrator

You are running a Replit-origin project's migration to portable hosting. This skill triages the project's current state, sequences the appropriate phase skills, and tracks what's done vs. what's open.

## Phase model

The work has three phases, in canonical order:

1. **`replit-audit`** — discover the Replit-specific surface area. Outputs an inventory of portability concerns, security debt, and deployment-coupling.
2. **`replit-localize`** — make the project runnable without Replit's implicit infrastructure. Ready-state criterion: `npm install && npm run setup && npm run dev` (or equivalent) works on a fresh clone.
3. **`replit-to-gcp`** — provision durable cloud infrastructure (Cloud Run + Cloud SQL + Secret Manager + WIF + GHA pipeline). Ready-state criterion: push to main triggers an automated deploy that serves traffic.

A project being migrated may be at any point on this path. **Start by triaging — don't assume a fresh-start.**

## Step 1 — Triage current state

Before invoking any phase skill, inspect the repository to determine which phases have started. Use the following signals:

| Signal | What it tells you |
| --- | --- |
| `.replit` file present | Still actively wired for Replit. Suggests audit is incomplete or hasn't started. |
| `replit.md` at repo root | Replit Agent–generated project context. Either still authoritative (audit not done) or stale (audit complete; should have been replaced by `docs/PROJECT_CONTEXT.md` or similar). |
| `attached_assets/` directory | Replit Agent's working directory for build specs and design docs. Usually safe to leave alone for historical record but should be excluded from runtime images. Runtime files referenced under this path = relocation needed (a `replit-localize` task). |
| `compose.yaml` or `docker-compose.yaml` defining a `postgres` service | Local Postgres set up — `replit-localize` has started. |
| `npm run setup` script in `package.json` that chains `db:up && db:migrate && db:seed` (or similar) | The three-command setup pattern from `replit-localize` is in place. |
| `migrations/` with a single recent baseline + `migrations-archive/` | Migration consolidation from `replit-localize` is complete. |
| `Dockerfile` at repo root | Production containerization started — early `replit-to-gcp` work. |
| `infra/` with Terraform files | GCP provisioning in progress. Inspect `infra/README.md` for the slice roadmap and which are shipped. |
| `.github/workflows/deploy.yml` referencing Workload Identity Federation | CI/CD pipeline complete. |
| `DELIVERED.md` at repo root | A previous engagement has run; read it before doing anything else — most of the work may already be done. |
| `OPEN_QUESTIONS.md` at repo root | Outstanding decisions that may block phases — read before invoking any phase skill. |

Run a quick file-and-content sweep (use `Read` on the suspected files; use `Bash` for `ls` checks). Produce a one-paragraph status summary before proceeding.

## Step 2 — Decide phase sequence based on triage

Apply this decision tree:

- **All Replit signals present, no localization signals** → standard sequence: `replit-audit` → `replit-localize` → `replit-to-gcp`.
- **Localization signals present, audit unclear** → light-pass `replit-audit` (look only for things the localize work didn't address: security debt, undocumented env vars, dead deps), then `replit-to-gcp`.
- **Localization complete, GCP work partial** → triage the `infra/` directory specifically (which slices are present, which are applied?), then resume `replit-to-gcp` at the next unshipped slice.
- **All three phases substantially complete** → switch into review/handoff mode: read `DELIVERED.md` and `OPEN_QUESTIONS.md`, surface remaining gaps, recommend prioritized next actions.

## Step 3 — Confirm intent with the user before destructive work

Before invoking `replit-localize` or `replit-to-gcp`, confirm with the user:

- **Target platform** — Cloud Run is the documented path in these skills, but the localized output also works on Railway, Fly.io, AWS App Runner, etc. If the user hasn't said, ask.
- **Whether to apply infrastructure** — `replit-to-gcp` writes Terraform but does NOT auto-apply. The user must explicitly authorize each `terraform apply`. Likewise for any `gcloud secrets versions add` with real credentials.
- **Data migration** — if there's production data on Replit's Neon Postgres that needs to come over, that's its own workstream (often deferred). Ask whether the new environment should be fresh or carry data.

## Step 4 — Invoke phase skills in order

When invoking a phase skill, pass the triage summary into the skill's context so it doesn't re-discover the same state. Each phase skill has its own ready-state criteria and surfaces its own gaps.

After each phase completes, **return control to this orchestrator** for a recheck before invoking the next phase. The phase skills are not pipelined — they are sequential with operator-decision gates.

## Step 5 — Capture deliverables and open questions explicitly

At any point in the engagement, two documents at the project root should reflect current state:

- **`DELIVERED.md`** — maps the migration's task plan to shipped artifacts (commits, files, infrastructure). Created by the phase skills as they complete work.
- **`OPEN_QUESTIONS.md`** — captures decisions the user owes (credentials to provide, ownership questions, product-side decisions blocking refactors). Each question gets an ID, severity, context, and recommendation.

Both files are client-facing handoff documents. Keep them current as the engagement proceeds.

## Discipline

- **Audit before changing.** When you see Replit-specific code, grep for usage before mutating. The `Investigate-before-changing` discipline from `replit-audit` applies everywhere.
- **Commit and push as you go.** Each cohesive piece of work lands as its own commit with a meaningful message. Don't batch.
- **Three-tier severity in `OPEN_QUESTIONS.md`** — 🔴 blocks production capability, 🟡 architectural/hygiene, 🔵 discussion/scope. Be honest about which is which.
- **Don't pretend completeness.** A phase isn't "done" until its ready-state criterion is objectively met. Partial completion is captured in `OPEN_QUESTIONS.md`, not glossed over in `DELIVERED.md`.

## What this skill does NOT do

- It does not implement Replit Agent–side fixes (refactoring an in-Replit project's code). The phase skills produce a portable artifact; the user runs the migration.
- It does not handle commercial decisions (billing account ownership, license selection, hosting-provider negotiations). It surfaces them as open questions and recommends, but the user decides.
- It does not migrate production data from Replit. That's Workstream E in the canonical WBS — separately scoped.

## Pointers

- Methodology overview: [`METHODOLOGY.md`](../../METHODOLOGY.md) in this skill pack.
- Reference case study: [`CASE_STUDY.md`](../../CASE_STUDY.md) — the migration this skill pack was distilled from.
- Phase skills: [`replit-audit/SKILL.md`](../replit-audit/SKILL.md) · [`replit-localize/SKILL.md`](../replit-localize/SKILL.md) · [`replit-to-gcp/SKILL.md`](../replit-to-gcp/SKILL.md).
