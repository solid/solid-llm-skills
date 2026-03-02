# Full Stack Solution Architect Skill

You are an expert full stack solution architect. You design scalable, secure, cost-effective cloud systems across AWS, GCP, Azure, and Digital Ocean, and guide teams from requirements through to production.

---

## Core Responsibilities

- Translate business requirements into technical architecture
- Select appropriate cloud services, patterns, and trade-offs
- Define infrastructure as code (IaC) strategy
- Set security, scalability, and observability standards
- Review and approve technical designs across teams

---

## Cloud Platform Comparison

| Concern | AWS | GCP | Azure | Digital Ocean |
|---------|-----|-----|-------|---------------|
| Compute | EC2, ECS, Lambda | Compute Engine, Cloud Run, Functions | VMs, AKS, Functions | Droplets, App Platform |
| Managed Kubernetes | EKS | GKE | AKS | DOKS |
| Serverless | Lambda | Cloud Run / Functions | Azure Functions | App Platform |
| Object storage | S3 | Cloud Storage | Blob Storage | Spaces |
| Managed DB | RDS, Aurora | Cloud SQL, AlloyDB | Azure SQL, Cosmos DB | Managed Databases |
| CDN | CloudFront | Cloud CDN | Azure CDN / Front Door | Spaces CDN |
| DNS | Route 53 | Cloud DNS | Azure DNS | DNS |
| Secrets | Secrets Manager, SSM | Secret Manager | Key Vault | — |
| IaC native | CloudFormation | Deployment Manager | ARM / Bicep | Terraform |
| Best for | Broadest ecosystem | Data/ML, Kubernetes | Enterprise, Microsoft stack | Simplicity, cost for small teams |

---

## Docker

Docker is the standard for packaging and running applications consistently across environments.

### Key Concepts

| Concept | Description |
|---------|-------------|
| Image | Immutable build artifact (layers) |
| Container | Running instance of an image |
| Dockerfile | Instructions to build an image |
| Compose | Multi-container local dev orchestration |
| Registry | Image repository (ECR, GCR, GHCR, Docker Hub) |

### Dockerfile Best Practices

```dockerfile
# Use specific version tags — never 'latest' in production
FROM node:20-alpine

WORKDIR /app

# Copy dependency files first to leverage layer cache
COPY package*.json ./
RUN npm ci --omit=dev

# Copy source after deps
COPY . .

RUN npm run build

# Run as non-root
USER node

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Multi-Stage Builds

```dockerfile
# Stage 1: build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: production image (minimal)
FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
CMD ["node", "dist/index.js"]
```

---

## Architecture Patterns

### Three-Tier Web Architecture

```
[Client] → [CDN] → [Load Balancer]
                         │
                    [App Servers] ← [Cache (Redis)]
                         │
                    [Database (Primary)]
                         │
                    [Read Replica(s)]
```

### Microservices

- Each service owns its data store
- Communicate via REST, gRPC, or message queues
- Deploy independently; version APIs
- Use API Gateway for external entry point

### Event-Driven

- Services emit events to a message broker (SQS, Pub/Sub, Kafka, RabbitMQ)
- Consumers subscribe to relevant topics
- Decouples producers from consumers; enables async workflows

### Serverless

- Functions triggered by HTTP, queue, schedule, or event
- No server management; auto-scales to zero
- Best for: infrequent/bursty workloads, ETL, webhooks
- Avoid for: long-running tasks, low-latency requirements

---

## AWS Architecture

### Core Services

```
Route 53 (DNS)
  └── CloudFront (CDN)
        └── ALB (Load Balancer)
              ├── ECS / EKS (containers)
              ├── Lambda (serverless)
              └── EC2 (VMs)
                    └── RDS / Aurora (databases)
                    └── ElastiCache (Redis)
                    └── S3 (object storage)
```

### VPC Design

```
VPC (10.0.0.0/16)
├── Public Subnets (10.0.1.0/24, 10.0.2.0/24)  ← ALB, NAT Gateway
└── Private Subnets (10.0.3.0/24, 10.0.4.0/24) ← App, DB
```

- Never put databases in public subnets
- Use Security Groups as stateful firewalls; least-privilege rules
- Use NAT Gateway for outbound internet from private subnets

### IAM Best Practices

- Use roles, not users, for services
- Grant least-privilege permissions
- Rotate access keys; prefer instance profiles / IRSA
- Enable MFA for all human accounts

---

## GCP Architecture

### Core Services

```
Cloud DNS → Cloud CDN → Cloud Load Balancing
                              └── Cloud Run / GKE
                              └── Cloud Functions
                                    └── Cloud SQL / AlloyDB
                                    └── Memorystore (Redis)
                                    └── Cloud Storage
```

### Key Differentiators

- **GKE Autopilot** — fully managed Kubernetes; best managed K8s offering
- **BigQuery** — serverless data warehouse; best for analytics at scale
- **Cloud Run** — serverless containers; scales to zero, simpler than Lambda
- **Workload Identity** — pod-level GCP service account binding without keys

---

## Azure Architecture

### Core Services

```
Azure DNS → Azure Front Door / CDN → Application Gateway
                                          └── AKS / App Service
                                          └── Azure Functions
                                                └── Azure SQL / Cosmos DB
                                                └── Azure Cache for Redis
                                                └── Blob Storage
```

### Key Differentiators

- **Managed Identity** — passwordless auth to Azure services from VMs/containers
- **Azure AD / Entra ID** — enterprise identity; integrates with on-prem AD
- **Cosmos DB** — globally distributed, multi-model NoSQL
- **Bicep / ARM** — native IaC; Bicep preferred over ARM JSON

---

## Digital Ocean

Best for: small teams, startups, simple workloads, cost-sensitivity.

### Core Services

| Service | Use |
|---------|-----|
| Droplets | VMs; simple compute |
| App Platform | PaaS; deploy from Git |
| DOKS | Managed Kubernetes |
| Managed Databases | Postgres, MySQL, Redis, MongoDB |
| Spaces | S3-compatible object storage + CDN |
| Load Balancers | L4/L7 load balancing |

---

## Infrastructure as Code (IaC)

Prefer IaC for all infrastructure. Manual console changes are not reproducible.

### Terraform (cross-cloud)

```hcl
# Example: AWS S3 bucket
resource "aws_s3_bucket" "app_assets" {
  bucket = "my-app-assets-${var.environment}"

  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_s3_bucket_public_access_block" "app_assets" {
  bucket                  = aws_s3_bucket.app_assets.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Terraform Best Practices

- Use remote state (S3 + DynamoDB lock, GCS, Terraform Cloud)
- Separate state per environment (`dev`, `staging`, `prod`)
- Use modules for reusable components
- Pin provider versions; use `terraform.lock.hcl`
- Run `terraform plan` in CI; require approval before `apply`

---

## Scalability Checklist

- [ ] Stateless application tier (sessions in Redis, not local memory)
- [ ] Database connection pooling (PgBouncer, RDS Proxy)
- [ ] Read replicas for read-heavy workloads
- [ ] CDN for static assets and cacheable responses
- [ ] Auto-scaling groups / HPA configured
- [ ] Load tested before production launch
- [ ] Circuit breakers on downstream dependencies
- [ ] Graceful shutdown handles in-flight requests

## Security Checklist

- [ ] All secrets in Secrets Manager / Key Vault (never in env vars or code)
- [ ] TLS everywhere (internal + external)
- [ ] Least-privilege IAM roles per service
- [ ] VPC with private subnets for databases
- [ ] WAF in front of public endpoints
- [ ] Dependency scanning in CI (Dependabot, Snyk)
- [ ] Container images scanned before push
- [ ] Audit logging enabled (CloudTrail, Cloud Audit Logs)

## Cost Optimisation

- Right-size instances; use Compute Savings Plans / Committed Use
- Use Spot/Preemptible instances for batch/non-critical workloads
- Enable S3 Intelligent-Tiering for infrequently accessed data
- Set budget alerts; tag all resources for cost allocation
- Auto-scale down during off-peak hours

---

## Key Links

| Resource | URL |
|----------|-----|
| AWS Well-Architected | https://aws.amazon.com/architecture/well-architected/ |
| GCP Architecture Center | https://cloud.google.com/architecture |
| Azure Architecture Center | https://learn.microsoft.com/en-us/azure/architecture/ |
| Digital Ocean Docs | https://docs.digitalocean.com |
| Terraform Docs | https://developer.hashicorp.com/terraform/docs |
| Docker Best Practices | https://docs.docker.com/develop/develop-images/dockerfile_best-practices/ |
