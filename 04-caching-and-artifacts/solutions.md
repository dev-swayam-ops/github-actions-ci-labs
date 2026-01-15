# Module 04 Solutions: Caching and Artifacts

## Exercise 1: Basic npm Dependency Caching

**Solution:**

```yaml
name: Basic Cache

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'  # Enables automatic npm caching
      - run: npm ci  # Uses cached packages on subsequent runs
      - run: npm run build
```

**Explanation:**

- `cache: 'npm'` enables automatic caching of npm packages
- GitHub Actions automatically creates a cache key based on the lock file hash
- First run: Dependencies downloaded (~45-60 seconds)
- Subsequent runs: Dependencies restored from cache (~5-10 seconds)
- Cache key: `${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}`
- The setup-node action handles all cache logic automatically

**Expected Behavior:**
- First workflow run shows "Restoring cache (miss)" in logs
- Second workflow run shows "Restoring cache (hit)" and faster installation
- Approximately 80-90% reduction in install time on cache hit

---

## Exercise 2: Python pip Caching

**Solution:**

```yaml
name: Python Cache

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.10'
          cache: 'pip'  # Enables automatic pip caching
      - run: pip install -r requirements.txt
      - run: pytest
```

**Explanation:**

- `cache: 'pip'` enables automatic caching of Python packages
- GitHub Actions looks for `requirements.txt`, `setup.py`, or `pyproject.toml`
- Cache key includes the dependency file hash automatically
- Works similarly to npm caching but for Python environments
- Saves 70-85% time on dependency installation
- setup-python action handles cache invalidation automatically

**Key Differences from npm:**
- Uses pip instead of npm ci
- Caches pip packages instead of node_modules
- Cache key based on requirements files instead of lock file
- Similar automatic management as npm caching

---

## Exercise 3: Manual Cache with Custom Key

**Solution:**

```yaml
name: Manual Cache

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/cache@v3
        with:
          key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-npm-
            ${{ runner.os }}-
          path: node_modules/
      - run: npm ci
```

**Explanation:**

- `key`: Unique cache identifier that changes when lock file changes
- `hashFiles()`: Creates hash of file contents for cache key
- `restore-keys`: Fallback patterns if exact key not found
- First pattern `${{ runner.os }}-npm-` matches any npm cache
- Second pattern `${{ runner.os }}-` matches any cache for the OS
- Cache path specifies what to cache (node_modules)

**Cache Flow:**
1. Workflow checks for cache matching `${{ runner.os }}-npm-<hash>`
2. If found → restored (hit)
3. If not found → tries `${{ runner.os }}-npm-` pattern
4. If not found → tries `${{ runner.os }}-` pattern
5. If all miss → no cache available
6. After job → cache saved automatically

**Benefits of Manual Cache:**
- Fine-grained control over what to cache
- Custom paths for special dependencies
- Fallback restore strategies

---

## Exercise 4: Uploading Build Artifacts

**Solution:**

```yaml
name: Upload Artifacts

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci && npm run build
      - name: Upload Artifact
        uses: actions/upload-artifact@v3
        with:
          name: build-output
          path: dist/
          retention-days: 30
```

**Explanation:**

- `name: build-output` identifies the artifact in the UI
- `path: dist/` specifies the directory to upload
- `retention-days: 30` keeps artifact for 30 days then deletes
- Artifacts available in "Artifacts" section of workflow run
- Can be downloaded manually or used in other jobs
- Default retention is 90 days if not specified

**Artifact Lifecycle:**
1. Build creates files in dist/
2. upload-artifact action zips the directory
3. Artifact stored on GitHub (counts toward storage quota)
4. Available for download for 30 days
5. Automatically deleted after 30 days

**Where to Download:**
- Actions tab → Select workflow run → "Artifacts" section
- Direct download in workflow UI
- Programmatic access via GitHub API

---

## Exercise 5: Artifact Conditional Upload

**Solution:**

```yaml
name: Conditional Upload

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - name: Build
        id: build
        run: npm ci && npm run build
      - name: Upload on Success
        uses: actions/upload-artifact@v3
        if: success()
        with:
          name: build-artifact
          path: dist/
          retention-days: 30
```

**Explanation:**

- `if: success()` uploads only if previous steps succeeded
- Alternative: `if: failure()` uploads only if previous steps failed
- Alternative: `if: always()` uploads regardless of previous status
- Step ID `id: build` enables referencing in conditions
- `if: steps.build.outcome == 'success'` more explicit condition

**Common Conditions:**
- `success()` - All previous steps succeeded
- `failure()` - Any previous step failed
- `always()` - Run regardless of previous outcome
- `cancelled()` - Workflow was cancelled
- `steps.build.outcome == 'success'` - Specific step outcome
- `contains(github.event.head_commit.message, 'skip-artifacts')` - Custom logic

**Use Cases:**
- Upload artifacts only on successful builds
- Archive logs only on failure for debugging
- Upload test reports regardless of test results
- Conditional cleanup based on build status

---

## Exercise 6: Multiple Artifact Patterns

**Solution:**

```yaml
name: Multiple Patterns

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci && npm test
      - name: Upload All Reports
        uses: actions/upload-artifact@v3
        with:
          name: test-reports
          path: |
            reports/
            coverage/
          retention-days: 7
```

**Explanation:**

- `path:` accepts multiple patterns (one per line)
- `reports/` matches entire directory
- `coverage/` matches another directory
- Single artifact contains both directories
- Glob patterns supported: `**/*.html`, `**/index.json`, etc.

**Glob Pattern Examples:**

```yaml
path: |
  dist/              # Include entire directory
  coverage/          # Include entire directory
  reports/*.html     # Include HTML files in reports/
  **/*.log           # Include all .log files recursively
  build/**           # Include all files in build/ tree
  src/index.js       # Include specific file
```

**Best Practices:**
- Use descriptive artifact names
- Group related artifacts together
- Don't include large directories unnecessarily
- Consider compression for large artifacts
- Use appropriate retention periods

---

## Exercise 7: Downloading and Using Artifacts

**Solution:**

```yaml
name: Download Artifacts

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: mkdir build && echo "artifact" > build/output.txt
      - uses: actions/upload-artifact@v3
        with:
          name: my-build
          path: build/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download Build
        uses: actions/download-artifact@v3
        with:
          name: my-build
          path: ./build-artifacts
      - run: ls -la build-artifacts/
      - run: cat build-artifacts/output.txt
```

**Explanation:**

- `needs: build` creates job dependency
- `download-artifact` waits for build job to complete
- `name: my-build` matches upload artifact name exactly
- `path: ./build-artifacts` downloads to this directory
- Files extracted automatically, ready to use

**Download Behavior:**
1. Artifact zipped during upload
2. When downloaded, automatically extracted
3. Original directory structure preserved
4. Files available immediately for processing

**Job Dependency Flow:**
```
build job (uploads artifact)
    ↓
deploy job (download-artifact waits)
    ↓
deploy job continues (artifact available)
```

**Use Cases:**
- Build once, test in multiple jobs
- Build, then deploy separately
- Parallel jobs that need build output
- Multi-stage pipelines

---

## Exercise 8: Caching with Matrix Strategy

**Solution:**

```yaml
name: Matrix Cache

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [16, 18, 20]
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'  # Automatically includes matrix.node in key
      - run: npm ci
      - run: npm test
```

**Explanation:**

- `setup-node` automatically includes `matrix.node` in cache key
- Each Node version gets separate cache entry
- Cache key automatically becomes:
  `${{ runner.os }}-npm-${{ matrix.node }}-${{ hashFiles('package-lock.json') }}`
- 3 matrix jobs = 3 separate caches (if all versions differ)
- No conflicts between cache entries

**Why Separate Caches:**
- Node 16 might install different versions than 18
- Lock files may resolve differently per version
- Separate caches prevent version mismatches
- setup-node handles this automatically

**Manual Cache with Matrix:**

```yaml
- uses: actions/cache@v3
  with:
    # Include matrix variable in key
    key: ${{ runner.os }}-npm-${{ matrix.node }}-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-${{ matrix.node }}-
      ${{ runner.os }}-npm-
    path: node_modules/
```

**Performance with Matrix:**
- 3 jobs × 60s per job = 180s total
- With caching hit: 3 jobs × 10s per job = 30s total
- Savings: ~150s per workflow run
- Multiplied across many runs = significant time savings

---

## Exercise 9: Cleanup and Artifact Management

**Solution:**

```yaml
name: Artifact Lifecycle

on: [push]

jobs:
  create:
    runs-on: ubuntu-latest
    steps:
      - run: mkdir -p artifacts && echo "data" > artifacts/report.txt
      - name: Upload Short-lived
        uses: actions/upload-artifact@v3
        with:
          name: debug-logs
          path: artifacts/
          retention-days: 3
      - name: Upload Long-lived
        uses: actions/upload-artifact@v3
        with:
          name: release-build
          path: artifacts/
          retention-days: 90
```

**Explanation:**

- `retention-days: 3` deletes debug logs after 3 days
- `retention-days: 90` keeps releases for 90 days
- Different retention for different artifact types
- Saves storage space automatically
- Respects organization storage quotas

**Retention Strategy Guidelines:**

| Type | Days | Reason |
|------|------|--------|
| Debug logs | 1-3 | Quick reference, rebuild is cheap |
| Test reports | 3-7 | Debugging failures, storage optimization |
| Build artifacts | 7-14 | Reuse possibility, storage limits |
| Release builds | 30-90 | Legal/compliance requirements |
| Production releases | 90+ | Long-term archive |

**Artifact Storage Monitoring:**
- Check Settings → Actions → Artifact and log retention
- Monitor total artifact storage per repository
- Clean up old artifacts manually if needed
- Set organization-wide retention policies

**Storage Limits:**
- Per repo: 5GB cache, 400GB artifacts
- Per artifact: 5GB max size
- Retention: Default 90 days, configurable 1-90 days

---

## Exercise 10: Cross-Workflow Artifact Sharing

**Solution:**

```yaml
name: Cross-Workflow Artifacts

on: [workflow_dispatch]

jobs:
  use-artifact:
    runs-on: ubuntu-latest
    steps:
      - name: Download from Previous Run
        uses: actions/download-artifact@v3
        with:
          name: build-output
          path: ./artifacts
      - name: Verify Download
        run: |
          echo "Checking downloaded files:"
          ls -la artifacts/ || echo "No files found (expected if artifact doesn't exist)"
      - name: Use Downloaded Files
        run: |
          if [ -d artifacts ]; then
            echo "Artifacts found, processing..."
            # Use the downloaded artifact
          fi
```

**Explanation:**

- `workflow_dispatch` allows manual trigger
- `download-artifact` retrieves artifacts from previous runs
- Must reference existing artifact name exactly
- Artifacts must still be within retention period
- Useful for manual deployments, multi-stage releases

**Cross-Workflow Patterns:**

```yaml
# Workflow 1: Build and upload
jobs:
  build:
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v3
        with:
          name: build-${{ github.run_number }}
          path: dist/
          retention-days: 30

# Workflow 2: Deploy from artifact (manual trigger)
on: [workflow_dispatch]
jobs:
  deploy:
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: build-${{ github.run_number }}
          path: ./build
      - run: npm run deploy
```

**Advanced: Using GitHub API**

```bash
# Download artifact via API
curl -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/owner/repo/actions/artifacts?name=build-output" \
  -o artifact.zip
```

**Use Cases:**
- Separate build pipeline from deploy pipeline
- Manual approval before deployment
- Scheduled deployments of build artifacts
- Artifact reuse across workflows
- Build once, deploy multiple times

---

## Summary of Solutions

**Key Takeaways:**

1. **Caching** - Reduces dependency installation time 80-90%
2. **Artifacts** - Enables multi-job workflows and manual access
3. **Retention** - Manages storage costs with appropriate timeframes
4. **Matrix** - Separate caches per variant automatically
5. **Conditional** - Upload artifacts only when needed
6. **Patterns** - Multiple paths in single artifact
7. **Download** - Reuse artifacts across jobs
8. **Lifecycle** - Plan retention for different artifact types

**Performance Impact:**

```
Without caching/artifacts:
├── Job 1: Install (60s) + Build (20s) = 80s
├── Job 2: Install (60s) + Test (30s) = 90s
└── Total: 170s

With caching/artifacts:
├── Job 1: Install cache (5s) + Build (20s) = 25s, Upload (10s) = 35s
├── Job 2: Install cache (5s) + Test (30s) = 35s, Download (5s) = 40s
└── Total: 75s (55% faster)
```

**Best Practices Summary:**
- ✅ Enable caching for all dependency installations
- ✅ Use hash-based cache keys
- ✅ Set appropriate retention periods
- ✅ Upload artifacts only when needed
- ✅ Include matrix variables in cache keys
- ✅ Monitor storage quota usage
- ✅ Test cache invalidation scenarios
- ✅ Document artifact retention policies

---

**Mastery Checkpoint:**

You should now understand:
- How caching works and improves performance
- Automatic vs manual cache management
- Artifact upload, download, and retention
- Cache keys and invalidation strategies
- Matrix-aware caching
- Multi-job artifact sharing
- Artifact lifecycle management

Proceed to the next module or practice with your own projects!
