---
name: replit-to-gcp-terraform-foundation
description: First Terraform slice for a Replit→GCP migration. Provisions provider plumbing, project variables, Artifact Registry API enablement, the Docker repository, and cleanup policies. No cost-incurring resources beyond Artifact Registry storage (free under 0.5 GB). Invoke as part of `replit-to-gcp` Step 3 or directly when the user asks "bootstrap Terraform for this GCP project" / "set up the AR repo." Writes files under `infra/`. Does NOT run `terraform apply`.
---

# Terraform Slice 1 — Foundation + Artifact Registry

The first Terraform slice for a Replit→GCP migration. Provisions just enough to give the production Dockerfile a concrete deploy target.

## What this slice creates

| Resource | Purpose |
| --- | --- |
| `google_project_service.artifactregistry` | API enable for `artifactregistry.googleapis.com` |
| `google_artifact_registry_repository.<app>` | Docker repository where CI pushes images |

Two resources. Plus the Terraform module scaffolding (provider config, variables, outputs, README, .gitignore).

## Files to write under `infra/`

### `versions.tf`

```hcl
terraform {
  # >= 1.5.0 is the last MPL release and is OpenTofu-compatible. If the org
  # standardizes on a specific tool/version later, narrow the range.
  required_version = ">= 1.5.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
  }
}
```

### `providers.tf`

```hcl
provider "google" {
  project = var.project_id
  region  = var.region
}
```

### `variables.tf`

```hcl
variable "project_id" {
  type        = string
  description = "GCP project ID. Must already exist; this config does not create the project."

  validation {
    condition     = length(var.project_id) > 0
    error_message = "project_id is required."
  }
}

variable "region" {
  type        = string
  description = "Default region for regional resources."
  default     = "us-central1"
}

variable "environment" {
  type        = string
  description = "Deployment environment. Used as a label and in resource naming."
  default     = "prod"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod."
  }
}
```

### `apis.tf`

```hcl
# Enables the GCP service APIs used by this slice. Each subsequent slice
# (Cloud SQL, Cloud Run, Secret Manager, IAM/WIF) adds its own API entries.
#
# disable_on_destroy = false: `terraform destroy` does NOT disable the API
# on the project. Avoids breaking other tenants of the same project.

resource "google_project_service" "artifactregistry" {
  service            = "artifactregistry.googleapis.com"
  disable_on_destroy = false
}
```

### `artifact_registry.tf`

```hcl
resource "google_artifact_registry_repository" "<app>" {
  location      = var.region
  repository_id = "<app>"
  format        = "DOCKER"
  description   = "<App Name> production container images"

  labels = {
    app         = "<app>"
    environment = var.environment
    managed_by  = "terraform"
  }

  cleanup_policies {
    id     = "keep-latest-10-tagged"
    action = "KEEP"
    most_recent_versions {
      keep_count = 10
    }
  }

  cleanup_policies {
    id     = "delete-untagged-after-7d"
    action = "DELETE"
    condition {
      tag_state  = "UNTAGGED"
      older_than = "604800s" # 7 days
    }
  }

  depends_on = [google_project_service.artifactregistry]
}
```

### `outputs.tf`

```hcl
output "artifact_registry_repository" {
  description = "Full repository ID for use in CI/CD image refs."
  value       = google_artifact_registry_repository.<app>.id
}

output "artifact_registry_url" {
  description = "Docker pull/push URL prefix."
  value       = "${var.region}-docker.pkg.dev/${var.project_id}/${google_artifact_registry_repository.<app>.repository_id}"
}
```

**Terraform string gotcha:** `${VAR}` in any string is parsed as interpolation, including `description` fields. For literal placeholder docs like `<artifact_registry_url>`, use angle brackets or escape with `$${...}`. `terraform validate` catches the syntax error but not until you try to init providers, so this surfaces late.

### `terraform.tfvars.example`

```hcl
# Copy to terraform.tfvars (gitignored) and fill in real values.
project_id  = "<project-id>"
region      = "us-central1"
environment = "prod"
```

### `.gitignore`

```
# Terraform working dir + provider plugins
.terraform/
.terraform.lock.hcl.bak

# State (must never be committed — contains secrets in later slices)
*.tfstate
*.tfstate.*
*.tfstate.backup

# Local variable files (project-specific values, sometimes secrets)
terraform.tfvars
*.auto.tfvars

# Plan files + crash logs
*.tfplan
crash.log
crash.*.log

# Real backend config — example template committed at backend.tf.example
backend.tf
```

Lock file (`.terraform.lock.hcl`) IS committed for provider-version reproducibility.

### `README.md`

Short. Apply procedure, prerequisites (gcloud auth, optional GCS state backend bootstrap — required from slice 2 onward), conventions (resource naming, labels, API enablement), the slice roadmap.

## Conventions to establish at slice 1

These carry forward through every subsequent slice:

- **`disable_on_destroy = false`** on every `google_project_service`. Destroy shouldn't disrupt other tenants of a shared project.
- **Label triple** on every resource that supports labels: `app`, `environment`, `managed_by = "terraform"`. Useful for cost-attribution filters.
- **State backend deferred to slice 2.** Slice 1 doesn't persist secrets; local state is acceptable for the bootstrap pass. Switch to GCS before any secret-containing resource lands.

## Validation (no cloud actions yet)

```bash
cd infra
terraform fmt -diff -check    # should produce no diff
terraform init -backend=false  # downloads providers; no state interaction
terraform validate             # should print "Success! The configuration is valid."
```

If `terraform validate` fails, fix before proceeding. Don't run `terraform apply` yet — that's the operator's decision (see Slice 2 README for the cost-decision gate).

## Cost on apply

$0/month at typical Replit-Agent scale. Artifact Registry storage is free under 0.5 GB; one Docker image (~140 MB compressed) plus a few historical tags fits comfortably.

## When to commit

```
feat(infra): Terraform foundation + Artifact Registry repo

First slice of Phase 3 infrastructure-as-code. Provisions provider
plumbing, project variables, AR API enablement, and the Docker
repository with cleanup policies. No cost-incurring resources
beyond AR storage (free under 0.5 GB).
```

## What this skill doesn't do

- Doesn't apply. Operator runs `terraform apply` per the runbook.
- Doesn't create the GCP project, billing account, or state bucket. Those are operator prerequisites.

## Parent skill

Invoked from `replit-to-gcp` Step 3. Slice 2 (`replit-to-gcp-terraform-cloud-sql`) builds directly on this scaffolding.
