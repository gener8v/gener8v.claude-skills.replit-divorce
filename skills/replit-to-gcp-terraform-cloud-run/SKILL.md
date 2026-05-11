---
name: replit-to-gcp-terraform-cloud-run
description: Fourth Terraform slice — provisions the Cloud Run service, its runtime service account with per-secret IAM, Cloud SQL Auth Proxy mount, env-var bindings (plain + secret), startup probe pointing at `/api/health`, and the org-policy override that allows `allUsers` invoker bindings under Workspace orgs with Domain Restricted Sharing. Invoke as part of `replit-to-gcp` Step 6 or when the user asks "deploy the Cloud Run service." Writes files under `infra/`.
---

# Terraform Slice 4 — Cloud Run + IAM

The slice that ties everything together. Cloud Run service, runtime service account, per-secret IAM, Cloud SQL Auth Proxy mount, public-invoker binding, org-policy override.

## What this slice creates

| Resource | Purpose |
| --- | --- |
| `google_service_account.runtime` | Identity Cloud Run instances run as |
| `google_project_iam_member` × 2 | `cloudsql.client`, `cloudsql.instanceUser` on runtime SA |
| `google_secret_manager_secret_iam_member` × N | Per-secret `secretAccessor` for runtime SA (one per app secret) |
| `google_cloud_run_v2_service.<app>` | The service itself |
| `google_cloud_run_v2_service_iam_member.public_invoker` | `allUsers` → `roles/run.invoker` |
| `google_project_organization_policy.allow_all_users` | Project-scoped override of Domain Restricted Sharing |

Plus three new API enables (`run`, `iam`, `orgpolicy`) and new variables for Cloud Run sizing.

## Files to write

### Update `apis.tf`

```hcl
resource "google_project_service" "run" {
  service            = "run.googleapis.com"
  disable_on_destroy = false
}

resource "google_project_service" "iam" {
  service            = "iam.googleapis.com"
  disable_on_destroy = false
}

# Needed for the project-scoped org-policy override
resource "google_project_service" "orgpolicy" {
  service            = "orgpolicy.googleapis.com"
  disable_on_destroy = false
}
```

### Update `variables.tf`

```hcl
variable "container_image" {
  type        = string
  description = "Cloud Run container image. Defaults to a public placeholder; CI replaces it on first deploy. Terraform ignores subsequent changes via lifecycle.ignore_changes."
  default     = "gcr.io/cloudrun/hello"
}

variable "cloud_run_min_instances" {
  type    = number
  default = 0   # T1: scale to zero
}

variable "cloud_run_max_instances" {
  type    = number
  default = 3
}

variable "cloud_run_cpu" {
  type    = string
  default = "1"
}

variable "cloud_run_memory" {
  type    = string
  default = "1Gi"
}

# Operator-supplied app config (non-secret)
variable "app_base_url" {
  type        = string
  description = "Public URL the app is reachable at. Placeholder OK for first apply; update once custom domain is set up."
}

variable "r2_endpoint" {
  type        = string
  description = "Cloudflare R2 S3-compatible endpoint URL."
}

variable "r2_bucket" {
  type    = string
}
```

### New `org_policy.tf`

```hcl
# Workspace orgs default-enforce Domain Restricted Sharing, which rejects
# allUsers and allAuthenticatedUsers in IAM bindings. The Cloud Run service
# needs allUsers for public ingress, so we override the constraint at PROJECT
# scope — leaves the org-wide default intact for other projects.

resource "google_project_organization_policy" "allow_all_users" {
  project    = var.project_id
  constraint = "iam.allowedPolicyMemberDomains"
  list_policy {
    allow {
      all = true
    }
  }
  depends_on = [google_project_service.orgpolicy]
}
```

### New `service_account.tf`

```hcl
# Distinct from the deploy SA (which lives in WIF slice). Runtime SA is what
# Cloud Run instances run as — needs CloudSQL client + per-secret accessor.

resource "google_service_account" "runtime" {
  account_id   = "<app>-${var.environment}-runtime"
  display_name = "<App> runtime (${var.environment})"
  depends_on   = [google_project_service.iam]
}

resource "google_project_iam_member" "runtime_cloudsql_client" {
  project = var.project_id
  role    = "roles/cloudsql.client"
  member  = "serviceAccount:${google_service_account.runtime.email}"
}

resource "google_project_iam_member" "runtime_cloudsql_instance_user" {
  project = var.project_id
  role    = "roles/cloudsql.instanceUser"
  member  = "serviceAccount:${google_service_account.runtime.email}"
}

# Per-secret accessor grants — granular, not project-wide. Uses a for_each
# over a local map of all app-secret IDs (DB-related from slice 2 and
# app-related from slice 3).
locals {
  runtime_secret_ids = {
    database_url            = google_secret_manager_secret.database_url.id
    db_app_password         = google_secret_manager_secret.db_app_password.id
    session_secret          = google_secret_manager_secret.session_secret.id
    api_key_signing_secret  = google_secret_manager_secret.api_key_signing_secret.id
    webhook_signing_secret  = google_secret_manager_secret.webhook_signing_secret.id
    encryption_key          = google_secret_manager_secret.encryption_key.id
    seed_init_token         = google_secret_manager_secret.seed_init_token.id
    r2_access_key_id        = google_secret_manager_secret.r2_access_key_id.id
    r2_secret_access_key    = google_secret_manager_secret.r2_secret_access_key.id
    anthropic_api_key       = google_secret_manager_secret.anthropic_api_key.id
  }
}

resource "google_secret_manager_secret_iam_member" "runtime_accessor" {
  for_each  = local.runtime_secret_ids
  secret_id = each.value
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.runtime.email}"
}
```

### New `cloud_run.tf`

```hcl
resource "google_cloud_run_v2_service" "<app>" {
  name     = "<app>-${var.environment}"
  location = var.region
  ingress             = "INGRESS_TRAFFIC_ALL"
  deletion_protection = false  # operator-readable revisions; flip on after launch

  labels = { app = "<app>", environment = var.environment, managed_by = "terraform" }

  template {
    service_account = google_service_account.runtime.email

    scaling {
      min_instance_count = var.cloud_run_min_instances
      max_instance_count = var.cloud_run_max_instances
    }

    containers {
      image = var.container_image

      ports {
        container_port = 8080
      }

      resources {
        limits = { cpu = var.cloud_run_cpu, memory = var.cloud_run_memory }
        cpu_idle          = true   # only billed during request handling
        startup_cpu_boost = true   # free; doubles CPU during cold start
      }

      # Startup probe — Cloud Run waits for /api/health to return 200 before
      # routing traffic to a new revision. NOT /healthz — GFE reserves it
      # at the edge. See replit-to-gcp-dockerfile for the experimental finding.
      startup_probe {
        http_get { path = "/api/health" }
        initial_delay_seconds = 5
        timeout_seconds       = 5
        period_seconds        = 10
        failure_threshold     = 12   # 2 min total — accommodates migrate-on-start
      }

      # Plain env vars (non-secret). PORT is INTENTIONALLY NOT here —
      # Cloud Run reserves PORT and injects it from ports.container_port.
      # Adding `env { name = "PORT" }` is rejected by the API.
      env { name = "NODE_ENV"     value = "production" }
      env { name = "DB_SSL_MODE"  value = "disable" }   # Auth Proxy = socket; no pg-level TLS
      env { name = "APP_BASE_URL" value = var.app_base_url }
      env { name = "R2_ENDPOINT"  value = var.r2_endpoint }
      env { name = "R2_BUCKET"    value = var.r2_bucket }

      # Secret env vars (Secret Manager refs). One block per app secret.
      env {
        name = "DATABASE_URL"
        value_source {
          secret_key_ref {
            secret  = google_secret_manager_secret.database_url.secret_id
            version = "latest"
          }
        }
      }

      env {
        name = "SESSION_SECRET"
        value_source {
          secret_key_ref {
            secret  = google_secret_manager_secret.session_secret.secret_id
            version = "latest"
          }
        }
      }

      # … one env block per remaining secret: api_key_signing_secret,
      # webhook_signing_secret, encryption_key, seed_init_token,
      # r2_access_key_id, r2_secret_access_key, anthropic_api_key

      # Cloud SQL Auth Proxy mount
      volume_mounts {
        name       = "cloudsql"
        mount_path = "/cloudsql"
      }
    }

    volumes {
      name = "cloudsql"
      cloud_sql_instance {
        instances = [google_sql_database_instance.main.connection_name]
      }
    }
  }

  # CI owns image deploys after first apply
  lifecycle {
    ignore_changes = [
      template[0].containers[0].image,
      client,
      client_version,
    ]
  }

  depends_on = [
    google_project_service.run,
    google_secret_manager_secret_iam_member.runtime_accessor,
    google_project_iam_member.runtime_cloudsql_client,
  ]
}

# Public ingress — allUsers can hit the URL. The app handles auth at the
# route layer. depends_on the org-policy override so the binding is allowed.
resource "google_cloud_run_v2_service_iam_member" "public_invoker" {
  project  = var.project_id
  location = google_cloud_run_v2_service.<app>.location
  name     = google_cloud_run_v2_service.<app>.name
  role     = "roles/run.invoker"
  member   = "allUsers"

  depends_on = [google_project_organization_policy.allow_all_users]
}
```

### Update `outputs.tf`

```hcl
output "cloud_run_service_name" {
  value = google_cloud_run_v2_service.<app>.name
}

output "cloud_run_service_url" {
  value = google_cloud_run_v2_service.<app>.uri
}

output "cloud_run_runtime_service_account" {
  description = "Runtime SA email. WIF slice grants the deploy SA `iam.serviceAccountUser` on this."
  value       = google_service_account.runtime.email
}
```

## Apply expectations

```bash
terraform plan -out=tfplan
# Slice 4 adds ~18 resources:
#   + 3 google_project_service (run, iam, orgpolicy)
#   + 1 google_project_organization_policy
#   + 1 google_service_account (runtime)
#   + 2 google_project_iam_member (cloudsql client + instance user)
#   + N google_secret_manager_secret_iam_member (one per runtime_secret_ids entry)
#   + 1 google_cloud_run_v2_service
#   + 1 google_cloud_run_v2_service_iam_member (public_invoker)

terraform apply tfplan
```

## First-apply gotchas

Three errors you will hit if anything is misconfigured. Each is documented in the runbook's troubleshooting section:

### Gotcha 1 — Reserved `PORT` env var

```
Error 400: The following reserved env names were provided: PORT.
```

Cloud Run reserves `PORT`. The `cloud_run.tf` template above intentionally omits it; if you add it, you'll hit this. Remove the explicit `env { name = "PORT" }` block.

### Gotcha 2 — `allUsers` blocked by Domain Restricted Sharing

```
Error 400: One or more users named in the policy do not belong to a
permitted customer, perhaps due to an organization policy.
```

The `google_project_organization_policy.allow_all_users` override resolves this. **Org policy changes propagate ~1–2 minutes** — the first apply that creates both the override and the public_invoker binding may fail because the override hasn't propagated yet. Two mitigations:

- `depends_on` from the binding to the override (already in the template).
- Retry loop: `until terraform apply -auto-approve; do sleep 30; done`.

### Gotcha 3 — Operator-owned secrets need placeholder versions

```
Error: Secret projects/.../secrets/<name>/versions/latest was not found
```

Cloud Run's `secret_key_ref { version = "latest" }` is resolved at service-creation time, not lazily. If any operator-owned secret has no versions yet, Cloud Run creation fails. Either populate real values first, or stash placeholder versions (`replit-to-gcp-terraform-secrets` documents the `printf 'PLACEHOLDER_REPLACE_BEFORE_USE' \| gcloud secrets versions add ...` pattern).

## Cost from this slice

| Resource | Cost |
| --- | --- |
| Cloud Run with `min_instances = 0` | **~$0/month at idle** |

Per-request billing only. At T1 traffic levels (idle most of the time, minimal real traffic), Cloud Run cost is well under the free tier.

`min_instances = 1` for instant cold-start adds ~$15–20/month.

## When to commit

```
feat(infra): slice 4 — Cloud Run service + IAM

Provisions the runtime service account with per-secret IAM (for_each over
all app secrets, granular accessor grants), the Cloud Run service with Cloud
SQL Auth Proxy mount + startup probe (/api/health, NOT /healthz), and the
project-scoped org-policy override allowing allUsers public invoker
bindings. Container image bootstrap uses gcr.io/cloudrun/hello placeholder;
lifecycle.ignore_changes hands image management to CI after first apply.
```

## What this skill doesn't do

- Doesn't deploy a real image. The placeholder serves at first apply; CI deploys the real image in `replit-to-gcp-gha-deploy`.
- Doesn't set up the deploy SA or WIF. That's slice 5.

## Parent skill

Invoked from `replit-to-gcp` Step 6. Slice 5 (`replit-to-gcp-terraform-wif`) grants the deploy SA `iam.serviceAccountUser` on the runtime SA created here.
