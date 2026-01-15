# Module 09 Solutions: Monorepo and Path Filters

## Exercise 1: Basic Monorepo Structure

**Solution:**

```bash
# Create directory structure
mkdir -p my-repo/services/{api,web}/src
cd my-repo
```

`package.json` (root):
```json
{
  "name": "my-monorepo",
  "version": "1.0.0",
  "private": true,
  "workspaces": [
    "services/api",
    "services/web"
  ]
}
```

`services/api/package.json`:
```json
{
  "name": "@myapp/api",
  "version": "1.0.0",
  "main": "src/index.js",
  "scripts": {
    "test": "echo 'API tests'",
    "build": "echo 'API build'"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

`services/web/package.json`:
```json
{
  "name": "@myapp/web",
  "version": "1.0.0",
  "main": "src/index.js",
  "scripts": {
    "test": "echo 'Web tests'",
    "build": "echo 'Web build'"
  },
  "dependencies": {
    "react": "^18.0.0"
  }
}
```

**Verify:**
```bash
npm install
# node_modules appears at root
# Both services linked
npm ls
```

---

## Exercise 2: Path Filter Triggers

**Solution:**

```yaml
name: API Workflow

on:
  push:
    branches: [main]
    paths:
      - 'services/api/**'
      - 'services/shared/**'
      - 'package*.json'
      - '.github/workflows/api-ci.yml'
  pull_request:
    branches: [main]
    paths:
      - 'services/api/**'
      - 'services/shared/**'
      - 'package*.json'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build API
        run: echo "Building API"
```

**How it works:**
- Pushes to main trigger if any file in listed paths changed
- PR also triggers on same paths
- Ignores changes to other services
- Changes to README.md alone don't trigger

**Test locally:**
```bash
# See what would trigger
git diff --name-only origin/main..HEAD
```

---

## Exercise 3: Working Directory Configuration

**Solution:**

```yaml
name: Multi-Service Build

on: [push]

jobs:
  api:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: services/api
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci --workspaces
      - run: npm test
      - run: npm run build
  
  web:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: services/web
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci --workspaces
      - run: npm test
      - run: npm run build
```

**Explanation:**
- `defaults.run.working-directory` applies to all steps
- `npm ci --workspaces` installs entire workspace from that directory
- Commands run in service context without path prefix
- Tests and build use service-specific scripts

---

## Exercise 4: Conditional Workflow Execution

**Solution:**

```yaml
name: Conditional Build

on:
  push:
    branches: [main]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      api-changed: ${{ steps.check.outputs.api }}
      web-changed: ${{ steps.check.outputs.web }}
    
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Check Changes
        id: check
        run: |
          CHANGED=$(git diff --name-only origin/main..HEAD)
          
          if echo "$CHANGED" | grep -q "^services/api/"; then
            echo "api=true" >> $GITHUB_OUTPUT
          else
            echo "api=false" >> $GITHUB_OUTPUT
          fi
          
          if echo "$CHANGED" | grep -q "^services/web/"; then
            echo "web=true" >> $GITHUB_OUTPUT
          else
            echo "web=false" >> $GITHUB_OUTPUT
          fi

  build-api:
    needs: detect-changes
    if: needs.detect-changes.outputs.api-changed == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building API"

  build-web:
    needs: detect-changes
    if: needs.detect-changes.outputs.web-changed == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building Web"
```

**Explanation:**
- `detect-changes` analyzes which services changed
- Outputs boolean for each service
- `build-api` runs only if API changed (using `if:`)
- `build-web` runs only if Web changed
- Efficient: skips unnecessary builds

---

## Exercise 5: Matrix Strategy for Services

**Solution:**

```yaml
name: Matrix Build

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [api, web, admin]
    
    defaults:
      run:
        working-directory: services/${{ matrix.service }}
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: services/${{ matrix.service }}/package-lock.json
      
      - run: npm ci --workspaces
      - run: npm run lint 2>/dev/null || true
      - run: npm test
      - run: npm run build
      
      - name: Summary
        run: echo "✓ ${{ matrix.service }} build complete"
```

**How it works:**
- Creates 3 parallel jobs (one per service)
- Each job runs in its service directory
- Cache includes service name (cache per service)
- All steps reference `${{ matrix.service }}`
- Jobs run in parallel for speed

---

## Exercise 6: Artifact Collection from Multiple Services

**Solution:**

```yaml
name: Collect Artifacts

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [api, web]
    
    defaults:
      run:
        working-directory: services/${{ matrix.service }}
    
    steps:
      - uses: actions/checkout@v3
      - name: Build Service
        run: |
          mkdir -p dist
          echo "Build output for ${{ matrix.service }}" > dist/output.txt
          echo "Service: ${{ matrix.service }}" >> dist/info.txt
      
      - name: Upload Artifact
        uses: actions/upload-artifact@v3
        with:
          name: ${{ matrix.service }}-build
          path: services/${{ matrix.service }}/dist
          retention-days: 7

  collect:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      - name: Download API Artifact
        uses: actions/download-artifact@v3
        with:
          name: api-build
          path: artifacts/api
      
      - name: Download Web Artifact
        uses: actions/download-artifact@v3
        with:
          name: web-build
          path: artifacts/web
      
      - name: Combine Artifacts
        run: |
          mkdir -p final-release
          cp -r artifacts/* final-release/
          ls -la final-release
      
      - name: Upload Combined
        uses: actions/upload-artifact@v3
        with:
          name: all-builds
          path: final-release
```

**Explanation:**
- Each service uploads with unique name
- Collect job downloads each separately
- Combines into single directory
- Final artifact contains all builds

---

## Exercise 7: Shared Dependency Management

**Solution:**

```yaml
on:
  push:
    paths:
      - 'services/api/**'
      - 'services/web/**'
      - 'shared/**'
      - 'package*.json'

jobs:
  test-shared:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: shared
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci --workspaces
      - run: npm test
  
  test-api:
    needs: test-shared
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: services/api
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci --workspaces
      - run: npm test
  
  test-web:
    needs: test-shared
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: services/web
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci --workspaces
      - run: npm test
```

**Explanation:**
- Path filter includes shared/**
- test-shared runs first
- API and web tests depend on shared
- If shared breaks, services fail tests
- Atomic validation of changes

---

## Exercise 8: Change Detection Script

**Solution:**

`scripts/detect-changes.sh`:
```bash
#!/bin/bash

# Get changed files
if [ -z "$GITHUB_BASE_REF" ]; then
  # Push event
  CHANGED=$(git diff --name-only ${{ github.event.before }}..${{ github.sha }})
else
  # PR event
  CHANGED=$(git diff --name-only origin/$GITHUB_BASE_REF..HEAD)
fi

# Check what changed
API_CHANGED=false
WEB_CHANGED=false
SHARED_CHANGED=false

if echo "$CHANGED" | grep -q "^services/api/"; then
  API_CHANGED=true
fi

if echo "$CHANGED" | grep -q "^services/web/"; then
  WEB_CHANGED=true
fi

if echo "$CHANGED" | grep -q "^shared/"; then
  SHARED_CHANGED=true
fi

# Output for GitHub Actions
echo "api=$API_CHANGED" >> $GITHUB_OUTPUT
echo "web=$WEB_CHANGED" >> $GITHUB_OUTPUT
echo "shared=$SHARED_CHANGED" >> $GITHUB_OUTPUT

# Display
echo "Changes detected:"
echo "  API: $API_CHANGED"
echo "  Web: $WEB_CHANGED"
echo "  Shared: $SHARED_CHANGED"
```

**Usage in workflow:**
```yaml
- name: Detect Changes
  id: changes
  run: bash scripts/detect-changes.sh

- name: Log Changes
  run: |
    echo "API Changed: ${{ steps.changes.outputs.api }}"
    echo "Web Changed: ${{ steps.changes.outputs.web }}"
```

---

## Exercise 9: Cache Optimization for Monorepo

**Solution:**

```yaml
name: Optimized Cache

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [api, web]
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: services/${{ matrix.service }}/package-lock.json
      
      # Cache npm cache directory
      - uses: actions/cache@v3
        with:
          path: ~/.npm
          key: npm-cache-${{ runner.os }}-${{ matrix.service }}-${{ hashFiles(format('services/{0}/package-lock.json', matrix.service)) }}
          restore-keys: |
            npm-cache-${{ runner.os }}-${{ matrix.service }}-
      
      # Cache build artifacts
      - uses: actions/cache@v3
        with:
          path: services/${{ matrix.service }}/dist
          key: build-${{ matrix.service }}-${{ github.sha }}
          restore-keys: |
            build-${{ matrix.service }}-
      
      - run: npm ci --workspaces
      - run: npm test
      - run: npm run build
```

**Explanation:**
- `cache: 'npm'` uses setup-node's automatic caching
- Additional cache for npm directory
- Build artifacts cached with commit SHA
- Separate caches per service
- Cache keys include service name

---

## Exercise 10: Cross-Service Dependency Validation

**Solution:**

```yaml
name: Dependency Check

on:
  push:
    paths:
      - 'services/**'
      - 'shared/**'
      - 'package*.json'

jobs:
  validate-dependencies:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci --workspaces
      
      - name: List All Dependencies
        run: npm list --all 2>&1 | head -50
      
      - name: Audit Dependencies
        run: npm audit --audit-level=moderate
      
      - name: Check for Duplicates
        run: npm list --depth=0 --all | grep " deduped"
      
      - name: Validate Shared References
        run: |
          echo "Checking services reference shared..."
          
          for service in services/api services/web; do
            if [ -f "$service/package.json" ]; then
              if grep -q '"@myapp/shared"' "$service/package.json"; then
                echo "✓ $service references shared"
              else
                echo "⚠ $service doesn't reference shared"
              fi
            fi
          done
      
      - name: Version Consistency
        run: |
          echo "Checking version consistency..."
          
          ROOT_VERSION=$(jq -r '.version' package.json)
          
          for service in services/*/package.json; do
            SERVICE_VERSION=$(jq -r '.version' "$service")
            if [ "$ROOT_VERSION" != "$SERVICE_VERSION" ]; then
              echo "⚠ Version mismatch in $service"
            fi
          done
      
      - name: Dependency Report
        run: |
          echo "## Dependency Report" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "### Services" >> $GITHUB_STEP_SUMMARY
          npm list --depth=0 >> $GITHUB_STEP_SUMMARY 2>&1 || true
```

**Explanation:**
- `npm list` shows all dependencies
- `npm audit` finds vulnerabilities
- Duplicate detection optimizes installs
- Cross-references validated
- Version consistency checked

---

## Summary

You now understand:
- ✅ Basic monorepo structure with npm workspaces
- ✅ Path filter triggers for selective workflows
- ✅ Working directory configuration per job
- ✅ Conditional execution based on changes
- ✅ Matrix strategies for multi-service builds
- ✅ Artifact collection from multiple services
- ✅ Shared dependency management
- ✅ Automated change detection
- ✅ Optimized caching strategies
- ✅ Cross-service dependency validation

Continue mastering monorepo CI/CD!
