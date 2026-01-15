# Module 09 Exercises: Monorepo and Path Filters

## Exercise 1: Basic Monorepo Structure

**Objective:** Create a simple monorepo with two services.

**Starter Code:**

```bash
# Repository structure to create
my-repo/
├── package.json                 # Root workspace
├── services/
│   ├── api/
│   │   ├── package.json        # TODO: API service
│   │   └── src/index.js
│   └── web/
│       ├── package.json        # TODO: Web service
│       └── src/index.js
└── .gitignore
```

**Requirements:**
- Create package.json in root with workspaces array
- Create separate package.json for each service
- Each service should have a unique name
- npm install should work from root

**Acceptance Criteria:**
- [ ] Root package.json has workspaces defined
- [ ] Both services have independent package.json
- [ ] npm install succeeds
- [ ] node_modules created at root
- [ ] Services are linked in workspace

**Hint:** npm workspaces is standard in npm v7+

---

## Exercise 2: Path Filter Triggers

**Objective:** Setup workflows triggered only on specific path changes.

**Starter Code:**

```yaml
name: API Workflow

on:
  push:
    branches: [main]
    paths:
      # TODO: Add paths for api service
      # TODO: Add paths for shared/common changes
      # TODO: Add root package.json
  pull_request:
    branches: [main]
    paths:
      # TODO: Same paths as push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build API
        run: echo "Building API"
```

**Requirements:**
- Trigger on services/api/** changes
- Trigger on services/shared/** changes
- Trigger on package*.json changes
- Same paths for push and pull_request

**Acceptance Criteria:**
- [ ] Workflow file is valid YAML
- [ ] Paths specified for both on.push and on.pull_request
- [ ] Pattern matches api service files
- [ ] Pattern matches shared files
- [ ] Pattern matches package files
- [ ] README.md changes don't trigger

**Hint:** Use glob patterns with ** for recursive matching.

---

## Exercise 3: Working Directory Configuration

**Objective:** Set working directory for multi-service builds.

**Starter Code:**

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
      
      # TODO: Install dependencies with workspace flag
      # TODO: Run tests for API
      # TODO: Build API service
  
  web:
    runs-on: ubuntu-latest
    # TODO: Set default working directory for web service
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci --workspaces
      # TODO: Run tests
      # TODO: Build service
```

**Requirements:**
- Set default working-directory per job
- Install workspace dependencies
- Run service-specific commands
- Tests run from correct directory

**Acceptance Criteria:**
- [ ] working-directory set for each job
- [ ] npm ci includes --workspaces flag
- [ ] Commands run in correct directory context
- [ ] No relative path errors
- [ ] Both jobs execute successfully

**Hint:** defaults.run.working-directory sets context for all steps.

---

## Exercise 4: Conditional Workflow Execution

**Objective:** Skip jobs based on which projects changed.

**Starter Code:**

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
          # TODO: Get list of changed files
          # TODO: Check if api service changed
          # TODO: Check if web service changed
          # TODO: Output results
          echo "api=true" >> $GITHUB_OUTPUT

  build-api:
    needs: detect-changes
    # TODO: Run only if API changed
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building API"

  build-web:
    needs: detect-changes
    # TODO: Run only if Web changed
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building Web"
```

**Requirements:**
- Detect which services changed
- Output detection results
- Skip jobs for unchanged services
- Use `if:` conditions for job execution

**Acceptance Criteria:**
- [ ] detect-changes job identifies changes
- [ ] Outputs set correctly (true/false)
- [ ] build-api runs only if API changed
- [ ] build-web runs only if Web changed
- [ ] Both can run if both changed

**Hint:** Use git diff to compare files, output to environment variable.

---

## Exercise 5: Matrix Strategy for Services

**Objective:** Build multiple services with a matrix strategy.

**Starter Code:**

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
      
      # TODO: Install dependencies
      # TODO: Run lint
      # TODO: Run tests
      # TODO: Build service
```

**Requirements:**
- Matrix includes api, web, admin services
- Working directory per matrix element
- Cache path includes service name
- All steps reference matrix.service

**Acceptance Criteria:**
- [ ] Matrix strategy creates 3 jobs
- [ ] Each job runs in correct directory
- [ ] Cache key includes service name
- [ ] All steps reference ${{ matrix.service }}
- [ ] Jobs run in parallel (or sequential with dependencies)

**Hint:** Use ${{ matrix.service }} in strings for dynamic paths.

---

## Exercise 6: Artifact Collection from Multiple Services

**Objective:** Collect build artifacts from multiple services.

**Starter Code:**

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
      
      # TODO: Upload artifact with service name
      # TODO: Use matrix.service in artifact name
      # TODO: Include dist folder

  collect:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      # TODO: Download all artifacts
      # TODO: Combine artifacts
      # TODO: Create final artifact
```

**Requirements:**
- Each service uploads its own artifact
- Artifact named with service identifier
- Collect job downloads all artifacts
- Combine artifacts into single output

**Acceptance Criteria:**
- [ ] Each job uploads artifact
- [ ] Artifact name includes service
- [ ] collect job downloads artifacts
- [ ] Artifacts successfully combined
- [ ] Final artifact contains all outputs

**Hint:** Use unique names for each artifact: `${{ matrix.service }}-artifact`

---

## Exercise 7: Shared Dependency Management

**Objective:** Manage shared code used by multiple services.

**Starter Code:**

```yaml
on:
  push:
    paths:
      - 'services/api/**'
      - 'services/web/**'
      - 'shared/**'              # Include shared folder
      - 'package*.json'          # Include root package

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      
      # TODO: Install with workspace support
      # TODO: Run shared module tests
      # TODO: Run API service tests
      # TODO: Run Web service tests
```

**Requirements:**
- Shared folder included in path filters
- npm workspaces installation
- Test shared modules before services
- Services test with shared updates

**Acceptance Criteria:**
- [ ] Path filter includes shared/**
- [ ] package*.json in path filter
- [ ] npm ci --workspaces used
- [ ] Shared tests run first
- [ ] All tests pass

**Hint:** Order tests: shared → api → web

---

## Exercise 8: Change Detection Script

**Objective:** Create script to detect which services changed.

**Starter Code:**

```bash
#!/bin/bash
# detect-changes.sh

# TODO: Get list of changed files
# TODO: Check if api folder changed
# TODO: Check if web folder changed
# TODO: Check if shared folder changed
# TODO: Output results to environment variable

CHANGED_FILES=$(git diff --name-only ${{ github.event.pull_request.base.sha }}...HEAD)

if echo "$CHANGED_FILES" | grep -q "^services/api/"; then
  API_CHANGED=true
fi

if echo "$CHANGED_FILES" | grep -q "^services/web/"; then
  WEB_CHANGED=true
fi

if echo "$CHANGED_FILES" | grep -q "^shared/"; then
  SHARED_CHANGED=true
fi

# TODO: Output to GitHub actions output format
```

**Requirements:**
- Detect changes in each service
- Detect shared module changes
- Output in GitHub Actions format
- Use in workflow conditions

**Acceptance Criteria:**
- [ ] Script identifies api changes
- [ ] Script identifies web changes
- [ ] Script identifies shared changes
- [ ] Outputs compatible with GitHub Actions
- [ ] Used in conditional job execution

**Hint:** Output format: `echo "var=value" >> $GITHUB_OUTPUT`

---

## Exercise 9: Cache Optimization for Monorepo

**Objective:** Setup efficient caching for monorepo builds.

**Starter Code:**

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
          # TODO: Set cache-dependency-path for each service
      
      # TODO: Cache shared modules separately
      # TODO: Cache service dependencies
      # TODO: Use matrix service in cache key
      
      - run: npm ci --workspaces
      - run: npm test
```

**Requirements:**
- Separate cache per service
- Cache includes matrix service
- Shared dependencies cached efficiently
- Different cache keys for different services

**Acceptance Criteria:**
- [ ] cache: 'npm' configured
- [ ] cache-dependency-path includes service
- [ ] Separate caches per service
- [ ] Cache hits on unchanged services
- [ ] Cache busts only for changed services

**Hint:** cache-dependency-path: `services/${{ matrix.service }}/package-lock.json`

---

## Exercise 10: Cross-Service Dependency Validation

**Objective:** Validate dependencies between monorepo services.

**Starter Code:**

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
      
      - run: npm ci --workspaces
      
      # TODO: Check for missing dependencies
      # TODO: Check for circular dependencies
      # TODO: Verify shared module references
      # TODO: Validate package versions match
      
      - name: Dependency Report
        run: |
          npm list --all
          # TODO: Custom validation script
```

**Requirements:**
- Verify all dependencies installed
- Check for missing packages
- Detect circular dependencies
- Validate cross-service references

**Acceptance Criteria:**
- [ ] npm list reports all dependencies
- [ ] No missing dependency warnings
- [ ] No circular dependency detection
- [ ] All services can reference shared
- [ ] Version conflicts identified

**Hint:** Use `npm audit` and `npm list` for validation.

---

## Summary

You now understand:
- ✅ Monorepo structure and organization
- ✅ Path filters for selective triggering
- ✅ Working directory configuration
- ✅ Conditional workflow execution
- ✅ Matrix strategies for services
- ✅ Artifact collection patterns
- ✅ Shared dependency management
- ✅ Change detection automation
- ✅ Cache optimization for monorepos
- ✅ Cross-service dependency validation

Continue mastering monorepo CI/CD!
