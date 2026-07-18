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
- `<DOCKERHUB_USERNAME>/<service-name>:<github.run_id>` (versioned)
- `<DOCKERHUB_USERNAME>/<service-name>:latest-<branch>` (branch alias)

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
