# 05-reusable-workflows-and-composite-actions

## Overview

This module teaches you how to build reusable workflows and composite actions to eliminate duplication and create a scalable CI/CD platform. You'll master workflow reuse, composite actions, and creating organization-wide standards.

---

## What You'll Learn

- Creating reusable workflows for different scenarios
- Calling reusable workflows from other workflows
- Understanding workflow inputs, outputs, and secrets
- Building composite actions for multi-step tasks
- Sharing actions across repositories
- Versioning and maintaining reusable components
- Best practices for action design
- Using marketplace actions effectively
- Testing reusable components
- Building action repositories

---

## Prerequisites

- Completion of **04-caching-and-artifacts** module
- Understanding of workflow syntax and jobs
- Knowledge of YAML structure
- Comfortable with GitHub Actions basics
- GitHub repository with workflow permissions

---

## Key Concepts

### 1. **Reusable Workflow**
A workflow file that can be called by other workflows, reducing duplication across repositories.

### 2. **Workflow Inputs**
Parameters passed to a reusable workflow (strings, booleans, choices, environments).

### 3. **Workflow Outputs**
Values returned from a reusable workflow to the calling workflow.

### 4. **Workflow Secrets**
Sensitive data passed to reusable workflows (passwords, tokens, API keys).

### 5. **Composite Action**
A custom action combining shell/Node.js steps into a single reusable unit.

### 6. **Action Metadata**
The `action.yml` file defining inputs, outputs, and step descriptions.

### 7. **Action Repository**
A public repository hosting an action for reuse across projects.

### 8. **Workflow Inheritance**
Pattern where workflows call other workflows to build complex pipelines.

### 9. **Job Outputs**
Data passed from one job to another using `jobs.<job-id>.outputs`.

### 10. **Marketplace Actions**
Community-contributed actions available via GitHub Marketplace.

---

## Hands-on Lab: Reusable Build and Test Pipeline

### Step 1: Create Reusable Build Workflow

Create `.github/workflows/build.yml`:

```yaml
name: Build Workflow (Reusable)

on:
  workflow_call:
    inputs:
      node-version:
        description: 'Node.js version'
        required: false
        default: '18'
        type: string
      build-command:
        description: 'Build command to run'
        required: false
        default: 'npm run build'
        type: string
    outputs:
      build-artifact-name:
        description: 'Name of uploaded build artifact'
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
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Run Build
        id: build
        run: |
          ${{ inputs.build-command }}
          echo "artifact-name=build-${{ github.run_id }}" >> $GITHUB_OUTPUT
      
      - name: Upload Build
        uses: actions/upload-artifact@v3
        with:
          name: build-${{ github.run_id }}
          path: dist/
          retention-days: 7
```

### Step 2: Create Reusable Test Workflow

Create `.github/workflows/test.yml`:

```yaml
name: Test Workflow (Reusable)

on:
  workflow_call:
    inputs:
      node-version:
        description: 'Node.js version'
        required: false
        default: '18'
        type: string
      test-command:
        description: 'Test command to run'
        required: false
        default: 'npm test'
        type: string
    secrets:
      test-token:
        description: 'Optional test token'
        required: false

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Run Tests
        env:
          TEST_TOKEN: ${{ secrets.test-token }}
        run: ${{ inputs.test-command }}
```

### Step 3: Create Calling Workflow

Create `.github/workflows/main-pipeline.yml`:

```yaml
name: Main Pipeline (Calls Reusable)

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

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

  verify:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Pipeline Summary
        run: |
          echo "✓ Build completed"
          echo "✓ Tests passed"
          echo "✓ Pipeline successful"
```

### Step 4: Commit and Push

```bash
git add .github/workflows/
git commit -m "Add reusable workflows for build and test"
git push origin main
```

### Step 5: Monitor Execution

**Expected Output:**

```
Workflow: Main Pipeline

├── build (uses ./.github/workflows/build.yml)
│   ├── Checkout ✓
│   ├── Setup Node ✓
│   ├── Install Dependencies ✓
│   ├── Run Build ✓
│   └── Upload Artifact ✓
│
├── test (depends on build, uses ./.github/workflows/test.yml)
│   ├── Checkout ✓
│   ├── Setup Node ✓
│   ├── Install Dependencies ✓
│   └── Run Tests ✓
│
└── verify (depends on test)
    └── Pipeline Summary ✓

All jobs completed successfully
```

---

## Validation Checklist

- [ ] Reusable build workflow accepts inputs
- [ ] Reusable test workflow accepts inputs and secrets
- [ ] Main pipeline successfully calls both workflows
- [ ] Build artifact is uploaded
- [ ] Outputs are passed between workflows
- [ ] Different inputs produce different outputs
- [ ] Secrets are handled securely
- [ ] All jobs complete in expected order

---

## Cleanup

```bash
rm .github/workflows/build.yml
rm .github/workflows/test.yml
rm .github/workflows/main-pipeline.yml
git add .github/workflows/
git commit -m "Remove reusable workflows"
git push origin main
```

---

## Common Mistakes

### ❌ Mistake 1: Incorrect Reusable Workflow Trigger

```yaml
# Wrong - uses push trigger
on:
  push:
    branches: [ main ]

# Correct - uses workflow_call trigger
on:
  workflow_call:
    inputs:
      node-version:
        type: string
```

### ❌ Mistake 2: Missing Input Type Declaration

```yaml
# Wrong - no type specified
inputs:
  node-version:
    description: 'Version'

# Correct - type required
inputs:
  node-version:
    type: string
    description: 'Version'
```

### ❌ Mistake 3: Incorrect Output Reference

```yaml
# Wrong - outputs use jobs syntax
outputs:
  artifact-name: ${{ jobs.build.outputs.build-name }}

# Correct - check actual job and step names
outputs:
  artifact-name: ${{ jobs.build.outputs.artifact-name }}
```

### ❌ Mistake 4: Not Passing Secrets to Called Workflow

```yaml
# Wrong - secret not passed
jobs:
  build:
    uses: ./.github/workflows/build.yml

# Correct - explicitly pass secrets
jobs:
  build:
    uses: ./.github/workflows/build.yml
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
```

### ❌ Mistake 5: Calling Workflow in Wrong Location

```yaml
# Wrong - workflow not in .github/workflows/
uses: ./workflows/build.yml

# Correct - workflows directory is .github/workflows
uses: ./.github/workflows/build.yml
```

---

## Troubleshooting

### Issue: Reusable workflow not found

**Solution:**
- Verify path: `./.github/workflows/workflow-name.yml`
- Ensure workflow file exists and is committed
- Check `on: workflow_call:` is set correctly
- Verify branch contains the reusable workflow

### Issue: Inputs not being passed correctly

**Solution:**
- Verify input names match exactly (case-sensitive)
- Ensure type is specified in workflow_call
- Check default values if input not required
- Validate input syntax in calling workflow

### Issue: Outputs empty or missing

**Solution:**
- Verify step has `id:` set
- Ensure output name set with `>> $GITHUB_OUTPUT`
- Check job name in output reference
- Verify workflow_call outputs section defined

### Issue: Secrets not available in reusable workflow

**Solution:**
- Explicitly pass secrets: `secrets: inherit` or individual secrets
- Verify secret name matches in both workflows
- Check repository secret is set in Settings
- Use `secrets.secret-name` syntax only in job context

### Issue: Reusable workflow timeout

**Solution:**
- Check workflow execution time
- Optimize caching and dependencies
- Split into smaller reusable workflows
- Set timeout-minutes to custom value

---

## Next Steps

✅ **You've completed module 05!**

**Congratulations! You've mastered reusable workflows and actions.**

**What's next?**

1. Explore advanced reusable components:
   - Composite actions
   - Action versioning
   - Publishing to Marketplace
   - Cross-repository actions

2. Advanced patterns:
   - Workflow orchestration
   - Conditional reusable workflows
   - Dynamic workflow generation
   - Composite actions with Node.js

3. Organization-level setup:
   - Workflow templates
   - Shared action repositories
   - Internal marketplace
   - Policy enforcement

4. Proceed to module 06:
   - **06-secrets-and-environment-management**
   - **07-security-scanning-and-compliance**

---

## Reusable Workflow Reference

### Basic Structure

```yaml
name: My Reusable Workflow

on:
  workflow_call:
    inputs:
      param-name:
        type: string
        required: true
        description: 'Parameter description'
    outputs:
      output-name:
        description: 'Output description'
        value: ${{ jobs.job-id.outputs.output-name }}
    secrets:
      secret-name:
        required: true
        description: 'Secret description'

jobs:
  main-job:
    runs-on: ubuntu-latest
    outputs:
      output-name: ${{ steps.step-id.outputs.output-name }}
    steps:
      - id: step-id
        run: echo "output-name=value" >> $GITHUB_OUTPUT
```

### Calling Reusable Workflow

```yaml
jobs:
  call-workflow:
    uses: ./.github/workflows/reusable.yml
    with:
      param-name: 'value'
    secrets:
      secret-name: ${{ secrets.SECRET }}

  call-with-inherit:
    uses: ./.github/workflows/reusable.yml
    secrets: inherit
```

### Input Types

| Type | Example | Usage |
|------|---------|-------|
| string | `'18'` | `${{ inputs.param }}` |
| boolean | `true` | `if: inputs.param` |
| choice | `'option1'` or `'option2'` | Selection menu |
| environment | `'prod'` or `'staging'` | Environment selector |

---

## Composite Action Reference

### Action Metadata (action.yml)

```yaml
name: 'My Composite Action'
description: 'What this action does'

inputs:
  input-name:
    description: 'Input description'
    required: true
    default: 'default-value'

outputs:
  output-name:
    description: 'Output description'
    value: ${{ steps.step-id.outputs.output-name }}

runs:
  using: 'composite'
  steps:
    - id: step-id
      run: echo "output-name=value" >> $GITHUB_OUTPUT
      shell: bash
```

### Using Composite Action

```yaml
- uses: ./local/path/to/action
  with:
    input-name: 'value'

- uses: owner/repo/path/to/action@v1
  with:
    input-name: 'value'
```

---

## Best Practices

| Practice | Benefit | Example |
|----------|---------|---------|
| Clear input names | Easy to understand | `node-version` not `v` |
| Default values | Less boilerplate in calls | `default: '18'` |
| Explicit types | Type safety | `type: string` |
| Document outputs | Clear return values | `description:` field |
| Version tags | Reproducibility | `uses: owner/repo@v1` |
| Secrets inheritance | Simpler code | `secrets: inherit` |
| Small focused workflows | Better reuse | One concern per workflow |

---

## Performance Optimization

| Strategy | Impact | Implementation |
|----------|--------|-----------------|
| Parallel reusable workflows | Linear speedup | Multiple `needs: []` |
| Conditional calls | Skip unnecessary work | `if: contains(...)` |
| Caching in reusables | Speed per call | `cache: 'npm'` in setup |
| Artifact pass-through | Avoid rebuilding | Download in dependent job |

---

## Additional Resources

- [Reusable Workflows Documentation](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
- [Composite Actions Guide](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action)
- [Workflow Syntax Reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [GitHub Marketplace Actions](https://github.com/marketplace?type=actions)

---

**Created:** January 2026 | **Level:** Intermediate-Advanced | **Estimated Time:** 60 minutes
