---
name: replit-to-gcp-platform-eval
description: Produce a written platform-evaluation document that justifies the Cloud Run + Cloud SQL + R2 + Secret Manager + Artifact Registry + GHA + WIF stack for a Replit-origin project, with three-tier cost projections backed by GCP's pricing pages. Invoke as part of `replit-to-gcp` Step 1 or directly when the user asks "write me a GCP architecture rec" / "how much will this cost on GCP?". Produces `docs/GCP_PLATFORM_EVALUATION.md`.
---

# Platform Evaluation Document

Produces a written evaluation that takes the application profile (from `replit-audit`) and maps it to a recommended GCP stack with three-tier cost projections. Output is `docs/GCP_PLATFORM_EVALUATION.md`.

## Why this document matters

It's the artifact the client takes to leadership. The cost projections justify the migration; the per-layer reasoning justifies the specific service choices. Without this document, the migration looks like "we picked GCP because the engineer liked it" — even when the engineering judgment is correct.

## Required input

The Phase 1 audit's `docs/INFRASTRUCTURE.md` (or `docs/REPLIT_AUDIT.md`) — specifically:

- The stack profile (Node + Express + Postgres + R2 + Anthropic, or whatever the actual shape is).
- The env-var contract.
- Scale assumptions (current customers, expected ramp).

**Don't re-derive this.** The audit document already has it. Read it.

## Document structure

```markdown
# GCP Platform Evaluation — <Project Name>

## 1. App profile
<copy the stack + scale profile from the audit, condensed>

## 2. Workload tiers (for cost projection)
T1 — pre-launch / staging: idle most of the time, minimal traffic
T2 — early production: sustained traffic from ~10 customers, hours/day
T3 — growth: ~10× T2 (~100 customers, business hours)

## 3. Layer-by-layer evaluation
3.1 Compute
3.2 Database
3.3 Object storage
3.4 Secret management
3.5 Container registry
3.6 CI/CD
3.7 Observability

## 4. Recommended stack
<one-row-per-layer table with the picks>

## 5. Per-service cost analysis
<one section per service with rate × units = monthly>

## 6. Total monthly cost summary
<one row per service, three columns T1/T2/T3, footer with total>

## 7. Out of scope
<things deliberately not evaluated: multi-region, AlloyDB, Spanner, GKE>
```

## Per-layer recommendations (the canonical picks)

For a Replit-Agent fullstack project, the recommendations are mostly determined by the stack:

| Layer | Recommendation | Why |
| --- | --- | --- |
| Compute | **Cloud Run** | Replit's `autoscale` deployment maps 1:1 to Cloud Run's "container scales to zero, billed per request." No architectural rework. |
| Database | **Cloud SQL Postgres (Enterprise edition)** | Direct replacement for Replit's auto-Postgres. Enterprise edition is the standard tier; Enterprise Plus is overkill at T1/T2. Same Postgres major version. |
| Object storage | **Cloudflare R2 (no change)** | R2 has $0 egress vs GCS's $0.12/GiB. For any document-heavy SaaS that ships PDFs to customers, this is the dominant cost factor at scale. Plus R2 is already wired via `@aws-sdk/client-s3`. **Don't migrate to GCS without a strong reason.** |
| Secret management | **Secret Manager** | No serious alternative on GCP. Workload Identity from Cloud Run → Secret Manager IAM roles is the canonical pattern. |
| Container registry | **Artifact Registry** | Replaces the legacy Container Registry. Cleanup policies (keep latest N tagged, delete untagged after T days) cap storage cost. |
| CI/CD | **GitHub Actions + Workload Identity Federation** | The repo is already on GitHub. 2,000 free Linux build-minutes/month for private repos. WIF lets GHA push and deploy without long-lived service account keys. Cloud Build is the alternative but adds a second SCM workflow. |
| Observability | **Cloud Logging + Cloud Monitoring** | Auto-from-Cloud-Run. No agent install. Generous free tier. |

## Cost analysis discipline

Three-tier projection per service. Each rate cited to its canonical GCP pricing page:

- [Cloud Run pricing](https://cloud.google.com/run/pricing)
- [Cloud SQL pricing](https://cloud.google.com/sql/pricing)
- [Secret Manager pricing](https://cloud.google.com/secret-manager/pricing)
- [Artifact Registry pricing](https://cloud.google.com/artifact-registry/pricing)
- [Cloud Logging / Monitoring pricing](https://cloud.google.com/stackdriver/pricing)
- [Internet egress pricing](https://cloud.google.com/vpc/network-pricing) (mostly $0 from Cloud Run at T1)

**Gotcha — GCP pricing pages are JS-rendered.** `WebFetch` against `cloud.google.com/<service>/pricing` returns scaffolding HTML, not the pricing tables. Workaround: use `WebSearch` for each specific rate (e.g., `"Cloud SQL PostgreSQL db-custom-1-3840 us-central1 pricing"`). Search-result snippets surface the actual numbers from cached pages and third-party blogs. Cite the canonical GCP pricing page in your document; the search is just the extraction mechanism.

## The headline insight to lead with

For a Cloud Run + managed-Postgres stack, **Cloud SQL is 70–85% of total monthly spend at every tier.** Compute is rounding error; secrets are rounding error; egress is rounding error (especially with R2 sticking around).

This reframes the cost-optimization conversation:

- ❌ "Should we use Cloud Run or GKE?" — low-impact; pick Cloud Run and move on.
- ✅ "How do we right-size the DB?" — high-impact; spend energy here.

Concretely, at T1 (~$52/mo total): Cloud SQL ~$50, everything else ~$2. At T3 (~$880/mo total): Cloud SQL ~$740, everything else ~$140. Optimization energy goes to DB tier, HA toggle, and committed-use discounts.

## Typical T1/T2/T3 totals

For a moderately-sized SaaS in `us-central1`, list price, no committed-use discounts:

| Tier | Cloud Run | Cloud SQL | Secrets | Registry | Egress | Logging | **Total** |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| T1 pre-launch | ~$0 | ~$50 | ~$0.06 | ~$0.10 | ~$0.50 | $0 | **~$52/mo** |
| T2 early prod | ~$10–15 | ~$220 | ~$0.30 | ~$0.40 | ~$3.50 | $0 | **~$240/mo** |
| T3 growth | ~$50 | ~$740 | ~$0.60 | ~$1 | ~$40 | ~$15 | **~$880/mo** |

Adjust the Cloud SQL numbers based on the actual sizing the application needs — these assume db-custom-1-3840 (1 vCPU / 3.75 GiB) at T1, doubled at T2, ~6× at T3.

## Out-of-scope section (always include)

The document should explicitly call out what wasn't evaluated:

- Multi-region failover (single `us-central1` is sufficient through T3; multi-region adds Load Balancer ~$18/mo + cross-region replication + DNS failover ≈ $200+/mo).
- AlloyDB (Google's higher-performance Postgres-compatible offering; price step up; revisit at T3 if query patterns demand).
- Spanner (too far up the relational scale; not a fit for this workload).
- GKE / Kubernetes (overkill for the workload; Cloud Run handles autoscale).
- Migrating R2 to GCS (would add $0.12/GiB egress; explicitly don't).

## When to commit

After the document is complete and the cost numbers are sourced:

```
docs: GCP platform evaluation for Phase 3 deployment

Written platform recommendation: Cloud Run + Cloud SQL Enterprise + R2
(unchanged) + Secret Manager + Artifact Registry + GHA/WIF. Three-tier
cost projection: ~$52 / ~$240 / ~$880 per month at T1/T2/T3, with Cloud
SQL representing 70-85% of spend at every tier. Citations to GCP
pricing pages inline.
```

## What this skill doesn't do

- Doesn't provision anything. Provisioning is the Terraform sub-skills.
- Doesn't compare with non-GCP platforms (Railway, Fly, AWS App Runner). If the WBS asked for that, it's a separate skill (not yet written).
- Doesn't bind cost numbers. They're projections; actual costs depend on real traffic. Re-verify before any contract or budget commitment.

## Parent skill

Invoked from `replit-to-gcp` Step 1. The output document is referenced by every subsequent sub-skill (sizing decisions, IAM roles, etc.).
