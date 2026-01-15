# Module 04 Cheatsheet: Caching and Artifacts

## Quick Reference

### Enable Built-in Caching

```yaml
# Node.js
uses: actions/setup-node@v3
with:
  node-version: '18'
  cache: 'npm'

# Python
uses: actions/setup-python@v4
with:
  python-version: '3.10'
  cache: 'pip'

# Java/Maven
uses: actions/setup-java@v3
with:
  java-version: '17'
  cache: 'maven'

# Java/Gradle
uses: actions/setup-gradle@v2
with:
  gradle-version: '7.0'
```

---

## Manual Cache

### Basic Manual Cache

```yaml
- uses: actions/cache@v3
  with:
    key: ${{ runner.os }}-cache-${{ hashFiles('**/lock-file') }}
    restore-keys: |
      ${{ runner.os }}-cache-
    path: ~/.cache/myapp
```

### Common Cache Paths

| Language | Path | Lock File |
|----------|------|-----------|
| npm | `node_modules/` | `package-lock.json` |
| Yarn | `node_modules/` | `yarn.lock` |
| pip | `~/.cache/pip/` | `requirements.txt` |
| Maven | `~/.m2/repository/` | `pom.xml` |
| Gradle | `~/.gradle/caches/` | `gradle.properties` |
| Composer | `~/.composer/cache/` | `composer.lock` |
| RubyGems | `~/.bundle/` | `Gemfile.lock` |

### Cache Key Functions

```yaml
# Hash file contents
${{ hashFiles('package-lock.json') }}
${{ hashFiles('**/requirements.txt') }}
${{ hashFiles('**/*.gradle.kts') }}

# With wildcards
${{ hashFiles('**/package*.json') }}

# Multiple files
${{ hashFiles('src/main/resources/**/pom.xml', 'src/main/resources/**/settings.xml') }}
```

---

## Upload Artifacts

### Single Directory

```yaml
- uses: actions/upload-artifact@v3
  with:
    name: build-output
    path: dist/
    retention-days: 30
```

### Multiple Paths

```yaml
- uses: actions/upload-artifact@v3
  with:
    name: reports
    path: |
      coverage/
      reports/
      test-results/
```

### Conditional Upload

```yaml
- uses: actions/upload-artifact@v3
  if: success()
  with:
    name: build
    path: build/

- uses: actions/upload-artifact@v3
  if: failure()
  with:
    name: debug-logs
    path: logs/
```

### Glob Patterns

```yaml
path: |
  dist/**/*.js          # All JS files recursively
  !dist/vendor/         # Exclude vendor
  src/index.js          # Specific file
  **/*.html             # All HTML files
  coverage/*.json       # JSON in coverage/
```

---

## Download Artifacts

### Download Single Artifact

```yaml
- uses: actions/download-artifact@v3
  with:
    name: build-output
    path: ./artifacts
```

### Download All Artifacts

```yaml
- uses: actions/download-artifact@v3
  with:
    path: ./all-artifacts
```

### Use in Another Job

```yaml
jobs:
  build:
    # ... upload artifact ...

  deploy:
    needs: build
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: build-output
          path: ./dist
      - run: npm run deploy
```

---

## Cache vs Artifact Comparison

| Feature | Cache | Artifact |
|---------|-------|----------|
| **Purpose** | Reuse dependencies | Store outputs |
| **Lifetime** | Workflow cache only | 90 days (configurable) |
| **Access** | Automatic in setup actions | Manual download |
| **Size limit** | 5GB per repo | 5GB per artifact |
| **Speed** | Fast hit (5-10s) | Slower (depends on size) |
| **Use case** | npm, pip, gems | Builds, reports, logs |
| **Retention** | Dynamic (7 days inactive) | Fixed (90 days default) |
| **Cost** | Counts to cache quota | Counts to storage quota |

---

## npm Caching Strategies

### Automatic (Recommended)

```yaml
- uses: actions/setup-node@v3
  with:
    node-version: '18'
    cache: 'npm'  # Automatic
```

### Manual Control

```yaml
- uses: actions/cache@v3
  with:
    key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
    path: node_modules/
```

### With Matrix

```yaml
strategy:
  matrix:
    node: [16, 18, 20]
steps:
  - uses: actions/setup-node@v3
    with:
      node-version: ${{ matrix.node }}
      cache: 'npm'  # Automatically includes matrix var
```

---

## Python Caching Strategies

### Automatic (Recommended)

```yaml
- uses: actions/setup-python@v4
  with:
    python-version: '3.10'
    cache: 'pip'  # Automatic
  
- run: pip install -r requirements.txt
```

### Manual Control

```yaml
- uses: actions/cache@v3
  with:
    key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
    path: ~/.cache/pip
```

---

## Artifact Retention Recommendations

| Type | Retention | Reason |
|------|-----------|--------|
| Debug logs | 1 day | Quick reference only |
| Test reports | 7 days | Debugging failures |
| Build artifacts | 14 days | Quick reuse |
| Release artifacts | 90 days | Legal/compliance |
| Docker images | 30 days | Push to registry instead |
| Security scans | 30 days | Compliance records |

---

## Performance Optimization

### Cache First Run

```bash
# Workflow executes
npm install → 60s (download)

# Next workflow run
npm install → 5s (from cache)
# Savings: 55s, 92% faster
```

### Artifact-Based CI/CD

```
Build Job (once)
    ├── Install cache hit → 5s
    ├── Build → 20s
    └── Upload artifact → 10s
    Total: 35s

Test Job 1 (parallel)
    ├── Download artifact → 5s
    ├── Run unit tests → 15s
    Total: 20s

Test Job 2 (parallel)
    ├── Download artifact → 5s
    ├── Run integration tests → 25s
    Total: 30s

Publish Job
    ├── Download artifact → 5s
    ├── Publish → 15s
    Total: 20s

Total time: 35s (build) + 30s (parallel tests) = 65s
Without artifacts: 35s + 20s + 25s + 15s = 95s
Savings: 30s, 32% faster
```

---

## Troubleshooting Quick Fix

### Cache Not Restoring

```yaml
# Check 1: Exact key match
key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
# If lock file changes, key changes

# Check 2: Path exists
- run: ls -la node_modules/ || echo "Cache miss or path wrong"

# Check 3: Include restore keys
restore-keys: |
  ${{ runner.os }}-npm-
  ${{ runner.os }}-

# Check 4: Check cache storage
# Settings → Actions → Artifact and log retention
```

### Artifact Upload Failing

```yaml
# Check 1: Path exists
- run: ls -la dist/ || mkdir -p dist/

# Check 2: File size reasonable
- run: du -sh dist/

# Check 3: Use if-no-files-found
- uses: actions/upload-artifact@v3
  with:
    path: dist/
    if-no-files-found: warn
```

### Artifact Download Missing

```yaml
# Check 1: Name exact match
# Upload: name: my-build
# Download: name: my-build  (EXACT)

# Check 2: In correct job
jobs:
  build:
    # uploads artifact
  deploy:
    needs: build  # Must depend on build job
    # downloads artifact

# Check 3: Still within retention
- run: echo "Artifact must be within retention days"
```

---

## Common Patterns

### Build Once, Test Anywhere

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci && npm run build
      - uses: actions/upload-artifact@v3
        with:
          name: dist
          path: dist/

  test-unit:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: dist
          path: dist/
      - run: npm test

  test-e2e:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: dist
          path: dist/
      - run: npm run test:e2e
```

### Matrix with Caching

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [16, 18, 20]
        os: [ubuntu, windows, macos]
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      - run: npm ci && npm test
```

### Release Pipeline

```yaml
jobs:
  build:
    # Build and upload as release artifact
    
  verify:
    needs: build
    # Download and verify artifact
    
  publish:
    needs: verify
    # Download release artifact and publish
```

---

## Environment Variables

### Cache-Related

```yaml
# Cache size
$GITHUB_CACHE_MAX_SIZE = 5GB per repo

# Artifact size
$GITHUB_ARTIFACT_MAX_SIZE = 5GB per artifact

# Check in workflow
- run: |
    echo "Cache directory: ${{ runner.tool_cache }}"
    echo "Workspace: ${{ github.workspace }}"
```

### Artifact-Related

```yaml
# Run ID for unique names
${{ github.run_id }}
${{ github.run_number }}
${{ github.sha }}

# Example
- uses: actions/upload-artifact@v3
  with:
    name: build-${{ github.run_id }}-${{ matrix.node }}
    path: dist/
```

---

## Limits and Quotas

| Item | Limit | Notes |
|------|-------|-------|
| Cache per repo | 5 GB | Shared, deleted if not used 7 days |
| Artifact per file | 5 GB | Individual artifact max |
| Artifacts per repo | 400 GB | Organization storage |
| Retention | 90 days | Free, 1-90 days configurable |
| Cache keys | Unlimited | But 5GB total |

---

## Pro Tips

**1. Version-specific caching**
```yaml
key: ${{ runner.os }}-${{ matrix.node }}-npm-${{ hashFiles('package-lock.json') }}
```

**2. Different retention per artifact**
```yaml
- uses: actions/upload-artifact@v3
  if: success()
  with:
    name: successful-build
    retention-days: 30

- uses: actions/upload-artifact@v3
  if: failure()
  with:
    name: failed-build
    retention-days: 3
```

**3. Conditional caching**
```yaml
- uses: actions/cache@v3
  if: contains(github.event.head_commit.message, 'skip-cache') == false
  with:
    key: npm-cache
    path: node_modules/
```

**4. Check cache stats**
```yaml
- run: |
    echo "Cache info:"
    npm cache verify
    du -sh ~/.npm
```

**5. Force cache refresh**
```bash
# Change key to force miss
key: ${{ runner.os }}-npm-v2-${{ hashFiles('package-lock.json') }}
# Add version number to invalidate
```

---

## API Commands

### Download via REST

```bash
# Get artifact download URL
curl -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/OWNER/REPO/actions/artifacts?name=ARTIFACT_NAME
```

### Delete Cache

```bash
# Via GitHub CLI
gh actions-cache delete CACHE_KEY --repo OWNER/REPO
```

---

**Last Updated:** January 2026 | **Module:** 04 | **Level:** Intermediate
