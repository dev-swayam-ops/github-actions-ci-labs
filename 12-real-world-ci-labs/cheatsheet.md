# Module 12 Cheatsheet: Real-World CI/CD Labs

## Quick Reference Tables

### Deployment Strategies Comparison

| Strategy | Downtime | Rollback | Risk | Setup Complexity |
|----------|----------|----------|------|-----------------|
| **Blue-Green** | Zero | Instant | Low | Medium |
| **Canary** | Zero | Gradual | Low | High |
| **Rolling** | Zero | Slow | Medium | High |
| **Recreate** | Yes | Fast | High | Low |
| **Shadow** | Zero | Instant | Very Low | Very High |

---

### Environment Promotion Workflow

```
Development (Auto) → Staging (Auto) → Production (Manual Approval)
├─ deploy-dev       ├─ deploy-staging    ├─ manual gate
├─ always live      ├─ smoke tests       ├─ approval required
└─ fast feedback    └─ validation        └─ prod deployment
```

**Branch mapping:**
```
develop branch  → Development environment (auto)
main branch     → Staging environment (auto)
main branch     → Production environment (manual approval)
```

---

### Conditional Deployment Patterns

| Scenario | Condition | Result |
|----------|-----------|--------|
| Main branch, tests pass | `success() && github.ref == 'refs/heads/main'` | Deploy |
| Main branch, tests fail | `failure()` | Skip deploy |
| Feature branch | `github.ref != 'refs/heads/main'` | Skip deploy |
| Tag pushed | `startsWith(github.ref, 'refs/tags/')` | Deploy release |
| Manual override | `github.event_name == 'workflow_dispatch'` | Deploy |

---

### Service Container Configuration

#### PostgreSQL Example

```yaml
services:
  postgres:
    image: postgres:15-alpine
    env:
      POSTGRES_PASSWORD: testpass
      POSTGRES_DB: testdb
    options: >-
      --health-cmd pg_isready
      --health-interval 10s
      --health-timeout 5s
      --health-retries 5
    ports:
      - 5432:5432
```

**Connection string:**
```
postgresql://postgres:testpass@postgres:5432/testdb
```

**Environment variable:**
```yaml
env:
  DATABASE_URL: ${{ env.DB_URL }}
```

---

#### MySQL Example

```yaml
services:
  mysql:
    image: mysql:8.0
    env:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: testdb
    options: >-
      --health-cmd="mysqladmin ping"
      --health-interval=10s
    ports:
      - 3306:3306
```

**Connection:**
```
mysql://root:rootpass@mysql:3306/testdb
```

---

#### Redis Example

```yaml
services:
  redis:
    image: redis:7-alpine
    options: >-
      --health-cmd "redis-cli ping"
    ports:
      - 6379:6379
```

**Connection:**
```
redis://redis:6379
```

---

### Docker Image Tagging Strategies

#### Strategy 1: Semantic Versioning

```yaml
tags: |
  ghcr.io/user/app:latest
  ghcr.io/user/app:v1.2.3
  ghcr.io/user/app:v1.2
  ghcr.io/user/app:v1
```

**Usage:** Release channels, version tracking

---

#### Strategy 2: Branch-based

```yaml
tags: |
  ghcr.io/user/app:main
  ghcr.io/user/app:develop
  ghcr.io/user/app:staging
```

**Usage:** Environment-specific images

---

#### Strategy 3: SHA-based

```yaml
tags: |
  ghcr.io/user/app:sha-a1b2c3d
  ghcr.io/user/app:latest
```

**Usage:** Exact commit tracking, rollback

---

#### Strategy 4: Combined

```yaml
tags: |
  ghcr.io/user/app:latest
  ghcr.io/user/app:main
  ghcr.io/user/app:v1.2.3
  ghcr.io/user/app:sha-a1b2c3d
```

**Usage:** Full flexibility

---

### Health Check Patterns

#### HTTP Health Endpoint

```yaml
- name: Health Check
  run: |
    curl -f http://localhost:3000/health \
      --max-time 5 \
      --retry 10 \
      --retry-delay 2 || exit 1
```

**Endpoint returns:**
```json
{
  "status": "ok",
  "timestamp": "2024-01-15T10:30:00Z",
  "version": "1.2.3"
}
```

---

#### Port Check

```bash
# Check if port is open
lsof -i :3000

# TCP connection test
nc -zv localhost 3000
```

---

#### Database Health

```bash
# PostgreSQL
psql -h localhost -U postgres -c "SELECT 1;"

# MySQL
mysqladmin -h localhost -u root ping

# Redis
redis-cli ping
```

---

### Approval Gate Configuration

#### GitHub Environments

```yaml
jobs:
  deploy-prod:
    environment:
      name: production
      url: https://myapp.com
    steps:
      - run: deploy
```

**In GitHub Settings:**
1. Settings → Environments → production
2. Add required reviewers
3. Set deployment branches
4. Configure secrets per environment

---

#### Manual Approval Example

```yaml
deploy-prod:
  needs: deploy-staging
  environment:
    name: production
    url: https://myapp.com
  steps:
    # Job waits for approval here
    - name: Deploy
      run: ./deploy.sh
```

**Behavior:**
- Workflow pauses after `needs:` completes
- GitHub sends approval request
- Authorized reviewers approve/reject
- Job continues or stops

---

### Secrets Management

#### Set Secrets in GitHub

```
Settings → Secrets and Variables → Actions → New repository secret

SECRET_NAME: secret_value
```

#### Use Secrets in Workflow

```yaml
env:
  API_KEY: ${{ secrets.API_KEY }}

steps:
  - name: Deploy
    run: deploy --key ${{ secrets.API_KEY }}
```

#### Environment-Specific Secrets

```yaml
jobs:
  deploy-prod:
    environment: production
    steps:
      - run: deploy
        env:
          DB_PASSWORD: ${{ secrets.PROD_DB_PASSWORD }}

  deploy-staging:
    environment: staging
    steps:
      - run: deploy
        env:
          DB_PASSWORD: ${{ secrets.STAGING_DB_PASSWORD }}
```

---

### Notification Templates

#### Slack Success Message

```json
{
  "text": "✅ Deployment Successful",
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Deployment Complete*\nApp: ${{ github.repository }}\nBranch: ${{ github.ref_name }}\nAuthor: ${{ github.actor }}"
      }
    }
  ]
}
```

---

#### Slack Failure Alert

```json
{
  "text": "❌ Deployment Failed",
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*⚠️ Deployment Failed*\nAction required from: ${{ github.actor }}"
      }
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": {"type": "plain_text", "text": "View Logs"},
          "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
        }
      ]
    }
  ]
}
```

---

## Common Patterns

### Multi-Region Deployment

```yaml
deploy-prod:
  strategy:
    matrix:
      region: [us-east-1, eu-west-1, ap-southeast-1]
  steps:
    - name: Deploy to ${{ matrix.region }}
      run: deploy --region ${{ matrix.region }}
```

---

### Database Migrations

```yaml
steps:
  - name: Run migrations
    run: npm run migrate:up
    env:
      DATABASE_URL: ${{ secrets.DATABASE_URL }}

  - name: Verify migrations
    run: npm run migrate:verify

  - name: Rollback on failure
    if: failure()
    run: npm run migrate:down
```

---

### Artifact-Based Deployments

```yaml
build:
  steps:
    - run: npm run build
    - uses: actions/upload-artifact@v3
      with:
        name: dist
        path: dist/

deploy:
  needs: build
  steps:
    - uses: actions/download-artifact@v3
      with:
        name: dist
        path: dist/
    - run: deploy
```

---

### Smoke Tests

```yaml
steps:
  - name: Deploy
    run: npm run deploy

  - name: Smoke tests
    run: |
      curl -f https://myapp.com/health
      curl -f https://myapp.com/api/status
      curl -f https://myapp.com/api/data
```

---

## Environment Variables Reference

### GitHub Context Variables

```
${{ github.repository }}       # owner/repo
${{ github.ref }}              # refs/heads/main
${{ github.ref_name }}         # main (branch name)
${{ github.sha }}              # commit SHA
${{ github.actor }}            # user who triggered
${{ github.event_name }}       # push, pull_request, etc.
${{ github.server_url }}       # https://github.com
```

### Runner Variables

```
${{ runner.os }}               # Linux, Windows, macOS
${{ runner.arch }}             # x64, arm64
${{ runner.name }}             # GitHub Runner 1
```

### Job Variables

```
${{ job.status }}              # success, failure, cancelled
${{ job.outputs.key }}         # Access job outputs
```

---

## Troubleshooting Guide

### Deployment Won't Start

```
❌ Problem: Deploy job never runs
✅ Check:
  - needs: job_name correct?
  - if: condition true?
  - Previous job passed?
  - Branch filtering correct?
```

### Secrets Not Available

```
❌ Problem: ${{ secrets.KEY }} is blank
✅ Solutions:
  - Verify secret exists in Settings
  - Check organization secrets (org/settings/secrets)
  - Use correct environment context
  - Secrets can't be read in pull requests from forks
```

### Service Container Not Connecting

```
❌ Problem: Connection refused
✅ Solutions:
  - Add health checks to service
  - Use service name as hostname (postgres, mysql, etc.)
  - Check ports are exposed
  - Verify environment variables
```

### Approval Not Requested

```
❌ Problem: Job doesn't wait for approval
✅ Check:
  - environment: name configured
  - Required reviewers set in GitHub
  - Job has needs: that completes first
  - Correct branch for deployment
```

### Docker Push Fails

```
❌ Problem: Permission denied
✅ Solutions:
  - Login before push
  - Check registry credentials
  - Verify ${{ secrets.GITHUB_TOKEN }} available
  - Check permissions in GitHub
```

---

## Performance Optimization

### Parallel Deployments

```yaml
jobs:
  deploy-us:
    steps:
      - run: deploy --region us-east-1

  deploy-eu:
    steps:
      - run: deploy --region eu-west-1

  # Both regions deploy in parallel
```

**Time saved:** Regions deploy simultaneously

---

### Skip Redundant Steps

```yaml
deploy:
  if: github.ref == 'refs/heads/main'
  # Skip for feature branches
```

---

### Conditional Artifact Download

```yaml
deploy:
  steps:
    - uses: actions/download-artifact@v3
      if: github.event_name == 'push'
      # Skip for pull requests
```

---

## Security Best Practices

### ✅ DO

```yaml
# ✓ Use secrets for sensitive data
env:
  API_KEY: ${{ secrets.API_KEY }}

# ✓ Limit secret exposure
- name: Deploy
  run: |
    echo "Deploying..."
    deploy  # API_KEY passed, not echoed

# ✓ Use environment-specific secrets
environment: production
env:
  DB_PASSWORD: ${{ secrets.PROD_DB_PASSWORD }}
```

### ❌ DON'T

```yaml
# ✗ Don't hardcode secrets
API_KEY: "sk_live_abc123"

# ✗ Don't echo secrets
- run: echo ${{ secrets.API_KEY }}

# ✗ Don't log secrets
- run: deploy --key ${{ secrets.API_KEY }}
  # Will show in logs if command fails
```

---

## Quick Start Templates

### Basic CI/CD Pipeline

```yaml
name: CI/CD

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci && npm test

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v3
      - run: npm ci && npm run deploy
```

---

### Full Enterprise Pipeline

```yaml
name: Enterprise Pipeline

on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm ci && npm run lint && npm test

  build:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm ci && npm run build
      - uses: docker/build-push-action@v4
        with:
          push: true

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - run: deploy --env staging

  deploy-prod:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: deploy --env production
```

---

## CLI Commands Reference

```bash
# Login to registry
echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin

# Push Docker image
docker push ghcr.io/${{ github.repository }}:latest

# Deploy with SSH
ssh -i ${{ secrets.SSH_KEY }} user@server "cd /app && git pull && npm install && npm run build"

# Health check
curl -f https://myapp.com/health --max-time 5 --retry 10 --retry-delay 2

# Database migration
npm run migrate:up -- --env production

# Slack notification
curl -X POST ${{ secrets.SLACK_WEBHOOK }} -d '{"text":"Deployed"}'
```

---

## Resources

- [GitHub Actions Docs](https://docs.github.com/actions)
- [Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments)
- [Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [Docker Build Push Action](https://github.com/docker/build-push-action)
- [Slack GitHub Action](https://github.com/slackapi/slack-github-action)

