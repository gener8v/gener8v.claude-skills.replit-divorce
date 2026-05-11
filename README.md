# Replit-Divorce Skills for Claude Code

**A skill pack for migrating Replit-origin projects to portable hosting** — GCP Cloud Run, Railway, or any container platform. Battle-tested patterns distilled from real Replit→GCP migrations of production SaaS applications.

Built by [Gener8tive](https://www.gener8tive.com).

---

## What this is

Replit is a phenomenal place to *build* a product. It is a less phenomenal place to *operate* one. Most production-bound Replit projects eventually need to leave the platform — and that migration has a consistent shape: the same audit findings, the same portability traps, the same operational decisions. This repository captures that shape as a set of [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills) so the next migration takes hours instead of weeks.

The skills here cover three phases:

- **`replit-audit`** — discover what makes a Replit project tick. Env-var contract, dependency surface, deployment-coupling, security debt.
- **`replit-localize`** — make the project runnable on a developer's machine without Replit's implicit infrastructure. Compose-based Postgres, dotenv strategy, dependency cleanup, migration consolidation, dev-credential sanitization.
- **`replit-to-gcp`** — deploy the localized project to Google Cloud Run with Cloud SQL, Secret Manager, Workload Identity Federation, and a GitHub Actions pipeline.

A meta-skill, **`orchestrator`**, triages a target project's current state and sequences the right phase skills.

## What's in the box

The pack has **24 skills** organized hierarchically: one orchestrator, three phase coordinators, and twenty sub-skills.

### Orchestrator

| Skill | When Claude invokes it |
| --- | --- |
| [`replit-divorce`](skills/replit-divorce/SKILL.md) | "Help me move this Replit project to GCP." Triages target state, sequences the right phase skills. |

### Phase coordinators

| Skill | What the phase produces |
| --- | --- |
| [`replit-audit`](skills/replit-audit/SKILL.md) | A written inventory of Replit-coupling, security debt, and portability concerns. Read-only. |
| [`replit-localize`](skills/replit-localize/SKILL.md) | A working `npm install && npm run setup && npm run dev` on a fresh clone. Modifies the codebase across multiple commits. |
| [`replit-to-gcp`](skills/replit-to-gcp/SKILL.md) | GCP Cloud Run + Cloud SQL + Secret Manager + WIF + GHA pipeline. Writes code; operator gates each `terraform apply`. |

### Sub-skills (independently invokable)

Each phase coordinator delegates to multiple sub-skills. When Claude already knows what sub-problem it's solving, it can invoke a sub-skill directly without going through the phase coordinator.

**Phase 1 — Audit (5 sub-skills):**
[`replit-audit-config`](skills/replit-audit-config/SKILL.md) ·
[`replit-audit-deps`](skills/replit-audit-deps/SKILL.md) ·
[`replit-audit-env-vars`](skills/replit-audit-env-vars/SKILL.md) ·
[`replit-audit-lifecycle-scripts`](skills/replit-audit-lifecycle-scripts/SKILL.md) ·
[`replit-audit-security`](skills/replit-audit-security/SKILL.md)

**Phase 2 — Localize (6 sub-skills):**
[`replit-localize-compose-postgres`](skills/replit-localize-compose-postgres/SKILL.md) ·
[`replit-localize-dotenv`](skills/replit-localize-dotenv/SKILL.md) ·
[`replit-localize-migration-consolidation`](skills/replit-localize-migration-consolidation/SKILL.md) ·
[`replit-localize-boot-verification`](skills/replit-localize-boot-verification/SKILL.md) ·
[`replit-localize-security-cleanup`](skills/replit-localize-security-cleanup/SKILL.md) ·
[`replit-localize-dep-cleanup`](skills/replit-localize-dep-cleanup/SKILL.md)

**Phase 3 — Deploy to GCP (9 sub-skills):**
[`replit-to-gcp-platform-eval`](skills/replit-to-gcp-platform-eval/SKILL.md) ·
[`replit-to-gcp-dockerfile`](skills/replit-to-gcp-dockerfile/SKILL.md) ·
[`replit-to-gcp-terraform-foundation`](skills/replit-to-gcp-terraform-foundation/SKILL.md) ·
[`replit-to-gcp-terraform-cloud-sql`](skills/replit-to-gcp-terraform-cloud-sql/SKILL.md) ·
[`replit-to-gcp-terraform-secrets`](skills/replit-to-gcp-terraform-secrets/SKILL.md) ·
[`replit-to-gcp-terraform-cloud-run`](skills/replit-to-gcp-terraform-cloud-run/SKILL.md) ·
[`replit-to-gcp-terraform-wif`](skills/replit-to-gcp-terraform-wif/SKILL.md) ·
[`replit-to-gcp-gha-deploy`](skills/replit-to-gcp-gha-deploy/SKILL.md) ·
[`replit-to-gcp-runbook`](skills/replit-to-gcp-runbook/SKILL.md)

## Quick start

Install the skill pack into Claude Code's user skills directory:

```bash
# Option 1: clone the whole pack
mkdir -p ~/.claude/skills
git clone https://github.com/gener8v/gener8v.claude-skills.replit-divorce \
  ~/.claude/skills/replit-divorce

# Option 2: cherry-pick individual skills
cp -R skills/replit-audit ~/.claude/skills/
cp -R skills/replit-localize ~/.claude/skills/
cp -R skills/replit-to-gcp ~/.claude/skills/
cp -R skills/replit-divorce ~/.claude/skills/        # the orchestrator
```

Then, in any Claude Code session targeting a Replit-origin project, ask for help:

> *"Move this Replit project to GCP."*
>
> *"Audit this codebase for Replit-specific coupling before we migrate."*
>
> *"Run `replit-localize` against this repo."*

Claude will find the right skill, triage the project's current state, and proceed.

## The methodology

The skills implement a four-step framework, in order:

1. **Audit** — what's Replit-specific in this codebase, and what's the blast radius if we leave?
2. **Localize** — can a new developer clone the repo and get the app running in ≤3 commands on their laptop?
3. **Deploy** — can we provision durable infrastructure on a real cloud, with CI/CD that anyone can hand off?
4. **Decouple ownership** — is anything in the new setup still implicitly tied to one person's account, credentials, or platform login?

Each step has objective ready-state criteria. The skills implement those criteria and surface gaps explicitly when they're encountered.

The full methodology is documented in [`METHODOLOGY.md`](METHODOLOGY.md). The first-application case study (Fall Line Specialty's "Agency Sidekick" — a multi-tenant insurance SaaS migrated end-to-end in 4 days of focused work) is in [`CASE_STUDY.md`](CASE_STUDY.md).

## Who this is for

- **Founders or technical leads** moving a Replit-built MVP to production-grade hosting.
- **Consultancies** taking on Replit→cloud migrations and looking for a reusable framework instead of starting from scratch.
- **Anyone who's bumped into "but we built it on Replit"** as a deployment blocker.

## Contributing

The skills are versioned by use. Each successful application of a skill that surfaces a new pattern, gotcha, or recipe is captured as an edit. Contributions welcome — see [`CONTRIBUTING.md`](CONTRIBUTING.md) for the pattern-capture conventions (TBD; this is a young repo).

## About Gener8tive

[Gener8tive](https://www.gener8tive.com) is a small consultancy focused on helping engineering leaders make their software simpler, faster, and more durable. We work with founders moving from "it works on my Replit" to "it serves real customers reliably," and with established teams paying down technical debt that's slowing them down.

We use Claude Code heavily in our own work and publish what we learn.

## License

[MIT](LICENSE). Use these skills freely; tell us if you find something missing.
