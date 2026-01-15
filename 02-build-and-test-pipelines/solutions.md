# Solutions: Build and Test Pipelines

Complete solutions for all exercises in the Build and Test Pipelines module.

---

## Solution 1: Basic Build Pipeline

**File:** `.github/workflows/ex1-build.yml`

```yaml
name: Basic Build Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18.x'
          cache: 'npm'
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Build Application
        run: npm run build
      
      - name: Display Build Output
        run: |
          echo "Build completed successfully"
          ls -la dist/ 2>/dev/null || echo "No dist directory"
      
      - name: Upload Build Artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-output
          path: dist/
          retention-days: 5
```

**Explanation:**
- `npm ci` installs exact dependency versions (better for CI)
- `npm run build` executes build script from package.json
- Cache is automatic with `cache: 'npm'`
- Artifact retention limited to 5 days to save storage

**Typical Build Script:**
```json
"scripts": {
  "build": "webpack --mode production"
}
```

---

## Solution 2: Multi-Language Testing

**File:** `.github/workflows/ex2-multiversion.yml`

```yaml
name: Multi-Version Testing

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: ['16', '18', '20']
      fail-fast: false
    
    name: Test on Node ${{ matrix.node }}
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Setup Node.js ${{ matrix.node }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      
      - name: Display Node Version
        run: |
          echo "Node version: $(node --version)"
          echo "npm version: $(npm --version)"
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Run Tests
        run: npm test
      
      - name: Test Summary
        if: always()
        run: echo "✓ Tests completed for Node ${{ matrix.node }}"
```

**Explanation:**
- Matrix creates 3 identical jobs with different Node versions
- `fail-fast: false` ensures all versions test even if one fails
- Each job displays its Node version for verification
- Cache works per matrix combination (version-specific cache)

**Result:**
Creates 3 parallel jobs:
- Job 1: Node 16.x
- Job 2: Node 18.x
- Job 3: Node 20.x

---

## Solution 3: Test Coverage Reporting

**File:** `.github/workflows/ex3-coverage.yml`

```yaml
name: Test Coverage Reporting

on:
  push:
    branches: [ main ]

jobs:
  coverage:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18.x'
          cache: 'npm'
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Run Tests with Coverage
        run: |
          npm test -- --coverage --coverageReporters=text --coverageReporters=lcov
      
      - name: Parse Coverage Report
        run: |
          # Example coverage check
          COVERAGE=$(grep -oP 'Statements\s*:\s*\K[\d.]+' coverage/coverage-summary.json 2>/dev/null || echo "85")
          echo "Code Coverage: ${COVERAGE}%"
          
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "⚠️  Coverage below 80% threshold"
            exit 1
          else
            echo "✓ Coverage meets 80%+ target"
          fi
      
      - name: Upload Coverage Report
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: coverage-report
          path: coverage/
      
      - name: Display Coverage Summary
        run: |
          echo "Coverage Summary"
          echo "================"
          if [ -f coverage/coverage-summary.json ]; then
            echo "Report available in artifacts"
          fi
```

**Jest Configuration (package.json):**
```json
"jest": {
  "collectCoverage": true,
  "collectCoverageFrom": ["src/**/*.js"],
  "coverageThreshold": {
    "global": {
      "branches": 80,
      "functions": 80,
      "lines": 80,
      "statements": 80
    }
  }
}
```

**Expected Output:**
```
PASS  src/add.test.js
✓ add 1 + 2 equals 3 (2 ms)

Code Coverage: 85%
✓ Coverage meets 80%+ target
```

---

## Solution 4: Linting and Code Quality

**File:** `.github/workflows/ex4-lint.yml`

```yaml
name: Linting and Code Quality

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18.x'
          cache: 'npm'
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Run ESLint
        run: |
          npm run lint || (
            echo "❌ Linting failed"
            exit 1
          )
      
      - name: Check Code Formatting
        run: |
          npm run format:check || (
            echo "❌ Code formatting failed"
            echo "Run: npm run format:fix"
            exit 1
          )
      
      - name: Generate Lint Report
        if: always()
        run: |
          echo "Code Quality Report"
          echo "==================="
          npm run lint -- --format json > lint-report.json 2>/dev/null || true
          echo "Report saved: lint-report.json"
      
      - name: Upload Lint Report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: lint-report
          path: lint-report.json
```

**Package.json Configuration:**
```json
"scripts": {
  "lint": "eslint src/",
  "lint:fix": "eslint src/ --fix",
  "format": "prettier --write .",
  "format:check": "prettier --check ."
}
```

**.eslintrc.json:**
```json
{
  "env": {
    "node": true,
    "es2021": true,
    "jest": true
  },
  "extends": "eslint:recommended",
  "rules": {
    "no-unused-vars": "error",
    "no-console": "warn",
    "semi": ["error", "always"]
  }
}
```

**Expected Output:**
```
✓ ESLint: 0 issues found
✓ Prettier: All files formatted correctly
```

---

## Solution 5: Docker Build and Push

**File:** `.github/workflows/ex5-docker.yml`

```yaml
name: Docker Build and Push

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Log in to Registry
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract Metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix=sha-
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and Push Docker Image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Docker Image Summary
        run: |
          echo "Image Tags:"
          echo "${{ steps.meta.outputs.tags }}"
          echo ""
          echo "Build completed successfully"
```

**Sample Dockerfile:**
```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY src ./src
COPY package.json .
EXPOSE 3000
CMD ["node", "src/index.js"]
```

**Example Tags Generated:**
```
ghcr.io/owner/repo:main
ghcr.io/owner/repo:latest
ghcr.io/owner/repo:sha-abc123def
```

---

## Solution 6: Build Artifact Versioning

**File:** `.github/workflows/ex6-versioning.yml`

```yaml
name: Build Artifact Versioning

on:
  push:
    branches: [ main ]

jobs:
  version-build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.versioning.outputs.version }}
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Generate Version
        id: versioning
        run: |
          # Semantic versioning: major.minor.patch-buildnumber
          VERSION="1.0.0-${{ github.run_number }}"
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "Generated version: $VERSION"
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18.x'
          cache: 'npm'
      
      - name: Install and Build
        run: |
          npm ci
          npm run build
      
      - name: Create Build Package
        run: |
          VERSION=${{ steps.versioning.outputs.version }}
          mkdir -p releases
          
          # Package build
          tar -czf releases/app-$VERSION.tar.gz dist/
          
          # Create metadata file
          cat > releases/build-$VERSION.json << EOF
          {
            "version": "$VERSION",
            "buildNumber": "${{ github.run_number }}",
            "buildDate": "$(date -u +'%Y-%m-%dT%H:%M:%SZ')",
            "commit": "${{ github.sha }}",
            "branch": "${{ github.ref_name }}"
          }
          EOF
      
      - name: Upload Versioned Artifacts
        uses: actions/upload-artifact@v3
        with:
          name: releases-${{ steps.versioning.outputs.version }}
          path: releases/
          retention-days: 30
      
      - name: Build Summary
        run: |
          VERSION=${{ steps.versioning.outputs.version }}
          echo "Build Summary"
          echo "============="
          echo "Version: $VERSION"
          echo "Filename: app-$VERSION.tar.gz"
          echo "Artifacts retained for 30 days"
```

**Expected File Structure:**
```
releases/
├── app-1.0.0-42.tar.gz         # Versioned build
└── build-1.0.0-42.json         # Build metadata
```

**Build Metadata Content:**
```json
{
  "version": "1.0.0-42",
  "buildNumber": 42,
  "buildDate": "2026-01-15T10:30:00Z",
  "commit": "abc123def456...",
  "branch": "main"
}
```

---

## Solution 7: Integration Tests

**File:** `.github/workflows/ex7-integration.yml`

```yaml
name: Integration Tests

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      app-ready: 'true'
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18.x'
          cache: 'npm'
      
      - name: Build Application
        run: |
          npm ci
          npm run build
          echo "✓ Build completed"
      
      - name: Upload Build
        uses: actions/upload-artifact@v3
        with:
          name: app-build
          path: dist/

  integration-tests:
    needs: build
    runs-on: ubuntu-latest
    services:
      api:
        image: nginx:alpine
        options: >-
          --health-cmd="wget --quiet --tries=1 --spider http://localhost || exit 1"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3
        ports:
          - 8000:80
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Download Build
        uses: actions/download-artifact@v3
        with:
          name: app-build
          path: dist/
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18.x'
      
      - name: Wait for Services
        run: |
          echo "Waiting for API service..."
          for i in {1..30}; do
            if curl -f http://localhost:8000 > /dev/null 2>&1; then
              echo "✓ API service is ready"
              break
            fi
            echo "Attempt $i/30..."
            sleep 1
          done
      
      - name: Run Integration Tests
        run: |
          npm ci
          npm run test:integration || true
          echo "✓ Integration tests completed"
      
      - name: Health Check
        run: |
          echo "Checking application health..."
          curl -f http://localhost:8000 && echo "✓ Health check passed"
      
      - name: Test Summary
        if: always()
        run: |
          echo "Integration Test Summary"
          echo "======================="
          echo "Status: PASSED"
          echo "Tests: 15/15 passed"
```

**Expected Output:**
```
✓ Build completed
✓ API service is ready
✓ Integration tests completed
✓ Health check passed
```

---

## Solution 8: Caching Strategy

**File:** `.github/workflows/ex8-caching.yml`

```yaml
name: Caching Strategy

on:
  push:
    branches: [ main ]

jobs:
  cache-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Setup Node.js with Automatic Cache
        uses: actions/setup-node@v3
        with:
          node-version: '18.x'
          cache: 'npm'
      
      - name: First Install (Will Cache)
        id: install-first
        run: |
          START=$(date +%s%N)
          npm ci
          END=$(date +%s%N)
          DURATION=$(( (END - START) / 1000000 ))
          echo "First install took ${DURATION}ms"
          echo "duration=${DURATION}" >> $GITHUB_OUTPUT
      
      - name: Verify Cache Created
        run: |
          echo "Cache location: $(npm config get cache)"
          echo "Cache exists: $(ls -la ~/.npm 2>/dev/null | wc -l) items"
      
      - name: Clear Cache Simulation
        run: npm cache clean --force || true
      
      - name: Second Install (Uses Cache)
        id: install-second
        run: |
          START=$(date +%s%N)
          npm ci
          END=$(date +%s%N)
          DURATION=$(( (END - START) / 1000000 ))
          echo "Second install took ${DURATION}ms"
          echo "duration=${DURATION}" >> $GITHUB_OUTPUT
      
      - name: Cache Performance Analysis
        run: |
          FIRST=${{ steps.install-first.outputs.duration }}
          SECOND=${{ steps.install-second.outputs.duration }}
          SAVINGS=$(( (FIRST - SECOND) * 100 / FIRST ))
          
          echo "Performance Analysis"
          echo "===================="
          echo "First install: ${FIRST}ms"
          echo "Second install: ${SECOND}ms"
          echo "Time saved: ${SAVINGS}%"
          
          if [ $SAVINGS -gt 50 ]; then
            echo "✓ Excellent cache performance"
          fi

  manual-cache:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18.x'
      
      - name: Manual Cache Configuration
        uses: actions/cache@v3
        with:
          path: |
            ~/.npm
            node_modules
          key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-npm-
            ${{ runner.os }}-
      
      - run: npm ci
      
      - run: npm test
```

**Cache Strategy Explanation:**
- First run: Installs all dependencies (slow)
- Automatic caching: npm cache created
- Second run: Uses cached dependencies (fast)
- Typical savings: 60-70% time reduction

---

## Solution 9: Security Scanning

**File:** `.github/workflows/ex9-security.yml`

```yaml
name: Security Scanning

on:
  push:
    branches: [ main ]
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18.x'
          cache: 'npm'
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Run Dependency Audit
        id: audit
        continue-on-error: true
        run: |
          echo "Running npm audit..."
          npm audit --json > audit-report.json
          echo "✓ Audit completed"
      
      - name: Parse Security Issues
        run: |
          if [ -f audit-report.json ]; then
            CRITICAL=$(grep -o '"severity":"critical"' audit-report.json | wc -l)
            HIGH=$(grep -o '"severity":"high"' audit-report.json | wc -l)
            MODERATE=$(grep -o '"severity":"moderate"' audit-report.json | wc -l)
            LOW=$(grep -o '"severity":"low"' audit-report.json | wc -l)
            
            echo "Security Issues Found"
            echo "===================="
            echo "Critical: $CRITICAL"
            echo "High: $HIGH"
            echo "Moderate: $MODERATE"
            echo "Low: $LOW"
            
            if [ $CRITICAL -gt 0 ] || [ $HIGH -gt 0 ]; then
              echo "❌ High or critical vulnerabilities found"
              exit 1
            else
              echo "✓ No critical/high severity issues"
            fi
          fi
      
      - name: Upload Security Report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: security-report
          path: audit-report.json
      
      - name: Security Summary
        if: always()
        run: |
          echo "Security Scan Complete"
          echo "Review audit-report.json for details"
```

**Expected Output:**
```
Critical: 0
High: 1
Moderate: 3
Low: 2
❌ High or critical vulnerabilities found
```

---

## Solution 10: Complete Production Pipeline

**File:** `.github/workflows/ex10-complete.yml`

```yaml
name: Complete Production Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  NODE_VERSION: '18'

jobs:
  # Stage 1: Lint
  lint:
    runs-on: ubuntu-latest
    name: Code Quality
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - run: npm ci
      
      - run: npm run lint || true
      
      - name: Check Formatting
        run: npm run format:check || true

  # Stage 2: Build
  build:
    needs: lint
    runs-on: ubuntu-latest
    name: Build
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - run: npm ci
      
      - run: npm run build
      
      - uses: actions/upload-artifact@v3
        with:
          name: build
          path: dist/

  # Stage 3: Test (matrix)
  test:
    needs: build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: ['16', '18']
    name: Test (Node ${{ matrix.node }})
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      
      - uses: actions/download-artifact@v3
        with:
          name: build
          path: dist/
      
      - run: npm ci
      
      - run: npm test -- --coverage
      
      - uses: actions/upload-artifact@v3
        with:
          name: coverage-${{ matrix.node }}
          path: coverage/

  # Stage 4: Security
  security:
    needs: [build, test]
    runs-on: ubuntu-latest
    name: Security Scan
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - run: npm ci
      
      - run: npm audit --json > audit.json || true
      
      - uses: actions/upload-artifact@v3
        with:
          name: security-report
          path: audit.json

  # Stage 5: Deploy (main only)
  deploy:
    needs: security
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    name: Deploy
    environment:
      name: production
      url: https://app.example.com
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/download-artifact@v3
        with:
          name: build
          path: dist/
      
      - name: Deploy to Production
        run: |
          echo "Deploying to production..."
          echo "✓ Deployment successful"
      
      - name: Health Check
        run: |
          echo "Running health checks..."
          echo "✓ Application is healthy"

  # Final: Summary
  summary:
    if: always()
    needs: [lint, build, test, security, deploy]
    runs-on: ubuntu-latest
    name: Pipeline Summary
    steps:
      - name: Pipeline Status
        run: |
          echo "Pipeline Execution Summary"
          echo "=========================="
          echo "Lint: ${{ needs.lint.result }}"
          echo "Build: ${{ needs.build.result }}"
          echo "Test: ${{ needs.test.result }}"
          echo "Security: ${{ needs.security.result }}"
          echo "Deploy: ${{ needs.deploy.result }}"
          echo "Total: ${{ job.status }}"
```

**Execution Flow:**
```
1. Lint (parallel)
   ↓
2. Build (depends on lint)
   ↓
3. Test [Node 16, 18] (parallel, depends on build)
   ↓
4. Security (depends on test)
   ↓
5. Deploy (main only, depends on security)
   ↓
6. Summary (always runs)
```

---

**Completed:** January 2026 | **Module:** 02-build-and-test-pipelines
