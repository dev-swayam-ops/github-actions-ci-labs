# Module 05 Solutions: Reusable Workflows and Composite Actions

## Exercise 1: Basic Reusable Build Workflow

**Solution:**

```yaml
name: Build (Reusable)

on:
  workflow_call:
    inputs:
      node-version:
        type: string
        required: false
        default: '18'
        description: 'Node.js version'
      build-command:
        type: string
        required: false
        default: 'npm run build'
        description: 'Build command'
    outputs:
      artifact-name:
        description: 'Name of build artifact'
        value: ${{ jobs.build.outputs.artifact-name }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.build.outputs.artifact-name }}
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'
      - run: npm ci
      - id: build
        run: |
          ${{ inputs.build-command }}
          echo "artifact-name=build-${{ github.run_id }}" >> $GITHUB_OUTPUT
      - uses: actions/upload-artifact@v3
        with:
          name: ${{ steps.build.outputs.artifact-name }}
          path: dist/
          retention-days: 7
```

**Explanation:**
- `on: workflow_call:` enables this as reusable
- Inputs specify parameters (strings with defaults)
- `outputs:` returns artifact name from job output
- `${{ inputs.node-version }}` references input
- Step output captured with `id: build`
- Artifact name parameterized via output

**Usage:**
```yaml
- uses: ./.github/workflows/build.yml
  with:
    node-version: '20'
    build-command: 'npm run build:prod'
```

---

## Exercise 2: Reusable Workflow with Secrets

**Solution:**

```yaml
name: Deploy (Reusable)

on:
  workflow_call:
    secrets:
      deploy-token:
        description: 'Deployment token'
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v3
      - name: Deploy Application
        env:
          DEPLOY_TOKEN: ${{ secrets.deploy-token }}
        run: |
          echo "Deploying with provided token"
          # Deploy command would use $DEPLOY_TOKEN
```

**Explanation:**
- `secrets:` section in workflow_call defines expected secrets
- `required: true` makes secret mandatory
- `env: DEPLOY_TOKEN: ${{ secrets.deploy-token }}` passes secret to step
- Secret values never logged to output
- Secrets inherited or explicitly passed by caller

**Usage:**
```yaml
- uses: ./.github/workflows/deploy.yml
  secrets:
    deploy-token: ${{ secrets.CI_DEPLOY_TOKEN }}
```

---

## Exercise 3: Calling Reusable Workflow

**Solution:**

```yaml
name: Main Pipeline

on: [push, pull_request]

jobs:
  build:
    uses: ./.github/workflows/build.yml
    with:
      node-version: '18'
      build-command: 'npm run build'

  test:
    needs: build
    uses: ./.github/workflows/test.yml
    with:
      node-version: '18'
      test-command: 'npm test'
    secrets:
      test-token: ${{ secrets.CI_TEST_TOKEN }}
```

**Explanation:**
- `uses: ./.github/workflows/file.yml` calls reusable workflow
- `with:` passes input values
- `secrets:` passes secrets (required for reusable to access)
- `needs: build` creates job dependency
- Reusable workflows execute like regular jobs
- Path must start with `./` for local workflows

**Syntax Breakdown:**
- `./.github/workflows/reusable.yml` - local reusable workflow
- `owner/repo/.github/workflows/reusable.yml@ref` - remote reusable
- `owner/repo/path/to/action@v1` - public action

---

## Exercise 4: Workflow Outputs and Dependencies

**Solution:**

```yaml
on:
  workflow_call:
    outputs:
      artifact-name:
        description: 'Name of uploaded artifact'
        value: ${{ jobs.build.outputs.artifact-name }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.build.outputs.artifact-name }}
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - id: build
        run: |
          npm ci && npm run build
          echo "artifact-name=build-${{ github.run_id }}" >> $GITHUB_OUTPUT
      - uses: actions/upload-artifact@v3
        with:
          name: ${{ steps.build.outputs.artifact-name }}
          path: dist/
```

**Usage in Caller:**

```yaml
jobs:
  build:
    uses: ./.github/workflows/build.yml
    
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Get Artifact Name
        run: echo "Artifact: ${{ needs.build.outputs.artifact-name }}"
```

**Explanation:**
- Workflow-level outputs reference job outputs: `value: ${{ jobs.build.outputs.artifact-name }}`
- Job outputs reference step outputs: `outputs: artifact-name: ${{ steps.build.outputs.artifact-name }}`
- Caller accesses via `needs.job-name.outputs.output-name`
- Creates data pipeline between workflows

---

## Exercise 5: Conditional Reusable Workflows

**Solution:**

```yaml
name: Smart Pipeline

on: [push, pull_request]

jobs:
  build:
    uses: ./.github/workflows/build.yml
    with:
      node-version: '18'

  test:
    needs: build
    uses: ./.github/workflows/test.yml
    with:
      node-version: '18'

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    uses: ./.github/workflows/deploy.yml
    secrets:
      deploy-token: ${{ secrets.DEPLOY_TOKEN }}
```

**Explanation:**
- `if:` condition on job level controls execution
- `github.ref == 'refs/heads/main'` checks branch
- `github.event_name == 'push'` checks trigger type
- Deploy only runs on push to main
- Pull requests skip deploy step
- Conditions evaluated before job execution

**Common Conditions:**
```yaml
# On main branch
if: github.ref == 'refs/heads/main'

# On any release
if: startsWith(github.ref, 'refs/tags/')

# On PR
if: github.event_name == 'pull_request'

# On success
if: success()

# Multiple conditions
if: github.event_name == 'push' && contains(github.ref, 'main')
```

---

## Exercise 6: Composite Action with Multiple Steps

**Solution:**

Create `.github/actions/test-action/action.yml`:

```yaml
name: 'Test Composite Action'
description: 'Runs comprehensive tests'

inputs:
  test-command:
    description: 'Test command to execute'
    required: false
    default: 'npm test'

outputs:
  test-result:
    description: 'Test result summary'
    value: ${{ steps.test.outputs.result }}

runs:
  using: 'composite'
  steps:
    - name: Install Dependencies
      run: npm ci
      shell: bash

    - id: test
      name: Run Tests
      run: |
        ${{ inputs.test-command }}
        echo "result=✓ Tests passed" >> $GITHUB_OUTPUT
      shell: bash

    - name: Test Summary
      run: echo "Test step completed successfully"
      shell: bash
```

**Explanation:**
- `runs: { using: 'composite' }` enables composite mode
- Each step must specify explicit `shell:` (bash, pwsh, etc.)
- Step IDs allow output capture: `id: test`
- Outputs reference step outputs: `value: ${{ steps.test.outputs.result }}`
- Supports input parameters like regular actions
- Runs in context of calling workflow

---

## Exercise 7: Using Composite Action Locally

**Solution:**

File structure:
```
.github/
├── actions/
│   └── test-action/
│       └── action.yml
└── workflows/
    └── main.yml
```

`main.yml`:
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: ./.github/actions/test-action
        id: test-action
        with:
          test-command: 'npm test'
      - name: Use Action Output
        run: echo "Result: ${{ steps.test-action.outputs.test-result }}"
```

**Explanation:**
- `./.github/actions/test-action` references local composite action
- `id: test-action` allows referencing outputs
- `with:` passes inputs to composite action
- `steps.test-action.outputs.test-result` accesses action output
- Local actions execute immediately in workflow context
- No artifact download required

---

## Exercise 8: Marketplace Action Integration

**Solution:**

```yaml
name: Build with Marketplace Actions

on:
  workflow_call:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - uses: simonecorsi/moxxi-action@v1
        with:
          args: 'run lint'
      
      - uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      
      - uses: coverallsapp/github-action@v2
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
      
      - uses: actions/upload-artifact@v3
        with:
          name: build-output
          path: dist/
```

**Common Marketplace Actions:**
- `actions/checkout@v3` - Clone repository
- `actions/setup-node@v3` - Setup Node.js
- `actions/setup-python@v4` - Setup Python
- `actions/upload-artifact@v3` - Upload artifacts
- `codecov/codecov-action@v3` - Code coverage
- `aquasecurity/trivy-action@v0` - Security scanning

**Explanation:**
- Version pinning (@v3, @v1) ensures reproducibility
- Marketplace actions have documented inputs
- Actions run sequentially in order defined
- Each action receives proper inputs
- Composed into single workflow

---

## Exercise 9: Reusable Workflow Matrix

**Solution:**

```yaml
name: Matrix Build

on: [push]

jobs:
  build:
    strategy:
      matrix:
        node: [16, 18, 20]
        os: [ubuntu-latest, windows-latest]
    uses: ./.github/workflows/build.yml
    with:
      node-version: ${{ matrix.node }}
      build-command: 'npm run build'
```

**Explanation:**
- Matrix creates cartesian product of dimensions
- `node: [16, 18, 20]` × `os: [ubuntu-latest, windows-latest]` = 6 jobs
- Each job gets unique matrix values
- `${{ matrix.node }}` and `${{ matrix.os }}` available in with:
- Reusable workflow receives appropriate values
- All jobs run in parallel

**Advanced Matrix:**
```yaml
strategy:
  matrix:
    node: [16, 18, 20]
    test-suite: [unit, integration, e2e]
  exclude:
    - node: 16
      test-suite: e2e
  include:
    - node: 18
      test-suite: performance
# Creates 8 jobs (9 - 1 exclude + 1 include)
```

---

## Exercise 10: Reusable Workflow Library

**Solution:**

Create three reusable workflows:

**`.github/workflows/reusable-build.yml`:**
```yaml
name: Build Reusable
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: '18'
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.build.outputs.artifact-name }}
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'
      - run: npm ci && npm run build
      - id: build
        run: echo "artifact-name=build-${{ github.run_id }}" >> $GITHUB_OUTPUT
      - uses: actions/upload-artifact@v3
        with:
          name: ${{ steps.build.outputs.artifact-name }}
          path: dist/
```

**`.github/workflows/reusable-test.yml`:**
```yaml
name: Test Reusable
on:
  workflow_call:
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci && npm test
```

**`.github/workflows/reusable-deploy.yml`:**
```yaml
name: Deploy Reusable
on:
  workflow_call:
    secrets:
      deploy-token:
        required: true
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v3
      - name: Deploy
        env:
          DEPLOY_TOKEN: ${{ secrets.deploy-token }}
        run: echo "Deploying application"
```

**`.github/workflows/main.yml` orchestrates all three:**
```yaml
name: Complete Pipeline
on: [push]

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      node-version: '18'

  test:
    needs: build
    uses: ./.github/workflows/reusable-test.yml

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    uses: ./.github/workflows/reusable-deploy.yml
    secrets:
      deploy-token: ${{ secrets.DEPLOY_TOKEN }}
```

**Explanation:**
- Library consists of 3 focused, reusable workflows
- Each has single responsibility
- Main workflow orchestrates as pipeline
- Dependencies control execution order
- Demonstrates complete workflow inheritance pattern
- Easily extendable with more reusables

---

## Summary

**Mastery Checkpoint:**

You now understand:
- ✅ Reusable workflow syntax and trigger
- ✅ Input/output definitions and usage
- ✅ Secret passing to reusable workflows
- ✅ Workflow dependencies and ordering
- ✅ Composite action creation and usage
- ✅ Marketplace action integration
- ✅ Matrix strategy with reusables
- ✅ Workflow orchestration patterns

**Advanced Topics:**
- Composite actions with Node.js
- Publishing actions to Marketplace
- Organization-wide action libraries
- Workflow templates
- Dynamic workflow generation

Continue practicing and exploring advanced patterns!
