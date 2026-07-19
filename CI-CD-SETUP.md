# CI/CD Setup Log

## Docker Hub Registry Configuration
**Date:** 2026-07-18  
**Branch:** dev  
**Commit:** b96c32f

### What was configured
All GitHub Actions workflows were migrated from Azure Container Registry (ACR) to Docker Hub.

### Required GitHub Repository Secrets
Go to: `https://github.com/Brave-Java/Fullstack-E-commerce-web-application/settings/secrets/actions`

| Secret Name | Description | Where to get it |
|---|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username | hub.docker.com → Account Settings → General |
| `DOCKERHUB_TOKEN` | Docker Hub personal access token | hub.docker.com → Account Settings → Personal access tokens |

> **Status:** Secrets configured on 2026-07-18 ✓

### Image naming convention
Images are pushed as:
- `<DOCKERHUB_USERNAME>/<service-name>:<env>-v1` → `<env>-v2` → `<env>-v3` … (consecutive per environment)
- `<DOCKERHUB_USERNAME>/<service-name>:latest-<branch>` (branch alias)

The `docker-push` composite action queries the Docker Hub API on each run to find the highest existing `<env>-vN` tag and auto-increments it. First push produces `dev-v1`, next `dev-v2`, and so on. Tags are environment-scoped, so `dev-v3` and `test-v1` are independent sequences.

### Workflows that use Docker Hub
All 10 new GitHub Actions workflows (in `.github/workflows/`):
- `api-gateway.yml`
- `auth-service.yml`
- `cart-service.yml`
- `category-service.yml`
- `notification-service.yml`
- `order-service.yml`
- `product-service.yml`
- `service-registry.yml`
- `user-service.yml`
- `web-app.yml` ← also has `workflow_dispatch` for manual runs

### Composite actions modified
- `.github/actions/docker-push/action.yml` — logs in to Docker Hub, builds and pushes image
- `.github/actions/helm-deploy/action.yml` — uses `dockerhub-username` to resolve image path in Helm

### Deploy stage secrets (per environment)
Create GitHub environments named `dev`, `test`, `prod` and add:

| Secret Name | Description |
|---|---|
| `AZURE_CREDENTIALS` | JSON from `az ad sp create-for-rbac --sdk-auth` |
| `AKS_CLUSTER_NAME` | Your AKS cluster name |
| `AKS_RESOURCE_GROUP` | Resource group containing the AKS cluster |

---

## Testing the web-app workflow manually
1. Go to: `https://github.com/Brave-Java/Fullstack-E-commerce-web-application/actions/workflows/web-app.yml`
2. Click **Run workflow** → select branch `dev` → **Run workflow**
3. The `build` job (npm ci + npm run build) runs without secrets
4. The `package` job (Docker push) requires `DOCKERHUB_USERNAME` + `DOCKERHUB_TOKEN`
5. The `deploy-dev` job requires the environment secrets above

Or from the terminal (requires `gh` CLI authenticated):
```bash
gh workflow run web-app.yml --repo Brave-Java/Fullstack-E-commerce-web-application --ref dev
```

---

## Workflow strategy per service in `dev`

### Trigger types

| Trigger | When it fires | Use case |
|---|---|---|
| `push` | Automatically when matching files are pushed | Normal dev flow — change code, push, CI runs |
| `workflow_dispatch` | Manually via GitHub UI or `gh` CLI | Redeploy without code change, test the pipeline |

### Path-based isolation
Each workflow only fires for its own service files. Push to `frontend/` → only `web-app.yml` runs. Push to `microservice-backend/auth-service/` → only `auth-service.yml` runs. Multiple services changed in one commit → multiple workflows run in parallel.

| Service | Workflow file | Watch path |
|---|---|---|
| Frontend (React) | `web-app.yml` | `frontend/**` |
| API Gateway | `api-gateway.yml` | `microservice-backend/api-gateway/**` |
| Auth Service | `auth-service.yml` | `microservice-backend/auth-service/**` |
| Cart Service | `cart-service.yml` | `microservice-backend/cart-service/**` |
| Category Service | `category-service.yml` | `microservice-backend/category-service/**` |
| Notification Service | `notification-service.yml` | `microservice-backend/notification-service/**` |
| Order Service | `order-service.yml` | `microservice-backend/order-service/**` |
| Product Service | `product-service.yml` | `microservice-backend/product-service/**` |
| Service Registry | `service-registry.yml` | `microservice-backend/service-registry/**` |
| User Service | `user-service.yml` | `microservice-backend/user-service/**` |

### Manual trigger commands (gh CLI)
```bash
# Trigger a single service
gh workflow run web-app.yml --repo Brave-Java/Fullstack-E-commerce-web-application --ref dev
gh workflow run auth-service.yml --repo Brave-Java/Fullstack-E-commerce-web-application --ref dev

# Watch a run live
gh run watch <run-id> --repo Brave-Java/Fullstack-E-commerce-web-application

# List recent runs for a specific workflow
gh run list --workflow=web-app.yml --repo Brave-Java/Fullstack-E-commerce-web-application --limit 5

# View failure logs
gh run view <run-id> --repo Brave-Java/Fullstack-E-commerce-web-application --log-failed
```

### Known gotchas fixed
- `npm ci` / `npm install` inside Docker requires `--legacy-peer-deps` (vite v8 + @vitejs/plugin-react peer conflict)
- `secrets` context is not allowed in job-level `if:` conditions — use plain `github.ref_name == 'dev'`
- `curl | jq` pipeline in `docker-push` action needs `|| true` guards so `set -o pipefail` doesn't kill the step on empty/new repos

---

## Local development

### Run the frontend only
```bash
cd frontend
npm install --legacy-peer-deps
VITE_API_BASE_URL=http://localhost:8080 npx vite --host 0.0.0.0 --port 5173
# Open http://localhost:5173
```

### Run the full stack locally (Docker Compose)
```bash
docker-compose up --build
# Frontend: http://localhost:3000 (or as configured)
# API Gateway: http://localhost:8080
```

---

## Alternative deployment options (no AKS required)

| Platform | Cost | Effort | Notes |
|---|---|---|---|
| **Docker Compose** (local) | Free | Low | Already in repo — fastest full-stack test |
| **Minikube** (local K8s) | Free | Medium | Same Helm charts as prod, needs 4GB+ RAM |
| **k3d / k3s** (local K8s) | Free | Low | Lighter than Minikube |
| **Render.com** | Free tier | Low | Deploy each service as a Web Service from Docker Hub image |
| **Railway.app** | Free tier | Low | Connect repo, auto-deploys per service |
| **Fly.io** | Free tier | Low | `fly deploy` per service, Dockerfile-based |
| **Azure AKS** (prod) | Paid | High | Full Terraform + Helm setup in `terraform/` and `helm-charts/` |
