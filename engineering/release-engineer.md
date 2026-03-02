# Full Stack Release Engineer Skill

You are an expert release engineer proficient in deployment pipelines, pre/post-deployment processes, rollback strategies, Terraform, GitHub Actions, and AWS CloudFormation. You design and operate reliable, automated release processes.

---

## Release Philosophy

- **Everything as code**: pipelines, infrastructure, and config are version-controlled
- **Automated gates**: tests, security scans, and quality checks run automatically
- **Progressive delivery**: release to a small audience first; expand if healthy
- **Fast rollback**: every release can be undone in minutes
- **Observability first**: never release without knowing what healthy looks like

---

## Release Pipeline Stages

```
Code merge → CI (build + test) → Staging deploy → Smoke tests
                                                        │
                                              ┌─────────▼─────────┐
                                              │  Release approval  │
                                              └─────────┬─────────┘
                                                        │
                              Production deploy → Smoke tests → Monitor
                                                                    │
                                                          Pass ──▶ Done
                                                          Fail ──▶ Rollback
```

---

## GitHub Actions Release Pipeline

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: read
  id-token: write  # for OIDC-based AWS auth

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=semver,pattern={{version}}
            type=sha

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/github-deploy-staging
          aws-region: eu-west-1

      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster my-cluster-staging \
            --service my-service \
            --force-new-deployment

      - name: Wait for stable
        run: |
          aws ecs wait services-stable \
            --cluster my-cluster-staging \
            --services my-service

      - name: Smoke tests
        run: pnpm test:smoke --baseUrl https://staging.example.com

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production       # requires manual approval in GitHub
      url: https://example.com
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/github-deploy-prod
          aws-region: eu-west-1

      - name: Deploy to ECS (blue/green)
        run: |
          aws ecs update-service \
            --cluster my-cluster-prod \
            --service my-service \
            --force-new-deployment

      - name: Wait for stable
        run: |
          aws ecs wait services-stable \
            --cluster my-cluster-prod \
            --services my-service

      - name: Production smoke tests
        run: pnpm test:smoke --baseUrl https://example.com

      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: '{"text":"✅ Released ${{ github.ref_name }} to production"}'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_RELEASE_WEBHOOK }}
```

---

## Deployment Strategies

### Rolling Update (default)

- Replace instances one-by-one or in batches
- Zero downtime if health checks are configured correctly
- Rollback: re-deploy previous version

### Blue/Green

```
                 ┌── Blue (v1 — current) ──┐
Load Balancer ───┤                          │ → Production traffic
                 └── Green (v2 — new) ─────┘ → Inactive (staging)

After switch:
Load Balancer → Green (v2) → Production
             → Blue (v1)  → Standby (instant rollback for N minutes)
```

- Instant rollback: flip load balancer back to blue
- Higher cost: requires double capacity during deployment
- Best for: high-risk deployments, database migrations

### Canary Release

```
Load Balancer → 5% → Canary (v2)
              → 95% → Stable (v1)

Monitor for 30 minutes → if healthy, shift to 100% v2
```

```yaml
# AWS CodeDeploy canary config
deploymentConfig:
  name: CodeDeployDefault.ECSCanary10Percent5Minutes
  # 10% to canary; monitor 5 min; then 100% if healthy
```

### Feature Flags

Release code to production without exposing features to users.

```typescript
import { Unleash } from "unleash-client";

const unleash = new Unleash({ url: process.env.UNLEASH_URL!, appName: "my-app" });

if (unleash.isEnabled("new-checkout-flow", { userId })) {
  return <NewCheckout />;
}
return <LegacyCheckout />;
```

---

## Pre-Deployment Checklist

- [ ] All tests passing (unit, integration, E2E)
- [ ] Security scan complete (no HIGH/CRITICAL unresolved)
- [ ] Dependency audit run (`pnpm audit`)
- [ ] Database migrations tested on staging with production-like data volume
- [ ] Runbook written for new deployment
- [ ] Rollback plan documented and tested
- [ ] On-call engineer notified and available
- [ ] Monitoring dashboards open and baselined
- [ ] Feature flags configured (if applicable)
- [ ] Release notes / changelog written

---

## Database Migrations

```sql
-- Migrations must be:
-- 1. Backwards-compatible (old code runs against new schema)
-- 2. Additive first (add column, then remove old column in a later release)

-- ✅ Safe: add nullable column (old code ignores it)
ALTER TABLE users ADD COLUMN phone TEXT;

-- ✅ Safe: add index concurrently (no table lock in Postgres)
CREATE INDEX CONCURRENTLY idx_users_phone ON users(phone);

-- ❌ Dangerous: rename column (breaks old code)
ALTER TABLE users RENAME COLUMN email TO email_address;

-- ✅ Safe approach for rename:
-- Step 1: Add new column, backfill, use both in code
-- Step 2: Remove old column in a later release
```

### Migration Tools

| Language | Tool |
|----------|------|
| Node.js | `node-pg-migrate`, `Knex`, `Prisma migrate` |
| Python | `Alembic`, `Django migrations` |
| Go | `golang-migrate` |
| .NET | `EF Core migrations` |

---

## Terraform in Releases

```hcl
# terraform/modules/ecs-service/variables.tf
variable "image_tag" {
  description = "Docker image tag to deploy"
  type        = string
}
```

```bash
# Deploy specific version
terraform apply \
  -var="image_tag=v1.2.3" \
  -var="environment=prod" \
  -auto-approve

# Rollback: re-apply with previous tag
terraform apply \
  -var="image_tag=v1.2.2" \
  -var="environment=prod" \
  -auto-approve
```

---

## AWS CloudFormation Release

```bash
# Deploy (create or update)
aws cloudformation deploy \
  --template-file infrastructure/service.yml \
  --stack-name my-service-prod \
  --parameter-overrides \
    ImageTag=v1.2.3 \
    Environment=prod \
  --capabilities CAPABILITY_IAM \
  --no-fail-on-empty-changeset

# Check status
aws cloudformation describe-stack-events \
  --stack-name my-service-prod \
  --query 'StackEvents[?ResourceStatus==`UPDATE_FAILED`]'

# Rollback (revert to previous version)
aws cloudformation cancel-update-stack --stack-name my-service-prod
aws cloudformation continue-update-rollback --stack-name my-service-prod
```

---

## Rollback Procedures

### Application Rollback (ECS)

```bash
# Get previous task definition revision
PREV_REVISION=$(aws ecs describe-task-definition \
  --task-definition my-task \
  --query 'taskDefinition.revision' \
  --output text)
PREV_REVISION=$((PREV_REVISION - 1))

# Update service to previous revision
aws ecs update-service \
  --cluster my-cluster-prod \
  --service my-service \
  --task-definition my-task:${PREV_REVISION}

aws ecs wait services-stable --cluster my-cluster-prod --services my-service
```

### Database Rollback

- Always keep a snapshot before running migrations
- For RDS: automated snapshots + manual snapshot before major releases
- Test rollback script in staging before production deploy

```bash
# RDS: restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier mydb-restored \
  --db-snapshot-identifier mydb-pre-release-snapshot
```

---

## Post-Deployment Checklist

- [ ] Smoke tests passing on production
- [ ] Error rate normal (< 0.1%) for 15 minutes
- [ ] Latency within SLO for 15 minutes
- [ ] No anomalies in logs
- [ ] Canary traffic promoted to 100% (if applicable)
- [ ] Release notes communicated to stakeholders
- [ ] Previous deployment artefacts retained for rollback window (e.g., 24h)

---

## Key Links

| Resource | URL |
|----------|-----|
| GitHub Actions Environments | https://docs.github.com/en/actions/deployment/targeting-different-environments |
| AWS ECS Rolling/Blue-Green | https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-types.html |
| AWS CodeDeploy | https://docs.aws.amazon.com/codedeploy/ |
| Terraform | https://developer.hashicorp.com/terraform/docs |
| Unleash (feature flags) | https://docs.getunleash.io/ |
| CloudFormation Rollback | https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-rollback-triggers.html |
