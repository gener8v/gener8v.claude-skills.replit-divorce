---
name: replit-to-gcp-terraform-secrets
description: Third Terraform slice — provisions the remaining application secrets in Secret Manager. Splits them into two ownership patterns: Terraform-owned (auto-generated cryptographic values for session signing, encryption keys, etc.) and operator-owned (external service credentials like R2, Anthropic, Stripe). Invoke as part of `replit-to-gcp` Step 5 or directly when the user asks "set up the app secrets." Writes files under `infra/`. Operator-owned secrets need placeholder versions added via `gcloud secrets versions add` before Cloud Run can be created in slice 4.
---

# Terraform Slice 3 — App Secrets in Secret Manager

The third Terraform slice. Provisions Cloud Run's runtime env-var secrets — everything beyond `DATABASE_URL` (which was slice 2).

## The two-bucket pattern

Split app secrets by **ownership of the value**:

| Bucket | Members | Pattern |
| --- | --- | --- |
| **Terraform-owned** (auto-generated cryptographic values) | `SESSION_SECRET`, signing keys, encryption keys, post-deploy seed tokens | `random_password` / `random_id` + secret + version, all in state |
| **Operator-owned** (external service credentials) | `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `ANTHROPIC_API_KEY`, Stripe, etc. | Secret resource only, no version (operator adds via `gcloud secrets versions add`) |

The distinction matters because:

- Terraform CAN'T generate external credentials. R2 keys come from Cloudflare's dashboard; Anthropic keys from console.anthropic.com.
- But Terraform CAN generate session secrets, encryption keys, and other cryptographically-random values with no external dependency. There's no reason to make the operator do that work manually.

Apply the two-bucket pattern explicitly: label operator-owned secrets with `populated = "operator"` so the distinction is visible in the Secret Manager UI.

## Non-secret config stays OUT of Secret Manager

`R2_ENDPOINT` (a URL), `R2_BUCKET` (a name), `APP_BASE_URL`, `NODE_ENV` — these are not secrets. They appear in network metadata anyway and have no security value in masking. They go straight on the Cloud Run env-var list in slice 4 — NOT in Secret Manager.

Mixing the two costs ~$0.06/secret/month per misclassification, hides the access audit signal under noise, and obscures the env-var contract for whoever reads the deploy config.

## Files to write

### New `secrets_app.tf`

```hcl
# Application secrets consumed by Cloud Run at boot.

# ============================================================
# Bucket A — Terraform-owned cryptographic secrets
# ============================================================
# These have no external dependency. The value just needs to be a strong
# random of the right shape. Rotation = `terraform taint` + `terraform apply`.

# Express session signing key. 64 random alphanumerics + safe specials.
resource "random_password" "session_secret" {
  length  = 64
  special = true
}

resource "random_password" "api_key_signing_secret" {
  length  = 64
  special = true
}

resource "random_password" "webhook_signing_secret" {
  length  = 64
  special = true
}

# Field-level encryption key. 32 bytes / 64 hex chars — AES-256 sized.
# Use random_id for hex-formatted symmetric keys.
resource "random_id" "encryption_key" {
  byte_length = 32
}

# Optional post-deploy seed gate. Alphanumeric-only — it's a token.
resource "random_password" "seed_init_token" {
  length  = 48
  special = false
}

resource "google_secret_manager_secret" "session_secret" {
  secret_id = "<app>-${var.environment}-session-secret"
  replication { auto {} }
  labels = { app = "<app>", environment = var.environment, managed_by = "terraform" }
  depends_on = [google_project_service.secretmanager]
}

resource "google_secret_manager_secret_version" "session_secret" {
  secret      = google_secret_manager_secret.session_secret.id
  secret_data = random_password.session_secret.result
}

# … same pattern for api_key_signing_secret, webhook_signing_secret,
# encryption_key (uses random_id.encryption_key.hex), seed_init_token

# ============================================================
# Bucket B — Operator-owned external-service secrets
# ============================================================
# Terraform creates the secret container. The operator adds the first
# version AFTER apply via:
#
#   printf '<value>' | gcloud secrets versions add <secret> --data-file=-
#
# Cloud Run will deploy with these referenced but instances FAIL TO START
# until at least one version exists. That's the right failure mode — a
# missing secret is loud, not silent.

resource "google_secret_manager_secret" "r2_access_key_id" {
  secret_id = "<app>-${var.environment}-r2-access-key-id"
  replication { auto {} }
  labels = {
    app         = "<app>"
    environment = var.environment
    managed_by  = "terraform"
    populated   = "operator"     # signals: no Terraform-managed version
  }
  depends_on = [google_project_service.secretmanager]
}

# … same pattern for r2_secret_access_key, anthropic_api_key, and any
# other external credential the audit identified

# NO google_secret_manager_secret_version resources for the operator
# bucket — the version is added post-apply by `gcloud secrets versions add`.
```

### Update `outputs.tf`

```hcl
# All Terraform-managed app-secret IDs (for slice 4 to reference)
output "secret_session_secret_id"          { value = google_secret_manager_secret.session_secret.id }
output "secret_api_key_signing_secret_id"  { value = google_secret_manager_secret.api_key_signing_secret.id }
# … one per Terraform-owned secret

# All operator-owned secret IDs (for slice 4 to reference)
output "secret_r2_access_key_id_id"         { value = google_secret_manager_secret.r2_access_key_id.id }
output "secret_r2_secret_access_key_id"     { value = google_secret_manager_secret.r2_secret_access_key.id }
output "secret_anthropic_api_key_id"        { value = google_secret_manager_secret.anthropic_api_key.id }

# Self-documenting post-apply checklist of secrets needing operator action
output "operator_populated_secrets" {
  description = "Secret IDs the operator must populate before Cloud Run will boot."
  value = [
    google_secret_manager_secret.r2_access_key_id.secret_id,
    google_secret_manager_secret.r2_secret_access_key.secret_id,
    google_secret_manager_secret.anthropic_api_key.secret_id,
    # … list every operator-owned secret
  ]
}
```

## Apply expectations

```bash
terraform plan -out=tfplan
# Through slice 3 expect to ADD ~14 resources beyond slice 2:
#   + 4 random_password + 1 random_id
#   + 5 google_secret_manager_secret (Terraform-owned) + 5 versions
#   + 3 google_secret_manager_secret (operator-owned, no versions)

terraform apply tfplan
```

## Post-apply: populating operator-owned secrets

For each operator-owned secret, the operator runs:

```bash
printf '<value>' | gcloud secrets versions add <app>-prod-r2-access-key-id     --data-file=-
printf '<value>' | gcloud secrets versions add <app>-prod-r2-secret-access-key --data-file=-
printf '<value>' | gcloud secrets versions add <app>-prod-anthropic-api-key    --data-file=-
```

**Critical: `printf`, not `echo`.** `echo "..." | ...` appends a trailing newline that bakes into the secret and surfaces hours later as opaque "authentication failed" errors from the third-party API. Every operator hits this once. Document in the runbook.

Verify each operator-owned secret has at least one version before proceeding to slice 4:

```bash
terraform output -json operator_populated_secrets | jq -r '.[]' | while read secret; do
  v=$(gcloud secrets versions list "$secret" --format='value(name)' --limit=1)
  echo "$secret → version $v"
done
```

Each line should show a version number. If any line says `→ ` (blank), Cloud Run will fail to create in slice 4.

## Bootstrap placeholder for slice 4 sequencing

If you need to apply slice 4 (Cloud Run) before the operator has real R2 / Anthropic values, populate placeholders first:

```bash
for s in $(terraform output -json operator_populated_secrets | jq -r '.[]'); do
  printf 'PLACEHOLDER_REPLACE_BEFORE_USE' | gcloud secrets versions add "$s" --data-file=-
done
```

Cloud Run will deploy and boot successfully with placeholders; only the code paths that actually call those external services fail at runtime. Replace with real values when available.

## Cost from this slice

| Resource | Cost |
| --- | --- |
| Secret Manager — 8 secrets (5 TF-owned + 3 operator) | ~$0.48/mo |

Plus ~10K access ops/month from Cloud Run (free under the 10K-op tier).

## When to commit

```
feat(infra): slice 3 — remaining app secrets in Secret Manager

Provisions Cloud Run's env-var secrets beyond DATABASE_URL. Split by
ownership: 5 Terraform-generated cryptographic values (session/signing
keys, encryption key, seed token) and 3 operator-populated containers
(R2 access key, R2 secret, Anthropic key). Non-secret config (R2
endpoint, R2 bucket, APP_BASE_URL) intentionally stays out of Secret
Manager — those land as plain env vars in slice 4.
```

## What this skill doesn't do

- Doesn't bind the secrets to Cloud Run. That's slice 4.
- Doesn't grant Cloud Run's runtime service account access to the secrets. Also slice 4.
- Doesn't populate operator-owned values. That's an operator action documented in the runbook.

## Parent skill

Invoked from `replit-to-gcp` Step 5. Slice 4 (`replit-to-gcp-terraform-cloud-run`) consumes every secret ID this slice outputs.
