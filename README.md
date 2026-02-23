# Hello World Flask — ECS Fargate Deployment Guide

Deploy a containerized Flask app to **AWS ECS Fargate** using **GitHub Actions** with **OIDC authentication** (no static AWS credentials).

## Architecture

```
GitHub (push to main)
  │
  ▼
GitHub Actions (OIDC → assume IAM role)
  │
  ├─► Build Docker image
  ├─► Push to ECR
  └─► Update ECS task definition & service
        │
        ▼
ECS Fargate (pulls image from ECR, runs container on port 8080)
```

## Prerequisites

- AWS account with console access
- GitHub repository with this code pushed
- Docker installed locally (for optional initial image push)
- AWS CLI installed locally (for optional initial image push)

---

## Phase 1 — Create ECR Repository

1. Go to **ECR** → **Create repository**
2. Configure:
   - Visibility: **Private**
   - Repository name: `hello-world-app`
   - Tag immutability: **Enabled** (prevents overwriting images)
   - Scan on push: **Enabled** (auto-scans for CVEs)
   - Encryption: **AES-256** (default)
3. Click **Create repository**
4. Note your repository URI: `<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/hello-world-app`

---

## Phase 2 — Configure GitHub OIDC Identity Provider

1. Go to **IAM** → **Identity providers** → **Add provider**
2. Configure:
   - Provider type: **OpenID Connect**
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Click **Get thumbprint**
   - Audience: `sts.amazonaws.com`
3. Click **Add provider**

---

## Phase 3 — Create IAM Roles

### 3.1 — GitHub Actions Role

1. Go to **IAM** → **Roles** → **Create role**
2. Trusted entity type: **Web identity**
3. Identity provider: `token.actions.githubusercontent.com`
4. Audience: `sts.amazonaws.com`
5. Click **Next** → skip policies for now → click **Next**
6. Role name: `GitHubActionsECSRole`
7. Click **Create role**

#### Edit the Trust Policy

Go to the role → **Trust relationships** → **Edit trust policy** and replace with:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<GITHUB_ORG>/<REPO_NAME>:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

> Replace `<ACCOUNT_ID>`, `<GITHUB_ORG>`, and `<REPO_NAME>` with your values.

#### Attach Inline Policy

Go to the role → **Permissions** → **Add permissions** → **Create inline policy** → **JSON**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRAuth",
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Sid": "ECRPush",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "arn:aws:ecr:us-east-1:<ACCOUNT_ID>:repository/hello-world-app"
    },
    {
      "Sid": "ECSDeployment",
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeTaskDefinition",
        "ecs:RegisterTaskDefinition",
        "ecs:UpdateService",
        "ecs:DescribeServices"
      ],
      "Resource": "*"
    },
    {
      "Sid": "PassRoleToECS",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskExecutionRole",
        "arn:aws:iam::<ACCOUNT_ID>:role/ecsTaskRole"
      ]
    }
  ]
}
```

Policy name: `GitHubActionsECSDeployPolicy`

### 3.2 — ECS Task Execution Role

This role lets ECS pull images from ECR and write logs to CloudWatch.

1. Go to **IAM** → **Roles** → **Create role**
2. Trusted entity: **AWS service** → **Elastic Container Service** → **Elastic Container Service Task**
3. Attach policy: `AmazonECSTaskExecutionRolePolicy` (AWS managed)
4. Role name: `ecsTaskExecutionRole`
5. Click **Create role**

### 3.3 — ECS Task Role (optional, recommended)

This role is for your application container to call AWS services at runtime (S3, DynamoDB, etc). Create it now even if not needed yet.

1. Go to **IAM** → **Roles** → **Create role**
2. Trusted entity: **AWS service** → **Elastic Container Service** → **Elastic Container Service Task**
3. Attach policies: none for now
4. Role name: `ecsTaskRole`
5. Click **Create role**

---

## Phase 4 — Create CloudWatch Log Group

1. Go to **CloudWatch** → **Log groups** → **Create log group**
2. Name: `/ecs/hello-world-app`
3. Retention: **30 days**
4. Click **Create**

---

## Phase 5 — Create ECS Cluster

1. Go to **ECS** → **Create cluster**
2. Cluster name: `hello-world-cluster`
3. Infrastructure: **AWS Fargate (serverless)** only — uncheck EC2
4. Click **Create**

---

## Phase 6 — Create ECS Task Definition

1. Go to **ECS** → **Task definitions** → **Create new task definition**
2. Task definition family: `hello-world-task`
3. Launch type: **AWS Fargate**
4. OS/Arch: **Linux/X86_64**
5. CPU: **0.25 vCPU**
6. Memory: **0.5 GB**
7. Task execution role: `ecsTaskExecutionRole`
8. Task role: `ecsTaskRole`
9. Container definition:
   - Name: `hello-world-container`
   - Image URI: `<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/hello-world-app:latest` (placeholder — GitHub Actions will update this on each deploy)
   - Port mapping: **8080** TCP
   - Essential: **Yes**
   - Log driver: `awslogs`
     - `awslogs-group`: `/ecs/hello-world-app`
     - `awslogs-region`: `us-east-1`
     - `awslogs-stream-prefix`: `ecs`
   - Health check (recommended):
     - Command: `CMD-SHELL, curl -f http://localhost:8080/health || exit 1`
     - Interval: 30s, Timeout: 5s, Retries: 3, Start period: 60s
10. Click **Create**

---

## Phase 7 — Create Security Group

1. Go to **VPC** → **Security groups** → **Create security group**
2. Name: `hello-world-ecs-sg`
3. VPC: your default VPC
4. Inbound rules:
   - Type: **Custom TCP**, Port: **8080**, Source: `0.0.0.0/0`
5. Outbound rules: **All traffic** (default, needed for ECR pulls)
6. Click **Create security group**

---

## Phase 8 — Create ECS Service

1. Go to **ECS** → your cluster (`hello-world-cluster`) → **Create service**
2. Launch type: **Fargate**
3. Task definition: `hello-world-task` (latest revision)
4. Service name: `hello-world-service`
5. Desired tasks: **1**
6. Networking:
   - VPC: default VPC
   - Subnets: select at least 2 public subnets
   - Security group: `hello-world-ecs-sg`
   - Public IP: **Enabled** (required for Fargate in public subnets to pull from ECR)
7. Click **Create service**

---

## Phase 9 — Bootstrap First Image

The GitHub Actions workflow downloads the current task definition, so ECS needs a valid image before the first automated deploy.

### Option A — Push manually from your machine

```bash
# Login to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com

# Build and push
docker build -t hello-world-app .
docker tag hello-world-app:latest <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/hello-world-app:initial
docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/hello-world-app:initial
```

Then update the task definition image URI to the `:initial` tag.

### Option B — Just push to main

Let GitHub Actions handle it. The ECS service will initially fail (no image) but self-heal after the first successful workflow run.

---

## Phase 10 — Update deploy.yml Placeholders

Open `.github/workflows/deploy.yml` and replace `YOUR_ACCOUNT_ID` on line 32 with your real AWS account ID:

```yaml
role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionsECSRole
```

---

## Phase 11 — Deploy

Push to `main` and watch the workflow run in the **Actions** tab of your GitHub repo.

```bash
git add -A
git commit -m "initial deploy"
git push origin main
```

Verify:
1. GitHub Actions → workflow should complete with green checkmarks
2. ECS → your service should show 1 running task
3. Grab the task's **public IP** from the ECS console → task → **Networking** tab
4. Open `http://<PUBLIC_IP>:8080` — you should see `{"message": "Hello, World!"}`
5. Open `http://<PUBLIC_IP>:8080/health` — you should see `{"status": "healthy"}`

---

## Security Checklist

- [x] **OIDC** — no static AWS credentials stored in GitHub
- [x] **Least-privilege IAM** — GitHub Actions role only has ECR push + ECS deploy permissions
- [x] **Trust policy scoped** — OIDC role restricted to specific repo and branch
- [x] **Non-root container** — Dockerfile runs as `appuser`
- [x] **Multi-stage build** — minimal final image, no build tools in runtime
- [x] **ECR scan on push** — automatic CVE scanning
- [x] **Tag immutability** — prevents image tag overwriting
- [x] **CloudWatch logs** — centralized logging with retention policy

## Future Improvements

| Improvement | Why |
|-------------|-----|
| Add an **ALB** (Application Load Balancer) | TLS termination, health checks, no public IP on tasks |
| Move tasks to **private subnets + NAT Gateway** | More secure network topology |
| Add **ECR lifecycle policy** | Auto-delete old/untagged images to save storage costs |
| Use **Fargate Spot** for non-prod | Up to 70% cost savings |
| Add `workflow_dispatch` trigger | Allows manual re-deploys without pushing code |
| Pin GitHub Actions to **commit SHAs** | Prevents supply chain attacks via tag hijacking |
| Add `concurrency` block to workflow | Prevents parallel deploys to the same service |
| Add GitHub **environment** with protection rules | Require approvals before production deploys |
