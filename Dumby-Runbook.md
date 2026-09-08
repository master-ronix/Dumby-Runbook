# Dumby — Local Runbook (Docker · Kubernetes · Terraform)

> Generated from the actual files in `Dumby-Suite-main.zip`. Every command,
> path, port, and env var below is taken from the repo's own Dockerfile,
> docker-compose files, k8s manifests, Terraform modules, and `.env.example`.
>
> **Why a runbook and not a live run?** The sandbox this was generated in has
> no Docker daemon, kubectl, terraform, pnpm, or Node 22 — and no way to install
> them (non-root, no apt). So this is a verified, copy-paste-ready guide for you
> to run on your own machine where those tools exist.

---

## 0. Prerequisites (install once on your machine)

| Tool | Min version | Why |
|------|-------------|-----|
| Docker + Docker Compose | Docker 24+ / Compose v2 | Build image, run Postgres + Redis + app locally |
| Node.js | 22+ (see `.nvmrc`) | Runtime engine enforced by `package.json` |
| pnpm | 9+ | Workspace package manager (corepack can provide it) |
| kubectl + kustomize | recent | Apply the k8s manifests |
| kind / minikube / k3d | any | A local cluster to apply manifests into |
| terraform | 1.5+ | Provision AWS RDS / ElastiCache / EKS (only for the cloud path) |
| AWS CLI + credentials | configured | Terraform `aws` provider (~> 5.0) needs auth |

The Docker path needs only Docker + (optionally) Node/pnpm for local dev.
The Kubernetes path adds kubectl + kustomize + a local cluster.
The Terraform path is AWS-only and adds terraform + AWS creds.

---

## 1. Prepare the repo

```bash
unzip Dumby-Suite-main.zip
cd Dumby-Suite-main
```

The build target lives at `apps/main/` (`@workspace/dumby`).
The Dockerfile build context is the **workspace root** (`Dumby-Suite-main/`), not `apps/main/`.

---

## 2. Path A — Docker Compose (recommended local run)

This is the fastest path. It brings up Postgres 16, Redis 7, the bot app, and
the Swastik deployment sidecar, wired together on a private network.

### 2.1 Create the env file

```bash
cd apps/main
cp .env.example .env
```

Defaults are safe for local dev: `DISCORD_ADAPTER=mock` means no real Discord
token is required (uses `MockGatewayAdapter`), Postgres/Redis use the dev
credentials baked into `docker-compose.yml`.

### 2.2 Build and start everything

There are two compose files. The canonical one is `apps/main/docker-compose.yml`.
`apps/main/docker/docker-compose.yml` is just a wrapper that includes it.

```bash
# from apps/main/
docker compose up -d --build
```

This starts four services:

| Service | Image | Ports | Notes |
|---------|-------|-------|-------|
| `postgres` | `postgres:16-alpine` | `127.0.0.1:5432` | user/db/pass = `dumby` / `dumby` / `dumby_dev_password` |
| `redis` | `redis:7-alpine` | `127.0.0.1:6379` | healthchecked via `redis-cli ping` |
| `app` (the bot) | built from `apps/main/Dockerfile` | `8080` (API), `8081` (health), `8082` (metrics) | builds from workspace root context |
| `swastik` | same image, different entry | `127.0.0.1:8085` | runs `node dist/entry/swastik.js` |

### 2.3 What the Dockerfile does

`apps/main/Dockerfile` (build context = repo root, run as `docker build -f apps/main/docker/Dockerfile .`):

1. `base` — `node:22-slim`, enables corepack + pnpm.
2. `deps` — `pnpm install --frozen-lockfile --prod` (runtime deps only, cached store).
3. `builder` — full install, copies `src/` + `scripts/`, runs `typecheck` then `tsc` build.
4. `runtime` — copies `node_modules` + `dist`, `EXPOSE 8080 8081 8082`, runs as `USER node`, `CMD node dist/entry/main.js`.

### 2.4 Run migrations + verify health

```bash
# run migrations inside the app container
docker compose exec app pnpm migrate

# liveness probe used by the compose healthcheck
curl http://localhost:8081/health/liveness

# swastik sidecar
curl http://localhost:8085/swastik/status
```

### 2.5 Stop / clean

```bash
docker compose down              # keep volumes
docker compose down -v          # wipe postgres + swastik volumes
```

### 2.6 Local dev without Docker (optional)

If you want hot reload on your host instead of inside Docker:

```bash
# from repo root
corepack enable
pnpm install
pnpm --filter @workspace/dumby dev      # tsx watch src/bootstrap/bootstrap.ts
```

You'll still need Postgres and Redis reachable — either point `.env` at
`localhost:5432` / `localhost:6379` (run just `postgres` + `redis` from compose),
or `docker compose up -d postgres redis` then `pnpm dev`.

---

## 3. Path B — Kubernetes (local cluster with kustomize)

The repo ships a full kustomize base + staging/production overlays under
`apps/main/deploy/k8s/`. The manifests assume an image named `dumby:latest`.

### 3.1 Build and load the image into your local cluster

```bash
# from repo root
docker build -f apps/main/docker/Dockerfile -t dumby:latest .

# kind
kind load docker-image dumby:latest

# or minikube
# minikube image load dumby:latest

# or k3d
# k3d image import dumby:latest
```

### 3.2 Create the namespace + secrets

The base manifests read env from a ConfigMap (`dumby-config`, already defined)
and a Secret (`dumby-secrets`) which is **not** in the repo — you must create it:

```bash
kubectl create namespace dumby-staging   # or dumby-production

kubectl -n dumby-staging create secret generic dumby-secrets \
  --from-literal=BOT_TOKEN='' \
  --from-literal=DATABASE_URL='postgres://dumby:dumby_dev_password@postgres:5432/dumby' \
  --from-literal=REDIS_URL='redis://redis:6379/0' \
  --from-literal=QUEUE_REDIS_URL='redis://redis:6379/1' \
  --from-literal=INTERNAL_API_KEY='dev_internal_api_key_change_me' \
  --from-literal=JWT_SIGNING_KEY='dev_jwt_signing_key_change_me_32chars_min' \
  --from-literal=ENCRYPTION_KEY_CURRENT='dev_encryption_key_current_32_bytes!'
```

> Note: the base `configmap.yaml` sets `DISCORD_ADAPTER: live` and ports
> 3000/9090/9091, which differ from the Docker compose defaults (8080/8081/8082).
> The k8s path is the "production-shaped" config.

### 3.3 Provide Postgres + Redis

The k8s manifests do **not** include a Postgres or Redis deployment — they
expect managed/external services. For a local cluster, install them quickly:

```bash
# Bitnami charts (or use the same compose containers port-forwarded)
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install postgres bitnami/postgresql --namespace dumby-staging \
  --set auth.postgresPassword=dumby_dev_password \
  --set auth.postgresDatabase=dumby
helm install redis bitnami/redis --namespace dumby-staging
```

Then point the `DATABASE_URL` / `REDIS_URL` secret values at the in-cluster
service DNS names (`postgres.dumby-staging.svc.cluster.local:5432`, etc.).

### 3.4 Apply the manifests

```bash
cd apps/main

# staging (1 replica, NODE_ENV=staging, debug logs)
kubectl apply -k deploy/k8s/overlays/staging

# or production (2 replicas, NODE_ENV=production, warn logs)
kubectl apply -k deploy/k8s/overlays/production
```

### 3.5 What gets created (base layer)

- `Deployment/dumby` — 2 replicas, rolling update (maxSurge 1 / maxUnavailable 0),
  `terminationGracePeriodSeconds: 190`, runAsNonRoot uid 1000.
  Three probes on the health port: `startup` → `/health/startup`,
  `liveness` → `/health/liveness`, `readiness` → `/health/readiness`.
  Mounts two PVCs (`swastik-staging`, `swastik-live`).
- `Service/dumby` — ClusterIP, ports 3000/9090/9091.
- `ConfigMap/dumby-config` — non-secret runtime config (see file for full list).
- `HPA/dumby` — 2–10 replicas on CPU 70% / memory 80%.
- `ServiceAccount/dumby` — `automountServiceAccountToken: false`.
- Two `PersistentVolumeClaim`s — `dumby-swastik-staging`, `dumby-swastik-live`,
  `ReadWriteMany`, 1Gi each. (A local cluster needs a RWX-capable CSI driver,
  or change `accessModes` to `ReadWriteOnce` for single-node testing.)

### 3.6 Verify

```bash
kubectl -n dumby-staging get pods,svc,hpa
kubectl -n dumby-staging rollout status deploy/dumby
kubectl -n dumby-staging port-forward svc/dumby 9090:9090
curl http://localhost:9090/health/liveness
```

---

## 4. Path C — Terraform (AWS cloud infra)

The Terraform layer provisions the **cloud infrastructure** (RDS, ElastiCache,
EKS) that the k8s manifests then deploy into. It is AWS-only (provider
`hashicorp/aws ~> 5.0`).

### 4.1 What exists

Three reusable modules under `apps/main/deploy/terraform/modules/`:

| Module | Creates | Defaults |
|--------|---------|----------|
| `database/` | `aws_db_subnet_group` + `aws_db_instance` (Postgres 16, RDS) | `db.t3.micro`, 7-day backups, encryption on, deletion protection on |
| `redis/` | `aws_elasticache_subnet_group` + `aws_elasticache_replication_group` (Redis 7.1) | `cache.t4g.micro`, 2 nodes, failover on, TLS in transit + at rest |
| `kubernetes/` | `aws_eks_cluster` + IAM role/attachment | node types `t3.medium`, 2 desired / 2 min / 10 max |

Environment root modules (`deploy/terraform/environments/{staging,production}/`)
are currently **reserved placeholders** — the README says "provider credentials
are supplied by operators." So you need to write a small `main.tf` to wire the
modules together, e.g.:

```hcl
# apps/main/deploy/terraform/environments/staging/main.tf
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket         = "YOUR-TFSTATE-BUCKET"
    key            = "dumby/staging/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "dumby-tf-locks"
    encrypt        = true
  }
}

provider "aws" { region = "ap-south-1" }

# you must supply: vpc_id, subnet_ids (private), sg ids
module "database" {
  source               = "../../modules/database"
  identifier           = "dumby-staging"
  db_username          = "dumby"
  db_password          = var.db_password     # mark as sensitive, pass via TF_VAR_
  subnet_ids           = var.private_subnet_ids
  vpc_security_group_ids = var.db_sg_ids
}

module "redis" {
  source            = "../../modules/redis"
  cluster_id        = "dumby-staging"
  subnet_ids        = var.private_subnet_ids
  security_group_ids = var.redis_sg_ids
}

module "kubernetes" {
  source        = "../../modules/kubernetes"
  cluster_name  = "dumby-staging"
  subnet_ids    = var.private_subnet_ids
}
```

### 4.2 Plan + apply

```bash
cd apps/main/deploy/terraform/environments/staging
export TF_VAR_db_password='...'            # from your secrets manager
terraform init
terraform plan
terraform apply
```

Outputs (`endpoint`, `primary_endpoint_address`, `cluster_endpoint`) are marked
sensitive; use `terraform output -json` to feed them into the k8s Secret from
section 3.2.

> Reminder: `aws_db_instance` has `deletion_protection = true` and
> `skip_final_snapshot = false`, so `terraform destroy` will refuse to delete
> without a final snapshot. Plan accordingly.

---

## 5. End-to-end order (all three together)

If you want the full "Docker → Kubernetes → Terraform" flow:

1. **Terraform** (Path C): `terraform apply` in the environment of choice →
   provisions EKS, RDS, ElastiCache. Get `cluster_endpoint`, DB endpoint,
   Redis endpoint from outputs.
2. **kubeconfig**: `aws eks update-kubeconfig --name dumby-staging --region ap-south-1`
3. **Docker build + push**: build `dumby:latest`, push to your registry (ECR),
   update the k8s Deployment `image:` field to point at the registry tag.
4. **Kubernetes** (Path B): `kubectl apply -k deploy/k8s/overlays/staging`
   with a `dumby-secrets` Secret populated from the Terraform outputs.
5. **Migrate**: `kubectl -n dumby-staging exec deploy/dumby -- node scripts/migrate.ts up`
6. **Verify**: port-forward the health port and curl `/health/readiness`.

For purely local testing, skip steps 1–2 and use Path A (Docker Compose) or
Path B with a local cluster + Bitnami Postgres/Redis.

---

## 6. Quick troubleshooting

- **`pnpm install` fails on `minimumReleaseAge`** — `pnpm-workspace.yaml` enforces
  a 1-day release-age gate. If a dep was published <24h ago, either wait or add
  it to `minimumReleaseAgeExclude` (read the comment block before doing so).
- **Node version mismatch** — engine requires `>=22.0.0`; use `nvm use` against
  `.nvmrc` (contains `22`).
- **RWX PVC stuck Pending in k8s** — local clusters often lack a RWX CSI driver.
  Edit `swastik-volumes.yaml` `accessModes` to `ReadWriteOnce` for single-node tests.
- **Mock vs live Discord** — `.env.example` ships `DISCORD_ADAPTER=mock` (no token
  needed); the k8s `configmap.yaml` sets `DISCORD_ADAPTER: live` (token required
  in `dumby-secrets`).
- **Port differences** — Docker compose uses 8080/8081/8082; k8s base uses
  3000/9090/9091. Don't mix them up when curling health endpoints.

---

## 7. File map (where everything lives)

```
Dumby-Suite-main/
├── apps/main/
│   ├── .env.example                      ← copy to .env for compose
│   ├── .nvmrc                            ← "22"
│   ├── Dockerfile                        ← (root-level, used by compose)
│   ├── docker-compose.yml                ← canonical compose (4 services)
│   ├── docker/
│   │   ├── Dockerfile                    ← multi-stage build (node:22-slim)
│   │   └── docker-compose.yml           ← wrapper, includes the one above
│   ├── deploy/
│   │   ├── k8s/base/                     ← deployment, service, cm, hpa, sa, pvcs
│   │   ├── k8s/overlays/staging/         ← 1 replica, debug, staging env
│   │   ├── k8s/overlays/production/      ← 2 replicas, warn, production env
│   │   └── terraform/
│   │       ├── modules/database/         ← RDS Postgres 16
│   │       ├── modules/redis/            ← ElastiCache Redis 7.1
│   │       ├── modules/kubernetes/       ← EKS + IAM
│   │       └── environments/{staging,production}/  ← reserved (write root here)
│   ├── src/bootstrap/bootstrap.ts        ← 18-stage startup
│   └── scripts/migrate.ts               ← knex migrations entry
├── pnpm-workspace.yaml                   ← workspace + catalog + overrides
└── package.json                          ← workspace root
```
