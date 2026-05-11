---
name: replit-to-gcp-terraform-cloud-sql
description: Second Terraform slice — provisions the production Cloud SQL Postgres instance, the application database and user, and the `DATABASE_URL` secret in Secret Manager (composed via `format(...)` against the instance's connection name). Public IP disabled by configuration but enabled at the API level (a GCP requirement); connectivity is only via the Cloud SQL Auth Proxy. Invoke as part of `replit-to-gcp` Step 4 or when the user asks "set up Cloud SQL for this project." Writes files under `infra/`. GCS state backend bootstrap is mandatory before applying — passwords land in state.
---

# Terraform Slice 2 — Cloud SQL + DATABASE_URL Secret

The slice that provisions the production database. This is when **the GCS state backend becomes mandatory** — `random_password` resources persist their generated values into `terraform.tfstate` in plaintext.

## What this slice creates

| Resource | Purpose |
| --- | --- |
| `random_password.db_root`, `random_password.db_app_user` | Generated passwords (length 32, persisted in state) |
| `google_sql_database_instance.main` | Postgres 16, ENTERPRISE edition, ZONAL |
| `google_sql_database.<dbname>` | The application database |
| `google_sql_user.<appuser>` | The application user |
| `google_secret_manager_secret.database_url` + version | Connection string in Secret Manager |
| `google_secret_manager_secret.db_root_password` + version | For ad-hoc psql sessions |
| `google_secret_manager_secret.db_app_password` + version | Separate so app-user can rotate without touching DATABASE_URL |

Plus three new API enables (`sqladmin`, `secretmanager`, `compute`) and the `random` provider in `versions.tf`.

## Pre-apply: GCS state bootstrap (mandatory from this slice forward)

```bash
PROJECT_ID="<project-id>"
gcloud storage buckets create gs://${PROJECT_ID}-tfstate \
  --location=us-central1 \
  --uniform-bucket-level-access \
  --soft-delete-duration=0s
gcloud storage buckets update gs://${PROJECT_ID}-tfstate --versioning
```

Then `cp backend.tf.example backend.tf` and edit. The `backend.tf` template:

```hcl
terraform {
  backend "gcs" {
    bucket = "<PROJECT_ID>-tfstate"
    prefix = "<app>/<environment>"
  }
}
```

`backend.tf` is gitignored; only `backend.tf.example` is committed.

After writing `backend.tf`, migrate state from local to GCS:

```bash
cd infra
terraform init -migrate-state
```

## Files to write

### Update `versions.tf` — add `random` provider

```hcl
required_providers {
  google = {
    source  = "hashicorp/google"
    version = "~> 6.0"
  }
  random = {
    source  = "hashicorp/random"
    version = "~> 3.6"
  }
}
```

### Update `apis.tf` — add three API enables

```hcl
resource "google_project_service" "sqladmin" {
  service            = "sqladmin.googleapis.com"
  disable_on_destroy = false
}

resource "google_project_service" "secretmanager" {
  service            = "secretmanager.googleapis.com"
  disable_on_destroy = false
}

# Cloud SQL transitively requires Compute APIs even when not using private IP.
resource "google_project_service" "compute" {
  service            = "compute.googleapis.com"
  disable_on_destroy = false
}
```

### Update `variables.tf` — add Cloud SQL sizing vars

```hcl
variable "db_tier" {
  type        = string
  description = "Cloud SQL machine type. db-custom-1-3840 = 1 vCPU / 3.75 GiB (T1 baseline)."
  default     = "db-custom-1-3840"
}

variable "db_storage_gb" {
  type        = number
  description = "Initial Cloud SQL SSD storage. Autoresize is on, so this is a floor."
  default     = 10
}

variable "db_high_availability" {
  type        = bool
  description = "ZONAL (T1, ~$50/mo) vs REGIONAL (HA failover, +~$110/mo)."
  default     = false
}

variable "db_deletion_protection" {
  type        = bool
  description = "Block accidental destroy. Set false only when explicitly recreating."
  default     = true
}
```

### New `cloud_sql.tf`

```hcl
# Cloud SQL Postgres for the application.
#
# Connection model: public IP enabled at the API level but with NO
# authorized_networks. The instance has a public IP allocated, but only the
# authenticated Cloud SQL Auth Proxy (which Cloud Run uses automatically)
# can reach it. No firewall holes. No VPC/peering setup.
#
# GCP requires at least one of public IP / private IP / PSC to be enabled.
# `ipv4_enabled = false` with no other connectivity is rejected with:
#   "At least one of Public IP or Private IP or PSC connectivity must be enabled."
# Public IP + empty authorized_networks is equivalent to private IP for our
# threat model, without the VPC/peering complexity.

resource "random_password" "db_root" {
  length      = 32
  special     = true
  min_special = 4
}

resource "random_password" "db_app_user" {
  length      = 32
  special     = true
  min_special = 4
}

resource "google_sql_database_instance" "main" {
  name             = "<app>-${var.environment}"
  database_version = "POSTGRES_16"
  region           = var.region

  deletion_protection = var.db_deletion_protection
  root_password       = random_password.db_root.result

  settings {
    tier              = var.db_tier
    edition           = "ENTERPRISE"
    availability_type = var.db_high_availability ? "REGIONAL" : "ZONAL"
    disk_type         = "PD_SSD"
    disk_size         = var.db_storage_gb
    disk_autoresize   = true

    backup_configuration {
      enabled                        = true
      point_in_time_recovery_enabled = true
      start_time                     = "07:00"  # 02:00 CT, low-traffic
      transaction_log_retention_days = 7
      backup_retention_settings {
        retained_backups = 7
        retention_unit   = "COUNT"
      }
    }

    maintenance_window {
      day          = 7  # Sunday
      hour         = 4  # 04:00 UTC
      update_track = "stable"
    }

    ip_configuration {
      # Public IP enabled with NO authorized_networks list — the only callers
      # are the authenticated Cloud SQL Auth Proxy. See comment at top.
      ipv4_enabled = true
    }

    database_flags {
      name  = "cloudsql.iam_authentication"
      value = "on"
    }

    insights_config {
      query_insights_enabled  = true
      query_string_length     = 1024
      record_application_tags = false  # privacy-default
      record_client_address   = false  # privacy-default
    }

    user_labels = {
      app         = "<app>"
      environment = var.environment
      managed_by  = "terraform"
    }
  }

  depends_on = [
    google_project_service.sqladmin,
    google_project_service.compute,
  ]
}

resource "google_sql_database" "<dbname>" {
  name     = "<dbname>"
  instance = google_sql_database_instance.main.name
}

resource "google_sql_user" "app" {
  name     = "<appuser>"
  instance = google_sql_database_instance.main.name
  password = random_password.db_app_user.result
}
```

### New `secrets.tf` — the three DB secrets

```hcl
# DATABASE_URL is composed via format() to use Cloud Run's Auth Proxy
# mount path: postgresql://USER:PASS@/DB?host=/cloudsql/CONNECTION_NAME
# The ?host=/cloudsql/... query string tells `pg` to use a Unix socket
# instead of TCP — no pg-level TLS needed (DB_SSL_MODE=disable).

resource "google_secret_manager_secret" "database_url" {
  secret_id = "<app>-${var.environment}-database-url"
  replication { auto {} }
  labels = {
    app         = "<app>"
    environment = var.environment
    managed_by  = "terraform"
  }
  depends_on = [google_project_service.secretmanager]
}

resource "google_secret_manager_secret_version" "database_url" {
  secret = google_secret_manager_secret.database_url.id
  secret_data = format(
    "postgresql://%s:%s@/%s?host=/cloudsql/%s",
    google_sql_user.app.name,
    google_sql_user.app.password,
    google_sql_database.<dbname>.name,
    google_sql_database_instance.main.connection_name,
  )
}

resource "google_secret_manager_secret" "db_root_password" {
  secret_id = "<app>-${var.environment}-db-root-password"
  replication { auto {} }
  labels = { /* same triple */ }
  depends_on = [google_project_service.secretmanager]
}

resource "google_secret_manager_secret_version" "db_root_password" {
  secret      = google_secret_manager_secret.db_root_password.id
  secret_data = random_password.db_root.result
}

resource "google_secret_manager_secret" "db_app_password" {
  secret_id = "<app>-${var.environment}-db-app-password"
  replication { auto {} }
  labels = { /* same triple */ }
  depends_on = [google_project_service.secretmanager]
}

resource "google_secret_manager_secret_version" "db_app_password" {
  secret      = google_secret_manager_secret.db_app_password.id
  secret_data = random_password.db_app_user.result
}
```

Using `format(...)` instead of building the connection string manually means Terraform's dependency graph automatically waits for all four resources (user, db, instance, password) to exist before the secret version is created. Any downstream change flows through to a new secret version automatically.

### Update `outputs.tf`

```hcl
output "cloud_sql_connection_name" {
  description = "PROJECT:REGION:INSTANCE — pass to Cloud Run as cloudsql_instances."
  value       = google_sql_database_instance.main.connection_name
}

output "cloud_sql_instance_name" {
  description = "Bare instance name. Use with `gcloud sql connect`."
  value       = google_sql_database_instance.main.name
}

output "secret_database_url_id" {
  value = google_secret_manager_secret.database_url.id
}

output "secret_db_root_password_id" {
  value = google_secret_manager_secret.db_root_password.id
}
```

## Apply expectations

```bash
terraform plan -out=tfplan
# Through slice 2 expect ~12 resources to add:
#   + 4 google_project_service (artifactregistry, sqladmin, secretmanager, compute)
#   + 1 google_artifact_registry_repository
#   + 2 random_password
#   + 1 google_sql_database_instance       ← takes ~10 minutes
#   + 1 google_sql_database, 1 google_sql_user
#   + 3 google_secret_manager_secret + 3 google_secret_manager_secret_version

terraform apply tfplan
```

**The Cloud SQL instance takes ~10 minutes to provision.** `terraform apply` will appear stuck on `google_sql_database_instance.main` for that whole window. **Do not kill the apply** — a partially-created instance leaves a stuck resource that needs manual cleanup. Document prominently in the runbook.

## Connecting to Cloud SQL after apply

Public IP is allocated but no authorized networks. Two paths:

- **Cloud Run** (production): mounts the instance via `cloudsql_instances` in the service config — handled in `replit-to-gcp-terraform-cloud-run`.
- **Operator workstation** (ad-hoc psql): `gcloud sql connect <instance> --user=<appuser> --database=<dbname>`. Transparently uses the Cloud SQL Auth Proxy. First connection provisions a temporary client cert (~30 sec); subsequent connections are instant.

For the root password: `gcloud secrets versions access latest --secret=<app>-prod-db-root-password`.

## Cost from this slice

| Resource | Cost |
| --- | --- |
| Cloud SQL `db-custom-1-3840` ZONAL + 10 GB SSD + backups + PITR | **~$50/mo** |
| Secret Manager — 3 secrets | ~$0.18/mo |

This is ~85% of the platform's total cost at T1. Tunable via `db_tier`, `db_high_availability`, `db_storage_gb`.

## When to commit

```
feat(infra): slice 2 — Cloud SQL + DATABASE_URL secret

Provisions Postgres 16 instance (ENTERPRISE, ZONAL, public IP enabled
but no authorized_networks — Auth-Proxy-only access), application
database and user, plus three Secret Manager entries: composed
DATABASE_URL, root password, app password. State now contains
random_password.result values, so GCS state backend is mandatory
before any apply.
```

## What this skill doesn't do

- Doesn't apply. Operator runs `terraform apply` after reviewing the plan and confirming the ~$50/mo cost commitment.
- Doesn't migrate data. Fresh DB; data migration is a separate workstream.
- Doesn't provision Cloud Run yet — that's slice 4, which depends on this and slice 3.

## Parent skill

Invoked from `replit-to-gcp` Step 4.
