---
name: replit-to-gcp-runbook
description: Produce the deployment runbook — the single end-to-end operator-facing document covering bootstrap → first deploy → custom domain → secret rotation → operations cookbook → rollback → decommission → troubleshooting. Invoke as part of `replit-to-gcp` Step 9 or directly when the user asks "write me a runbook" / "how does someone else operate this?". Produces `docs/DEPLOYMENT_RUNBOOK.md`.
---

# Deployment Runbook

The single end-to-end document for taking a clean GCP project to a serving production deployment, plus the operations cookbook for steady-state running.

Distinct from the per-slice READMEs in `infra/` and `.github/workflows/` — those are reference; this is the narrative.

## Document structure

`docs/DEPLOYMENT_RUNBOOK.md` should have these sections, in this order:

```markdown
# Deployment Runbook — <Project Name>

## Prerequisites
<workstation tooling table; what the operator must provide from FLS>

## Phase A — One-time infrastructure bootstrap
A1. Authenticate gcloud
A2. Bootstrap the Terraform state bucket
A3. Configure terraform.tfvars
A4. Apply Terraform (~10 min for Cloud SQL)
A5. Re-apply with real APP_BASE_URL once known
A6. Populate operator-owned secrets

## Phase B — First deploy
B1. Wire up GitHub repo variables
B2. Trigger the first deploy
B3. Verify the deployment

## Phase C — Custom domain (optional)

## Phase D — Production secret rotation
Frame as decision: Path 1 (fresh DB, rotate now) vs
Path 2a (preserve old encryption key) vs
Path 2b (re-encryption migration)

## Operations cookbook
- View Cloud Run logs
- Open a psql session via `gcloud sql connect`
- Read a secret value (audit log caveat)
- Rotate a Terraform-owned secret
- Add a new env var
- Resize Cloud Run / Cloud SQL
- Manual Cloud SQL backup + restore
- Cost monitoring

## Rollback
Cloud Run revision traffic shift; for schema-affecting recoveries,
`gcloud sql instances clone --point-in-time=...` (NOT `gcloud sql
backups restore` which clobbers the live instance)

## Decommission
`terraform destroy` with the deletion_protection toggle gotcha

## Troubleshooting
Top 5 failure modes with diagnostics
```

Total target length: 400–500 lines. Worth being thorough — this is the doc someone reads at 2am when a deploy is failing.

## Key content for each section

### Phase A — one-time bootstrap

The four-step sequence:

```bash
# A1. Authenticate
gcloud auth login
gcloud auth application-default login
gcloud config set project <PROJECT_ID>

# A2. State bucket
PROJECT_ID="<project-id>"
gcloud storage buckets create gs://${PROJECT_ID}-tfstate \
  --location=us-central1 \
  --uniform-bucket-level-access \
  --soft-delete-duration=0s
gcloud storage buckets update gs://${PROJECT_ID}-tfstate --versioning

# Then write infra/backend.tf (copy from backend.tf.example, fill in)

# A3. terraform.tfvars
cd infra
cp terraform.tfvars.example terraform.tfvars
# Edit: project_id, app_base_url (placeholder OK), r2_endpoint, r2_bucket

# A4. Apply (~10 min)
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

**Critical inline notes:**
- "Cloud SQL apply takes ~10 minutes. Do not interrupt — a partially-created instance leaves a stuck resource that needs manual cleanup."
- "First apply will hit ≥1 error you couldn't catch with `terraform validate`. Expected. See Troubleshooting."

For A6 (operator-owned secrets), the `printf 'value' | gcloud secrets versions add ... --data-file=-` recipe with the **`echo -n` newline-trap warning** prominently called out.

### Phase B — first deploy

The six `gh variable set` commands sourcing from `terraform output -raw`. Then `git push origin main` or `gh workflow run deploy.yml`. Then two curl checks.

### Phase C — custom domain

```bash
gcloud beta run domain-mappings create \
  --service=<service-name> \
  --domain=<domain> \
  --region=<region>
```

The command prints DNS records to add. After DNS validates, update `app_base_url` in tfvars and re-apply.

### Phase D — secret rotation

Frame as an explicit decision:

> Which path is right depends on whether the new production starts with a fresh, empty database or carries data over from the existing Replit deployment.

**Path 1 — Fresh DB.** Recommended for greenfield. Terraform-generated secrets are already fresh (never existed on Replit). Operator-owned secrets (R2, Anthropic) can be brand-new — populate the new ones; revoke the old once the new GCP deploy is serving real traffic. The Replit deployment can stay alive in parallel until DNS flips.

**Path 2a — Data migration, preserve old encryption key.** If carrying data over: load the legacy `*_ENCRYPTION_KEY` value into Secret Manager. Production runs with the legacy key from day one. Rotation is a separate planned migration (decrypt with old, re-encrypt with new).

**Path 2b — Data migration with re-encryption migration.** Run a one-shot Cloud Run Job before serving traffic that decrypts every encrypted row with the old key and re-encrypts with the new. Higher engineering cost; cleaner steady state.

Recommend Path 1 for greenfield. Flag Path 2's re-encryption cost up front.

### Operations cookbook

Each operation as a single shell-block recipe. Examples:

```bash
# View Cloud Run logs (last 50, newest first)
SERVICE=$(cd infra && terraform output -raw cloud_run_service_name)
REGION=$(grep '^region' infra/terraform.tfvars | cut -d'"' -f2)
gcloud run services logs read "$SERVICE" --region="$REGION" --limit=50

# Open psql session (uses Cloud SQL Auth Proxy automatically)
INSTANCE=$(cd infra && terraform output -raw cloud_sql_instance_name)
gcloud sql connect "$INSTANCE" --user=<app-user> --database=<dbname>

# Read a secret value (logged in Cloud Audit Logs — use sparingly)
gcloud secrets versions access latest --secret=<app>-prod-database-url

# Rotate a Terraform-owned secret
cd infra
terraform taint random_password.session_secret
terraform apply
# Note: invalidates all existing user sessions; schedule for low-traffic window

# Resize Cloud Run / Cloud SQL — edit terraform.tfvars + terraform apply
```

### Rollback — the clone-not-restore distinction

`gcloud sql backups restore` **replaces** the live instance. For non-destructive recovery, use `gcloud sql instances clone --point-in-time=...` to create a new instance from PITR, validate it, swap `DATABASE_URL`, then decommission the old. This is the rollback the operator usually wants.

```bash
INSTANCE=$(cd infra && terraform output -raw cloud_sql_instance_name)

# Clone to a NEW instance at a specific point in time (non-destructive)
gcloud sql instances clone "$INSTANCE" "${INSTANCE}-restored" \
  --point-in-time="2026-05-08T12:00:00Z"

# Update DATABASE_URL to point at the clone, deploy, validate, then
# swap names / decommission old.
```

### Decommission

```bash
cd infra
# Set deletion_protection = false on Cloud SQL first
sed -i.bak 's|db_deletion_protection = true|db_deletion_protection = false|' terraform.tfvars
terraform apply

# Then destroy
terraform destroy
```

Backups, Artifact Registry images, and the state bucket survive `terraform destroy` (we set `disable_on_destroy = false` on API enables, and the buckets aren't Terraform-managed). Clean up manually if appropriate.

### Troubleshooting — five common failure modes

1. **`terraform apply` hangs on `google_sql_database_instance.main`** — Expected; ~10 min provisioning. Watch in another terminal: `gcloud sql operations list --instance=<INSTANCE>`.
2. **Cloud Run revision stuck in `Revision failed to become ready`** — Check `gcloud run services logs read` for the cause. Common: missing operator-populated secret; migration failure; missing IAM grant.
3. **`gh workflow run` rejects with "Workflow not found"** — The workflow file isn't on the default branch yet. Push `.github/workflows/deploy.yml` to `main` first.
4. **Deploy succeeds but `GET /` returns the placeholder hello image** — `gcloud run deploy` didn't run with the new image tag. Check the workflow logs around the deploy step. Most likely `vars.ARTIFACT_REGISTRY_URL` is empty; verify with `gh variable list`.
5. **`gcloud sql connect` hangs forever** — First invocation provisions a client cert (~60 sec). If still hung after 90s, may be behind a proxy: `gcloud config set proxy/disable_proxy true`.

## Cost monitoring section

```bash
# Set a project-level budget alert (one-time, via console):
# Billing → Budgets & alerts → Create Budget
# Recommended threshold: 50%, 90%, 100% of expected monthly total
```

Then describe what to watch (Cloud SQL utilization in Cloud SQL → Insights) and when to evaluate committed-use discounts (~3 months of stable T2 traffic).

## When to commit

```
docs: deployment runbook — clean project → serving prod, end-to-end

Single end-to-end procedure for taking a clean GCP project to a serving
production deployment, plus the operations cookbook. Distinct from the
per-slice READMEs — those are reference; this is the narrative.

~500 lines, structured as Phase A (bootstrap) → Phase B (first deploy)
→ Phase C (custom domain) → Phase D (secret rotation, with the three
explicit paths) → operations cookbook → rollback (clone-not-restore
distinction) → decommission → troubleshooting (top 5 failure modes).
```

## What this skill doesn't do

- Doesn't execute any of the runbook steps. It's a document.
- Doesn't keep itself up to date. As infrastructure changes, the runbook needs to follow. Treat as living documentation.

## Parent skill

Invoked from `replit-to-gcp` Step 9 — the final piece of the GCP migration. After this skill, the orchestrator returns control to the user with a complete handoff bundle: working infrastructure + automated CI/CD + operator documentation.
