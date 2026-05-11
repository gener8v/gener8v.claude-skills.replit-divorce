---
name: replit-to-gcp-gha-deploy
description: Build the GitHub Actions push-to-main → Cloud Run deploy workflow that uses Workload Identity Federation for auth (no JSON keys), Buildx with GHA layer cache, two-tag push (`<sha>` + `:latest`), synchronous `gcloud run deploy`, and a curl smoke test. Invoke as part of `replit-to-gcp` Step 8 or directly when the user asks "set up CI/CD" / "automate deploys." Writes files under `.github/workflows/`.
---

# GitHub Actions Deploy Workflow

The CI/CD workflow that closes the loop on Phase 3 — every push to `main` builds, pushes, and deploys to Cloud Run automatically. Authentication is WIF (provisioned by `replit-to-gcp-terraform-wif`), so there are no GCP service-account JSON keys in GitHub Secrets.

## Files to write

### `.github/workflows/deploy.yml`

```yaml
# Build → push → deploy on every commit to main.
#
# Auth is via Workload Identity Federation. The `id-token: write` permission
# below is what lets the runner request the OIDC token that WIF exchanges
# for short-lived GCP credentials. No GCP service-account JSON keys.
#
# Required repository variables (set once via `gh variable set` after
# `terraform apply`):
#
#   GCP_PROJECT_ID            ← terraform.tfvars project_id
#   GCP_REGION                ← terraform.tfvars region
#   WIF_PROVIDER              ← terraform output -raw wif_provider
#   DEPLOY_SERVICE_ACCOUNT    ← terraform output -raw wif_deploy_service_account
#   CLOUD_RUN_SERVICE         ← terraform output -raw cloud_run_service_name
#   ARTIFACT_REGISTRY_URL     ← terraform output -raw artifact_registry_url
#
# No GitHub Secrets are needed.

name: deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  id-token: write   # required for WIF OIDC token exchange

# Serialize deploys to the same ref so two pushes in quick succession don't
# race. cancel-in-progress=false: let in-flight deploys finish rather than
# abandoning a partially-applied rollout.
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false

env:
  IMAGE_TAG: ${{ github.sha }}
  IMAGE: ${{ vars.ARTIFACT_REGISTRY_URL }}/<app>:${{ github.sha }}
  IMAGE_LATEST: ${{ vars.ARTIFACT_REGISTRY_URL }}/<app>:latest

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 20

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Authenticate to GCP via Workload Identity Federation
        id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.WIF_PROVIDER }}
          service_account: ${{ vars.DEPLOY_SERVICE_ACCOUNT }}

      - name: Set up gcloud
        uses: google-github-actions/setup-gcloud@v2

      - name: Configure Docker for Artifact Registry
        run: gcloud auth configure-docker ${{ vars.GCP_REGION }}-docker.pkg.dev --quiet

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          # AMD64 only — Cloud Run runs Linux amd64. Multi-platform default
          # doubles build time for no benefit.
          platforms: linux/amd64
          tags: |
            ${{ env.IMAGE }}
            ${{ env.IMAGE_LATEST }}
          # GHA cache survives across runs in the same repo.
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy to Cloud Run
        # `gcloud run deploy` is synchronous: waits for the new revision to
        # be ready and serving. If migrate-on-start fails or a secret is
        # missing, this step fails and the previous revision keeps serving.
        run: |
          gcloud run deploy "${{ vars.CLOUD_RUN_SERVICE }}" \
            --image="${{ env.IMAGE }}" \
            --region="${{ vars.GCP_REGION }}" \
            --project="${{ vars.GCP_PROJECT_ID }}" \
            --quiet

      - name: Smoke test
        run: |
          URL=$(gcloud run services describe "${{ vars.CLOUD_RUN_SERVICE }}" \
            --region="${{ vars.GCP_REGION }}" \
            --project="${{ vars.GCP_PROJECT_ID }}" \
            --format='value(status.url)')
          echo "Service URL: $URL"

          # GET / — SPA shell or app root, expect 200 with non-trivial body.
          STATUS=$(curl -s -o /tmp/body -w '%{http_code}' "$URL/")
          BYTES=$(wc -c < /tmp/body)
          echo "GET /               → HTTP $STATUS  $BYTES bytes"
          if [ "$STATUS" != "200" ] || [ "$BYTES" -lt 100 ]; then
            echo "::error::Smoke test failed: GET / returned HTTP $STATUS, $BYTES bytes"
            exit 1
          fi

          # GET /api/health — explicit health endpoint
          STATUS=$(curl -s -o /tmp/body -w '%{http_code}' "$URL/api/health")
          echo "GET /api/health     → HTTP $STATUS"
          if [ "$STATUS" != "200" ]; then
            echo "::error::Health check failed: /api/health returned HTTP $STATUS"
            exit 1
          fi

          # GET /api/auth/me without a session — expect 401 JSON.
          # (Adjust based on the app's actual auth-gate response.)
          STATUS=$(curl -s -o /tmp/body -w '%{http_code}' "$URL/api/auth/me")
          echo "GET /api/auth/me    → HTTP $STATUS"
          if [ "$STATUS" != "401" ]; then
            echo "::error::Auth gate broken: /api/auth/me returned HTTP $STATUS, expected 401"
            cat /tmp/body
            exit 1
          fi

      - name: Summary
        if: always()
        run: |
          {
            echo "## Deploy summary"
            echo ""
            echo "| Field | Value |"
            echo "| --- | --- |"
            echo "| Image | \`${{ env.IMAGE }}\` |"
            echo "| Service | \`${{ vars.CLOUD_RUN_SERVICE }}\` |"
            echo "| Region | \`${{ vars.GCP_REGION }}\` |"
            echo "| Triggered by | ${{ github.event_name }} |"
            echo "| Commit | ${{ github.sha }} |"
          } >> "$GITHUB_STEP_SUMMARY"
```

### `.github/workflows/README.md`

```markdown
# GitHub Actions workflows

Production deploy is fully automated via Workload Identity Federation. No
long-lived service-account JSON keys are stored in GitHub.

## One-time setup (after first `terraform apply`)

Six GitHub repository **variables** (not secrets — none of these values
are sensitive; they appear in `gcloud` command output anyway):

```bash
cd infra
gh variable set GCP_PROJECT_ID         --body "$(grep '^project_id' terraform.tfvars | cut -d'"' -f2)"
gh variable set GCP_REGION             --body "$(grep '^region'     terraform.tfvars | cut -d'"' -f2)"
gh variable set WIF_PROVIDER           --body "$(terraform output -raw wif_provider)"
gh variable set DEPLOY_SERVICE_ACCOUNT --body "$(terraform output -raw wif_deploy_service_account)"
gh variable set CLOUD_RUN_SERVICE      --body "$(terraform output -raw cloud_run_service_name)"
gh variable set ARTIFACT_REGISTRY_URL  --body "$(terraform output -raw artifact_registry_url)"
```

Verify with `gh variable list`.

## Why variables not secrets

All six values appear in `gcloud` command output during deploys (which are
in the workflow logs). Treating them as secrets adds masking noise without
adding security. Reserve secrets for actual secrets — there are none for
this workflow because WIF replaces them.

## Workflow shape

deploy.yml triggers on push to main + manual dispatch:

1. WIF auth via `google-github-actions/auth@v2`
2. Configure Docker for Artifact Registry
3. Build with Buildx + GHA layer cache (amd64 only)
4. Push two tags: `<sha>` and `:latest`
5. `gcloud run deploy` (synchronous — fails if new revision can't start)
6. Smoke test: GET /, GET /api/health, GET /api/auth/me
7. Markdown summary appended to the workflow run page
```

## Local validation before push

```bash
# YAML parses cleanly
npx --yes -p js-yaml -c "js-yaml .github/workflows/deploy.yml" | head

# actionlint catches common issues GitHub's UI surfaces only on first run
# (action versioning, expression typos, syntax)
brew install actionlint
actionlint .github/workflows/deploy.yml
# Expect: no findings
```

These two checks together catch most workflow bugs without burning a real CI run.

## First-deploy expectations

The workflow's first run (with cold Buildx cache) takes ~2–3 minutes:

| Step | Approximate time |
| --- | ---: |
| Checkout | 5s |
| WIF auth | 5s |
| Docker setup | 10s |
| Build + push (cold cache) | 2 min |
| `gcloud run deploy` | 30–45 sec |
| Smoke test | 5s |
| **Total** | **~2m30s** |

Warm-cache runs: ~1m30s.

## Failure modes

- **`vars.WIF_PROVIDER` unset** → auth step fails with "workload_identity_provider is required." → Run the `gh variable set` recipe.
- **Cloud Run new revision fails to start** → `gcloud run deploy` step fails; previous revision keeps serving. Check `gcloud run services logs read` for the cause (most common: missing secret value, schema migration failure).
- **`/healthz` smoke test returns Google 404** → you used `/healthz` instead of `/api/health`. GFE reserves `/healthz`. See `replit-to-gcp-dockerfile`.

## When to commit

```
feat(ci): GitHub Actions deploy workflow

Push to main → automatic Cloud Run deploy. WIF auth (no JSON keys),
Buildx + GHA layer cache, two-tag push (<sha> + :latest), synchronous
gcloud run deploy, curl smoke test mirroring the local docker-run
verification. Six repo variables document the one-time setup.

Validated locally: js-yaml parses cleanly, actionlint clean.
```

## What this skill doesn't do

- Doesn't run the workflow. It triggers on push to main; the operator pushes a commit (or runs `gh workflow run deploy.yml`) when ready.
- Doesn't enforce branch protection or required reviews. Configure those in GitHub repo settings.
- Doesn't handle a separate dev/prod environment split. That's a follow-on once a dev environment exists.

## Parent skill

Invoked from `replit-to-gcp` Step 8. The runbook (Step 9) documents how to wire the GitHub variables and trigger the first deploy.
