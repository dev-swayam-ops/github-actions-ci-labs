# Module 09: Monorepo and Path Filters

## Overview

Managing multiple projects in a single Git repository (monorepo) with GitHub Actions requires intelligent workflow triggering. This module teaches you how to use path filters to run workflows only when relevant code changes, optimizing CI/CD performance and reducing unnecessary builds.

### What You'll Learn

1. **Monorepo Structure** - Organizing multiple projects in one repository
2. **Path Filters** - Triggering workflows on specific file/folder changes
3. **Conditional Workflows** - Using `if` conditions to skip or run jobs
4. **Matrix Builds** - Building multiple projects simultaneously
5. **Artifact Management** - Handling artifacts from multiple projects
6. **Dependency Management** - Managing cross-project dependencies
7. **Change Detection** - Identifying which projects changed
8. **Caching Strategies** - Optimizing monorepo builds

### Prerequisites

- Basic GitHub Actions knowledge (Modules 01-02)
- Understanding of workflow syntax
- Git fundamentals
- Multi-project repository setup

---

## Key Concepts

### What is a Monorepo?

**Definition:** A single Git repository containing multiple independent projects or services.

**Structure Example:**
```
my-monorepo/
├── services/
│   ├── api/
│   │   ├── package.json
│   │   └── src/
│   ├── auth/
│   │   ├── package.json
│   │   └── src/
│   └── ui/
│       ├── package.json
│       └── src/
├── shared/
│   ├── utils/
│   └── types/
└── .github/workflows/
```

**Advantages:**
- Unified version control
- Atomic cross-project commits
- Shared dependencies
- Single CI/CD system

**Challenges:**
- Workflow complexity
- Build optimization
- Dependency tracking

### Path Filters in GitHub Actions

**Purpose:** Run workflows only when specific files or directories change.

**Trigger On Specific Paths:**
```yaml
on:
  push:
    paths:
      - 'services/api/**'        # Any file in api folder
      - 'shared/**'              # Any file in shared folder
      - 'package.json'           # Specific file
```

**Exclude Paths:**
```yaml
on:
  push:
    paths:
      - 'services/api/**'
    paths-ignore:
      - '**.md'                  # Ignore markdown
      - '.gitignore'
```

**How It Works:**
1. GitHub compares changed files to path patterns
2. If ANY changed file matches, workflow triggers
3. Can combine multiple path patterns
4. Paths are relative to repository root

### Monorepo Build Patterns

**Pattern 1: Separate Workflows**
- One workflow file per project
- Triggered only when that project's files change
- Simple, isolated

**Pattern 2: Matrix Workflow**
- Single workflow with matrix strategy
- Builds all projects, skips unchanged ones
- More complex, but unified

**Pattern 3: Dynamic Workflow**
- Detects changes, triggers appropriate workflows
- Uses change detection script
- Most powerful but most complex

### Change Detection

Identifying which projects changed:
```bash
# Get list of changed files
git diff --name-only origin/main..HEAD > changed.txt

# Check if service changed
if grep -q "services/api/" changed.txt; then
  echo "API service changed"
fi
```

### Workspace and Project Setup

**npm Workspaces:**
```json
{
  "workspaces": [
    "services/api",
    "services/auth",
    "services/ui"
  ]
}
```

**Benefits:**
- Shared node_modules
- Single package-lock.json
- Unified dependency management

---

## Hands-on Lab: Monorepo with Path Filters

### Part 1: Create Monorepo Structure

**Step 1: Initialize Repository**

```bash
mkdir my-monorepo && cd my-monorepo
git init
npm init -y
```

**Step 2: Setup Workspace**

`package.json`:
```json
{
  "name": "my-monorepo",
  "version": "1.0.0",
  "private": true,
  "workspaces": [
    "services/api",
    "services/auth",
    "shared/utils"
  ]
}
```

**Step 3: Create Services**

```bash
# API Service
mkdir -p services/api
cd services/api
npm init -y

# package.json for api
cat > package.json << 'EOF'
{
  "name": "@myapp/api",
  "version": "1.0.0",
  "scripts": {
    "test": "echo 'API tests passed'",
    "lint": "echo 'API linting passed'",
    "build": "echo 'API build complete'"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF

# Create source files
mkdir src
echo "console.log('API Server');" > src/index.js

cd ../..
```

```bash
# Auth Service
mkdir -p services/auth
cd services/auth
npm init -y

cat > package.json << 'EOF'
{
  "name": "@myapp/auth",
  "version": "1.0.0",
  "scripts": {
    "test": "echo 'Auth tests passed'",
    "lint": "echo 'Auth linting passed'",
    "build": "echo 'Auth build complete'"
  },
  "dependencies": {
    "jsonwebtoken": "^9.0.0"
  }
}
EOF

mkdir src
echo "console.log('Auth Service');" > src/index.js

cd ../..
```

```bash
# Shared Utils
mkdir -p shared/utils
cd shared/utils
npm init -y

cat > package.json << 'EOF'
{
  "name": "@myapp/utils",
  "version": "1.0.0",
  "scripts": {
    "test": "echo 'Utils tests passed'"
  }
}
EOF

mkdir src
echo "module.exports = { helper: () => {} };" > src/index.js

cd ../..
```

**Step 4: Install Dependencies**

```bash
npm install
# All workspaces installed, shared node_modules
```

### Part 2: Create Path Filter Workflows

**Create workflow directory:**
```bash
mkdir -p .github/workflows
```

`.github/workflows/api-ci.yml` - Triggers only on API changes:

```yaml
name: API CI

on:
  push:
    branches: [main]
    paths:
      - 'services/api/**'
      - 'shared/**'
      - 'package*.json'
  pull_request:
    branches: [main]
    paths:
      - 'services/api/**'
      - 'shared/**'
      - 'package*.json'

jobs:
  test:
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
      - run: npm run lint
      - run: npm run test
      - run: npm run build
```

`.github/workflows/auth-ci.yml` - Triggers only on Auth changes:

```yaml
name: Auth CI

on:
  push:
    branches: [main]
    paths:
      - 'services/auth/**'
      - 'shared/**'
      - 'package*.json'
  pull_request:
    branches: [main]
    paths:
      - 'services/auth/**'
      - 'shared/**'
      - 'package*.json'

jobs:
  test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: services/auth
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci --workspaces
      - run: npm run lint
      - run: npm run test
      - run: npm run build
```

### Part 3: Validate

**Push changes:**
```bash
git add .
git commit -m "Initial monorepo setup"
git push origin main
```

**Expected Results:**
- ✅ Both workflows trigger on initial push
- ✅ API workflow runs when services/api/* changes
- ✅ Auth workflow runs when services/auth/* changes
- ✅ Both run when shared/* or package.json changes
- ✅ Neither runs when only README.md changes

**Success Criteria:**
- [ ] Monorepo structure created
- [ ] npm workspaces configured
- [ ] Path filters working correctly
- [ ] Workflows trigger appropriately
- [ ] Jobs run in correct directory context
- [ ] Dependency caching functional

---

## Common Mistakes

### 1. Missing Workspace Dependencies
```yaml
# ✗ Bad: Only installs one service
- run: npm ci
  working-directory: services/api

# ✓ Good: Installs entire workspace
- run: npm ci --workspaces
```

### 2. Incorrect Path Patterns
```yaml
# ✗ Bad: Pattern too specific
paths:
  - 'services/api/src/**'     # Misses package.json changes

# ✓ Good: Include all relevant paths
paths:
  - 'services/api/**'
  - 'shared/**'
  - 'package*.json'
```

### 3. Wrong Working Directory
```yaml
# ✗ Bad: Tests run from root
- run: npm test

# ✓ Good: Run from service directory
- run: npm test
  working-directory: services/api
```

### 4. Forgetting Shared Dependencies
```yaml
# ✗ Bad: Only triggers on service changes
paths:
  - 'services/api/**'

# ✓ Good: Include shared dependencies
paths:
  - 'services/api/**'
  - 'shared/**'
```

### 5. No Change Detection for Matrix
```yaml
# ✗ Bad: Always runs all matrix jobs
strategy:
  matrix:
    service: [api, auth]

# ✓ Good: Skip if unchanged (see exercises)
- if: contains(env.CHANGED_SERVICES, matrix.service)
  run: npm test
```

---

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| "All workflows trigger" | Paths too broad | Make patterns more specific |
| "Workflow doesn't trigger" | Path pattern doesn't match | Test with `git diff` locally |
| "Dependencies not installed" | Missing `--workspaces` flag | Add to npm install commands |
| "Wrong directory context" | Missing `working-directory` | Set for all steps or job |
| "Cache miss" | Different cache keys per service | Use service-specific keys |
| "Build failure in one service" | Shared dependency broken | Check shared/* folder |
| "Matrix runs all jobs" | No conditional skip | Implement change detection |

---

## Summary Checklist

- ✅ Monorepo structure organized by service
- ✅ npm workspaces or package manager configured
- ✅ Path filters on appropriate file patterns
- ✅ Workflows trigger only when relevant
- ✅ Working directory set correctly for each job
- ✅ Shared dependencies included in path filters
- ✅ Node cache configured per workflow
- ✅ Change detection implemented (if using matrix)
- ✅ Documentation for monorepo structure
- ✅ Dependencies between services documented

---

## Resources

- npm Workspaces: https://docs.npmjs.com/cli/v9/using-npm/workspaces
- GitHub Actions Paths: https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#onpushpull_requestpaths
- Monorepo Best Practices: https://monorepo.tools
- Git Path Filters: https://git-scm.com/docs/git-diff#Documentation/git-diff.txt---name-only
