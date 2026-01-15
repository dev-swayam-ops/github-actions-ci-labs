# Cheatsheet: Workflow Fundamentals

Quick reference for GitHub Actions workflow advanced features.

---

## Trigger Events (on:)

```yaml
on:
  push:
    branches: [ main, develop ]
    paths:
      - 'src/**'
      - 'package.json'
  
  pull_request:
    branches: [ main ]
  
  schedule:
    - cron: '0 2 * * *'  # Daily 2 AM UTC
  
  workflow_dispatch:
    inputs:
      environment:
        required: true
        type: choice
        options: [dev, prod]
  
  release:
    types: [created, published]
  
  issues:
    types: [opened, labeled]
```

---

## Complete Workflow Template

```yaml
name: Workflow Name

on:
  push:
    branches: [ main ]
  workflow_dispatch:

env:
  REGISTRY: ghcr.io

jobs:
  job-name:
    runs-on: ubuntu-latest
    environment: production
    permissions:
      contents: read
      packages: write
    
    outputs:
      output-key: ${{ steps.step-id.outputs.value }}
    
    strategy:
      matrix:
        node: [16, 18, 20]
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node }}
      
      - id: step-id
        run: |
          echo "value=hello" >> $GITHUB_OUTPUT
```

---

## Job Configuration

| Key | Purpose | Example |
|-----|---------|---------|
| `runs-on` | Runner environment | `ubuntu-latest`, `self-hosted` |
| `needs` | Job dependencies | `needs: [build, test]` |
| `if` | Conditional execution | `if: success()` |
| `timeout-minutes` | Max execution time | `timeout-minutes: 60` |
| `environment` | Deployment environment | `environment: production` |
| `permissions` | GitHub token permissions | `permissions: { contents: read }` |
| `strategy` | Matrix/fail settings | `strategy: { matrix: {...} }` |
| `outputs` | Job-level outputs | `outputs: { key: value }` |
| `concurrency` | Limit concurrent runs | `concurrency: ci-${{ github.ref }}` |

---

## Step Configuration

```yaml
steps:
  - name: Step Name
    run: command
    shell: bash
    working-directory: ./src
    env:
      VAR: value
    if: always()
    continue-on-error: false
    id: step-id
    timeout-minutes: 10
```

| Key | Purpose | Example |
|-----|---------|---------|
| `name` | Step identifier | `"Build application"` |
| `run` | Shell command | `npm test` |
| `uses` | Action reference | `actions/checkout@v3` |
| `with` | Action inputs | `node-version: '18'` |
| `env` | Step variables | `KEY: value` |
| `shell` | Shell type | `bash`, `pwsh`, `sh` |
| `id` | Reference identifier | `id: build` |
| `if` | Conditional | `if: success()` |

---

## Matrix Strategy

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [16, 18, 20]
    include:
      - os: macos-latest
        node: 18
    exclude:
      - os: windows-latest
        node: 16
  
  fail-fast: false  # Continue all if one fails
  max-parallel: 4   # Limit parallel jobs
```

**Creating Job Combinations:**

```yaml
strategy:
  matrix:
    os: [ubuntu, windows]        # 2 values
    node: [16, 18, 20]           # 3 values
# Creates 2 × 3 = 6 job combinations
```

**Access Matrix Values:**

```yaml
runs-on: ${{ matrix.os }}
with:
  node-version: ${{ matrix.node }}
```

---

## Conditions (if:)

```yaml
# Event-based
if: github.event_name == 'push'
if: github.event_name == 'pull_request'
if: github.event_name == 'schedule'

# Branch-based
if: github.ref == 'refs/heads/main'
if: startswith(github.ref, 'refs/tags/')
if: contains(github.ref, 'release')

# Status-based
if: success()
if: failure()
if: always()
if: cancelled()

# Context-based
if: github.actor == 'octocat'
if: github.repository == 'owner/repo'

# Combined
if: github.event_name == 'push' && github.ref == 'refs/heads/main'
if: success() || failure()  # Run regardless
```

---

## Context Variables

### GitHub Context

| Variable | Example | Use |
|----------|---------|-----|
| `github.repository` | `owner/repo` | Logging, Docker tags |
| `github.repository_owner` | `owner` | Permissions, notifications |
| `github.ref` | `refs/heads/main` | Branch/tag detection |
| `github.ref_name` | `main` | Branch name only |
| `github.sha` | `abc123def456...` | Commit identification |
| `github.event_name` | `push`, `pull_request` | Trigger type |
| `github.actor` | `octocat` | User attribution |
| `github.workspace` | `/home/runner/work/repo` | Working directory |
| `github.run_id` | `123456789` | Workflow run ID |
| `github.run_number` | `42` | Sequential run number |

### Runner Context

| Variable | Example | Use |
|----------|---------|-----|
| `runner.os` | `Linux`, `Windows`, `macOS` | OS-specific commands |
| `runner.arch` | `X86`, `ARM64` | Architecture detection |
| `runner.name` | `Hosted Agent` | Runner identification |
| `runner.tool_cache` | `/opt/hostedtoolcache` | Tool paths |

### Job Status

| Function | Returns | Use |
|----------|---------|-----|
| `success()` | true | Previous succeeded |
| `failure()` | true | Previous failed |
| `always()` | true | Always runs |
| `cancelled()` | true | Workflow cancelled |

---

## Job Dependencies and Outputs

### Using needs:

```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.meta.outputs.version }}
    steps:
      - id: meta
        run: echo "version=1.0.0" >> $GITHUB_OUTPUT

  job2:
    needs: job1
    steps:
      - run: echo ${{ needs.job1.outputs.version }}
```

### Multiple Dependencies:

```yaml
needs: [job1, job2, job3]  # All must complete
```

---

## Workflow Inputs and Secrets

### Inputs (workflow_dispatch):

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - dev
          - staging
          - prod
      version:
        description: 'Release version'
        required: false
        default: '1.0.0'
```

### Access Inputs:

```yaml
echo ${{ github.event.inputs.environment }}
echo ${{ github.event.inputs.version }}
```

### Secrets:

```yaml
with:
  api-key: ${{ secrets.API_KEY }}

env:
  DATABASE_PASSWORD: ${{ secrets.DB_PASSWORD }}
```

---

## Artifacts Management

### Upload:

```yaml
- uses: actions/upload-artifact@v3
  with:
    name: my-artifact
    path: dist/
    retention-days: 5
```

### Download:

```yaml
- uses: actions/download-artifact@v3
  with:
    name: my-artifact
    path: ./downloaded/
```

### Multiple Artifacts:

```yaml
# Upload
- run: |
    mkdir artifacts
    echo "build" > artifacts/build.txt
    echo "test" > artifacts/test.txt

- uses: actions/upload-artifact@v3
  with:
    name: results
    path: artifacts/

# Download all
- uses: actions/download-artifact@v3
  with:
    name: results
```

---

## Caching Dependencies

```yaml
- uses: actions/setup-node@v3
  with:
    node-version: '18'
    cache: 'npm'  # Automatic npm caching

# Manual caching
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
```

**Cache Strategies:**

```yaml
# Key with hash (miss if dependencies change)
key: ${{ runner.os }}-${{ hashFiles('package-lock.json') }}

# Fallback keys for partial match
restore-keys: |
  ${{ runner.os }}-node-
  ${{ runner.os }}-
```

---

## Environments and Deployments

```yaml
jobs:
  deploy:
    environment:
      name: production
      url: https://myapp.com
      deployment_branch_policy:
        protected_branches: true
    environment_variables:
      NODE_ENV: production
```

**Environment Protection:**
- Require approval before deployment
- Restrict to specific branches
- Set environment-specific secrets

---

## Common GitHub Actions

| Action | Purpose | Version |
|--------|---------|---------|
| `actions/checkout` | Clone repo | v3 |
| `actions/setup-node` | Node.js | v3 |
| `actions/setup-python` | Python | v4 |
| `actions/setup-go` | Go | v4 |
| `actions/cache` | Cache files | v3 |
| `actions/upload-artifact` | Store files | v3 |
| `actions/download-artifact` | Retrieve files | v3 |
| `docker/build-push-action` | Docker build | v4 |
| `codecov/codecov-action` | Code coverage | v3 |
| `actions/create-release` | Create release | v1 |

---

## Running Multiple Commands

```yaml
- run: |
    echo "Line 1"
    echo "Line 2"
    npm install
    npm test

# OR using semicolons (not recommended)
- run: npm install; npm test
```

---

## Expressions and Logic

```yaml
# If-then-else (not native, use conditional steps)
steps:
  - if: github.ref == 'refs/heads/main'
    run: npm run deploy

# String operations
${{ contains(github.ref, 'release') }}
${{ startswith(github.ref, 'refs/tags/') }}
${{ endsWith(github.ref, 'main') }}

# Array operations
${{ fromJson('["a","b","c"]') }}
${{ toJson(matrix.os) }}
```

---

## Debugging and Logging

```yaml
# Debug mode
env:
  RUNNER_DEBUG: '1'

# Custom logging
- run: |
    echo "::notice file=path/to/file.js,line=10,col=15::This is a notice"
    echo "::warning file=path/to/file.js::This is a warning"
    echo "::error file=path/to/file.js::This is an error"

# Masking secrets in logs
- run: echo "::add-mask::${{ secrets.SECRET_VALUE }}"
```

---

## Workflow File Naming

- Location: `.github/workflows/`
- Naming: Use lowercase with hyphens: `build-test.yml`
- Extension: `.yml` or `.yaml`
- Example: `.github/workflows/ci-pipeline.yml`

---

## Performance Optimization

| Technique | Impact | Method |
|-----------|--------|--------|
| Caching | 50-70% faster | Use `cache:` in setup |
| Parallel jobs | Scales linearly | Use matrix strategy |
| Skip unchanged | Save time | Use path filters |
| Concurrent limit | Cost control | Use `concurrency:` |
| Container cache | 30-40% faster | Use `cache-from: type=gha` |

---

## Common Patterns

### Lint, Build, Test Pattern

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm run lint

  build:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm run build
      - uses: actions/upload-artifact@v3
        with:
          name: build
          path: dist/

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: build
      - run: npm test
```

### Conditional Deploy Pattern

```yaml
deploy:
  needs: test
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest
  steps:
    - run: echo "Deploying to production"
```

---

## Useful Links

| Resource | URL |
|----------|-----|
| Syntax Reference | https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions |
| Trigger Events | https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows |
| Expressions | https://docs.github.com/en/actions/learn-github-actions/expressions |
| Context | https://docs.github.com/en/actions/learn-github-actions/contexts |
| Marketplace | https://github.com/marketplace?type=actions |

---

**Cheatsheet Version:** 1.0 | **Updated:** January 2026 | **Module:** 01-workflow-fundamentals
