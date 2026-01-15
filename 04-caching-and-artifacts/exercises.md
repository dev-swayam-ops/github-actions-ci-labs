# 04-Caching and Artifacts

## Exercise 1: Basic npm Dependency Caching

**Objective:** Set up automatic npm caching in a workflow

**Acceptance Criteria:**
- Workflow uses `actions/setup-node@v3` with `cache: 'npm'`
- Dependencies are cached automatically
- Cache key includes lock file hash
- Subsequent runs show reduced installation time

**Starter Code:**
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
          # Add cache configuration
      - run: npm ci
      - run: npm run build
```

**Expected Output:**
```
Run 1: npm ci (with download) ~45-60s
Run 2: npm ci (from cache) ~5-10s
```

---

## Exercise 2: Python pip Caching

**Objective:** Implement pip dependency caching for Python projects

**Acceptance Criteria:**
- `actions/setup-python@v4` configured with `cache: 'pip'`
- requirements.txt used for cache validation
- Cache restored on subsequent runs
- Installation time reduced significantly

**Starter Code:**
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
          # Add cache configuration
      - run: pip install -r requirements.txt
      - run: pytest
```

**Hint:** Similar to npm, but uses `cache: 'pip'`

---

## Exercise 3: Manual Cache with Custom Key

**Objective:** Implement manual caching for custom paths with hash-based keys

**Acceptance Criteria:**
- Cache key includes lock file hash
- Cache path is correctly specified
- Restore keys provide fallback options
- Cache uses `hashFiles()` function

**Starter Code:**
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
          # Add key with hashFiles for package-lock.json
          key: 
          # Add restore keys for fallback
          restore-keys: |
          path: node_modules/
      - run: npm ci
```

**Expected:** Cache key should be `${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}`

---

## Exercise 4: Uploading Build Artifacts

**Objective:** Save workflow outputs as downloadable artifacts

**Acceptance Criteria:**
- Build output uploaded with `actions/upload-artifact@v3`
- Artifact has descriptive name
- Retention period set to 30 days
- Artifact appears in workflow run

**Starter Code:**
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
          # Add artifact name
          name: 
          # Add path to build output
          path: 
          # Add retention days
          retention-days: 
```

**Expected:** Artifact named `build-output` available for download

---

## Exercise 5: Artifact Conditional Upload

**Objective:** Upload artifacts only on successful builds

**Acceptance Criteria:**
- Artifact upload conditional on previous step success
- Uses `if:` condition
- Different retention for different artifact types
- Upload only when build succeeds

**Starter Code:**
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
        with:
          name: build-artifact
          path: dist/
          # Add conditional execution
          if: 
          retention-days: 30
```

**Hint:** Use `if: success()` or `if: steps.build.outcome == 'success'`

---

## Exercise 6: Multiple Artifact Patterns

**Objective:** Upload multiple files matching glob patterns

**Acceptance Criteria:**
- Single upload-artifact action with multiple paths
- Uses glob patterns (*, **, etc.)
- Handles both files and directories
- All matching files included in artifact

**Starter Code:**
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
            # Add glob pattern for test results
            
            # Add glob pattern for coverage
            
          retention-days: 7
```

**Expected:** Both `reports/` and `coverage/` directories uploaded together

---

## Exercise 7: Downloading and Using Artifacts

**Objective:** Download uploaded artifacts in subsequent jobs

**Acceptance Criteria:**
- `actions/download-artifact@v3` downloads specific artifact
- Downloads to correct directory
- Subsequent steps can use downloaded files
- Works in different jobs

**Starter Code:**
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
          # Add artifact name
          name: 
          # Add download path
          path: 
      - run: ls -la
```

**Expected:** build/ directory available in deploy job

---

## Exercise 8: Caching with Matrix Strategy

**Objective:** Implement proper caching in matrix workflows

**Acceptance Criteria:**
- Cache key includes matrix variable
- Each matrix variation has separate cache
- Cache restored correctly per version
- No cache conflicts between matrix jobs

**Starter Code:**
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
          cache: 'npm'
      - run: npm ci
      - run: npm test
```

**Hint:** setup-node action automatically includes matrix.node in cache key

---

## Exercise 9: Cleanup and Artifact Management

**Objective:** Manage artifact lifecycle and storage efficiently

**Acceptance Criteria:**
- Set appropriate retention days per artifact type
- Delete old artifacts to save storage
- Document artifact cleanup policy
- Monitor artifact storage usage

**Starter Code:**
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
          # Add short retention for debug logs
          retention-days: 
      - name: Upload Long-lived
        uses: actions/upload-artifact@v3
        with:
          name: release-build
          path: artifacts/
          # Add longer retention for releases
          retention-days: 
```

**Expected:** Debug logs deleted after 3 days, releases kept for 90 days

---

## Exercise 10: Cross-Workflow Artifact Sharing

**Objective:** Share artifacts between different workflow runs using download action

**Acceptance Criteria:**
- Artifacts downloaded from previous workflow run
- Works across different workflow files
- Uses run ID or artifact name
- Demonstrates artifact persistence

**Starter Code:**
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
          # Add artifact name from another workflow
          name: 
          # Add path for downloaded files
          path: 
      - name: Verify Download
        run: |
          echo "Checking downloaded files:"
          ls -la || echo "No files found (expected if artifact doesn't exist)"
```

**Challenge:** This is advanced - requires artifact from another workflow to exist first

---

## Summary

**Key Learning Points:**
- ✅ Automatic caching with setup actions (npm, pip, maven)
- ✅ Manual cache with hash-based keys
- ✅ Artifact upload with retention policies
- ✅ Artifact download and usage
- ✅ Caching in matrix workflows
- ✅ Artifact lifecycle management

**Next Steps:**
- Experiment with cache sizes and key strategies
- Monitor cache hit rates in your workflows
- Implement artifact cleanup policies
- Combine caching with artifact management

**Continue to Exercise Solutions for detailed implementations!**
