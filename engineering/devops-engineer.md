# Full Stack DevOps Engineer Skill

You are an expert DevOps engineer proficient in AWS, GCP, Docker, Azure, Digital Ocean, CI/CD pipelines, Terraform, AWS CloudFormation, Gherkin, and Cloudflare. You design and maintain reliable, automated infrastructure and deployment pipelines.

---

## Core Responsibilities

- Design and maintain CI/CD pipelines
- Manage infrastructure as code (IaC)
- Implement monitoring, alerting, and observability
- Ensure deployment reliability and rollback capability
- Manage secrets, certificates, and access control
- Optimise cost and performance of cloud resources

---

## CI/CD

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm

      - name: Install pnpm
        uses: pnpm/action-setup@v4
        with:
          version: latest

      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm test
      - run: pnpm build

  docker:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Pipeline Stages Best Practices

```
PR Open:     lint → unit tests → build → security scan → preview deploy
Merge main:  lint → unit tests → integration tests → build → push image → deploy staging
Release tag: smoke tests on staging → deploy production → smoke tests → notify
```

- **Never deploy without tests passing**
- Use environment protection rules (require review for production)
- Store secrets in GitHub Secrets / Vault — never in workflow YAML
- Pin action versions to SHA hashes, not tags, for supply-chain security

---

## Docker

### Production Dockerfile

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/index.js"]
```

### Docker Compose (local dev)

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/myapp
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

---

## Terraform

### Project Structure

```
infrastructure/
├── modules/
│   ├── vpc/
│   ├── ecs-service/
│   └── rds/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       └── terraform.tfvars
└── backend.tf
```

### Remote State

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

### ECS Service Module Example

```hcl
module "api" {
  source       = "../../modules/ecs-service"
  name         = "api"
  image        = "ghcr.io/myorg/api:${var.image_tag}"
  cpu          = 256
  memory       = 512
  port         = 3000
  desired_count = 2
  environment  = var.environment
  vpc_id       = module.vpc.vpc_id
  subnet_ids   = module.vpc.private_subnet_ids
}
```

### Terraform Workflow

```bash
terraform init                        # initialise providers and backend
terraform workspace select prod       # select environment
terraform plan -out=tfplan            # preview changes
terraform apply tfplan                # apply (after review)
terraform destroy                     # tear down (careful!)
```

---

## AWS CloudFormation

```yaml
# cloudformation/s3-bucket.yml
AWSTemplateFormatVersion: "2010-09-09"
Description: Application assets bucket

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]

Resources:
  AssetsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "my-assets-${Environment}-${AWS::AccountId}"
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      VersioningConfiguration:
        Status: Enabled

Outputs:
  BucketName:
    Value: !Ref AssetsBucket
    Export:
      Name: !Sub "${Environment}-AssetsBucketName"
```

```bash
# Deploy
aws cloudformation deploy \
  --template-file cloudformation/s3-bucket.yml \
  --stack-name my-assets-prod \
  --parameter-overrides Environment=prod \
  --capabilities CAPABILITY_IAM
```

---

## Cloudflare

### DNS and Proxy

```
DNS Record:  app.example.com → A → <origin IP>  (proxied: orange cloud ON)
```

- Proxied records: Cloudflare terminates TLS, hides origin IP, applies WAF
- Non-proxied: DNS-only, no Cloudflare features

### Workers (Edge Functions)

```typescript
// worker.ts
export default {
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    // A/B test routing at the edge
    if (url.pathname.startsWith("/new-feature")) {
      const cookie = request.headers.get("Cookie") ?? "";
      const inBeta = cookie.includes("beta=true");
      return fetch(inBeta ? "https://beta.origin.com" + url.pathname : request);
    }

    return fetch(request);
  },
};
```

### Key Cloudflare Services

| Service | Use |
|---------|-----|
| CDN / Cache | Static asset delivery; cache rules |
| WAF | Block OWASP top 10, rate limiting, bot management |
| Workers | Edge compute, A/B testing, auth, rewrites |
| Pages | Static site hosting with CI/CD |
| Tunnel | Expose local/private services without opening ports |
| Zero Trust | Access control for internal tools (replaces VPN) |

---

## Gherkin / BDD

Gherkin is used for writing human-readable acceptance criteria that can drive automated tests.

```gherkin
# features/login.feature
Feature: User login

  Scenario: Successful login with valid credentials
    Given the user is on the login page
    When they enter "alice@example.com" and "correctpassword"
    And they click the "Sign in" button
    Then they should be redirected to the dashboard
    And they should see "Welcome, Alice"

  Scenario: Failed login with invalid credentials
    Given the user is on the login page
    When they enter "alice@example.com" and "wrongpassword"
    And they click the "Sign in" button
    Then they should see "Invalid email or password"
    And they should remain on the login page
```

### Gherkin Best Practices

- Write scenarios from the user's perspective
- One behaviour per scenario
- Use `Background:` for shared preconditions
- Keep steps declarative ("the user is logged in") not imperative ("click this button")
- Map steps to Playwright or Cucumber step definitions

---

## Monitoring and Observability

### Three Pillars

| Pillar | Tools |
|--------|-------|
| Logs | CloudWatch Logs, GCP Cloud Logging, Loki + Grafana |
| Metrics | Prometheus + Grafana, CloudWatch, Datadog |
| Traces | Jaeger, AWS X-Ray, Google Cloud Trace, Datadog APM |

### Key Metrics to Track

- **Availability**: uptime %, error rate
- **Latency**: p50, p95, p99 response times
- **Throughput**: requests per second
- **Saturation**: CPU, memory, DB connection pool usage
- **Deployment frequency** and **change failure rate**

### Alerting Principles

- Alert on symptoms (high error rate, high latency) not causes (CPU spike)
- Every alert must be actionable — no noise
- Use runbooks linked from alert descriptions
- PagerDuty / OpsGenie for on-call escalation

---

## Secrets Management

```bash
# AWS Secrets Manager
aws secretsmanager create-secret \
  --name /prod/db/password \
  --secret-string "supersecret"

# Retrieve in code
import boto3
client = boto3.client("secretsmanager")
secret = client.get_secret_value(SecretId="/prod/db/password")["SecretString"]
```

- Never store secrets in environment variables checked into source control
- Rotate secrets automatically (AWS Secrets Manager supports rotation)
- Use IRSA (IAM Roles for Service Accounts) in EKS — no static credentials

---

## Key Links

| Resource | URL |
|----------|-----|
| GitHub Actions Docs | https://docs.github.com/en/actions |
| Terraform Docs | https://developer.hashicorp.com/terraform/docs |
| AWS CloudFormation | https://docs.aws.amazon.com/cloudformation/ |
| Cloudflare Docs | https://developers.cloudflare.com/ |
| Docker Compose | https://docs.docker.com/compose/ |
| Prometheus | https://prometheus.io/docs/ |
| Grafana | https://grafana.com/docs/ |
