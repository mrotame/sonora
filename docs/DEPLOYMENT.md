# Sonora - Deployment and Operations

> **Document purpose:** operational reference for how Sonora is built, deployed, and kept within free-tier limits. The decision itself is ADR 0011 in `TECH_STACK.md`.
>
> **Public repository:** this file must not contain account identifiers (project IDs or numbers, billing accounts, service account emails, e-mail addresses, service URLs). Those values live only in the GitHub repository variables and in the GCP console. Use placeholders such as `<PROJECT_ID>` here.

## 1. Overview

| Part | Host | Workflow | Trigger |
|---|---|---|---|
| Frontend (static `vite build` output) | GitHub Pages, served under `/<repo-lowercase>/` | `.github/workflows/deploy-frontend.yml` | Push to `main` outside backend-only paths, or manual run |
| Backend (`server/`, Docker) | Google Cloud Run, `us-central1` | `.github/workflows/deploy-backend.yml` | Push to `main` touching `server/`, `shared/`, `Dockerfile`, `.dockerignore`, `package*.json` or the workflow, or manual run |

Both workflows run `npm ci` and the tests before deploying. A commit that changes both sides runs both workflows in parallel.

The browser calls the backend cross-origin: the frontend build receives the backend origin as `VITE_API_BASE_URL`, and the backend returns CORS headers only for `ALLOWED_ORIGIN`.

## 2. GitHub repository configuration

Settings > Secrets and variables > Actions > **Variables** (not Secrets; the workflows read `vars.*`):

| Variable | Used by | Meaning |
|---|---|---|
| `GCP_PROJECT_ID` | backend | GCP project ID |
| `GCP_REGION` | backend | Cloud Run region (keep a free-tier US region) |
| `CLOUD_RUN_SERVICE` | backend | Cloud Run service name |
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | backend | `projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/<POOL>/providers/<PROVIDER>` |
| `GCP_SERVICE_ACCOUNT` | backend | Deployer service account email |
| `ALLOWED_ORIGIN` | backend | Frontend origin without path, e.g. `https://<owner>.github.io` |
| `BACKEND_URL` | frontend | Cloud Run service URL, no trailing slash; the frontend workflow fails early if empty |

Settings > Pages > Source must be **GitHub Actions**.

No secrets or keys are stored in GitHub: GitHub authenticates to GCP through Workload Identity Federation (OIDC), and the RapidAPI key lives in GCP Secret Manager (Section 3).

## 3. GCP setup (one-time, already done)

- APIs enabled: Cloud Run, Cloud Build, Artifact Registry, IAM Credentials, STS, Billing Budgets.
- Deployer service account roles: `run.admin`, `cloudbuild.builds.editor`, `storage.admin`, `artifactregistry.admin`, `iam.serviceAccountUser`, `serviceusage.serviceUsageConsumer`.
- Workload Identity pool and OIDC provider with issuer `https://token.actions.githubusercontent.com` and an attribute condition that accepts only this repository. The deployer service account grants `roles/iam.workloadIdentityUser` to that repository's principal set.
- The default Compute service account has `roles/cloudbuild.builds.builder`.
- A billing account is linked (required even within the free tier).
- **RapidAPI key (ADR 0012):** Secret Manager secret `rapidapi-key` holds the key of the "YouTube Audio Downloader & Converter" subscription. The Cloud Run runtime service account (default Compute service account) has `roles/secretmanager.secretAccessor` on it. The deploy workflow maps it to the `RAPIDAPI_KEY` environment variable (`rapidapi-key:latest`), so a new key version takes effect on the next deploy. A deploy fails if the secret does not exist. Local runs have no `RAPIDAPI_KEY` and use yt-dlp.

## 4. What a backend deploy does

1. Uploads the repository (filtered by `.dockerignore`) to the bucket `run-sources-<PROJECT_ID>-<REGION>`.
2. Cloud Build builds the `Dockerfile` (Node 24 slim + the latest yt-dlp release) and pushes the image to Artifact Registry `cloud-run-source-deploy/<SERVICE>`, tagged `latest`. Older images stay, untagged.
3. Cloud Run creates a new revision and routes 100% of traffic to it. Previous revisions stay inactive.

Service flags: `--allow-unauthenticated --max-instances=1 --cpu=1 --memory=512Mi --timeout=120 --cpu-boost`. Image environment: `HOST=0.0.0.0`, `TRUST_PROXY=1`, `NODE_ENV=production`; Cloud Run injects `PORT`.

Because yt-dlp is installed at build time, a redeploy (manual run of the backend workflow) is how yt-dlp is updated when YouTube breaks extraction.

## 5. Free-tier guards

Applied with `deploy/gcp-free-tier-guards.sh` (2026-11-16):

- Budget alert at a low amount with 50% and 100% thresholds. It only **notifies**; it does not stop anything.
- Artifact Registry cleanup policy: keep the 2 most recent images, delete others older than 1 day.
- Source bucket lifecycle: delete uploads older than 1 day.

Cleanup runs periodically (about daily), not at deploy time. Rollback is possible only to revisions whose image still exists.

Script notes: in this environment `gcloud billing budgets create` needed `--billing-project=<PROJECT_ID>`; rerunning the whole script creates a duplicate budget. Steps 2 and 3 need the repository and bucket, which exist only after the first deploy.

Do **not** add `--no-cpu-throttling` or `--min-instances` without a cost review: both bill CPU outside requests.

## 6. Known limitations (to validate)

- **CPU throttling:** Cloud Run throttles CPU outside requests; yt-dlp jobs run in the background after `POST /api/jobs` returns and may stall between polls.
- **Egress counter:** the monthly egress quota state is saved in `/tmp`, which is in memory on Cloud Run and is lost when the instance stops, so the counter does not really cap monthly egress.
- **Datacenter IP:** YouTube blocks yt-dlp on Cloud Run with its bot check (observed 2026-11-16, error `BLOCKED`). Production uses the RapidAPI provider instead (ADR 0012); yt-dlp is only the fallback when `RAPIDAPI_KEY` is empty.
- **RapidAPI free plans** (one key, one subscription per API on the same account), tried in this order:
  1. "YouTube to MP3 Converter API": 4 requests per month. One Sonora job uses 1 request.
  2. "YouTube Audio Downloader & Converter": 25 conversions and 300 requests per month, both hard limits (no overage billing). One Sonora job uses 1 conversion and about 3 to 10 requests.
  When a quota runs out the provider returns 429 and the backend falls back to the next one; if all fail the user sees the last error (`QUOTA_EXCEEDED` when the last quota is used up).
- **Client IP:** rate limiting uses the last `X-Forwarded-For` entry; its correctness behind the Cloud Run proxy is not verified.
- In-memory jobs are lost when the instance restarts (accepted, ADR 0008).

## 7. Useful commands

```sh
# Service URL, revisions, logs
gcloud run services describe <SERVICE> --project <PROJECT_ID> --region <REGION> --format="value(status.url)"
gcloud run revisions list --service <SERVICE> --project <PROJECT_ID> --region <REGION>
gcloud run services logs read <SERVICE> --project <PROJECT_ID> --region <REGION> --limit 50

# Roll back to a previous revision
gcloud run services update-traffic <SERVICE> --project <PROJECT_ID> --region <REGION> --to-revisions <REVISION>=100

# Stored images and builds
gcloud artifacts docker images list <REGION>-docker.pkg.dev/<PROJECT_ID>/cloud-run-source-deploy --include-tags
gcloud builds list --project <PROJECT_ID> --region <REGION> --limit 5

# Smoke test: an unknown job must return 404; the CORS preflight must return 204 with the allowed origin
curl -s -o /dev/null -w "%{http_code}\n" <BACKEND_URL>/api/jobs/unknown
curl -si -X OPTIONS <BACKEND_URL>/api/jobs -H "Origin: <ALLOWED_ORIGIN>" -H "Access-Control-Request-Method: POST"
```

On Windows Git Bash, prefix local builds that set `BASE_PATH` with `MSYS_NO_PATHCONV=1`, or the path is rewritten to a Windows path.

Local production build matching the Pages deploy:

```sh
BASE_PATH=/<repo-lowercase>/ VITE_API_BASE_URL=<BACKEND_URL> npm run build
```
