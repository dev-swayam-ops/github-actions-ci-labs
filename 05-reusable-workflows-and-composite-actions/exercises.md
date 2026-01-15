# Module 05 Exercises: Reusable Workflows and Composite Actions

## Exercise 1: Basic Reusable Build Workflow

**Objective:** Create a reusable workflow that can be called from other workflows

**Acceptance Criteria:**
- Workflow uses `on: workflow_call:`
- Accepts `node-version` input (type: string)
- Accepts `build-command` input (type: string)
- Outputs artifact name
- Calls `actions/setup-node@v3` with cache
- Uploads build artifacts

**Starter Code:**
```yaml
name: Build (Reusable)

on:
  # Add workflow_call trigger

jobs:
  build:
    runs-on: ubuntu-latest
    # Add outputs section
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          # Add node-version input
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v3
        with:
          # Add artifact name
          path: dist/
          retention-days: 7
```

**Expected Output:**
Reusable workflow that can be called with: `uses: ./.github/workflows/build.yml`

---

## Exercise 2: Reusable Workflow with Secrets

**Objective:** Accept and use secrets in a reusable workflow

**Acceptance Criteria:**
- Workflow_call includes secrets parameter
- Secret passed to job environment
- Secret used in deploy step
- Secret access controlled via workflow_call

**Starter Code:**
```yaml
name: Deploy (Reusable)

on:
  workflow_call:
    # Add secrets definition for deploy-token
    
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v3
      - name: Deploy
        # Add secret to environment
        run: echo "Deploying with token"
```

**Expected:** Secret passed via `secrets: { deploy-token: ${{ secrets.DEPLOY_TOKEN }} }`

---

## Exercise 3: Calling Reusable Workflow

**Objective:** Call a reusable workflow from a main workflow

**Acceptance Criteria:**
- Main workflow uses `uses: ./.github/workflows/reusable.yml`
- Passes inputs with `with:`
- Passes secrets with `secrets:`
- Creates job dependency with `needs:`
- Receives outputs from reusable workflow

**Starter Code:**
```yaml
name: Main Pipeline

on: [push]

jobs:
  build:
    # Add workflow_call usage
    # Pass node-version input
    # Pass build-command input
    
  test:
    needs: build
    # Call test workflow
    # Pass node-version input
    # Pass secrets via secrets:
```

**Expected:** Main workflow successfully calls 2 reusable workflows in sequence

---

## Exercise 4: Workflow Outputs and Dependencies

**Objective:** Use workflow outputs from one reusable workflow in another

**Acceptance Criteria:**
- First workflow outputs artifact name
- Second workflow receives artifact name
- Artifact name used in dependent job
- Jobs execute in correct order

**Starter Code:**
```yaml
on:
  workflow_call:
    outputs:
      # Define output for artifact name

jobs:
  build:
    outputs:
      # Add step output with artifact name
    steps:
      - id: build
        run: |
          echo "artifact-name=build-123" >> $GITHUB_OUTPUT
      - uses: actions/upload-artifact@v3
        with:
          # Use step output for artifact name
```

**Expected:** Output value accessible via `jobs.build.outputs.artifact-name`

---

## Exercise 5: Conditional Reusable Workflows

**Objective:** Conditionally call reusable workflows based on triggers

**Acceptance Criteria:**
- Main workflow checks event type
- Different reusables called for push vs pull_request
- Uses `if:` condition on job
- Only relevant workflows execute

**Starter Code:**
```yaml
name: Smart Pipeline

on: [push, pull_request]

jobs:
  build:
    uses: ./.github/workflows/build.yml
    with:
      node-version: '18'

  deploy:
    needs: build
    if: # Add condition for main branch and push event
    uses: ./.github/workflows/deploy.yml
    secrets:
      deploy-token: ${{ secrets.DEPLOY_TOKEN }}
```

**Hint:** Use `github.ref == 'refs/heads/main'` and `github.event_name == 'push'`

---

## Exercise 6: Composite Action with Multiple Steps

**Objective:** Create a composite action that combines multiple steps

**Acceptance Criteria:**
- `action.yml` file defines inputs and outputs
- Composite action uses `runs: { using: 'composite' }`
- Includes at least 3 steps
- Each step has explicit shell
- Outputs defined with step reference

**Starter Code:**
```yaml
# action.yml
name: 'Test Composite Action'
description: 'Runs tests'

inputs:
  test-command:
    # Add input definition

outputs:
  test-result:
    description: 'Test result'
    # Reference step output

runs:
  using: 'composite'
  steps:
    - run: echo "Installing dependencies"
      shell: bash
    - id: test
      # Add test command
      shell: bash
    - run: echo "Test completed"
      shell: bash
```

**Expected:** Composite action with 3 steps and output from middle step

---

## Exercise 7: Using Composite Action Locally

**Objective:** Use a composite action from the same repository

**Acceptance Criteria:**
- Composite action stored in `.github/actions/my-action/action.yml`
- Main workflow uses `uses: ./.github/actions/my-action`
- Passes inputs with `with:`
- Receives outputs from action

**Starter Code:**
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: ./.github/actions/test-action
        with:
          test-command: 'npm test'
      - name: Use Action Output
        # Reference action output
        run: echo "Test result: ${{ steps.action.outputs.result }}"
```

**Expected:** Custom composite action executes and provides output

---

## Exercise 8: Marketplace Action Integration

**Objective:** Use popular marketplace actions in reusable workflow

**Acceptance Criteria:**
- Reusable workflow uses 3+ marketplace actions
- Actions have appropriate versions (@v3, @v1, etc.)
- Actions receive proper inputs
- Workflow demonstrates action composition

**Starter Code:**
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
      - # Add action for linting
      - # Add action for testing
      - # Add action for artifact upload
```

**Expected:** Multiple marketplace actions working together in pipeline

---

## Exercise 9: Reusable Workflow Matrix

**Objective:** Combine reusable workflows with matrix strategy

**Acceptance Criteria:**
- Calling workflow uses matrix in job
- Passes matrix variable to reusable workflow
- Multiple job instances run in parallel
- Each gets separate artifact

**Starter Code:**
```yaml
jobs:
  build:
    strategy:
      matrix:
        node: [16, 18, 20]
    uses: ./.github/workflows/build.yml
    with:
      node-version: ${{ matrix.node }}
      build-command: 'npm run build'
```

**Expected:** 3 parallel builds with different Node versions

---

## Exercise 10: Reusable Workflow Library

**Objective:** Create set of reusable workflows for common tasks

**Acceptance Criteria:**
- At least 3 reusable workflows created
- Each has specific purpose (build, test, deploy)
- All accept inputs and secrets appropriately
- Main workflow orchestrates all three
- Demonstrates workflow inheritance pattern

**Starter Code:**
```yaml
# build.yml - builds application
# test.yml - runs tests
# deploy.yml - deploys to environment
# main.yml - orchestrates all three

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
    if: success()
    uses: ./.github/workflows/deploy.yml
    secrets:
      api-key: ${{ secrets.DEPLOY_API_KEY }}
```

**Challenge:** Create a complete 3-stage pipeline using reusables

---

## Summary

**Key Learning Points:**
- ✅ Creating reusable workflows with inputs/outputs
- ✅ Passing secrets securely
- ✅ Job dependencies and ordering
- ✅ Composite actions for code reuse
- ✅ Marketplace action integration
- ✅ Matrix strategy with reusables
- ✅ Workflow orchestration patterns

**Next Steps:**
- Publish actions to GitHub Marketplace
- Create organization-wide action library
- Implement workflow templates
- Build composite actions in Node.js

**Continue to Exercise Solutions for complete implementations!**
