---
name: replit-to-gcp
description: Phase 3 of the Replit-divorce workflow. Coordinates the nine sub-skills that provision GCP infrastructure (Cloud Run + Cloud SQL + Secret Manager + WIF) for a Replit-localized project, plus a GitHub Actions deploy pipeline and an operator-facing runbook. Use when the user asks to "deploy to GCP," "set up Cloud Run," "provision the cloud infra." Writes code; does NOT auto-apply — operator decisions gate each `terraform apply`.
---

# Replit → GCP Deployment — Phase 3 Coordinator

You are running Phase 3 of the Replit-divorce workflow. The ready-state criterion is objective:

> A push to `main` triggers an automated GitHub Actions workflow that builds, pushes, and deploys the application to Cloud Run; the resulting URL serves the SPA and the auth gate works.

This phase **writes Terraform, Dockerfile, GHA workflow, and operator-facing documentation. It does NOT run `terraform apply` autonomously.** Every cost-incurring action requires explicit user authorization.

## Prerequisites

- `replit-localize` (Phase 2) is complete — `npm install && npm run setup && npm run dev` works on a fresh clone.
- The user has a GCP project (or will create one) with billing enabled.
- The user has `gcloud`, `terraform` (≥1.5 or OpenTofu), `gh`, and Docker installed.
- The user can provide R2 credentials (or equivalent object-storage), Anthropic API key, and any other external credentials identified in `replit-audit-env-vars`.

## Mental model — Replit → GCP service mapping

There is no architectural rework. Replit's deployment shape maps 1:1 onto GCP services:

| Replit | GCP |
| --- | --- |
| `autoscale` deployment | Cloud Run |
| Auto-provisioned Postgres | Cloud SQL Postgres |
| Replit Secrets | Secret Manager |
| Replit's auto-HTTPS | Cloud Run native HTTPS |
| `[postMerge]` hook | GitHub Actions workflow |
| Replit's auto-deploy | GHA push-to-main triggers `gcloud run deploy` |

Lift-and-shift. No Kubernetes, no custom load balancer, no VPC at v1.

## Sub-skill sequence

Nine sub-skills, in this order. Each is its own commit:

| # | Sub-skill | Output |
| --- | --- | --- |
| 1 | [`replit-to-gcp-platform-eval`](../replit-to-gcp-platform-eval/SKILL.md) | `docs/GCP_PLATFORM_EVALUATION.md` — written platform recommendation + three-tier cost projection |
| 2 | [`replit-to-gcp-dockerfile`](../replit-to-gcp-dockerfile/SKILL.md) | Multi-stage alpine Dockerfile + `.dockerignore` + `DB_SSL_MODE` env override + `/api/health` endpoint |
| 3 | [`replit-to-gcp-terraform-foundation`](../replit-to-gcp-terraform-foundation/SKILL.md) | Provider plumbing, variables, API enables, Artifact Registry repo (slice 1) |
| 4 | [`replit-to-gcp-terraform-cloud-sql`](../replit-to-gcp-terraform-cloud-sql/SKILL.md) | Cloud SQL instance + DB + user + `DATABASE_URL` secret (slice 2). GCS state backend becomes mandatory here. |
| 5 | [`replit-to-gcp-terraform-secrets`](../replit-to-gcp-terraform-secrets/SKILL.md) | App secrets — Terraform-owned cryptographic + operator-populated external (slice 3) |
| 6 | [`replit-to-gcp-terraform-cloud-run`](../replit-to-gcp-terraform-cloud-run/SKILL.md) | Cloud Run service + runtime SA + per-secret IAM + Auth Proxy mount + org-policy override (slice 4) |
| 7 | [`replit-to-gcp-terraform-wif`](../replit-to-gcp-terraform-wif/SKILL.md) | Workload Identity Federation pool + provider + deploy SA + IAM (slice 5) |
| 8 | [`replit-to-gcp-gha-deploy`](../replit-to-gcp-gha-deploy/SKILL.md) | `.github/workflows/deploy.yml` + `.github/workflows/README.md` |
| 9 | [`replit-to-gcp-runbook`](../replit-to-gcp-runbook/SKILL.md) | `docs/DEPLOYMENT_RUNBOOK.md` — end-to-end procedure + ops cookbook |

Each sub-skill is independently invokable — if the user already has Terraform and just needs the GHA workflow, invoke `replit-to-gcp-gha-deploy` directly. For a full Phase 3, run all nine.

## Author code; do not apply

This phase **writes code only.** No `terraform apply`, no `gcloud secrets versions add` with real values, no `gh variable set`. Those are operator actions documented in the runbook (sub-skill 9).

The reason: each one is an explicit cost commitment, a real-credential entry, or a "this changes who can deploy" action. Those decisions belong to the operator, not the skill.

## Cost commitment summary (when the operator applies)

At T1 (pre-launch / staging traffic), the platform total is approximately:

| Service | Cost/mo |
| --- | ---: |
| Cloud SQL (db-custom-1-3840, ZONAL) | ~$50 |
| Secret Manager (~11 secrets) | ~$0.66 |
| Artifact Registry | ~$0.10 |
| Cloud Run (min_instances=0) | ~$0 |
| **Total** | **~$51/mo** |

Cloud SQL is 70–85% of total spend at every tier. The runbook documents the cost in the Phase A bootstrap section so the operator goes in eyes-open.

## First-apply gotchas to expect

`terraform validate` doesn't exercise GCP API rules. The first real `terraform apply` will hit ≥1 of these:

| Error | Documented in |
| --- | --- |
| `At least one of Public IP or Private IP or PSC connectivity must be enabled` | `replit-to-gcp-terraform-cloud-sql` — set `ipv4_enabled = true` with empty `authorized_networks` |
| `The following reserved env names were provided: PORT` | `replit-to-gcp-terraform-cloud-run` — Cloud Run reserves PORT |
| `One or more users named in the policy do not belong to a permitted customer` | `replit-to-gcp-terraform-cloud-run` — project-scoped org-policy override |
| `Secret ... was not found` on Cloud Run creation | `replit-to-gcp-terraform-secrets` — operator-owned secrets need placeholder v1 |
| `/healthz` returns Google's 404 page | `replit-to-gcp-dockerfile` — use `/api/health` instead |

All five are normal Phase-3-of-the-first-migration friction. None are blockers.

## Open questions to surface

Throughout Phase 3, capture decisions the user needs to make in an `OPEN_QUESTIONS.md` at the project root. Common items:

- 🔴 R2 access key + secret (operator provides)
- 🔴 Anthropic API key (operator provides)
- 🔴 R2 endpoint URL + bucket name (operator provides — feeds tfvars)
- 🟡 Custom domain mapping
- 🟡 GCP billing account ownership (don't leave it on the developer's personal billing)
- 🟡 IAM admin / bus-factor backup
- 🟡 Incident contact + paging
- 🟡 GitHub branch protection / required reviews
- 🔵 Dev environment provisioning (separate from prod)
- 🔵 Open-signup vs invite-only

Three-tier severity:
- 🔴 blocks production capability
- 🟡 architectural / hygiene
- 🔵 discussion / scope

## DELIVERED.md and OPEN_QUESTIONS.md as handoff artifacts

By the end of Phase 3, two documents at the project root should exist:

- **`DELIVERED.md`** — maps the migration's task plan (often a WBS from the engagement) to shipped artifacts (commits, files, infrastructure). Keep current as each sub-skill completes.
- **`OPEN_QUESTIONS.md`** — captures the unresolved decisions and information needs. Each question gets an ID, severity, context, recommendation, and effort estimate.

These are the client-facing handoff documents. Treat them as first-class deliverables, not afterthoughts.

## Output to surface when complete

Return to the user (or the orchestrator) a summary:

1. The nine commits + their roles.
2. Live infrastructure resources (Cloud Run URL, Cloud SQL instance, AR repo, runtime + deploy SAs, WIF provider).
3. Cost commitment from this point (~$51/month idle baseline at T1).
4. Pointer to `DELIVERED.md` and `OPEN_QUESTIONS.md` for the full handoff.

Then ask whether the user wants to:

- Provide real R2 / Anthropic credentials now (close 🔴 open questions).
- Set up a custom domain.
- Provision a separate dev environment.
- Pause for ownership/access decisions before further changes.

## Parent skill

Invoked from `replit-divorce` (the top-level orchestrator). On completion, return control to the orchestrator for the decision on whether to start dev-environment provisioning, branch protection, secret rotation, or related follow-on work.
