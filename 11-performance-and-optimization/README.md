# Module 12: Real-World CI/CD Labs# Module 11: Performance and Optimization






























































































































































































































































































































































- Container Registries: https://docs.docker.com/docker-hub- Monitoring: https://prometheus.io- Infrastructure as Code: https://www.terraform.io- Deployment Strategies: https://martinfowler.com/bliki/DeploymentPipeline.html- GitHub Actions Best Practices: https://docs.github.com/en/actions/guides## Resources---- ✅ Monitoring integrated- ✅ Artifacts built and stored- ✅ Secrets managed securely- ✅ Failure handling implemented- ✅ Deployment validation included- ✅ Manual approval for production- ✅ Security scanning integrated- ✅ Quality gates enforced- ✅ Multi-environment support- ✅ Complete pipeline automated## Summary Checklist---```# Production requires manual approval from Actions tab# Expected: deploy-dev, deploy-staging, deploy-prod jobs rungit push origin maingit merge feature/testgit checkout main# Merge to main# Expected: All checks required before mergegh pr create --base main --head feature/test# Create PR# Expected: quality + security + build jobs rungit push origin feature/testgit checkout -b feature/test# Push to develop```bash### Validation```          echo "✓ Health check passed"          curl -f https://myapp.com/health || exit 1        run: |      - name: Smoke test                echo "✓ Production deployment complete"          sleep 2          echo "Deploying to production: ${{ needs.build.outputs.image }}"        run: |      - name: Deploy              run: echo "Production deployment approved"      - name: Wait for approval    steps:          url: https://myapp.com      name: production    environment:    name: Deploy to Production    runs-on: ubuntu-latest    if: github.ref == 'refs/heads/main'    needs: [quality, build, deploy-staging]  deploy-prod:          echo "✓ Staging deployment complete"          sleep 2          echo "Deploying to staging: ${{ needs.build.outputs.image }}"        run: |      - name: Deploy    steps:          url: https://staging.myapp.com      name: staging    environment:    name: Deploy to Staging    runs-on: ubuntu-latest    if: github.ref == 'refs/heads/main'    needs: [quality, build]  deploy-staging:          echo "✓ Dev deployment complete"          sleep 2          # Simulate deployment          echo "Deploying to dev: ${{ needs.build.outputs.image }}"        run: |      - name: Deploy    steps:          name: development    environment:    name: Deploy to Dev    runs-on: ubuntu-latest    needs: build  deploy-dev:          cache-to: type=gha,mode=max          cache-from: type=gha            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}          tags: |          push: ${{ github.event_name != 'pull_request' }}          context: .        with:        uses: docker/build-push-action@v4        id: image      - name: Build and push                password: ${{ secrets.GITHUB_TOKEN }}          username: ${{ github.actor }}          registry: ${{ env.REGISTRY }}        with:        uses: docker/login-action@v2      - name: Log in to Registry              uses: docker/setup-buildx-action@v2      - name: Set up Docker Buildx            - uses: actions/checkout@v3    steps:          tag: ${{ steps.image.outputs.tag }}      image: ${{ steps.image.outputs.image }}    outputs:          packages: write      contents: read    permissions:    name: Build Docker Image    runs-on: ubuntu-latest    needs: quality  build:          scan-ref: '.'          scan-type: 'fs'        with:      - uses: aquasecurity/trivy-action@master      - run: npm audit      - uses: actions/checkout@v3    steps:        name: Security Scan    runs-on: ubuntu-latest  security:          files: ./coverage/lcov.info        with:        uses: codecov/codecov-action@v3      - name: Upload coverage            - run: npm run build      - run: npm test      - run: npm run lint      - run: npm ci                cache: 'npm'          node-version: '18'        with:      - uses: actions/setup-node@v3      - uses: actions/checkout@v3    steps:        name: Code Quality    runs-on: ubuntu-latest  quality:jobs:  IMAGE_NAME: ${{ github.repository }}/app  REGISTRY: ghcr.ioenv:    branches: [main]  pull_request:    branches: [main, develop]  push:on:name: Node.js Web App Pipeline```yaml`.github/workflows/nodejs-pipeline.yml`:### Complete Workflow```Cloud Hosting (AWS/GCP/Azure)  ↓  └─ Deploy to Production (manual approval)  ├─ Deploy to Staging (auto on main)  ├─ Deploy to Dev  ├─ Build Docker Image  ├─ Lint & TestGitHub Actions (workflow)  ↓GitHub (code push)```### ArchitectureDeploy a Node.js web app through dev → staging → production with full testing.### Scenario## Lab 1: Node.js Web Application Pipeline---- Troubleshooting guide- Validation steps- Expected results- Setup instructions- Complete workflow fileEach lab includes:Provision cloud infrastructure, configure, deploy, monitor.### Lab 3: Infrastructure as Code (Advanced)Deploy multiple services with dependencies and health checks.### Lab 2: Multi-Service Architecture (Intermediate)Deploy a Node.js application with testing and staging.### Lab 1: Node.js Web App (Beginner)## Real-World Labs Overview---```      API_KEY: ${{ secrets.PROD_API_KEY }}      DATABASE_PASSWORD: ${{ secrets.PROD_DB_PASSWORD }}    secrets:      url: https://myapp.com      name: production    environment:  deploy:jobs:```yaml**Pattern: Environment Secrets**### Secrets Management at Scale```    # Only deploy if tests passed and on main    if: success() && github.ref == 'refs/heads/main'  deploy:jobs:```yaml**Pattern 3: Conditional Deployment**```  # Workflow continues, job marked failed  continue-on-error: true- run: npm test```yaml**Pattern 2: Continue with Status**```  # If fails, entire workflow stops- run: npm test```yaml**Pattern 1: Fail Fast**### Failure Handling Patterns- Gradual traffic shift- Zero downtime- Replace instances one at a time**Rolling Deployment:**- Rollback if metrics bad- Gradually increase to 100%- Monitor metrics- Route 5% traffic to new version**Canary Deployment:**- Instant rollback possible- Switch traffic after validation- Old version (blue) + new version (green)**Blue-Green Deployment:**### Deployment Strategies```rollback-ready (keep previous version)  ↓production environment (manual, requires approval)  ↓staging environment (auto deploy)  ↓dev environment (auto deploy)  ↓main branch (latest code)```### Environment Promotion```Monitoring & Alerts  ↓Production Deployment (manual approval)  ↓Staging Deployment (auto)  ↓Dev Deployment  ↓Security Scanning  ↓Build Artifacts  ↓Code Quality (lint, test)  ↓Code Push```**Typical Flow:**### Complete CI/CD Pipeline Architecture## Key Concepts---- Cloud/infrastructure familiarity helpful- Basic deployment knowledge- Understanding of core GitHub Actions concepts- All modules 01-11 completed### Prerequisites8. **Production Resilience** - Zero-downtime deployments, canary releases7. **Cross-Functional Workflows** - QA, deployment, notifications6. **Monitoring & Observability** - Metrics, logs, and alerts5. **Security Integration** - Secrets, signing, vulnerability scanning4. **Error Handling** - Graceful failure management and rollbacks3. **Infrastructure Automation** - Deploying and managing infrastructure2. **Multi-Environment Deployment** - Dev, staging, production environments1. **End-to-End Pipeline** - Complete CI/CD workflows from code to production### What You'll LearnReal-world CI/CD pipelines combine multiple concepts from previous modules into complete, production-ready systems. This module provides comprehensive labs that simulate actual enterprise scenarios: deploying web applications, managing infrastructure, handling multiple environments, and responding to failures.## Overview
## Overview

Optimizing GitHub Actions workflows reduces build times, costs, and resource usage while improving developer experience. This module teaches you how to profile workflows, parallelize jobs, optimize caching, and implement best practices for lightning-fast CI/CD pipelines.

### What You'll Learn

1. **Workflow Profiling** - Identifying performance bottlenecks
2. **Job Parallelization** - Running jobs concurrently
3. **Caching Strategies** - Optimizing dependency and artifact caching
4. **Build Optimization** - Reducing build times through efficiency
5. **Resource Management** - Managing runners and costs
6. **Artifact Optimization** - Minimizing artifact storage
7. **Matrix Strategy Tuning** - Optimizing parallel matrix builds
8. **Monitoring Performance** - Tracking and measuring improvements

### Prerequisites

- GitHub Actions fundamentals (Modules 01-02)
- Workflow syntax knowledge
- Understanding of caching (Module 04)
- Build pipeline experience

---

## Key Concepts

### Workflow Performance Metrics

**Important Metrics:**
- Total execution time (queue + run)
- Job duration
- Step duration
- Artifact upload/download time
- Cache hit rate

**Where to Find Metrics:**
- GitHub Actions tab → Workflow run
- Each job shows total time
- Each step shows duration
- Logs show detailed timing

### Parallelization Patterns

**Sequential Execution (slow):**
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test
      - run: npm run lint
      - run: npm run build  # Waits for previous steps
      - run: docker build . # Waits for all previous
```

**Parallel Execution (fast):**
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
```

### Caching Performance

**Cache Strategy:**
```
First build: 30s (no cache)
Second build: 5s (cache hit)
→ 6x faster with proper caching
```

**Cache Hit Rate:**
- High hit rate: >80% (excellent)
- Medium hit rate: 50-80% (good)
- Low hit rate: <50% (needs improvement)

### Matrix Optimization

**Unoptimized matrix (slow):**
```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node: ['16', '18', '20']
    # Creates 9 jobs, all run sequentially
```

**Optimized matrix (fast):**
```yaml
strategy:
  matrix:
    include:
      - os: ubuntu-latest
        node: '18'     # Most important combo
      - os: windows-latest
        node: '18'
      - os: ubuntu-latest
        node: '20'
    # Fewer jobs, runs in parallel
```

### Build Bottlenecks

**Common Bottlenecks:**
1. Dependency installation (slow npm/pip)
2. Test execution (slow tests)
3. Build compilation (TypeScript, Go, etc.)
4. Docker image building
5. Upload/download steps

**Solutions:**
1. Cache dependencies
2. Run tests in parallel
3. Use build optimization flags
4. Multi-stage Docker builds
5. Use artifacts wisely

---

## Hands-on Lab: Optimize Slow Workflow

### Part 1: Create Slow Workflow (Baseline)

`.github/workflows/slow-workflow.yml`:

```yaml
name: Slow Workflow (Baseline)

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm install           # ~20s first time
      - run: npm run lint          # ~5s
      - run: npm test              # ~15s
      - run: npm run build         # ~10s
      # Total: ~50s
```

**Measure baseline:**
```bash
git push origin main
# Watch workflow in Actions tab
# Note total execution time
```

**Expected: ~50-60 seconds**

### Part 2: Optimize with Caching

`.github/workflows/optimized-workflow.yml`:

```yaml
name: Optimized Workflow

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'  # ← Enable caching
      
      - run: npm ci              # Faster than npm install
      - run: npm run lint
      - run: npm test
      - run: npm run build
```

**Measure optimized:**
```bash
git push origin main
# First run: similar time (cache creation)
# Second run: ~30-35s (cache hit!)
# Improvement: ~40% faster
```

### Part 3: Parallelize Jobs

`.github/workflows/parallel-workflow.yml`:

```yaml
name: Parallel Workflow

on: [push]

jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - uses: actions/upload-artifact@v3
        with:
          name: node_modules
          path: node_modules
          retention-days: 1

  lint:
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: node_modules
          path: node_modules
      - run: npm run lint

  test:
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: node_modules
          path: node_modules
      - run: npm test

  build:
    needs: [test, lint]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: node_modules
          path: node_modules
      - run: npm run build
```

**Expected: ~20-25s** (lint and test run in parallel)

### Part 4: Validate & Measure

**Compare all three:**

| Workflow | Time | Improvement |
|----------|------|-------------|
| Slow | 50-60s | Baseline |
| Optimized | 30-40s | 33% faster |
| Parallel | 20-25s | 60% faster |

**Measurement:**
```bash
# In GitHub Actions, view each workflow
# Compare "Total time" reported
# Calculate improvement percentage
```

### Part 5: Cleanup

```bash
# Keep optimized workflow
git rm .github/workflows/slow-workflow.yml
git rm .github/workflows/parallel-workflow.yml

# Or keep all for learning reference
git add .
git commit -m "docs: workflow optimization examples"
```

---

## Common Mistakes

### 1. No Caching
```yaml
# ✗ Bad: Downloads dependencies every time
- run: npm install

# ✓ Good: Uses cache
- uses: actions/setup-node@v3
  with:
    cache: 'npm'
```

### 2. Sequential Everything
```yaml
# ✗ Bad: All steps block each other
jobs:
  ci:
    steps:
      - run: npm test
      - run: npm run build

# ✓ Good: Parallel jobs
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test
  build:
    needs: test
    steps:
      - run: npm run build
```

### 3. Matrix Overkill
```yaml
# ✗ Bad: Tests every combination (27 jobs!)
strategy:
  matrix:
    os: [ubuntu, windows, macos]
    node: ['16', '18', '20']
    python: ['3.8', '3.9', '3.10']

# ✓ Good: Only needed combinations (6 jobs)
strategy:
  matrix:
    include:
      - os: ubuntu-latest
        node: '18'
      - os: windows-latest
        node: '18'
      - os: macos-latest
        node: '18'
```

### 4. Large Artifacts
```yaml
# ✗ Bad: Uploads entire build folder (500MB)
- uses: actions/upload-artifact@v3
  with:
    path: .

# ✓ Good: Only necessary files (50MB)
- uses: actions/upload-artifact@v3
  with:
    path: dist/
```

### 5. Wrong Cache Key
```yaml
# ✗ Bad: Cache invalidates on any file change
key: ${{ runner.os }}-${{ hashFiles('**/') }}

# ✓ Good: Only invalidate on dependency change
key: ${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
```

---

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| "Workflow still slow" | No caching | Enable cache in setup-node |
| "Cache miss" | Key doesn't match files | Include all package files in hash |
| "Jobs not parallel" | Missing `needs:` | Add job dependencies, not steps |
| "Artifacts too large" | Including everything | Only upload needed files |
| "Setup takes too long" | No artifact sharing | Use upload-artifact for shared deps |
| "Tests still slow" | No parallelization | Split into separate test jobs |
| "Matrix explodes" | Too many combinations | Use include/exclude |

---

## Summary Checklist

- ✅ Workflow profiled and baseline measured
- ✅ Dependency caching enabled
- ✅ Jobs parallelized where possible
- ✅ Matrix strategy optimized
- ✅ Build steps streamlined
- ✅ Artifact sizes minimized
- ✅ Cache hit rate monitored
- ✅ Sequential dependencies eliminated
- ✅ Resource usage optimized
- ✅ Performance improvements documented

---

## Resources

- GitHub Actions Performance: https://docs.github.com/en/actions/learn-github-actions
- Workflow Commands: https://docs.github.com/en/actions/using-workflows
- Caching Best Practices: https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows
- Job Dependencies: https://docs.github.com/en/actions/using-jobs/using-jobs-in-a-workflow
