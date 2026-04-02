---
name: deployment
description: "Deploy applications to local dev, cloud dev, or production environments. Dev mode uses Docker Compose (local) or cloud CLI (sandbox). Prod mode generates Terraform modules and GitHub Actions pipelines. Supports AWS and GCP. Replaces docker-deployment skill. Use this skill when the user asks to deploy, set up infrastructure, create a CI/CD pipeline, or move an app to production."
---

# Deployment Skill

## When to Skip

- Apps Script — deployed via Extensions > Apps Script, no infrastructure needed
- Quick prototype running locally with `docker-compose up` — already handled by `backend` skill
- The user just wants to write code, not deploy yet

## When to Use

- Moving from local dev to a cloud or production environment
- Setting up CI/CD pipelines
- Provisioning cloud infrastructure (databases, storage, networking)
- Production readiness review

---

Deploy applications across environments: local dev, cloud dev sandbox, and production. Dev is fast and manual. Prod is 100% automated through Terraform and GitHub Actions.

## Safety Rule — Read Only

**CRITICAL: Never write to cloud infrastructure directly.**

- **Allowed:** Read commands to inspect cloud state (`gcloud describe`, `aws describe`, `gcloud list`, `aws list`, `terraform plan`)
- **Not allowed:** Write commands (`gcloud create`, `aws create`, `terraform apply`, `gcloud deploy`, resource deletion, IAM modifications)
- **What to do instead:** Generate the Terraform files, GitHub Actions workflows, and CLI commands. Present write commands as suggestions. Only execute a write command when the user explicitly instructs you to run it.
- **GitHub Actions** is the mechanism that writes to cloud in production — triggered by PR merge, not by direct CLI execution.

This applies to both dev and prod. Even in a cloud dev sandbox, suggest the command and let the user decide.

## Step 0: Identify the Deployment Target

Ask or infer from context:

| Target | When | How |
|--------|------|-----|
| **Local dev** | Development, prototyping, POC | Docker Compose, no cloud, no Terraform |
| **Cloud dev sandbox** | Testing cloud services, staging | CLI commands (manual), disposable resources |
| **Production (self-hosted)** | K8s on own infrastructure | K8s manifests + GitHub Actions |
| **Production (GCP)** | Cloud Run, GKE, Cloud SQL | Terraform + GitHub Actions |
| **Production (AWS)** | ECS/Fargate, EKS, RDS | Terraform + GitHub Actions |

## Step 1: Evaluate Defaults

Before generating anything, evaluate each infrastructure component using the self-hosted vs cloud framework from the `backend` skill. Consider:

- Operational complexity vs engineering time
- Cost (money and people)
- Vendor lock-in risk
- Scale requirements
- Data sovereignty constraints

State evaluations briefly. Confirm defaults or recommend alternatives with reasons.

## Step 2: Credentials & Secrets Strategy

Before generating any deployment config, ask: **how will credentials flow in each environment?**

### Principle: No long-lived keys anywhere.

| Environment | Method | Details |
|-------------|--------|---------|
| **Local dev** | `.env` file (gitignored) | Database passwords, API keys for local services. Never committed. |
| **CI/CD → GCP** | Workload Identity Federation | GitHub proves its identity to GCP directly — no service account JSON keys. Configure once in GCP IAM + GitHub Actions. |
| **CI/CD → AWS** | OIDC role assumption | GitHub assumes an IAM role via OIDC — no access keys stored. Configure once in AWS IAM + GitHub Actions. |
| **App runtime (cloud)** | Cloud Secret Manager | GCP Secret Manager or AWS Secrets Manager. App reads secrets at startup. Mounted into Cloud Run containers or ECS task definitions. |
| **App runtime (K8s)** | Kubernetes Secrets + Sealed Secrets | For self-hosted K8s. Use Sealed Secrets or external-secrets-operator to sync from a secret manager. |

### CI/CD Auth Patterns

**GCP — Workload Identity Federation (preferred):**
```yaml
# GitHub Actions
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/PROJECT_NUM/locations/global/workloadIdentityPools/github/providers/github
    service_account: deploy@project.iam.gserviceaccount.com
```

**GCP — Service account key (fallback, avoid if possible):**
```yaml
- uses: google-github-actions/auth@v2
  with:
    credentials_json: ${{ secrets.GCP_SA_KEY }}
```

**AWS — OIDC (preferred):**
```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::ACCOUNT_ID:role/github-deploy
    aws-region: eu-west-1
```

### App Runtime Secrets

**GCP — Secret Manager in Cloud Run:**
```hcl
# Terraform
resource "google_secret_manager_secret" "db_url" {
  secret_id = "database-url"
  replication { auto {} }
}

# Referenced in Cloud Run service
env {
  name = "DATABASE_URL"
  value_source {
    secret_key_ref {
      secret  = google_secret_manager_secret.db_url.secret_id
      version = "latest"
    }
  }
}
```

**AWS — Secrets Manager in ECS:**
```hcl
# Terraform
resource "aws_secretsmanager_secret" "db_url" {
  name = "app/database-url"
}

# Referenced in ECS task definition
secrets = [
  {
    name      = "DATABASE_URL"
    valueFrom = aws_secretsmanager_secret.db_url.arn
  }
]
```

### Rules

| Do | Don't |
|----|-------|
| Use Workload Identity Federation (GCP) or OIDC (AWS) for CI/CD | Store service account keys or access keys in GitHub Secrets |
| Use Cloud Secret Manager for app runtime secrets | Put secrets in environment variables, tfvars, or code |
| Use `.env` files (gitignored) for local dev only | Commit `.env` files or use them in production |
| Rotate secrets via the secret manager | Embed long-lived credentials anywhere |
| Use least-privilege IAM roles for CI/CD service accounts | Give CI/CD admin access |

---

## Dev Mode

### Local Dev (Docker Compose)

The `backend` skill already scaffolds a `docker-compose.yml`. Deployment extends it:

```yaml
# docker-compose.yml — local dev
services:
  api:
    build: .
    ports:
      - "8080:8080"
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - .:/src  # Hot reload in dev

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: dev
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 2s
      timeout: 5s
      retries: 5
    volumes:
      - pgdata:/var/lib/postgresql/data

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    volumes:
      - ./frontend/src:/app/src  # Hot reload

volumes:
  pgdata:
```

Add SeaweedFS, Redis, or other services as needed:

```yaml
  seaweedfs:
    image: chrislusf/seaweedfs
    command: "server -s3"
    ports:
      - "8333:8333"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

**Dev workflow:**
```bash
docker compose up -d          # Start everything
docker compose logs -f api    # Follow API logs
docker compose down           # Stop everything
docker compose down -v        # Stop and wipe data
```

### Cloud Dev Sandbox

Use cloud CLI to inspect existing resources, then suggest commands to create disposable dev resources.

#### GCP — Read commands (always allowed)

```bash
# Discover project and region
gcloud config get-value project
gcloud config get-value compute/region

# List existing resources
gcloud sql instances list
gcloud run services list
gcloud storage buckets list
gcloud iam service-accounts list

# Inspect a specific service
gcloud run services describe SERVICE_NAME --region REGION
gcloud sql instances describe INSTANCE_NAME
```

#### AWS — Read commands (always allowed)

```bash
# Discover account and region
aws sts get-caller-identity
aws configure get region

# List existing resources
aws rds describe-db-instances
aws ecs list-clusters
aws s3 ls
aws iam list-roles

# Inspect a specific service
aws ecs describe-services --cluster CLUSTER --services SERVICE
aws rds describe-db-instances --db-instance-identifier NAME
```

#### Suggested write commands (present to user, never execute)

```bash
# GCP dev sandbox — suggest these, don't run
gcloud sql instances create dev-myapp --database-version=POSTGRES_16 --tier=db-f1-micro --region=REGION
gcloud run deploy myapp-dev --source . --region=REGION --allow-unauthenticated

# AWS dev sandbox — suggest these, don't run
aws rds create-db-instance --db-instance-identifier dev-myapp --engine postgres --db-instance-class db.t4g.micro
aws ecs create-service ...
```

---

## Prod Mode

Production is fully automated. The deliverables are:
1. **Terraform modules** — provision infrastructure
2. **GitHub Actions workflows** — build, plan, apply, deploy
3. **No manual steps** — everything runs through CI/CD

### Step 2: Inspect Existing Infrastructure

Use read-only CLI commands to understand what exists before generating Terraform. This avoids creating duplicate resources or conflicting configurations.

**GCP checklist:**
```bash
gcloud projects describe PROJECT_ID
gcloud compute networks list
gcloud compute networks subnets list
gcloud sql instances list
gcloud container clusters list  # GKE
gcloud run services list
gcloud iam service-accounts list
gcloud services list --enabled  # Which APIs are enabled
```

**AWS checklist:**
```bash
aws sts get-caller-identity
aws ec2 describe-vpcs
aws ec2 describe-subnets
aws rds describe-db-instances
aws ecs list-clusters
aws eks list-clusters
aws iam list-roles
```

Document findings before proceeding to Terraform generation.

### Step 3: Generate Terraform

#### Project Structure

```
terraform/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       └── backend.tf
├── modules/
│   ├── database/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── app/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── storage/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── iam/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── README.md
```

#### Terraform Patterns

**State backend — always remote:**

```hcl
# GCP
terraform {
  backend "gcs" {
    bucket = "myproject-terraform-state"
    prefix = "prod"
  }
}

# AWS
terraform {
  backend "s3" {
    bucket         = "myproject-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

**Database module (GCP example):**

```hcl
resource "google_sql_database_instance" "main" {
  name             = var.instance_name
  database_version = "POSTGRES_16"
  region           = var.region

  settings {
    tier              = var.tier
    availability_type = var.environment == "prod" ? "REGIONAL" : "ZONAL"

    backup_configuration {
      enabled                        = true
      point_in_time_recovery_enabled = var.environment == "prod"
    }

    ip_configuration {
      ipv4_enabled    = false
      private_network = var.vpc_id
    }

    database_flags {
      name  = "max_connections"
      value = var.max_connections
    }
  }

  deletion_protection = var.environment == "prod"
}

resource "google_sql_database" "app" {
  name     = var.db_name
  instance = google_sql_database_instance.main.name
}

resource "google_sql_user" "app" {
  name     = var.db_user
  instance = google_sql_database_instance.main.name
  password = var.db_password
}
```

**Database module (AWS example):**

```hcl
resource "aws_db_instance" "main" {
  identifier     = var.instance_name
  engine         = "postgres"
  engine_version = "16"
  instance_class = var.instance_class

  db_name  = var.db_name
  username = var.db_user
  password = var.db_password

  multi_az               = var.environment == "prod"
  backup_retention_period = var.environment == "prod" ? 7 : 1
  deletion_protection    = var.environment == "prod"

  vpc_security_group_ids = [var.db_security_group_id]
  db_subnet_group_name   = var.db_subnet_group

  storage_encrypted = true
}
```

**App deployment (GCP Cloud Run):**

```hcl
resource "google_cloud_run_v2_service" "app" {
  name     = var.service_name
  location = var.region

  template {
    containers {
      image = var.image

      env {
        name  = "DATABASE_URL"
        value_source {
          secret_key_ref {
            secret  = google_secret_manager_secret.db_url.secret_id
            version = "latest"
          }
        }
      }

      resources {
        limits = {
          cpu    = var.cpu
          memory = var.memory
        }
      }

      startup_probe {
        http_get {
          path = "/health"
        }
      }

      liveness_probe {
        http_get {
          path = "/health"
        }
      }
    }

    scaling {
      min_instance_count = var.environment == "prod" ? 1 : 0
      max_instance_count = var.max_instances
    }
  }
}
```

**App deployment (AWS ECS Fargate):**

```hcl
resource "aws_ecs_service" "app" {
  name            = var.service_name
  cluster         = var.cluster_id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.environment == "prod" ? 2 : 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = var.private_subnets
    security_groups = [var.app_security_group_id]
  }

  load_balancer {
    target_group_arn = var.target_group_arn
    container_name   = var.service_name
    container_port   = 8080
  }
}

resource "aws_ecs_task_definition" "app" {
  family                   = var.service_name
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = var.cpu
  memory                   = var.memory
  execution_role_arn       = var.execution_role_arn
  task_role_arn            = var.task_role_arn

  container_definitions = jsonencode([{
    name  = var.service_name
    image = var.image
    portMappings = [{ containerPort = 8080 }]
    healthCheck = {
      command     = ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
      interval    = 30
      timeout     = 5
      retries     = 3
      startPeriod = 60
    }
    secrets = [
      {
        name      = "DATABASE_URL"
        valueFrom = var.db_url_secret_arn
      }
    ]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = var.log_group
        "awslogs-region"        = var.region
        "awslogs-stream-prefix" = var.service_name
      }
    }
  }])
}
```

#### Terraform Rules

| Do | Don't |
|----|-------|
| Use modules for reusable components | Inline all resources in one file |
| Use `terraform.tfvars` per environment | Hardcode environment-specific values |
| Remote state with locking | Local state files |
| Tag all resources with project, environment, owner | Leave resources untagged |
| Use `deletion_protection` in prod | Allow accidental resource deletion |
| Reference secrets from Secret Manager | Put secrets in tfvars or code |
| Use private networking (VPC, private IPs) | Expose databases to the internet |

### Step 4: Generate GitHub Actions Pipeline

#### CI/CD Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io  # or gcr.io / ECR
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - run: go test ./... -v -count=1

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name == 'push' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  terraform-plan:
    needs: test
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: terraform/environments/prod
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3

      # GCP auth
      - uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      # OR AWS auth
      # - uses: aws-actions/configure-aws-credentials@v4
      #   with:
      #     role-to-arn: ${{ secrets.AWS_ROLE_ARN }}
      #     aws-region: ${{ vars.AWS_REGION }}

      - run: terraform init
      - run: terraform plan -no-color
        id: plan

      - uses: actions/github-script@v7
        with:
          script: |
            const plan = `${{ steps.plan.outputs.stdout }}`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Terraform Plan\n\`\`\`\n${plan}\n\`\`\``
            });

  terraform-apply:
    needs: [build]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval in GitHub
    defaults:
      run:
        working-directory: terraform/environments/prod
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - run: terraform init
      - run: terraform apply -auto-approve

  deploy:
    needs: [build, terraform-apply]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      # GCP Cloud Run
      - uses: google-github-actions/auth@v2
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      - uses: google-github-actions/deploy-cloudrun@v2
        with:
          service: myapp
          image: ${{ needs.build.outputs.image_tag }}
          region: ${{ vars.GCP_REGION }}

      # OR AWS ECS
      # - uses: aws-actions/amazon-ecs-deploy-task-definition@v1
      #   with:
      #     task-definition: task-def.json
      #     service: myapp
      #     cluster: myapp-cluster

  migrate:
    needs: [deploy]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          # Run database migrations against prod
          # Use a one-off container or Cloud Run job
          echo "Run migrations here"
```

#### Pipeline Flow

```
PR opened:
  test → terraform plan → comment plan on PR

PR merged to main:
  test → build + push image → terraform apply → deploy app → run migrations
```

### Step 5: Production Checklist

Before going live, verify:

| Category | Check |
|----------|-------|
| **Networking** | Database on private network, not public |
| **TLS** | HTTPS everywhere, managed certificates |
| **Secrets** | All secrets in Secret Manager, none in code or env files |
| **Auth** | Service accounts with least-privilege IAM |
| **Health checks** | `/health` and `/ready` endpoints configured |
| **Backups** | Database backups enabled, point-in-time recovery in prod |
| **Monitoring** | Structured logging to Cloud Logging / CloudWatch |
| **Scaling** | Min/max instances configured, auto-scaling rules set |
| **Deletion protection** | Enabled on database and critical resources |
| **Rollback** | Previous image tag available, can redeploy in < 5 minutes |
| **DNS** | Domain configured, pointing to load balancer / Cloud Run |
| **Budget alerts** | Cost alerts configured on the cloud project |

---

## Self-Hosted Production (K8s)

When deploying to self-hosted Kubernetes instead of cloud-managed services:

### K8s Manifests

```
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patch-replicas.yaml
    └── prod/
        ├── kustomization.yaml
        ├── patch-replicas.yaml
        └── patch-resources.yaml
```

Use Kustomize for environment-specific overlays. Avoid Helm unless the project needs templating beyond what Kustomize provides.

### GitHub Actions for K8s

```yaml
deploy:
  steps:
    - uses: actions/checkout@v4
    - run: |
        kubectl apply -k k8s/overlays/prod
        kubectl rollout status deployment/myapp -n myapp --timeout=300s
```

---

## Rules

**Cross-cutting guardrails:** Follow all rules in `guardrails.md` — security, cost, data integrity, process, resilience, and maintenance.

| Do | Don't |
|----|-------|
| Read cloud state via CLI to understand existing infra | Write to cloud without explicit user instruction |
| Generate Terraform and present it for review | Run `terraform apply` directly |
| Use GitHub Actions for all prod deployments | Deploy manually from a laptop |
| Use GitHub environment protection rules for prod | Allow auto-deploy to prod without approval |
| Tag all cloud resources (project, env, owner) | Create untagged resources |
| Use private networking for databases | Expose databases publicly |
| Store Terraform state remotely with locking | Use local state files |
| Use `deletion_protection` in prod | Allow accidental resource deletion |
| Set budget alerts on cloud projects | Deploy without cost monitoring |
| Present suggested write commands to the user | Execute write commands autonomously |

## Handoff

### Input
- Built application from `backend` and `frontend` skills (Dockerfile, docker-compose.yml)
- Test suite and CI workflow from `testing` skill
- Deployment target decision from `software-architecture` skill

### Output
- **Dev:** Docker Compose config, cloud sandbox CLI commands (suggested, not executed)
- **Prod:** Terraform modules, GitHub Actions pipeline (test → build → plan → apply → deploy → migrate)
- Production checklist

### Previous skills
- `software-architecture` — deployment target decision
- `backend` — Dockerized Go service
- `frontend` — Frontend build artifacts
- `testing` — CI workflow to extend with deploy steps

### Next skills
This is the **final skill** in the pipeline. No downstream skills.

### Full Pipeline

```
software-architecture (entry point)
    ├── backend ──────────┐
    ├── frontend ─────────┤
    ├── mobile ───────────┤
    └── google-apps-script ├── testing → deployment (exit point)
```

## Continuous Improvement

### Skill Health Check

At the end of every use, evaluate and report if any of the following apply:

- **Skill gap:** "This project needs [X] and no current skill covers it. Consider creating a `[name]` skill." — e.g., an observability/monitoring skill, a database migration management skill.
- **Skill split:** "This skill's [section] is growing complex enough to be its own skill." — e.g., if Terraform patterns become extensive enough to warrant a dedicated `infrastructure` skill separate from app deployment.
- **Skill overlap:** "This skill and `[other skill]` both cover [topic]. Consider consolidating."

Only report when genuinely applicable. Don't force observations.

### Learnings

Corrections and refinements discovered during use. When the user overrides a recommendation or a default doesn't fit, record it here so future uses benefit.

_(Empty — will accumulate over time)_
