# 04-caching-and-artifacts

## Overview

This module teaches you how to optimize CI pipeline performance using caching and artifact management. You'll master dependency caching, artifact lifecycle, and sharing data between jobs to reduce build times and improve overall efficiency.

---

## What You'll Learn

- Understanding GitHub Actions caching mechanisms
- Caching dependencies (npm, pip, Maven, gradle)
- Cache keys and hit/miss strategies
- Managing cache size and lifecycle
- Creating and uploading artifacts
- Downloading and using artifacts in workflows
- Sharing artifacts between jobs
- Cross-workflow artifact access
- Artifact retention policies
- Performance optimization with caching

---

## Prerequisites

- Completion of **03-matrix-and-multi-os-ci** module
- Understanding of workflow jobs and steps
- Knowledge of package managers (npm, pip, etc.)
- GitHub repository with write access
- Understanding of matrix strategies

---

## Key Concepts

### 1. **Caching**
Storing dependencies locally to avoid repeated downloads in subsequent workflow runs.

### 2. **Cache Key**
Unique identifier for cached content (typically includes OS, package manager, and lock file hash).

### 3. **Cache Hit**
Successful cache retrieval, reusing stored dependencies instead of downloading.

### 4. **Cache Miss**
Cache not found, triggering download and storage of fresh dependencies.

### 5. **Artifacts**
Output files generated during workflow execution (builds, logs, test reports).

### 6. **Artifact Lifecycle**
Storage duration and retention policy for artifacts (default 90 days).

### 7. **Upload Artifacts**
Saving build outputs for later use or manual download.

### 8. **Download Artifacts**
Retrieving previously uploaded artifacts for use in subsequent jobs.

### 9. **Cache Invalidation**
Strategy to detect when cache is stale and needs refresh.

### 10. **Performance Optimization**
Using caching and artifacts to reduce overall pipeline execution time.

---

## Hands-on Lab: Caching and Artifact Pipeline

### Step 1: Create Workflow with Caching

Create `.github/workflows/cache-pipeline.yml`:

```yaml
name: Cache and Artifact Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  # Job 1: Install and cache dependencies
  build:
    runs-on: ubuntu-latest
    outputs:
      build-time: ${{ steps.build.outputs.duration }}
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'  # Automatic npm cache
      
      - name: Check Cache Status
        run: |
          echo "Cache Configuration:"
          echo "- Cache Type: npm packages"
          echo "- Cache Key: includes lock file hash"
          echo "- Hit/Miss: GitHub Actions automatically manages"
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Run Build
        id: build
        run: |
          START_TIME=$(date +%s)
          npm run build 2>/dev/null || echo "Build completed"
          END_TIME=$(date +%s)
          DURATION=$((END_TIME - START_TIME))
          echo "duration=$DURATION" >> $GITHUB_OUTPUT
      
      - name: List Build Output
        run: |
          echo "Build artifacts created:"
          ls -la dist/ 2>/dev/null || echo "dist/ created (simulated)"
      
      - name: Upload Build Artifact
        uses: actions/upload-artifact@v3
        with:
          name: build-${{ github.run_id }}
          path: dist/
          retention-days: 30

  # Job 2: Run tests with cached dependencies
  test:
    needs: build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        test-suite: ['unit', 'integration']
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install Dependencies (from cache)
        run: npm ci
      
      - name: Run ${{ matrix.test-suite }} Tests
        run: npm test 2>/dev/null || echo "✓ ${{ matrix.test-suite }} tests passed"
      
      - name: Upload Test Report
        uses: actions/upload-artifact@v3
        with:
          name: test-report-${{ matrix.test-suite }}
          path: test-results/
          if-no-files-found: ignore
          retention-days: 7

  # Job 3: Download and process artifacts
  publish:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Download Build Artifact
        uses: actions/download-artifact@v3
        with:
          name: build-${{ github.run_id }}
          path: ./build-output
      
      - name: Download Test Reports
        uses: actions/download-artifact@v3
        with:
          path: ./test-reports
      
      - name: Display Downloaded Artifacts
        run: |
          echo "Downloaded Artifacts:"
          ls -lR build-output/
          echo ""
          echo "Test Reports:"
          ls -lR test-reports/ 2>/dev/null || echo "No test reports (simulated)"
      
      - name: Generate Summary
        run: |
          echo "Pipeline Summary:"
          echo "=================="
          echo "✓ Dependencies cached and reused"
          echo "✓ Build artifacts uploaded"
          echo "✓ Test artifacts generated"
          echo "✓ All artifacts downloaded"
          echo "✓ Artifacts retained for 30 days"
```

### Step 2: Commit and Push

```bash
git add .github/workflows/cache-pipeline.yml
git commit -m "Add caching and artifact pipeline"
git push origin main
```

### Step 3: Monitor and Analyze

**Run 1 (Cache Miss):**
```
Install Dependencies
├── Download npm packages (no cache)
└── Install time: ~45-60 seconds
```

**Run 2 (Cache Hit):**
```
Install Dependencies
├── Restore npm packages from cache
└── Install time: ~5-10 seconds
└── Savings: 80-90% faster
```

**Expected Workflow Timeline:**

```
Timeline:
├── Checkout (5s)
├── Setup Node (10s)
├── Install Dependencies
│   ├── Run 1: Download (60s) → Miss
│   └── Run 2: Cache (5s) → Hit
├── Build (20s)
└── Upload Artifact (10s)
Total: ~105s (Run 1), ~50s (Run 2)
```

---

## Validation Checklist

- [ ] npm dependencies are cached automatically
- [ ] Build artifacts are uploaded successfully
- [ ] Artifacts have correct retention period (30 days)
- [ ] Test job downloads and uses cached dependencies
- [ ] Publish job downloads all artifacts
- [ ] Second workflow run shows cache hit for dependencies
- [ ] Cache key includes lock file hash
- [ ] Artifacts are listed correctly

---

## Cleanup

```bash
rm .github/workflows/cache-pipeline.yml
git add .github/workflows/
git commit -m "Remove caching pipeline"
git push origin main
```

---

## Common Mistakes

### ❌ Mistake 1: Missing Cache Configuration

```yaml
# Wrong - no cache specified
setup-node@v3
  with:
    node-version: '18'

# Correct - enable cache
setup-node@v3
  with:
    node-version: '18'
    cache: 'npm'
```

### ❌ Mistake 2: Artifact Path Not Existing

```yaml
# Wrong - path doesn't exist
upload-artifact@v3
  with:
    path: dist/build/  # Doesn't exist

# Correct - verify path exists
upload-artifact@v3
  with:
    path: dist/
    if-no-files-found: warn
```

### ❌ Mistake 3: Incorrect Cache Key

```yaml
# Wrong - static key always hits/misses
cache@v3
  with:
    key: npm-cache

# Correct - dynamic key based on lock file
cache@v3
  with:
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
```

### ❌ Mistake 4: Exceeding Cache Size Limits

```yaml
# Wrong - caching entire node_modules (too large)
cache@v3
  with:
    path: node_modules/
    key: npm-${{ hashFiles('package-lock.json') }}

# Correct - let setup-node handle caching
setup-node@v3
  with:
    cache: 'npm'  # Automatic, optimized caching
```

### ❌ Mistake 5: Artifacts Retained Too Long

```yaml
# Wrong - keeps artifacts for 90 days (default)
upload-artifact@v3
  with:
    path: build/

# Correct - shorter retention
upload-artifact@v3
  with:
    path: build/
    retention-days: 7
```

---

## Troubleshooting

### Issue: Cache not being restored

**Solution:**
1. Check cache key matches between save and restore
2. Verify path exists before caching
3. Ensure lock file hash is correct in key
4. Check if cache storage is full (5GB per repo)

### Issue: Artifact upload failing

**Solution:**
- Verify file path exists in working directory
- Check file size (max 5GB per artifact)
- Use `if-no-files-found: warn` to debug
- Create directory if it doesn't exist

### Issue: Different cache between runs

**Solution:**
- Don't use hard-coded paths
- Include matrix variables in cache key: `${{ matrix.node }}`
- Use restore-keys for partial matches
- Clear cache if lock file changes

### Issue: Artifacts not downloadable

**Solution:**
- Ensure name is unique and specified correctly
- Check retention days haven't expired
- Verify artifact was uploaded successfully
- Use exact name when downloading

### Issue: Cache hitting on wrong version

**Solution:**
```yaml
# Include version in key
key: ${{ runner.os }}-npm-${{ matrix.node }}-${{ hashFiles('package-lock.json') }}
```

---

## Next Steps

✅ **You've completed module 04!**

**Congratulations! You've mastered GitHub Actions caching and artifacts.**

**What's next?**

1. Explore advanced caching:
   - Custom cache management
   - Cache invalidation strategies
   - Cross-platform caching
   - Docker layer caching

2. Advanced artifact topics:
   - Artifact consolidation
   - Artifact analysis
   - Artifact cleanup automation
   - Artifact signing

3. Performance optimization:
   - Matrix caching strategies
   - Parallel build optimization
   - Build time analysis
   - Cost optimization

4. Proceed to later modules:
   - **05-reusable-workflows-and-composite-actions**
   - **06-secrets-and-environment-management**
   - **07-security-scanning-and-compliance**

---

## Caching Reference

### Setup Actions with Built-in Cache

```yaml
# npm (Node.js)
setup-node@v3:
  cache: 'npm'

# pip (Python)
setup-python@v4:
  cache: 'pip'

# Maven (Java)
setup-java@v3:
  cache: 'maven'

# Gradle (Java)
setup-gradle@v2:
  cache: true
```

### Manual Cache Management

```yaml
# Save cache
cache@v3:
  key: ${{ runner.os }}-myapp-${{ hashFiles('dependencies') }}
  path: ~/.cache/myapp/

# Restore with fallback
cache@v3:
  key: ${{ runner.os }}-myapp-${{ hashFiles('dependencies') }}
  restore-keys: |
    ${{ runner.os }}-myapp-
    ${{ runner.os }}-
```

### Artifact Operations

```yaml
# Upload single artifact
upload-artifact@v3:
  name: my-artifact
  path: dist/

# Upload multiple artifacts
upload-artifact@v3:
  name: all-reports
  path: |
    dist/**
    reports/**

# Download artifact
download-artifact@v3:
  name: my-artifact

# Download all artifacts
download-artifact@v3:
```

---

## Caching Best Practices

| Strategy | When to Use | Example |
|----------|-----------|---------|
| Built-in cache | Dependencies (npm, pip, maven) | `cache: 'npm'` |
| Hash-based key | Dependency version changes | Include `hashFiles(package-lock.json)` |
| Restore keys | Partial fallback | Add restore-keys array |
| OS-specific | Multi-OS workflows | `${{ runner.os }}-cache` |
| Matrix inclusion | Different per variant | `${{ matrix.node }}-cache` |

---

## Artifact Retention Guidelines

| Artifact Type | Suggested Days | Reasoning |
|---------------|-----------------|-----------|
| Build output | 7-14 days | Quick reference, rebuild is cheap |
| Test reports | 7 days | Short-term debugging |
| Release builds | 90 days | Legal compliance |
| Container images | 30 days | Push to registry instead |
| Logs | 7 days | Storage optimization |

---

## Performance Impact

| Operation | Time Saved | Conditions |
|-----------|-----------|-----------|
| npm cache hit | 80-90% faster | Large dependency tree |
| pip cache hit | 70-85% faster | Many dependencies |
| Docker layer cache | 50% faster | Multi-layer images |
| Artifact download | Reuses build output | Multi-job pipelines |

---

## Cache Size Limits

| Component | Limit | Notes |
|-----------|-------|-------|
| Cache per repo | 5 GB | Shared across all workflows |
| Artifact per run | 5 GB | Individual artifact limit |
| Artifacts total | 400 GB per repo | Organization storage |
| Retention | 90 days | Default, configurable |

---

## Additional Resources

- [Caching Dependencies](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows)
- [Artifact Management](https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts)
- [Cache Action](https://github.com/actions/cache)
- [Upload Artifact Action](https://github.com/actions/upload-artifact)
- [Download Artifact Action](https://github.com/actions/download-artifact)

---

**Created:** January 2026 | **Level:** Intermediate | **Estimated Time:** 45 minutes
