# 01-workflow-fundamentals

## Overview

This module deepens your understanding of GitHub Actions workflows. You'll explore different trigger types, advanced job configurations, and how to orchestrate complex automation pipelines.

---

## What You'll Learn

- Different trigger events and when to use them
- Advanced workflow syntax and configurations
- Job dependencies and concurrency control
- Using actions from the GitHub Marketplace
- Working with artifacts and caching
- Understanding workflow permissions and security
- Debugging and monitoring workflows
- Matrix builds and environment management

---

## Prerequisites

- Completion of **00-setup-and-basics** module
- GitHub repository with write access
- VS Code or similar text editor
- Comfortable with YAML syntax
- Basic understanding of CI/CD concepts

---

## Key Concepts

### 1. **Workflow Triggers**
Events that initiate workflow execution. Can be webhook events, scheduled, or manual.

### 2. **Workflow Dispatch**
Manual trigger that allows you to start workflows from the GitHub UI or API.

### 3. **Actions**
Reusable code units that perform specific tasks. Can be created by GitHub or the community.

### 4. **Artifacts**
Files created during workflow execution that can be downloaded or used in subsequent jobs.

### 5. **Caching**
Storing dependencies to speed up workflow execution.

### 6. **Job Dependencies**
Using `needs` keyword to create sequential job execution.

### 7. **Matrix Strategy**
Running jobs across multiple configurations (Node versions, OS, etc.) simultaneously.

### 8. **Context and Secrets**
Accessing workflow information and secure variables.

---

## Hands-on Lab: Build a Multi-Stage Workflow

### Step 1: Create the Workflow File

Create `.github/workflows/comprehensive-workflow.yml`:

```yaml
name: Comprehensive CI Workflow

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'development'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # Job 1: Validate and Build
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Set Build Variables
        id: meta
        run: |
          echo "tags=v${{ github.run_number }}" >> $GITHUB_OUTPUT
          echo "Build tag: v${{ github.run_number }}"
      
      - name: Run Build
        run: |
          echo "Building application..."
          mkdir -p build
          echo "Build successful" > build/status.txt
      
      - name: Upload Build Artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-artifacts
          path: build/
          retention-days: 5

  # Job 2: Test (depends on build)
  test:
    needs: build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Download Artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-artifacts
          path: build/
      
      - name: Run Tests
        run: |
          echo "Running tests on Node ${{ matrix.node-version }}..."
          echo "✓ Unit tests passed"
          echo "✓ Integration tests passed"
      
      - name: Upload Test Results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results-node-${{ matrix.node-version }}
          path: build/

  # Job 3: Deploy (depends on test)
  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Download Artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-artifacts
      
      - name: Verify Build Status
        run: cat status.txt
      
      - name: Deploy to Production
        run: |
          echo "Deploying to production..."
          echo "✓ Deployment successful"

  # Job 4: Notify (runs regardless of previous jobs)
  notify:
    runs-on: ubuntu-latest
    always:
    if: always()
    steps:
      - name: Workflow Status
        run: |
          echo "Workflow completed!"
          echo "Commit: ${{ github.sha }}"
          echo "Actor: ${{ github.actor }}"
          echo "Branch: ${{ github.ref }}"
```

### Step 2: Push and Monitor

```bash
git add .github/workflows/comprehensive-workflow.yml
git commit -m "Add comprehensive CI workflow"
git push origin main
```

### Step 3: Manual Trigger

1. Go to **Actions** tab
2. Click **Comprehensive CI Workflow**
3. Click **Run workflow** dropdown
4. Select environment (optional)
5. Click **Run workflow**

### Expected Output:

Workflow completes with:
- ✅ Build job succeeds
- ✅ Test job runs for 3 Node versions
- ✅ Deploy job runs (only on main branch)
- ✅ Notify job always runs
- Artifacts available for download

---

## Validation Checklist

- [ ] Workflow file is syntactically correct
- [ ] All triggers work (push, PR, manual dispatch)
- [ ] Job dependencies execute in correct order
- [ ] Matrix strategy creates multiple jobs
- [ ] Artifacts upload and download successfully
- [ ] Conditional jobs execute correctly
- [ ] Build completes in under 2 minutes
- [ ] You can download artifacts from Actions tab

---

## Cleanup

```bash
# Remove workflow file
rm .github/workflows/comprehensive-workflow.yml

# Commit changes
git add .github/workflows/
git commit -m "Remove comprehensive workflow"
git push origin main
```

---

## Common Mistakes

### ❌ Mistake 1: Forgetting Job Dependency

```yaml
# Wrong - jobs run in parallel unpredictably
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building"
  
  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Testing"  # May run before build!
```

```yaml
# Correct - test waits for build
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building"
  
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Testing"  # Waits for build
```

### ❌ Mistake 2: Missing Artifact Upload

Tests might use build artifacts but forget to download them:

```yaml
# Wrong - artifact not downloaded
test:
  needs: build
  steps:
    - run: cat build/output.txt  # File doesn't exist!
```

```yaml
# Correct - download before use
test:
  needs: build
  steps:
    - uses: actions/download-artifact@v3
      with:
        name: artifacts
    - run: cat build/output.txt  # Now file exists
```

### ❌ Mistake 3: Using Old Action Versions

```yaml
# Wrong - outdated action
- uses: actions/checkout@v2

# Correct - latest stable version
- uses: actions/checkout@v3
```

### ❌ Mistake 4: Incorrect Matrix Syntax

```yaml
# Wrong
matrix:
  - node: [16, 18, 20]

# Correct
matrix:
  node: [16, 18, 20]
```

### ❌ Mistake 5: Secrets Not Passed to Steps

```yaml
# Wrong - environment variable not passed
env:
  MY_SECRET: ${{ secrets.API_KEY }}
steps:
  - run: echo $MISSING_VAR

# Correct - use exported variable
env:
  MY_SECRET: ${{ secrets.API_KEY }}
steps:
  - run: echo $MY_SECRET
```

---

## Troubleshooting

### Issue: Job doesn't wait for dependency

**Solution:**
- Ensure `needs:` syntax is correct
- Check job names are spelled exactly
- Remember `needs:` is required for sequential execution

### Issue: Artifacts not found in download step

**Solution:**
- Verify upload step uses `actions/upload-artifact`
- Check artifact name matches between upload and download
- Ensure artifact exists before download step

### Issue: Matrix jobs not creating multiple runs

**Solution:**
- Check matrix syntax: `matrix: { key: [value1, value2] }`
- Verify matrix is under `strategy:` section
- Ensure proper indentation

### Issue: Workflow won't trigger

**Solution:**
- Check branch name matches `branches:` filter
- Verify event type is in `on:` section
- For scheduled triggers, check cron syntax at [crontab.guru](https://crontab.guru)

### Issue: Action fails with permission denied

**Solution:**
- Check runner permissions
- Ensure files have correct paths
- Verify working directory is correct

---

## Next Steps

✅ **You've completed module 01!**

**What's next?**

1. Proceed to **02-build-and-test-pipelines** to learn:
   - Language-specific build tools
   - Running actual tests
   - Code coverage and reporting
   - Performance optimization
   
2. Advanced topics to explore:
   - Custom actions
   - Reusable workflows
   - Container workflows
   - Docker integration

3. Practice exercises:
   - Build workflows for your language
   - Integrate with package managers
   - Add code quality tools

---

## Trigger Events Reference

| Event | Syntax | When | Use Case |
|-------|--------|------|----------|
| Push | `on: push:` | Commit pushed | Build on every commit |
| Pull Request | `on: pull_request:` | PR created/updated | Validate changes |
| Schedule | `on: schedule:` | Cron time | Nightly builds |
| Manual | `on: workflow_dispatch:` | Manual trigger | On-demand tasks |
| Release | `on: release:` | GitHub release | Build on release |

---

## Actions Library

Popular reusable actions:

| Action | Purpose | Example |
|--------|---------|---------|
| `actions/checkout` | Clone repository | `uses: actions/checkout@v3` |
| `actions/setup-node` | Setup Node.js | `uses: actions/setup-node@v3` |
| `actions/upload-artifact` | Store files | `uses: actions/upload-artifact@v3` |
| `actions/download-artifact` | Retrieve files | `uses: actions/download-artifact@v3` |
| `actions/cache` | Cache dependencies | `uses: actions/cache@v3` |

---

## Quick Reference

### Matrix Strategy

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [16.x, 18.x, 20.x]
    # Creates 6 job combinations (2 OS × 3 Node versions)
```

### Job Outputs

```yaml
jobs:
  job1:
    outputs:
      output-key: ${{ steps.step-id.outputs.value }}
    steps:
      - id: step-id
        run: echo "value=hello" >> $GITHUB_OUTPUT

  job2:
    needs: job1
    steps:
      - run: echo ${{ needs.job1.outputs.output-key }}
```

### Conditional Execution

```yaml
if: github.ref == 'refs/heads/main'
if: github.event_name == 'push'
if: success()  # Previous step succeeded
if: failure()  # Previous step failed
```

---

## Additional Resources

- [Trigger Events Documentation](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)
- [Actions Marketplace](https://github.com/marketplace?type=actions)
- [Workflow Syntax Reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Creating Custom Actions](https://docs.github.com/en/actions/creating-actions)

---

**Created:** January 2026 | **Level:** Intermediate | **Estimated Time:** 45 minutes
