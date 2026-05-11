---
name: replit-to-gcp-terraform-wif
description: Fifth Terraform slice — provisions Workload Identity Federation for GitHub Actions so CI can authenticate to GCP via short-lived OIDC tokens instead of long-lived service-account JSON keys. Invoke as part of `replit-to-gcp` Step 7 or when the user asks "set up WIF for GitHub Actions." Writes files under `infra/`. Outputs include a ready-to-paste auth snippet for the GHA workflow.
---

# Terraform Slice 5 — Workload Identity Federation

The final Terraform slice. Lets GitHub Actions authenticate to GCP via short-lived OIDC tokens — no JSON service-account keys in GitHub secrets, no key rotation cadence to maintain.

## Architecture

```
GitHub Actions runner
    │  (presents OIDC token; iss=token.actions.githubusercontent.com)
    ▼
google_iam_workload_identity_pool_provider.github
    │  (validates iss/aud, maps claims like `repository` and `ref` onto Google attributes)
    │  (attribute_condition restricts to OWNER/REPO)
    ▼
google_iam_workload_identity_pool.github
    │  (principal = principalSet://...attribute.repository/OWNER/REPO)
    ▼
google_service_account.deploy
    │  (workloadIdentityUser binding; deploy SA is distinct from runtime SA)
    ▼
Project IAM grants: artifactregistry.writer + run.developer
    │
    ▼
Plus act-as: deploy SA has iam.serviceAccountUser on runtime SA
    │  (required to `gcloud run services update` services that run as runtime SA)
    ▼
GitHub Actions runs as deploy SA for the duration of the workflow.
```

## What this slice creates

| Resource | Purpose |
| --- | --- |
| `google_iam_workload_identity_pool.github` | Logical container for federated identities |
| `google_iam_workload_identity_pool_provider.github` | GitHub OIDC validator + attribute mapper |
| `google_service_account.deploy` | Distinct from runtime SA; CI's identity |
| `google_service_account_iam_member.deploy_workload_identity_user` | Binds GHA principal to deploy SA |
| `google_project_iam_member.deploy_artifactregistry_writer` | CI can push images |
| `google_project_iam_member.deploy_run_developer` | CI can update Cloud Run services |
| `google_service_account_iam_member.deploy_actas_runtime` | CI can deploy services running as runtime SA |

Plus two new API enables (`iamcredentials`, `sts`).

## Files to write

### Update `apis.tf`

```hcl
# WIF requires the IAM Credentials and Security Token Service APIs.
resource "google_project_service" "iamcredentials" {
  service            = "iamcredentials.googleapis.com"
  disable_on_destroy = false
}

resource "google_project_service" "sts" {
  service            = "sts.googleapis.com"
  disable_on_destroy = false
}
```

### Update `variables.tf`

```hcl
variable "github_owner" {
  type        = string
  description = "GitHub organization or user owning the repo."
}

variable "github_repo" {
  type        = string
  description = "Repository name (no owner prefix)."
}
```

### New `wif.tf`

```hcl
# Workload Identity Federation. GHA authenticates with its short-lived OIDC
# token; the pool exchanges it for short-lived GCP credentials.

# Need the project NUMBER (not ID) for the principalSet:// URL.
# data "google_project" "current" is the canonical way to pick it up.
data "google_project" "current" {}

resource "google_iam_workload_identity_pool" "github" {
  workload_identity_pool_id = "github-actions"
  display_name              = "GitHub Actions"
  depends_on = [
    google_project_service.iamcredentials,
    google_project_service.sts,
  ]
}

resource "google_iam_workload_identity_pool_provider" "github" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github"
  display_name                       = "GitHub OIDC"

  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.actor"      = "assertion.actor"
    "attribute.aud"        = "assertion.aud"
    "attribute.repository" = "assertion.repository"
    "attribute.ref"        = "assertion.ref"
  }

  # Scope to this single repo. Without the condition, any GHA-hosted runner
  # from any repo could exchange tokens against the pool — noise floor we
  # don't want, even if no role is granted.
  attribute_condition = "assertion.repository == \"${var.github_owner}/${var.github_repo}\""

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }
}

# Separate from runtime SA. Keeps CI permissions tight (deploy SA doesn't
# need DB access; runtime SA doesn't need image-push permissions).
resource "google_service_account" "deploy" {
  account_id   = "<app>-${var.environment}-deploy"
  display_name = "<App> CI deploy (${var.environment})"
  depends_on   = [google_project_service.iam]
}

# WIF binding — only the specific GitHub repo can impersonate this SA.
resource "google_service_account_iam_member" "deploy_workload_identity_user" {
  service_account_id = google_service_account.deploy.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/projects/${data.google_project.current.number}/locations/global/workloadIdentityPools/${google_iam_workload_identity_pool.github.workload_identity_pool_id}/attribute.repository/${var.github_owner}/${var.github_repo}"
}

# Deploy SA permissions: push images + update Cloud Run services.
resource "google_project_iam_member" "deploy_artifactregistry_writer" {
  project = var.project_id
  role    = "roles/artifactregistry.writer"
  member  = "serviceAccount:${google_service_account.deploy.email}"
}

resource "google_project_iam_member" "deploy_run_developer" {
  project = var.project_id
  role    = "roles/run.developer"
  member  = "serviceAccount:${google_service_account.deploy.email}"
}

# act-as: deploy SA must be allowed to deploy services that run as the
# runtime SA. Without this, `gcloud run services update` fails with
# "Permission denied to use service account ...".
resource "google_service_account_iam_member" "deploy_actas_runtime" {
  service_account_id = google_service_account.runtime.name
  role               = "roles/iam.serviceAccountUser"
  member             = "serviceAccount:${google_service_account.deploy.email}"
}
```

### Update `outputs.tf`

```hcl
output "wif_provider" {
  description = "Pass to google-github-actions/auth as workload_identity_provider."
  value       = google_iam_workload_identity_pool_provider.github.name
}

output "wif_deploy_service_account" {
  description = "Pass to google-github-actions/auth as service_account."
  value       = google_service_account.deploy.email
}

# Ready-to-paste workflow snippet
output "github_actions_auth_snippet" {
  description = "Drop-in auth step for the GitHub Actions workflow."
  value = <<-EOT
    - id: auth
      uses: google-github-actions/auth@v2
      with:
        workload_identity_provider: ${google_iam_workload_identity_pool_provider.github.name}
        service_account: ${google_service_account.deploy.email}
  EOT
}
```

The heredoc output means the operator can `terraform output -raw github_actions_auth_snippet` and paste straight into their workflow YAML — no manual URL composition, no risk of typo.

## Common pitfall: project ID vs project NUMBER

The `principalSet://iam.googleapis.com/projects/<NUMBER>/...` URL needs the project **NUMBER**, not the ID. `var.project_id` is the ID (string like `<app>-prod`). The NUMBER (numeric like `943129270781`) comes from `data "google_project" "current" {}` → `.number`. Use the data source — without it the binding silently matches nothing and CI can't authenticate with no obvious error.

## Apply expectations

```bash
terraform plan -out=tfplan
# Slice 5 adds ~7 resources:
#   + 2 google_project_service (iamcredentials, sts)
#   + 1 google_iam_workload_identity_pool
#   + 1 google_iam_workload_identity_pool_provider
#   + 1 google_service_account (deploy)
#   + 2 google_project_iam_member (artifactregistry.writer, run.developer)
#   + 2 google_service_account_iam_member (workloadIdentityUser, deploy_actas_runtime)

terraform apply tfplan
```

## Cost from this slice

$0. WIF is free. Service accounts are free. IAM bindings are free.

## When to commit

```
feat(infra): slice 5 — Workload Identity Federation for GHA

Lets GitHub Actions authenticate to GCP via short-lived OIDC tokens.
No GCP service-account JSON keys in GitHub secrets, no key rotation
cadence. Pool + provider + deploy SA + IAM grants
(artifactregistry.writer, run.developer, act-as runtime SA) all
scoped via attribute_condition to a single repo.

Output github_actions_auth_snippet is a ready-to-paste workflow auth
step — operator runs `terraform output -raw
github_actions_auth_snippet` and pastes into .github/workflows/.
```

## What this skill doesn't do

- Doesn't write the GHA workflow itself. That's `replit-to-gcp-gha-deploy`.
- Doesn't set the GitHub repo variables. The operator runs `gh variable set` post-apply — documented in `replit-to-gcp-gha-deploy`'s README.

## Parent skill

Invoked from `replit-to-gcp` Step 7. The GHA workflow consumes `wif_provider` and `wif_deploy_service_account` outputs from this slice.
