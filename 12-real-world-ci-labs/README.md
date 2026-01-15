# Module 12: Real-World CI/CD Labs

## Overview

This module provides comprehensive hands-on labs covering real-world CI/CD scenarios. You'll learn to build enterprise-grade pipelines combining multiple GitHub Actions concepts with practical production patterns.

## What You'll Learn

### Core Concepts

1. **End-to-End Pipelines** - Complete application workflow from code to production
2. **Multi-Environment Deployments** - Dev, staging, and production environments
3. **Deployment Strategies** - Blue-green, canary, and rolling deployments
4. **Approval Gates** - Manual approvals for production deployments
5. **Service Dependencies** - Testing with real database services
6. **Docker Integration** - Building and pushing container images
7. **Health Checks** - Validating deployments with health endpoints
8. **Failure Handling** - Rollback procedures and error recovery
9. **Notifications** - Alerting teams on deployment status
10. **Monitoring** - Tracking pipeline execution and deployment metrics

## Learning Path

### Beginner Topics
- Static website deployments to GitHub Pages
- Basic health checks and validation
- Simple notification patterns
- Environment configuration

### Intermediate Topics
- Service container integration with databases
- Docker image building and registry push
- Multi-environment promotion workflows
- Conditional deployment logic

### Advanced Topics
- Canary deployment strategy with gradual rollout
- Blue-green deployment with instant rollback
- Enterprise-grade pipeline architecture
- Production resilience patterns

## Hands-On Labs

### Lab 1: Node.js Web Application Pipeline

**Scenario:** Deploy a Node.js web application with full CI/CD pipeline

**Architecture:**
```
Code Push
  ↓
Quality Checks (lint, test)
  ↓
Build Artifacts
  ↓
Security Scanning
  ↓
Docker Image Build
  ↓
Deploy to Development (auto)
  ↓
Deploy to Staging (auto on main)
  ↓
Manual Approval
  ↓
Deploy to Production
  ↓
Health Checks & Monitoring
```

**Complete Workflow:**

```yaml
name: Node.js CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/myapp

jobs:
  # ═══════════════════════════════════════════════════════════
  # PHASE 1: CODE QUALITY
  # ═══════════════════════════════════════════════════════════
  
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Lint code
        run: npm run lint
      
      - name: Run tests
        run: npm test -- --coverage
      
      - name: Upload coverage
        uses: actions/upload-artifact@v3
        with:
          name: coverage
          path: coverage/
          retention-days: 7

  # ═══════════════════════════════════════════════════════════
  # PHASE 2: SECURITY SCANNING
  # ═══════════════════════════════════════════════════════════
  
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run npm audit
        run: npm audit --audit-level=moderate
      
      - name: CodeQL analysis
        uses: github/codeql-action/init@v2
        with:
          languages: 'javascript'
      
      - name: Perform analysis
        uses: github/codeql-action/analyze@v2

  # ═══════════════════════════════════════════════════════════
  # PHASE 3: BUILD ARTIFACTS
  # ═══════════════════════════════════════════════════════════
  
  build:
    needs: [quality, security]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install and build
        run: |
          npm ci
          npm run build
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: dist
          path: dist/
          retention-days: 7
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Log in to registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and push image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ═══════════════════════════════════════════════════════════
  # PHASE 4: DEPLOYMENT TO DEV
  # ═══════════════════════════════════════════════════════════
  
  deploy-dev:
    needs: build
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: development
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/download-artifact@v3
        with:
          name: dist
          path: dist/
      
      - name: Deploy to development
        run: |
          echo "🚀 Deploying to development..."
          echo "Branch: ${{ github.ref_name }}"
          echo "Commit: ${{ github.sha }}"
          # Deployment command here
      
      - name: Health check
        run: |
          echo "🏥 Checking health..."
          sleep 2
          curl -f http://dev.myapp.com/health || exit 1
          echo "✓ Dev deployment healthy"

  # ═══════════════════════════════════════════════════════════
  # PHASE 5: DEPLOYMENT TO STAGING
  # ═══════════════════════════════════════════════════════════
  
  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/download-artifact@v3
        with:
          name: dist
          path: dist/
      
      - name: Deploy to staging
        run: |
          echo "🚀 Deploying to staging..."
          echo "Branch: ${{ github.ref_name }}"
          echo "Commit: ${{ github.sha }}"
          # Deployment command here
      
      - name: Run smoke tests
        run: |
          echo "🧪 Running smoke tests..."
          sleep 2
          curl -f https://staging.myapp.com/health || exit 1
          curl -f https://staging.myapp.com/api/status || exit 1
          echo "✓ Staging tests passed"

  # ═══════════════════════════════════════════════════════════
  # PHASE 6: DEPLOYMENT TO PRODUCTION (MANUAL APPROVAL)
  # ═══════════════════════════════════════════════════════════
  
  deploy-prod:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/download-artifact@v3
        with:
          name: dist
          path: dist/
      
      - name: Deploy to production
        run: |
          echo "🚀 DEPLOYING TO PRODUCTION"
          echo "Branch: ${{ github.ref_name }}"
          echo "Commit: ${{ github.sha }}"
          echo "Triggered by: ${{ github.actor }}"
          # Production deployment command here
      
      - name: Health check
        run: |
          echo "🏥 Checking production health..."
          sleep 3
          curl -f https://myapp.com/health || exit 1
          echo "✓ Production deployment successful"
      
      - name: Notify deployment
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "✅ Production Deployment Successful",
              "blocks": [{
                "type": "section",
                "text": {"type": "mrkdwn", "text": "*✅ Deployed to Production*\nVersion: ${{ github.sha }}\nAuthor: ${{ github.actor }}"}
              }]
            }
```

**Key Features:**
- ✓ Lint and test quality checks
- ✓ Security scanning (npm audit, CodeQL)
- ✓ Docker image building and caching
- ✓ Development auto-deployment
- ✓ Staging auto-deployment with tests
- ✓ Production manual approval gate
- ✓ Health checks after each deployment
- ✓ Slack notifications

## Real-World Scenarios

### Scenario 1: Blue-Green Deployment
Deploy new version alongside existing version, test, then switch traffic.
- **File:** See Exercise 8 in exercises.md
- **Benefits:** Instant rollback, zero downtime
- **Complexity:** Medium

### Scenario 2: Canary Deployment
Gradually roll out new version to percentage of users, monitor metrics.
- **File:** See Exercise 7 in exercises.md
- **Benefits:** Early error detection, safe rollout
- **Complexity:** High

### Scenario 3: Database Migrations
Safely run migrations with rollback capability.
- **Pattern:** Run migrations before deployment
- **Rollback:** Automatic on failure

### Scenario 4: Service Dependencies
Test against real services (databases, caches, etc.).
- **Pattern:** Service containers in GitHub Actions
- **Benefits:** Production-like testing

## Common Deployment Patterns

### Pattern 1: Sequential Promotion
```
Code → Test → Dev → Staging → Production
```

### Pattern 2: Parallel Environments
```
Code → Test → {Dev, Staging, Production}
```

### Pattern 3: Approval-Based
```
Code → Test → Staging → [Manual Approval] → Production
```

### Pattern 4: Feature Branch
```
Feature Branch → Staging
Main Branch → Production
```

## Environment Management

### Development
- Auto-deploy on `develop` branch
- Fast feedback
- Minimal validation
- Resource constraints acceptable

### Staging
- Auto-deploy on `main` branch
- Production-like environment
- Smoke tests required
- Load testing optional

### Production
- Manual approval required
- Restricted deployment windows
- Health checks mandatory
- Monitoring required
- Rollback capability

## Failure Handling

### Health Check Failure
```yaml
if: failure()
  then: Automatic rollback to previous version
```

### Database Migration Failure
```yaml
if: migration fails
  then: Automatic rollback
  and: Block deployment
```

### Approval Timeout
```yaml
if: approval not granted within 24 hours
  then: Workflow expires
```

## Monitoring and Observability

### Deployment Metrics
- Deployment frequency
- Lead time for changes
- Mean time to recovery
- Change failure rate

### Health Monitoring
- API availability
- Response times
- Error rates
- Database connectivity

## Best Practices

1. **Approval Gates** - Always require approval for production
2. **Health Checks** - Validate every deployment
3. **Rollback Plan** - Always have rollback strategy
4. **Notifications** - Alert teams on all deployments
5. **Testing** - Test before promotion to next environment
6. **Monitoring** - Monitor after deployment
7. **Secrets** - Use environment-specific secrets
8. **Artifacts** - Keep artifacts for rollback
9. **Documentation** - Document deployment procedures
10. **Runbooks** - Create incident response guides

## Exercises

This module includes 10 comprehensive exercises:

1. Static website deployment
2. Database integration testing
3. Docker image building
4. Multi-environment deployments
5. Health checks and validation
6. Deployment notifications
7. Canary deployment strategy
8. Blue-green deployment
9. Scheduled maintenance
10. Complete enterprise pipeline

See `exercises.md` for detailed instructions.

## Solutions

Complete implementations for all exercises in `solutions.md`:

- Full workflow files
- Step-by-step explanations
- Real-world variations
- Troubleshooting tips

## Quick Reference

See `cheatsheet.md` for:

- Deployment strategy comparison
- Environment promotion patterns
- Service container configurations
- Approval gate setup
- Health check examples
- Docker tagging strategies
- Slack notification templates
- Troubleshooting guide

## Next Steps

1. Start with Lab 1: Node.js Web Application Pipeline
2. Work through exercises 1-5 for fundamentals
3. Explore exercises 6-8 for advanced patterns
4. Complete exercise 10 for full enterprise pipeline
5. Adapt patterns to your specific technology stack

## Key Takeaways

✓ Build production-grade CI/CD pipelines
✓ Implement multiple environment strategy
✓ Deploy with confidence using health checks
✓ Recover quickly using rollback procedures
✓ Notify teams of deployment status
✓ Monitor and validate deployments
✓ Handle failures gracefully
✓ Secure sensitive data with secrets
✓ Scale to multiple services
✓ Achieve high deployment velocity

## Resources

- GitHub Actions Documentation
- Container Registry (ghcr.io)
- Deployment Strategies
- Database Migration Tools
- Health Check Best Practices

## Exercises & Solutions

📝 **exercises.md** - 10 hands-on labs
✅ **solutions.md** - Complete implementations
📋 **cheatsheet.md** - Quick reference guide

---

**Duration:** 3-4 hours for all exercises
**Difficulty:** Advanced
**Prerequisites:** Modules 0-10

Continue building production-ready CI/CD systems!
