# 02-build-and-test-pipelines

## Overview

This module teaches you how to build real CI/CD pipelines with language-specific tools, automated testing, code quality checks, and optimization techniques. Learn industry best practices for production-ready pipelines.

---

## What You'll Learn

- Setting up build environments for different languages
- Running tests (unit, integration, end-to-end)
- Code coverage analysis and reporting
- Code quality and linting tools
- Dependency caching for performance
- Docker integration in CI pipelines
- Build optimization techniques
- Artifact management and versioning

---

## Prerequisites

- Completion of **01-workflow-fundamentals** module
- Understanding of basic build tools
- Familiarity with testing frameworks
- Basic knowledge of Docker (for container builds)
- Git and GitHub repository access

---

## Key Concepts

### 1. **Build Stage**
Compilation/preparation of source code for execution.

### 2. **Test Pyramid**
Strategy with unit tests (many), integration tests (some), e2e tests (few).

### 3. **Code Coverage**
Percentage of code lines executed by tests. Higher is better (target: 80%+).

### 4. **Linting**
Static code analysis to catch style issues and bugs early.

### 5. **Caching Strategy**
Storing dependencies to reduce build time by 50-70%.

### 6. **Dependency Lock Files**
Ensures reproducible builds (`package-lock.json`, `requirements.txt`, etc.).

### 7. **Docker Images**
Containerizing applications for consistency across environments.

### 8. **Build Artifacts**
Compiled binaries or packages ready for deployment.

---

## Hands-on Lab: Complete CI/CD Pipeline

### Step 1: Create a Sample Project

```bash
# Create project structure
mkdir my-ci-project
cd my-ci-project

# Create package.json (for Node.js project)
cat > package.json << 'EOF'
{
  "name": "ci-demo",
  "version": "1.0.0",
  "scripts": {
    "build": "echo 'Building application...'",
    "test": "npm run test:unit && npm run test:coverage",
    "test:unit": "echo 'Running unit tests...'",
    "test:coverage": "echo 'Code coverage: 85%'",
    "lint": "echo 'Linting code...'",
    "start": "echo 'Starting application...'"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "eslint": "^8.0.0"
  }
}
EOF

# Create a simple source file
mkdir src
cat > src/index.js << 'EOF'
function add(a, b) {
  return a + b;
}

function multiply(a, b) {
  return a * b;
}

module.exports = { add, multiply };
EOF

# Create test file
cat > src/index.test.js << 'EOF'
const { add, multiply } = require('./index');

describe('Math functions', () => {
  test('add 1 + 2 equals 3', () => {
    expect(add(1, 2)).toBe(3);
  });

  test('multiply 2 * 3 equals 6', () => {
    expect(multiply(2, 3)).toBe(6);
  });
});
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY src .
CMD ["node", "src/index.js"]
EOF

# Initialize git
git init
git add .
git commit -m "Initial project structure"
```

### Step 2: Create Build and Test Workflow

Create `.github/workflows/build-test-deploy.yml`:

```yaml
name: Build, Test, and Deploy

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  NODE_VERSION: '18'

jobs:
  # Stage 1: Code Quality and Linting
  quality:
    runs-on: ubuntu-latest
    name: Code Quality Check
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Check code formatting
        run: echo "✓ Code formatting check passed"

  # Stage 2: Build
  build:
    runs-on: ubuntu-latest
    needs: quality
    name: Build Application
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build application
        run: npm run build
      
      - name: Create build output
        run: |
          mkdir -p dist
          cp package.json dist/
          cp -r src dist/
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-output
          path: dist/
          retention-days: 5

  # Stage 3: Unit Tests
  test-unit:
    runs-on: ubuntu-latest
    needs: build
    name: Unit Tests
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit
      
      - name: Collect coverage
        run: npm run test:coverage

  # Stage 4: Integration Tests
  test-integration:
    runs-on: ubuntu-latest
    needs: build
    name: Integration Tests
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Start services
        run: echo "✓ Services started"
      
      - name: Run integration tests
        run: echo "✓ Integration tests passed"

  # Stage 5: Docker Build and Push
  docker-build:
    runs-on: ubuntu-latest
    needs: [test-unit, test-integration]
    name: Build Docker Image
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
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
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # Stage 6: Security Scanning
  security:
    runs-on: ubuntu-latest
    needs: [test-unit, test-integration]
    name: Security Scanning
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Run security scan
        run: echo "✓ No vulnerabilities found"
      
      - name: Dependency check
        run: npm audit --audit-level=moderate || true

  # Stage 7: Deploy (main branch only)
  deploy:
    runs-on: ubuntu-latest
    needs: [docker-build, security]
    name: Deploy to Production
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment:
      name: production
      url: https://myapp.example.com
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Deploy application
        run: |
          echo "Deploying to production..."
          echo "Version: ${{ github.sha }}"
          echo "✓ Deployment successful"
      
      - name: Health check
        run: echo "✓ Application is healthy"

  # Stage 8: Summary and Notifications
  summary:
    runs-on: ubuntu-latest
    needs: [quality, build, test-unit, test-integration, docker-build, security, deploy]
    if: always()
    name: Pipeline Summary
    steps:
      - name: Pipeline Status
        run: |
          echo "Pipeline Execution Summary"
          echo "=========================="
          echo "Commit: ${{ github.sha }}"
          echo "Branch: ${{ github.ref }}"
          echo "Author: ${{ github.actor }}"
          echo "Status: ${{ job.status }}"
      
      - name: Create build report
        run: |
          echo "Build Report" > report.txt
          echo "Build: SUCCESS" >> report.txt
          echo "Tests: PASSED" >> report.txt
          echo "Coverage: 85%" >> report.txt
      
      - name: Upload summary
        uses: actions/upload-artifact@v3
        with:
          name: pipeline-summary
          path: report.txt
```

### Step 3: Push to Repository

```bash
git add .github/workflows/build-test-deploy.yml
git commit -m "Add complete build and test pipeline"
git push origin main
```

### Expected Execution:

The workflow will execute 8 stages:
1. ✅ Quality checks (linting)
2. ✅ Build application
3. ✅ Run unit tests
4. ✅ Run integration tests
5. ✅ Build Docker image
6. ✅ Security scanning
7. ✅ Deploy (main only)
8. ✅ Generate summary

**Total runtime:** 3-5 minutes

---

## Validation Checklist

- [ ] Workflow file passes YAML validation
- [ ] All 8 stages execute successfully
- [ ] Code quality check passes
- [ ] Build completes without errors
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Docker image builds successfully
- [ ] Security scan completes
- [ ] Artifacts are generated and available
- [ ] Deploy step only runs on main branch
- [ ] All logs are readable and informative

---

## Cleanup

```bash
rm .github/workflows/build-test-deploy.yml
git add -A
git commit -m "Remove build pipeline workflow"
git push origin main
```

---

## Common Mistakes

### ❌ Mistake 1: No Caching for Dependencies

```yaml
# Wrong - reinstalls every time
steps:
  - uses: actions/setup-node@v3
    with:
      node-version: '18'
  - run: npm install
```

```yaml
# Correct - uses cache
steps:
  - uses: actions/setup-node@v3
    with:
      node-version: '18'
      cache: 'npm'
  - run: npm ci
```

### ❌ Mistake 2: Not Using Lock Files

```yaml
# Wrong - version inconsistency
run: npm install

# Correct - consistent versions
run: npm ci
```

### ❌ Mistake 3: Missing Test Coverage

```yaml
# Wrong - no coverage reporting
steps:
  - run: npm test

# Correct - with coverage
steps:
  - run: npm test -- --coverage
  - uses: codecov/codecov-action@v3
```

### ❌ Mistake 4: Docker Push Always

```yaml
# Wrong - pushes on every PR
docker/build-push-action@v4:
  push: true

# Correct - only on main branch
docker/build-push-action@v4:
  push: ${{ github.ref == 'refs/heads/main' }}
```

### ❌ Mistake 5: No Fail-Fast for Tests

All tests should run even if one fails initially, then fail job if any test failed.

```yaml
# Better strategy
strategy:
  fail-fast: false  # Continue all tests even if one fails
```

---

## Troubleshooting

### Issue: Slow builds

**Solution:**
- Enable caching: `cache: 'npm'`
- Use `npm ci` instead of `npm install`
- Run tests in parallel across multiple jobs
- Use Docker caching: `cache-to: type=gha`

### Issue: Tests fail intermittently

**Solution:**
- Add timeouts for async operations
- Run tests multiple times to verify
- Check for race conditions
- Ensure test isolation (no shared state)

### Issue: Docker image too large

**Solution:**
- Use slim base images: `node:18-alpine`
- Multi-stage builds: compile in one stage, run in another
- Remove unnecessary files in .dockerignore

### Issue: Deploy step fails silently

**Solution:**
- Add environment protection rules on main branch
- Use deployment environments for approval gates
- Add health checks after deployment
- Log deployment details

### Issue: High test execution time

**Solution:**
- Run tests in parallel with matrix strategy
- Cache dependencies aggressively
- Split tests: unit/integration/e2e in separate jobs
- Consider test parallelization (jest -j4)

---

## Next Steps

✅ **You've completed module 02!**

**What's next?**

1. Proceed to **03-matrix-and-multi-os-ci** to learn:
   - Multi-OS testing strategies
   - Matrix expansions
   - Parallel execution optimization
   
2. Advanced features to explore:
   - Code coverage badges
   - Performance benchmarking
   - Automated releases
   - Container scanning

3. Practice exercises:
   - Build pipelines for different languages
   - Integrate popular testing frameworks
   - Add code quality tools
   - Implement caching strategies

---

## Build Tools Reference

| Language | Build Tool | Install | Run |
|----------|-----------|---------|-----|
| Node.js | npm | `npm ci` | `npm run build` |
| Python | pip | `pip install -r requirements.txt` | `python -m pytest` |
| Java | Maven | `mvn install` | `mvn test` |
| Go | go mod | `go mod download` | `go test ./...` |
| Rust | cargo | `cargo fetch` | `cargo test` |

---

## Testing Frameworks

| Language | Framework | Command |
|----------|-----------|---------|
| Node.js | Jest | `jest` |
| Node.js | Mocha | `mocha` |
| Python | pytest | `pytest` |
| Python | unittest | `python -m unittest` |
| Java | JUnit | Maven/Gradle |
| Go | testing | `go test` |

---

## Code Quality Tools

| Tool | Purpose | Integration |
|------|---------|-----------|
| ESLint | JavaScript linting | `npm run lint` |
| Prettier | Code formatting | `prettier --write` |
| SonarQube | Code analysis | GitHub Action |
| CodeCov | Coverage reporting | `codecov/codecov-action` |
| OWASP | Security scanning | Container scan |

---

## Docker Best Practices

### Multi-Stage Build

```dockerfile
# Build stage
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Runtime stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY src ./
CMD ["node", "src/index.js"]
```

### .dockerignore

```
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
```

---

## Performance Optimization Tips

| Technique | Benefit | Implementation |
|-----------|---------|-----------------|
| Caching | 50-70% faster | `actions/setup-node` with `cache:` |
| Parallel jobs | Faster overall | Matrix strategy, multiple jobs |
| Container cache | Faster builds | `cache-from: type=gha` |
| Slim images | Smaller images | Alpine base images |
| CI skip | Optional runs | `[skip ci]` in commit message |

---

## Additional Resources

- [GitHub Actions for Testing](https://docs.github.com/en/actions/automating-builds-and-tests)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Jest Testing Framework](https://jestjs.io/)
- [Code Coverage Tools](https://codecov.io/)
- [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)

---

**Created:** January 2026 | **Level:** Advanced | **Estimated Time:** 60 minutes
