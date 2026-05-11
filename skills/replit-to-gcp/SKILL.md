---
name: replit-to-gcp
description: Provision GCP Cloud Run + Cloud SQL + Secret Manager + Workload Identity Federation for a Replit-localized project, plus a GitHub Actions deploy pipeline. Use after `replit-localize` is complete or when the user says "deploy to GCP," "set up Cloud Run," "provision the cloud infra," or any variant. Writes Terraform under `infra/`, a production Dockerfile, a GHA workflow, and a deployment runbook. Does NOT auto-apply — operator decisions gate each `terraform apply`.
---

# Replit → GCP Deployment

You are provisioning durable cloud infrastructure for a Replit-localized project. The ready-state criterion:

> A push to `main` triggers an automated GitHub Actions workflow that builds, pushes, and deploys the application to Cloud Run; the resulting URL serves the SPA and the auth gate works.

This skill **writes Terraform code, Dockerfile, GHA workflow, and operator-facing documentation. It does NOT run `terraform apply` autonomously.** Every cost-incurring action requires explicit user authorization.

## Prerequisites

- `replit-localize` is complete (three-command setup works locally).
- The user has a GCP project (or is willing to create one), with billing enabled.
- The user has `gcloud`, `terraform` (≥1.5 or OpenTofu), and `gh` installed.
- The user has push access to the GitHub repo.

The user also needs to provide:

- Cloudflare R2 credentials (or equivalent object-storage credentials for the chosen platform).
- Any API keys the application calls (Anthropic, Stripe, etc.) — captured during `replit-audit`.

## Mental model — Replit → GCP service mapping

Replit's deployment shape maps one-to-one onto GCP services. There is no architectural rework.

| Replit | GCP |
| --- | --- |
| `autoscale` deployment | Cloud Run (scales to zero, per-request billing) |
| Auto-provisioned Postgres | Cloud SQL Postgres |
| Replit Secrets | Secret Manager |
| Replit's auto-HTTPS | Cloud Run native HTTPS |
| `[postMerge]` hook | GitHub Actions workflow |
| Replit's auto-deploy | GHA push-to-main triggers `gcloud run deploy` |

Use this 1:1 mapping when explaining the approach to a stakeholder. There's no need to evaluate Kubernetes, custom load balancers, or VPC architectures at v1.

## Work plan

Seven cohesive deliverables, each as its own commit:

1. **Platform evaluation document** (`docs/GCP_PLATFORM_EVALUATION.md`) — the rationale + three-tier cost projection.
2. **Production Dockerfile** (`Dockerfile` + `.dockerignore`) — multi-stage, alpine-based, ~450 MB.
3. **Terraform slice 1** — provider plumbing + Artifact Registry.
4. **Terraform slice 2** — Cloud SQL + the `DATABASE_URL` secret.
5. **Terraform slice 3** — remaining app secrets (Terraform-owned cryptographic + operator-populated external).
6. **Terraform slice 4** — Cloud Run service + service account + IAM.
7. **Terraform slice 5** — Workload Identity Federation for GitHub Actions.
8. **GitHub Actions deploy workflow** + operator docs.
9. **Deployment runbook** (`docs/DEPLOYMENT_RUNBOOK.md`).

Each Terraform slice is committed independently. The full `infra/` directory has ~9 Terraform files at the end, provisioning ~40+ GCP resources.

## Step 1 — Platform evaluation

Produce `docs/GCP_PLATFORM_EVALUATION.md`. Pull the application profile straight from `docs/INFRASTRUCTURE.md` (the `replit-audit` output). Sections:

- App profile (stack, deployment shape, scale assumptions).
- Layer-by-layer candidate analysis: compute, database, object storage, secrets, container registry, CI/CD, observability.
- Recommended stack (Cloud Run + Cloud SQL Enterprise + Cloudflare R2 + Secret Manager + Artifact Registry + GitHub Actions + WIF + Cloud Logging).
- Three-tier cost analysis: T1 pre-launch (~$52/mo), T2 early-prod (~$240/mo), T3 growth (~$880/mo). **Cloud SQL is 70–85% of total spend at every tier** — reframe optimization conversations around DB right-sizing, HA toggle, and committed-use discounts.
- Out of scope (multi-region, AlloyDB, Spanner, GKE).

**On object storage:** if the project already uses Cloudflare R2 (typical for Replit-origin), **keep it.** Don't migrate to GCS. R2 has $0 egress vs GCS's $0.12/GiB; for any document-heavy SaaS that's a real line item. R2 is API-compatible with S3 via the AWS SDK; the app already speaks it.

**Gotcha on GCP pricing pages:** they're JS-rendered, so `WebFetch` returns scaffolding HTML rather than pricing tables. Use `WebSearch` per-rate (e.g., `"Cloud SQL PostgreSQL pricing db-custom-1-3840 us-central1"`) to extract numbers; cite the canonical GCP pricing page in the doc itself.

## Step 2 — Production Dockerfile

Multi-stage, alpine-based:

```Dockerfile
# syntax=docker/dockerfile:1.7
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --no-audit --no-fund
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production \
    PORT=8080
RUN apk add --no-cache bash    # for start-prod.sh's `set -o pipefail`
COPY package.json package-lock.json ./
RUN npm ci --omit=dev --no-audit --no-fund && npm cache clean --force
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/migrations ./migrations
COPY scripts/db-migrate.js scripts/post-deploy.js scripts/start-prod.sh ./scripts/
RUN chown -R node:node /app && chmod +x ./scripts/start-prod.sh
USER node
EXPOSE 8080
CMD ["bash", "scripts/start-prod.sh"]
```

`.dockerignore` excludes `node_modules`, `dist`, `.env*`, `attached_assets/`, `migrations-archive/`, `docs/`, `compose.yaml`, `.replit*`, `.git`, etc.

**The `replit-localize` dep cleanup is a prerequisite for an efficient image.** Without it, the runtime image will be ~1 GB instead of ~450 MB because every `@radix-ui/*` and React package will end up in production `node_modules`.

**SSL gate refactor:** the localized codebase likely has `ssl: process.env.NODE_ENV === "production" ? { rejectUnauthorized: false } : false` in `server/db.ts` and `scripts/db-migrate.js`. Refactor to a `DB_SSL_MODE` env var (`disable | no-verify | verify`):

```ts
function resolveSsl() {
  const mode = process.env.DB_SSL_MODE
    ?? (process.env.NODE_ENV === "production" ? "no-verify" : "disable");
  switch (mode) {
    case "disable": return false;
    case "no-verify": return { rejectUnauthorized: false };
    case "verify": return { rejectUnauthorized: true };
    default: throw new Error(`Invalid DB_SSL_MODE: ${mode}`);
  }
}
```

Cloud SQL via the Auth Proxy uses a Unix socket — no pg-level TLS — so `DB_SSL_MODE=disable` is the right Cloud Run value.

**Healthcheck endpoint:** add `GET /api/health` (NOT `/healthz`) registered before any middleware. Returns `200 ok` text/plain, no DB dependency. **Important:** Google Front End (fronting Cloud Run) reserves `/healthz` at the edge and returns 404 to external requests — confirmed experimentally. Use `/api/health` to avoid the trap.

Commit: `feat(deploy): production Dockerfile + DB_SSL_MODE override + /api/health`.

## Step 3 — Terraform foundation (slice 1)

Under `infra/`:

- `versions.tf` — `required_version = ">= 1.5.0"` (last MPL release; OpenTofu-compatible), Google provider `~> 6.0`, hashicorp/random `~> 3.6`.
- `providers.tf` — `provider "google" { project = var.project_id; region = var.region }`.
- `variables.tf` — `project_id` (required, validated), `region` (default `us-central1`), `environment` (default `prod`, validated `dev|staging|prod`).
- `apis.tf` — enable `artifactregistry.googleapis.com` with `disable_on_destroy = false`.
- `artifact_registry.tf` — `google_artifact_registry_repository` in DOCKER format with cleanup policies (keep latest 10 tagged, delete untagged after 7 days).
- `outputs.tf` — repo ID and Docker pull/push URL prefix.
- `terraform.tfvars.example` — committed example, gitignored real values.
- `.gitignore` — `.terraform/`, `*.tfstate*`, `*.tfvars` (real), `*.tfplan`.
- `README.md` — apply procedure + slice roadmap.

**Convention notes:**
- `disable_on_destroy = false` on every `google_project_service` — destroy shouldn't disrupt other tenants.
- Label triple on every resource that supports labels: `app`, `environment`, `managed_by = "terraform"`.
- State backend deferred to slice 2 — bootstrap slice can use local state.

**Terraform string gotcha:** `${VAR}` in any string is parsed as interpolation, including `description` fields. For literal placeholder docs, use `<var>` or escape with `$${...}`.

Commit: `feat(infra): Terraform foundation + Artifact Registry repo`.

## Step 4 — Cloud SQL + DATABASE_URL secret (slice 2)

Adds `sqladmin`, `secretmanager`, `compute` API enables. New `cloud_sql.tf`:

- `random_password.db_root` and `random_password.db_app_user` (length 32, persisted in state — **this is when the GCS state backend becomes mandatory**).
- `google_sql_database_instance "main"` — Postgres 16, ENTERPRISE, ZONAL by default (T1 cost; expose `db_high_availability` variable for T2+). **`ip_configuration.ipv4_enabled = true` with NO `authorized_networks`** — the API rejects "no connectivity options" so you must enable public IP, but without authorized networks only the Cloud SQL Auth Proxy can reach the instance. Enable backups + 7-day PITR + `cloudsql.iam_authentication = on`.
- `google_sql_database "<dbname>"` + `google_sql_user "<appuser>"`.

New `secrets.tf`:

- 3 secrets: `<project>-<env>-database-url`, `-db-root-password`, `-db-app-password`.
- DATABASE_URL composed via `format("postgresql://%s:%s@/%s?host=/cloudsql/%s", user.name, user.password, db.name, instance.connection_name)` — the `?host=/cloudsql/...` query string tells `pg` to use a Unix socket (no TLS).

New `backend.tf.example` (GCS backend template; gitignore the real `backend.tf`). Update README to make GCS state backend bootstrap mandatory before this slice's apply.

**Cloud SQL apply takes ~10 minutes** — document prominently so the operator doesn't kill the apply assuming it hung.

Commit: `feat(infra): slice 2 — Cloud SQL + DATABASE_URL secret`.

## Step 5 — Remaining app secrets (slice 3)

Split app secrets into two ownership patterns:

**Terraform-owned (auto-generated cryptographic values):** session signing keys, webhook signing keys, encryption keys, post-deploy seed tokens. Use `random_password` (length 64, special on) or `random_id { byte_length = 32 }` for hex-formatted symmetric encryption keys. Each gets a secret + version managed by Terraform.

**Operator-owned (external service credentials):** R2 access key/secret, Anthropic API key, Stripe keys. Create the `google_secret_manager_secret` resource but **NOT** a version. Label them `populated = "operator"` so the distinction shows in Secret Manager UI. The operator runs `gcloud secrets versions add` post-apply.

**`echo "..." | gcloud secrets versions add` bakes a trailing newline.** Document `printf '...' \| gcloud secrets versions add ... --data-file=-` or `echo -n` in the runbook. Every operator hits this once.

**Non-secret config stays out of Secret Manager.** R2 endpoint URL, R2 bucket name, `APP_BASE_URL`, `NODE_ENV` → plain Cloud Run env vars. Saves $0.06/secret/month per misclassification and keeps the access audit signal clean.

Commit: `feat(infra): slice 3 — remaining app secrets in Secret Manager`.

## Step 6 — Cloud Run service + IAM (slice 4)

Adds `run`, `iam` API enables.

`service_account.tf`:

- `google_service_account.runtime` (`<project>-<env>-runtime`) — distinct from the deploy SA.
- Project-level IAM: `cloudsql.client` + `cloudsql.instanceUser` on the runtime SA.
- `for_each` over a local map of all app-secret IDs → per-secret `secretmanager.secretAccessor` bindings. Granular > project-wide.

`cloud_run.tf`:

- `google_cloud_run_v2_service "ask"` with `template.service_account = runtime.email`.
- Scaling 0–3 by default (configurable via vars).
- Resources 1 vCPU / 1 GiB, `cpu_idle = true`, `startup_cpu_boost = true`.
- `container_port = 8080` (do NOT add `env { name = "PORT" }` — Cloud Run **reserves** `PORT` and rejects the env block; it injects `PORT` itself based on `container_port`).
- 6 plain env vars: `NODE_ENV`, `DB_SSL_MODE=disable`, `APP_BASE_URL`, `R2_ENDPOINT`, `R2_BUCKET`, plus any other non-secret config.
- One `env { value_source { secret_key_ref { ... } } }` block per app secret (DATABASE_URL, signing secrets, R2 credentials, Anthropic key, etc.).
- Cloud SQL Auth Proxy mount via `volume_mounts { name = "cloudsql"; mount_path = "/cloudsql" }` + `volumes { name = "cloudsql"; cloud_sql_instance { instances = [...connection_name] } }`. Matches DATABASE_URL's `?host=/cloudsql/...`.
- `startup_probe.http_get.path = "/api/health"` (NOT `/healthz` — see Dockerfile step). 5s initial delay, 5s timeout, 10s period, 12 failures = 2 min total tolerance (accommodates migrate-on-start).
- `lifecycle.ignore_changes = [template[0].containers[0].image, client, client_version]` — CI owns image deploys after first apply.

Plus a `google_cloud_run_v2_service_iam_member` granting `roles/run.invoker` to `allUsers` for public ingress.

**Org-policy override required for `allUsers`:** Workspace orgs default-enforce Domain Restricted Sharing (`iam.allowedPolicyMemberDomains`), which rejects `allUsers` and `allAuthenticatedUsers`. Add a project-scoped override:

```hcl
resource "google_project_organization_policy" "allow_all_users" {
  project    = var.project_id
  constraint = "iam.allowedPolicyMemberDomains"
  list_policy { allow { all = true } }
}
```

Enable `orgpolicy.googleapis.com` in `apis.tf`. Add `depends_on` from the public_invoker binding to the override.

**Org policy changes propagate ~1–2 min** — the apply that creates both the override and the binding may fail once on propagation lag. Wrap in a retry loop: `until terraform apply -auto-approve; do sleep 30; done`.

**Bootstrap image:** default `var.container_image` to `gcr.io/cloudrun/hello`. With `min_instances = 0`, no instance starts until traffic, so the placeholder doesn't matter at create time — CI replaces it on first deploy.

Commit: `feat(infra): slice 4 — Cloud Run service + IAM`.

## Step 7 — Workload Identity Federation (slice 5)

Adds `iamcredentials`, `sts` API enables. `wif.tf`:

- `data "google_project" "current"` — needed for the project NUMBER (not ID; the `principalSet://` URL requires NUMBER).
- `google_iam_workload_identity_pool "github"`.
- `google_iam_workload_identity_pool_provider "github"` — issuer `https://token.actions.githubusercontent.com`. Map `assertion.sub`, `assertion.actor`, `assertion.aud`, `assertion.repository`, `assertion.ref`. **`attribute_condition = "assertion.repository == 'OWNER/REPO'"`** scopes to this single repo.
- `google_service_account.deploy` — distinct from the runtime SA.
- WIF binding (`workloadIdentityUser`) via `principalSet://iam.googleapis.com/projects/<NUMBER>/locations/global/workloadIdentityPools/github-actions/attribute.repository/OWNER/REPO`.
- Deploy SA roles: `artifactregistry.writer` + `run.developer`.
- act-as: deploy SA gets `roles/iam.serviceAccountUser` on the runtime SA.

Outputs: `wif_provider`, `wif_deploy_service_account`, and a heredoc `github_actions_auth_snippet` ready to paste into the workflow.

Commit: `feat(infra): slice 5 — Workload Identity Federation for GHA`.

## Step 8 — GitHub Actions deploy workflow

`.github/workflows/deploy.yml`:

```yaml
name: deploy
on:
  push: { branches: [main] }
  workflow_dispatch:
permissions:
  contents: read
  id-token: write   # required for WIF
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false
env:
  IMAGE: ${{ vars.ARTIFACT_REGISTRY_URL }}/<app-name>:${{ github.sha }}
jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.WIF_PROVIDER }}
          service_account: ${{ vars.DEPLOY_SERVICE_ACCOUNT }}
      - uses: google-github-actions/setup-gcloud@v2
      - run: gcloud auth configure-docker ${{ vars.GCP_REGION }}-docker.pkg.dev --quiet
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          platforms: linux/amd64   # Cloud Run is amd64; multi-platform wastes 50%
          tags: |
            ${{ env.IMAGE }}
            ${{ vars.ARTIFACT_REGISTRY_URL }}/<app-name>:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - run: |
          gcloud run deploy "${{ vars.CLOUD_RUN_SERVICE }}" \
            --image="${{ env.IMAGE }}" \
            --region="${{ vars.GCP_REGION }}" \
            --project="${{ vars.GCP_PROJECT_ID }}" --quiet
      - run: |
          # Smoke test — `gcloud run deploy` is synchronous, but confirm.
          URL=$(gcloud run services describe "${{ vars.CLOUD_RUN_SERVICE }}" --region="${{ vars.GCP_REGION }}" --project="${{ vars.GCP_PROJECT_ID }}" --format='value(status.url)')
          curl -sSf -o /dev/null "$URL/" && echo "GET / → 200"
          [ "$(curl -s -o /dev/null -w '%{http_code}' "$URL/api/auth/me")" = "401" ] && echo "auth gate ok"
```

Plus `.github/workflows/README.md` documenting the one-time setup — six `gh variable set` commands sourcing values from `terraform output -raw`:

```bash
gh variable set GCP_PROJECT_ID         --body "$(grep '^project_id' infra/terraform.tfvars | cut -d'"' -f2)"
gh variable set GCP_REGION             --body "$(grep '^region'     infra/terraform.tfvars | cut -d'"' -f2)"
gh variable set WIF_PROVIDER           --body "$(cd infra && terraform output -raw wif_provider)"
gh variable set DEPLOY_SERVICE_ACCOUNT --body "$(cd infra && terraform output -raw wif_deploy_service_account)"
gh variable set CLOUD_RUN_SERVICE      --body "$(cd infra && terraform output -raw cloud_run_service_name)"
gh variable set ARTIFACT_REGISTRY_URL  --body "$(cd infra && terraform output -raw artifact_registry_url)"
```

Use GitHub **variables**, NOT secrets — the values aren't sensitive (they appear in `gcloud` command output anyway) and treating them as secrets adds masking noise without security benefit.

Validate locally with `actionlint` (`brew install actionlint`) before push.

Commit: `feat(ci): GitHub Actions deploy workflow`.

## Step 9 — Deployment runbook

`docs/DEPLOYMENT_RUNBOOK.md` — the single end-to-end procedure from clean GCP project to serving production. Structure:

- Prerequisites (workstation tooling + what the user must provide).
- **Phase A** — One-time infrastructure bootstrap (gcloud auth → state bucket → tfvars → apply → operator-secret population).
- **Phase B** — First deploy (six `gh variable set` → push or workflow_dispatch → smoke).
- **Phase C** — Custom domain mapping.
- **Phase D** — Production secret rotation, explicit Path 1 (fresh DB) vs Path 2a (preserve old encryption key) vs Path 2b (re-encryption migration).
- Operations cookbook (logs, psql, secret access, rotation, env-var changes, sizing, backup/restore, cost monitoring).
- Rollback — Cloud Run revision traffic shift; for schema-affecting recoveries, `gcloud sql instances clone --point-in-time=...` (NOT `gcloud sql backups restore` which replaces the live instance).
- Decommission.
- Troubleshooting — top 5 failure modes with diagnostics.

Commit: `docs: deployment runbook — clean project → serving prod`.

## First-apply gotchas to expect

`terraform validate` doesn't exercise GCP API rules. The first real `terraform apply` will hit ≥1 error you couldn't catch locally. Documented failure modes from the canonical migration:

| Error | Fix |
| --- | --- |
| `Error 400: At least one of Public IP or Private IP or PSC connectivity must be enabled` | Set `ipv4_enabled = true` with empty `authorized_networks`. |
| `The following reserved env names were provided: PORT` | Remove the `env { name = "PORT" }` block; Cloud Run injects PORT from `ports.container_port`. |
| `One or more users named in the policy do not belong to a permitted customer, perhaps due to an organization policy` | Add the project-scoped `iam.allowedPolicyMemberDomains` override (slice 4). |
| Cloud Run service creation fails: `Secret ... was not found` | Operator-owned secrets need a placeholder v1 before Cloud Run can be created. Add `printf 'PLACEHOLDER_REPLACE_BEFORE_USE' \| gcloud secrets versions add ...` for each. |

Document these in the runbook's troubleshooting section.

## Operator decisions to surface

Throughout this skill, capture decisions the user needs to make in an `OPEN_QUESTIONS.md` at the project root. Examples that arise from the canonical migration:

- 🔴 R2 access key + secret (operator provides)
- 🔴 Anthropic API key (operator provides)
- 🔴 R2 endpoint URL + bucket name (operator provides — feeds into `terraform.tfvars`)
- 🟡 Custom domain mapping
- 🟡 GCP billing account ownership (don't leave on the developer's personal billing)
- 🟡 IAM admin / bus-factor backup
- 🟡 Incident contact + paging
- 🟡 GitHub branch protection / required reviews
- 🔵 Dev environment provisioning (separate from prod)
- 🔵 Open-signup vs invite-only

Three-tier severity: 🔴 blocks production capability · 🟡 architectural/hygiene · 🔵 discussion/scope.

## Output to surface

When complete, summarize for the user:

1. The nine commits + their roles.
2. Live infrastructure resources (Cloud Run URL, Cloud SQL instance, Artifact Registry, WIF provider, runtime + deploy SAs).
3. Cost commitment from this point (~$51/month idle baseline).
4. Pointer to `DELIVERED.md` (Workstream A–D mapping) and `OPEN_QUESTIONS.md` (operator decisions).

Then ask whether the user wants to:

- Provide real R2/Anthropic credentials now (close the 🔴 open questions).
- Set up a custom domain.
- Provision a separate dev environment (B2 from canonical WBS).
- Pause for ownership/access decisions before further changes.
