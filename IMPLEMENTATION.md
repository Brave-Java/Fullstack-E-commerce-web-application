# 🛠️ Implementation Guide — Purely E-Commerce

> **Purpose:** This is the single source of truth for all implementation work added on top of the original project.
> Any change — file added, modified, or deleted — **must be recorded here before it is committed**.
>
> **Rule:** Do not `git commit` anything without first updating the [Changelog](#-changelog) and [Pending Work](#5-pending-work) sections.

---

## 📑 Table of Contents

1. [Project Context](#1-project-context)
2. [Branch Strategy](#2-branch-strategy)
3. [Local Development Setup](#3-local-development-setup)
4. [Azure DevOps CI/CD Setup](#4-azure-devops-cicd-setup)
5. [Pending Work](#5-pending-work)
6. [Epic 637 - Local Validation Gate (Before ArgoCD)](#6-epic-637---local-validation-gate-before-argocd)
7. [Commit Workflow](#7-commit-workflow)
8. [Changelog](#-changelog)

---

## 1. Project Context

**Purely** is a cloud-first microservices e-commerce application. The original project targets **AWS EKS** with **GitHub Actions** CI/CD (`main` branch). This implementation layer adds:

- A full **local development stack** via Docker Compose (run everything with one command)
- An **Azure DevOps CI/CD pipeline** layer targeting **Azure Kubernetes Service (AKS)**
- A **multi-environment promotion flow**: `dev` → `test` → `prod`

The `main` branch is **never touched**. All new work lives in dedicated branches.

---

## 2. Branch Strategy

```
main   ─────────────────────────────────────  (original, read-only, never modified)
  │
  └── dev    ◄── all new work starts here  ✅ created
        │
        └── test   ◄── QA / integration     ⏳ pending
              │
              └── prod  ◄── live production  ⏳ pending
```

| Branch | Purpose | Deploys to |
|--------|---------|------------|
| `main` | Original project — read only | AWS EKS (original GitHub Actions) |
| `dev` | Active development | AKS `dev` namespace |
| `test` | QA / pre-production | AKS `test` namespace |
| `prod` | Live production | AKS `prod` namespace |

### Commands — create `test` and `prod` branches (pending)

```bash
# Create test from dev
git checkout dev
git checkout -b test
git push -u origin test

# Create prod from test
git checkout test
git checkout -b prod
git push -u origin prod
```

> ⚠️ Never push directly to `main`. Open a Pull Request if a fix must go there.

---

## 3. Local Development Setup

### New files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Starts the full stack locally with one command |
| `docker/nginx-dev.conf` | nginx proxy: `/api/*` → `api-gateway:8080` |
| `.env.example` | Template for secrets required by Docker Compose |

### Service port map

| Service | Host Port | URL |
|---------|-----------|-----|
| Frontend | 3000 | http://localhost:3000 |
| API Gateway | 8080 | http://localhost:8080 |
| Eureka Dashboard | 8761 | http://localhost:8761 |
| Auth Service | 9030 | http://localhost:9030 |
| Category Service | 9000 | http://localhost:9000 |
| Product Service | 9010 | http://localhost:9010 |
| Notification Service | 9020 | http://localhost:9020 |
| User Service | 9050 | http://localhost:9050 |
| Cart Service | 9060 | http://localhost:9060 |
| Order Service | 9070 | http://localhost:9070 |

### Commands — first-time local setup

```bash
# 1. Switch to dev branch
git checkout dev

# 2. Create your .env file from the template
cp .env.example .env

# 3. Edit .env and fill in your credentials
#    SPRING_MAIL_USERNAME=your-email@gmail.com
#    SPRING_MAIL_PASSWORD=your-gmail-app-password

# 4. Build and start the full stack
docker compose up --build

# 5. Verify all services appear in Eureka
#    → open http://localhost:8761

# 6. Open the application
#    → open http://localhost:3000
```

### Commands — daily usage

```bash
# Start without rebuilding images
docker compose up

# Rebuild and start a single service after a code change
docker compose up --build auth-service

# Run in the background
docker compose up -d

# Stop all containers (keep data)
docker compose down

# Stop and wipe all data (MongoDB volume full reset)
docker compose down -v

# Follow logs for one service
docker compose logs -f auth-service

# List running containers and their status
docker compose ps
```

### How the nginx proxy works locally

In Kubernetes the frontend and API Gateway are separate pods behind an Ingress. Locally they are separate Docker containers.

`docker/nginx-dev.conf` is volume-mounted into the frontend container, overriding its default nginx config. It adds a `proxy_pass` so any browser request to `/api/*` is forwarded to `api-gateway:8080` inside the Docker network — replicating Kubernetes ingress behaviour exactly.

```
Browser → localhost/api/... → nginx (frontend container) → api-gateway:8080/api/...
```

---

## 4. Azure DevOps CI/CD Setup

### New files

```
azure-pipelines/
├── templates/
│   ├── java-build.yml          ← reusable: Maven build + unit tests
│   ├── docker-build-push.yml   ← reusable: Docker build + push to ACR
│   └── helm-deploy.yml         ← reusable: Helm upgrade/install to AKS
├── service-registry.yml
├── api-gateway.yml
├── auth-service.yml
├── user-service.yml
├── category-service.yml
├── product-service.yml
├── cart-service.yml
├── order-service.yml
├── notification-service.yml
└── web-app.yml
```

### Pipeline flow (per service)

```
push to dev / test / prod
         │
         ▼
  Stage 1 ── Build & Test (Maven / npm)
         │
         ▼
  Stage 2 ── Docker build + push to ACR
             tags: $(Build.BuildId)  and  latest-<branch>
         │
         ▼
  Stage 3 ── Helm upgrade --install on AKS
             namespace = branch name (dev | test | prod)
             uses values.yaml + values-<env>.yaml   ← pending
             secrets injected from variable group
```

### Step-by-step Azure setup

#### Step 1 — Create Azure resources

```bash
# Login
az login

# Resource group
az group create --name purely-rg --location eastus

# Azure Container Registry
az acr create --resource-group purely-rg --name purelyacr --sku Basic

# AKS cluster (attach ACR so it can pull images without extra auth)
az aks create \
  --resource-group purely-rg \
  --name purely-aks \
  --node-count 2 \
  --attach-acr purelyacr \
  --generate-ssh-keys

# Download kubeconfig
az aks get-credentials --resource-group purely-rg --name purely-aks
```

#### Step 2 — Create Kubernetes namespaces

```bash
kubectl create namespace dev
kubectl create namespace test
kubectl create namespace prod
```

#### Step 3 — Create variable groups in Azure DevOps

Go to **Azure DevOps → Pipelines → Library → + Variable group**

Create three groups: `purely-dev`, `purely-test`, `purely-prod`

Each group must have these variables:

| Variable | Example value | Mark as secret |
|----------|--------------|:-:|
| `ACR_REGISTRY` | `purelyacr.azurecr.io` | |
| `ACR_SERVICE_CONNECTION` | `acr-service-connection` | |
| `AKS_CLUSTER_NAME` | `purely-aks` | |
| `AKS_RESOURCE_GROUP` | `purely-rg` | |
| `AZURE_SERVICE_CONNECTION` | `azure-service-connection` | |
| `SPRING_DATA_MONGODB_URI_AUTH` | `mongodb+srv://...` | ✅ |
| `SPRING_DATA_MONGODB_URI_CATEGORY` | `mongodb+srv://...` | ✅ |
| `SPRING_DATA_MONGODB_URI_PRODUCT` | `mongodb+srv://...` | ✅ |
| `SPRING_DATA_MONGODB_URI_CART` | `mongodb+srv://...` | ✅ |
| `SPRING_DATA_MONGODB_URI_ORDER` | `mongodb+srv://...` | ✅ |
| `SPRING_MAIL_USERNAME` | `your@email.com` | ✅ |
| `SPRING_MAIL_PASSWORD` | `app-password` | ✅ |

#### Step 4 — Create service connections in Azure DevOps

Go to **Project Settings → Service connections → New service connection**

1. **Docker Registry** → Azure Container Registry → name it `acr-service-connection`
2. **Azure Resource Manager** → your subscription → name it `azure-service-connection`

#### Step 5 — Register pipelines in Azure DevOps

Go to **Pipelines → New Pipeline → Existing YAML file** and register each one:

| Pipeline name | YAML path |
|--------------|-----------|
| `purely-service-registry` | `azure-pipelines/service-registry.yml` |
| `purely-api-gateway` | `azure-pipelines/api-gateway.yml` |
| `purely-auth-service` | `azure-pipelines/auth-service.yml` |
| `purely-user-service` | `azure-pipelines/user-service.yml` |
| `purely-category-service` | `azure-pipelines/category-service.yml` |
| `purely-product-service` | `azure-pipelines/product-service.yml` |
| `purely-cart-service` | `azure-pipelines/cart-service.yml` |
| `purely-order-service` | `azure-pipelines/order-service.yml` |
| `purely-notification-service` | `azure-pipelines/notification-service.yml` |
| `purely-web-app` | `azure-pipelines/web-app.yml` |

> Save each pipeline but **do not run** until variable groups exist.

#### Step 6 — Trigger first deployment

```bash
git checkout dev
git add .
git commit -m "ci: trigger initial dev deployment"
git push -u origin dev
```

---

## 5. Pending Work

> Update the status column here whenever a task is started or completed.

| # | Task | Branch | Status |
|---|------|--------|--------|
| 1 | Create `test` branch from `dev` | — | ⬜ Not started |
| 2 | Create `prod` branch from `test` | — | ⬜ Not started |
| 3 | Add `values-dev.yaml`, `values-test.yaml`, `values-prod.yaml` to every Helm chart (10 charts × 3 files = 30 files) | `dev` | ⬜ Not started |
| 4 | Add branch protection rules in Azure DevOps (require PR to merge into `test` and `prod`) | Azure DevOps | ⬜ Not started |
| 5 | Add manual approval gate on the `prod` environment in Azure DevOps | Azure DevOps | ⬜ Not started |
| 6 | Load sample data into local MongoDB on first `docker compose up` | `dev` | ⬜ Not started |
| 7 | End-to-end smoke test locally (place an order, verify email) | Local | ⬜ Not started |
| 8 | Epic 637.1 - Local preflight checks complete | Local | ⬜ Not started |
| 9 | Epic 637.2 - Clean Docker restart baseline complete | Local | ⬜ Not started |
| 10 | Epic 637.3 - All containers healthy in Docker and Eureka | Local | ⬜ Not started |
| 11 | Epic 637.4 - Service health checks passed for all services | Local | ⬜ Not started |
| 12 | Epic 637.5 - Local E2E order and notification verification complete | Local | ⬜ Not started |
| 13 | Epic 637.6 - Test evidence captured and archived | Local | ⬜ Not started |

---

## 6. Epic 637 - Local Validation Gate (Before ArgoCD)

> This epic is the hard gate. Do not start any ArgoCD epic/task until all Epic 637 tasks below are completed and marked done.

### Goal

Validate the full application stack locally, including service startup, discovery, health, routing, and end-to-end order flow.

### Definition of Done

- All Docker services are up and stable.
- Eureka shows all expected backend services as registered.
- Health checks pass for API Gateway + all backend services.
- Frontend is reachable and can call backend through API Gateway.
- End-to-end order flow is verified and notification service behavior is confirmed.
- Evidence (logs + command outputs + screenshots) is saved.

### Service Testing Runbook

This is the practical test flow to run from the terminal and Postman before any new development starts.

#### Architecture Under Test

```text
Postman / browser / curl
  -> http://localhost:8080/api/*
    -> API Gateway (port 8080)
      -> StripPrefix=2
        -> Target service
```

#### Services In Scope

- Service Registry (Eureka) - `http://localhost:8761`
- Auth Service - `http://localhost:9030` via gateway route `/api/auth-service`
- Category Service - `http://localhost:9000` via gateway route `/api/category-service`
- Product Service - `http://localhost:9010` via gateway route `/api/product-service`
- User Service - `http://localhost:9050` via gateway route `/api/user-service`
- Cart Service - `http://localhost:9060` via gateway route `/api/cart-service`
- Order Service - `http://localhost:9070` via gateway route `/api/order-service`
- Notification Service - `http://localhost:9020` via gateway route `/api/notification-service`
- Frontend - `http://localhost:3000`

#### Postman Setup

Create one Postman environment with these variables:

- `frontend` = `http://localhost:3000`
- `gateway` = `http://localhost:8080`
- `eureka` = `http://localhost:8761`
- `auth` = `http://localhost:9030`
- `category` = `http://localhost:9000`
- `product` = `http://localhost:9010`
- `notification` = `http://localhost:9020`
- `user` = `http://localhost:9050`
- `cart` = `http://localhost:9060`
- `order` = `http://localhost:9070`
- `token` = empty initially

#### Terminal Steps

1. Confirm the stack is up:

```bash
docker compose ps
```

2. Confirm all backend services are healthy:

```bash
curl -fsS http://localhost:8080/actuator/health
curl -fsS http://localhost:9030/actuator/health
curl -fsS http://localhost:9000/actuator/health
curl -fsS http://localhost:9010/actuator/health
curl -fsS http://localhost:9020/actuator/health
curl -fsS http://localhost:9050/actuator/health
curl -fsS http://localhost:9060/actuator/health
curl -fsS http://localhost:9070/actuator/health
```

3. Verify Eureka registration:

```text
http://localhost:8761
```

#### Postman Steps by Service

1. Frontend and gateway sanity
- Open `{{frontend}}` in a browser and confirm the UI loads.
- Add a simple Postman request for `GET {{gateway}}/actuator/health` and expect HTTP 200.

2. Eureka dashboard check
- Add a Postman request for `GET {{eureka}}` and confirm the dashboard HTML loads.

3. Auth service
- Create `POST {{gateway}}/auth/signin` and `POST {{gateway}}/auth/signup` requests.
- Save the returned JWT into `{{token}}`.

4. Category and product services
- Create `GET {{gateway}}/category-service/**` and `GET {{gateway}}/product-service/**` requests for the read-only endpoints used by the UI.
- If the exact path is unknown, read it from the browser network tab or from the service OpenAPI docs.

5. User, cart, and order services
- Create authenticated requests with header `Authorization: Bearer {{token}}`.
- Validate the user profile, cart operations, and order creation/retrieval endpoints through the gateway.

6. Notification service
- After placing an order, inspect `notification-service` logs and verify the email flow was triggered.

#### Definition of Smart Testing

Yes, this is a smart way to test the system if you use Postman as a service-level validation layer, not as the only check.

- Postman is good for request/response verification, auth flows, headers, tokens, and regressions.
- `curl` is better for fast health checks and CI-friendly smoke tests.
- Eureka and container checks confirm startup and service discovery.
- Browser testing is still needed for the real frontend flow.

The strongest approach is to combine all four: Docker status, Eureka, curl health checks, and Postman functional tests.

### Azure DevOps Backlog and Sprint Setup (CLI)

Use this once to create Epic 637 and all child tasks, then assign them to a sprint.

```bash
# 1) Login (required)
az login

# 2) Set defaults (replace placeholders)
az devops configure --defaults organization=https://dev.azure.com/<your-org> project="<your-project>"

# 3) Discover team and sprint path (optional helper)
az boards iteration team list --team "<your-team>" -o table

# 4) Create the epic
EPIC_ID=$(az boards work-item create \
  --type Epic \
  --title "Epic 637: Local Validation Gate Before ArgoCD" \
  --description "Validate all local services, health, discovery, and E2E flow before starting any ArgoCD epic/tasks." \
  --query id -o tsv)

echo "Created Epic ID: $EPIC_ID"

# 5) Create child Product Backlog Items (or use type Task if your process requires)
PBI1=$(az boards work-item create --type "Product Backlog Item" --title "637.1 Local preflight checks" --query id -o tsv)
PBI2=$(az boards work-item create --type "Product Backlog Item" --title "637.2 Clean Docker restart baseline" --query id -o tsv)
PBI3=$(az boards work-item create --type "Product Backlog Item" --title "637.3 Container and Eureka validation" --query id -o tsv)
PBI4=$(az boards work-item create --type "Product Backlog Item" --title "637.4 Service health matrix validation" --query id -o tsv)
PBI5=$(az boards work-item create --type "Product Backlog Item" --title "637.5 E2E order and notification smoke test" --query id -o tsv)
PBI6=$(az boards work-item create --type "Product Backlog Item" --title "637.6 Evidence capture and sign-off" --query id -o tsv)

# 6) Link children to epic
for CHILD in "$PBI1" "$PBI2" "$PBI3" "$PBI4" "$PBI5" "$PBI6"; do
  az boards work-item relation add --id "$EPIC_ID" --relation-type Child --target-id "$CHILD"
done

# 7) Assign all items to sprint (replace iteration path)
ITERATION_PATH="<your-project>\\<team-or-program>\\<sprint-name>"

for ITEM in "$EPIC_ID" "$PBI1" "$PBI2" "$PBI3" "$PBI4" "$PBI5" "$PBI6"; do
  az boards work-item update --id "$ITEM" --fields "System.IterationPath=$ITERATION_PATH"
done

echo "Epic and child tasks added to backlog and sprint successfully."
```

Notes:

- If your process template does not support `Product Backlog Item`, use `Task` or `User Story`.
- To target a different team backlog, include `--team "<your-team>"` in create/list commands.
- Replace the iteration path with one returned by `az boards iteration team list`.

### Task 637.1 - Local Preflight

Run from repository root:

```bash
docker context ls
docker compose version
docker --version
cp -n .env.example .env
```

Then edit `.env` and set:

```env
SPRING_MAIL_USERNAME=your_email@gmail.com
SPRING_MAIL_PASSWORD=your_app_password
```

Pass criteria:

- Docker context is `default`.
- Docker Compose is available.
- `.env` exists with required mail variables.

### Task 637.2 - Clean Docker Baseline

If you have stale/locked containers, reset Docker first:

```bash
docker compose down -v --remove-orphans
```

If `permission denied` or stuck stop/kill errors appear:

```bash
sudo systemctl restart docker
```

Then return to repository and start clean:

```bash
docker compose up -d --build
```

Pass criteria:

- `docker compose up` completes without port-allocation errors.

### Task 637.3 - Container + Discovery Validation

Check container states:

```bash
docker compose ps
```

Open Eureka dashboard:

```text
http://localhost:8761
```

Expected services in Eureka:

- API-GATEWAY
- AUTH-SERVICE
- USER-SERVICE
- CATEGORY-SERVICE
- PRODUCT-SERVICE
- CART-SERVICE
- ORDER-SERVICE
- NOTIFICATION-SERVICE

Pass criteria:

- All expected services are `UP` in Eureka.
- No service is continuously restarting.

### Task 637.4 - Health Check Matrix

Run these checks:

```bash
curl -fsS http://localhost:8080/actuator/health
curl -fsS http://localhost:9030/actuator/health
curl -fsS http://localhost:9050/actuator/health
curl -fsS http://localhost:9000/actuator/health
curl -fsS http://localhost:9010/actuator/health
curl -fsS http://localhost:9060/actuator/health
curl -fsS http://localhost:9070/actuator/health
curl -fsS http://localhost:9020/actuator/health
curl -I http://localhost:3000
```

If any check fails, inspect logs:

```bash
docker compose logs -f <service-name>
```

Pass criteria:

- Every service health endpoint returns HTTP 200.
- Frontend returns an HTTP success/redirect response.

### Task 637.5 - E2E Smoke Test (Order + Notification)

Manual browser flow:

1. Open `http://localhost:3000`.
2. Sign up/sign in.
3. Browse categories/products.
4. Add product(s) to cart.
5. Place an order.

Verify notification service activity:

```bash
docker compose logs --tail=200 notification-service
```

Pass criteria:

- Order is created successfully.
- Notification service logs show a send attempt/success for order flow.

### Task 637.6 - Evidence Collection and Sign-off

Capture and save evidence:

```bash
docker compose ps
docker compose logs --tail=200 service-registry
docker compose logs --tail=200 api-gateway
docker compose logs --tail=200 auth-service
docker compose logs --tail=200 user-service
docker compose logs --tail=200 category-service
docker compose logs --tail=200 product-service
docker compose logs --tail=200 cart-service
docker compose logs --tail=200 order-service
docker compose logs --tail=200 notification-service
docker compose logs --tail=200 frontend
```

Sign-off checklist:

- [ ] All Epic 637 tasks marked complete in Pending Work.
- [ ] Changelog updated with test date and outcomes.
- [ ] Explicit statement added: "ArgoCD epic can proceed."

---

## 7. Commit Workflow

### Before every commit

1. Update [Changelog](#-changelog): add a new entry with today's date, what changed and why, and which files were affected.
2. Update [Pending Work](#5-pending-work): mark completed items, add new ones.
3. Stage `IMPLEMENTATION.md` together with all other changed files.

### Commands

```bash
# Stage everything including this document
git add IMPLEMENTATION.md <other-files>

# Commit with a descriptive message
git commit -m "type(scope): short description"

# Push to remote
git push
```

### Commit message convention

```
type(scope): short description

Types:   feat | fix | chore | ci | docs | refactor | test
Scopes:  auth-service | docker | helm | pipeline | all | ...

Examples:
  feat(cart-service): add item quantity validation
  ci(auth-service): fix ACR push step
  chore(helm): add values-dev.yaml for all charts
  docs(implementation): update pending work status
```

---

## 📋 Changelog

---

### 2026-07-08 - Add Epic 637 local validation gate plan

**Branch:** `dev`

#### Summary

Added a complete step-by-step Epic 637 execution guide for validating all services locally before any ArgoCD work begins. The guide includes preflight checks, clean Docker restart steps, service and Eureka validation, health check matrix, E2E order flow verification, evidence/sign-off criteria, and Azure DevOps CLI commands to create the epic/tasks and assign them to a sprint backlog.

#### Files modified

| File | Change |
|------|--------|
| `IMPLEMENTATION.md` | Added Epic 637 section, expanded pending tasks, updated TOC section numbering |

---

### 2026-06-27 — Fix broken Docker base image + port conflict

**Branch:** `dev`

#### Summary

`openjdk:21-jdk-slim` does not exist on Docker Hub (the `openjdk` official image was deprecated and this tag was never published). Replaced with `eclipse-temurin:21-jdk-alpine` in all 9 service Dockerfiles. Frontend port changed from `80` to `3000` to avoid conflict with a system-level pgadmin/nginx process occupying port 80 on the host.

#### Files modified

| File | Change |
|------|--------|
| `microservice-backend/*/Dockerfile` (all 9) | `FROM openjdk:21-jdk-slim` → `FROM eclipse-temurin:21-jdk-alpine` |
| `docker-compose.yml` | Frontend port mapping `80:80` → `3000:80` |
| `IMPLEMENTATION.md` | Updated port map and local setup commands to reflect port 3000 |

---

### 2026-06-27 — Initial implementation setup

**Branch:** `dev` | **Commit:** `42bf415`

#### Summary

The original project only supports local runs via `mvn spring-boot:run` per service and AWS EKS deployment via GitHub Actions. This commit adds Docker Compose for easy local development and Azure DevOps pipelines for a multi-environment cloud deployment flow, without touching `main`.

#### Files added

| File | Description |
|------|-------------|
| `IMPLEMENTATION.md` | This document |
| `docker-compose.yml` | Full local stack: MongoDB + 9 microservices + frontend |
| `docker/nginx-dev.conf` | nginx reverse proxy: local frontend → API Gateway |
| `.env.example` | Secret template (email credentials) |
| `azure-pipelines/templates/java-build.yml` | Reusable Maven build + test template |
| `azure-pipelines/templates/docker-build-push.yml` | Reusable Docker build + ACR push template |
| `azure-pipelines/templates/helm-deploy.yml` | Reusable Helm deploy to AKS template |
| `azure-pipelines/service-registry.yml` | Pipeline: Service Registry |
| `azure-pipelines/api-gateway.yml` | Pipeline: API Gateway |
| `azure-pipelines/auth-service.yml` | Pipeline: Auth Service |
| `azure-pipelines/user-service.yml` | Pipeline: User Service |
| `azure-pipelines/category-service.yml` | Pipeline: Category Service |
| `azure-pipelines/product-service.yml` | Pipeline: Product Service |
| `azure-pipelines/cart-service.yml` | Pipeline: Cart Service |
| `azure-pipelines/order-service.yml` | Pipeline: Order Service |
| `azure-pipelines/notification-service.yml` | Pipeline: Notification Service |
| `azure-pipelines/web-app.yml` | Pipeline: React Frontend |

#### Files modified

| File | Description |
|------|-------------|
| `README.md` | Reformatted for readability — no content removed |

#### Pending items identified

- Helm per-environment values files (Pending Work #3)
- `test` and `prod` branches (Pending Work #1 and #2)
- Azure DevOps approval gates (Pending Work #4 and #5)
