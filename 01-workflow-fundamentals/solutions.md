# Solutions: Workflow Fundamentals

Complete solutions for all exercises in the Workflow Fundamentals module.

---

## Solution 1: Scheduled Workflows

**File:** `.github/workflows/ex1-scheduled.yml`

```yaml
name: Scheduled Workflow

on:
  schedule:
    - cron: '0 2 * * *'      # Daily at 2:00 AM UTC
    - cron: '0 9 * * 1'      # Every Monday at 9:00 AM UTC

jobs:
  scheduled-job:
    runs-on: ubuntu-latest
    steps:
      - name: Print Current Date
        run: |
          echo "Current Date and Time:"
          date
      
      - name: Print Trigger Info
        run: |
          echo "Event: ${{ github.event_name }}"
          echo "Trigger: Scheduled workflow"
          echo "Workflow run: ${{ github.run_id }}"
```

**Explanation:**
- `schedule:` section contains cron expressions
- Each cron expression creates separate trigger
- Cron format: `minute hour day month weekday`
- Only scheduled events trigger this workflow
- Great for nightly builds, backups, cleanup tasks

**Testing:**
Visit [crontab.guru](https://crontab.guru) to generate or validate cron expressions.

---

## Solution 2: Workflow Dispatch

**File:** `.github/workflows/ex2-manual.yml`

```yaml
name: Manual Trigger Workflow

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to target'
        required: true
        default: 'dev'
        type: choice
        options:
          - dev
          - staging
          - prod
      version:
        description: 'Application version'
        required: false
        default: '1.0.0'
        type: string

jobs:
  manual-job:
    runs-on: ubuntu-latest
    steps:
      - name: Display Inputs
        run: |
          echo "Environment: ${{ github.event.inputs.environment }}"
          echo "Version: ${{ github.event.inputs.version }}"
      
      - name: Conditional Logic Based on Environment
        run: |
          if [ "${{ github.event.inputs.environment }}" = "prod" ]; then
            echo "⚠️  Deploying to PRODUCTION"
          else
            echo "✓ Deploying to ${{ github.event.inputs.environment }}"
          fi
```

**Explanation:**
- `workflow_dispatch` allows manual triggering from Actions UI
- `inputs:` define parameters with validation
- `choice` type shows dropdown in UI
- Access inputs via `github.event.inputs.<name>`
- `required: true` makes input mandatory
- Useful for manual deployments, backups, reports

**User Experience:**
When triggering from Actions:
1. Click workflow name
2. Click "Run workflow"
3. Fill in inputs
4. Click "Run workflow" button
5. Values passed to jobs

---

## Solution 3: Job Outputs

**File:** `.github/workflows/ex3-outputs.yml`

```yaml
name: Job Outputs Workflow

on:
  push:
    branches: [ main ]

jobs:
  job-one:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.number }}
      build-time: ${{ steps.build.outputs.timestamp }}
    steps:
      - name: Generate Version
        id: version
        run: |
          echo "number=1.2.3" >> $GITHUB_OUTPUT
      
      - name: Generate Build Timestamp
        id: build
        run: |
          echo "timestamp=$(date)" >> $GITHUB_OUTPUT

  job-two:
    needs: job-one
    runs-on: ubuntu-latest
    steps:
      - name: Use Job Outputs
        run: |
          echo "Version from job-one: ${{ needs.job-one.outputs.version }}"
          echo "Build time: ${{ needs.job-one.outputs.build-time }}"
      
      - name: Conditional Based on Output
        if: needs.job-one.outputs.version == '1.2.3'
        run: echo "✓ Version check passed"
```

**Explanation:**
- `outputs:` in job exposes values to other jobs
- `${{ steps.step-id.outputs.key }}` accesses step outputs
- Step must have `id:` to reference it
- `echo "key=value" >> $GITHUB_OUTPUT` sets output
- `needs.job-name.outputs.key` accesses output from another job
- Enables passing build info between stages

**Common Use Cases:**
- Pass version number from build to deploy
- Share build artifacts path
- Communicate test results
- Share generated configuration

---

## Solution 4: Matrix Strategy

**File:** `.github/workflows/ex4-matrix.yml`

```yaml
name: Matrix Strategy Workflow

on:
  push:
    branches: [ main ]

jobs:
  matrix-job:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [16, 18, 20]
    name: Node ${{ matrix.node }} on ${{ matrix.os }}
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node }}
      
      - name: Display Configuration
        run: |
          echo "OS: ${{ runner.os }}"
          echo "Node: ${{ matrix.node }}"
          echo "Runner: ${{ runner.name }}"
      
      - name: Run Node Specific Commands
        run: node --version && npm --version

  summary:
    needs: matrix-job
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Summary
        run: echo "✓ Matrix build completed (9 variations: 3 OS × 3 Node versions)"
```

**Explanation:**
- `strategy: matrix:` creates job combinations
- Each unique combination creates separate job
- 3 OS × 3 Node = 9 total jobs
- Access matrix values: `${{ matrix.key }}`
- `runs-on: ${{ matrix.os }}` uses matrix value for runner
- Runs all jobs in parallel (unless limited)
- Great for multi-version testing

**Benefits:**
- Test across multiple versions/OS simultaneously
- Parallel execution reduces total time
- Single config maintains all variations
- Easy to add/remove versions

---

## Solution 5: Conditional Jobs

**File:** `.github/workflows/ex5-conditional.yml`

```yaml
name: Conditional Jobs Workflow

on:
  push:
    branches: [ main, develop ]

jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - name: Setup Phase
        run: echo "✓ Setup completed for ${{ github.ref_name }} branch"

  deploy-dev:
    needs: setup
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    steps:
      - name: Deploy to Development
        run: echo "Deploying to development environment..."

  deploy-prod:
    needs: setup
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to Production
        run: echo "⚠️  Deploying to PRODUCTION environment..."

  notify:
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Notification
        run: |
          echo "Workflow Status: ${{ job.status }}"
          echo "Branch: ${{ github.ref_name }}"
          echo "Commit: ${{ github.sha }}"
```

**Explanation:**
- `if: condition` controls job execution
- `always()` runs regardless of previous status
- `success()` runs only on success
- `failure()` runs only on failure
- `github.ref` is full ref like `refs/heads/main`
- `github.ref_name` is just branch name like `main`
- `needs:` can depend on always-running jobs

**Conditional Expressions:**
- `github.ref == 'refs/heads/main'` - exact branch match
- `startswith(github.ref, 'refs/tags/')` - match tags
- `github.event_name == 'pull_request'` - PR only
- `success()` - previous step succeeded
- `cancelled()` - workflow was cancelled

---

## Solution 6: Artifact Management

**File:** `.github/workflows/ex6-artifacts.yml`

```yaml
name: Artifact Management Workflow

on:
  push:
    branches: [ main ]

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - name: Create Test Data
        run: |
          mkdir -p artifacts
          echo "Test data generated at $(date)" > artifacts/test-data.txt
          echo "Build number: ${{ github.run_number }}" >> artifacts/test-data.txt
      
      - name: Upload Artifacts
        uses: actions/upload-artifact@v3
        with:
          name: test-artifacts
          path: artifacts/
          retention-days: 7

  download:
    needs: generate
    runs-on: ubuntu-latest
    steps:
      - name: Download Artifacts
        uses: actions/download-artifact@v3
        with:
          name: test-artifacts
          path: downloaded/
      
      - name: Verify Download
        run: |
          echo "Files in download:"
          ls -la downloaded/
          echo "Contents:"
          cat downloaded/test-data.txt

  verify:
    needs: download
    runs-on: ubuntu-latest
    steps:
      - name: Download Artifacts Again
        uses: actions/download-artifact@v3
        with:
          name: test-artifacts
          path: verify/
      
      - name: Final Verification
        run: |
          if [ -f verify/test-data.txt ]; then
            echo "✓ Artifact exists"
            grep "Test data" verify/test-data.txt && echo "✓ Content verified"
          else
            echo "✗ Artifact missing"
            exit 1
          fi
```

**Explanation:**
- `actions/upload-artifact@v3` stores files
- `actions/download-artifact@v3` retrieves files
- `retention-days` controls how long artifacts stored
- Multiple jobs can download same artifact
- Artifacts visible in Actions tab under run details
- Great for sharing build output, test results, logs

**Best Practices:**
- Always specify artifact name (not auto-generated)
- Use short retention for large artifacts
- Download only needed artifacts
- Include metadata in artifact (version, date)

---

## Solution 7: Environment Variables

**File:** `.github/workflows/ex7-env.yml`

```yaml
name: Environment Variables Workflow

on:
  push:
    branches: [ main ]

env:
  WORKFLOW_VAR: "workflow-level"
  SHARED_VAR: "from-workflow"

jobs:
  env-test:
    runs-on: ubuntu-latest
    env:
      JOB_VAR: "job-level"
      SHARED_VAR: "from-job"
    steps:
      - name: Workflow Level Variables
        run: |
          echo "WORKFLOW_VAR=$WORKFLOW_VAR"
          echo "Value: ${{ env.WORKFLOW_VAR }}"
      
      - name: Job Level Variables
        run: |
          echo "JOB_VAR=$JOB_VAR"
          echo "JOB_VAR via context: ${{ env.JOB_VAR }}"
      
      - name: Step Level Variables
        env:
          STEP_VAR: "step-level"
        run: |
          echo "STEP_VAR=$STEP_VAR"
          echo "Accessible in this step only"
      
      - name: Variable Scope and Precedence
        env:
          SHARED_VAR: "from-step"
        run: |
          echo "Workflow SHARED_VAR: from-workflow"
          echo "Job SHARED_VAR: from-job"
          echo "Step SHARED_VAR: from-step"
          echo "Actual value: $SHARED_VAR"
          echo "(Step overrides job, job overrides workflow)"
      
      - name: Access All Available
        run: |
          echo "Available in all steps of this job:"
          echo "WORKFLOW_VAR: $WORKFLOW_VAR"
          echo "JOB_VAR: $JOB_VAR"
          echo "Runner: ${{ runner.os }}"
          echo "Repo: ${{ github.repository }}"

  another-job:
    runs-on: ubuntu-latest
    steps:
      - name: Only Workflow Variables
        run: |
          echo "Workflow var: $WORKFLOW_VAR"
          echo "Job var not available: $JOB_VAR (empty)"
```

**Explanation:**
- **Workflow level:** Available to all jobs and steps
- **Job level:** Available to all steps in that job
- **Step level:** Available only to that specific step
- **Precedence:** Step > Job > Workflow
- Variables accessed via `$VAR_NAME` in shell or `${{ env.VAR_NAME }}` in expressions
- Variables are case-sensitive

**Scope Rules:**
- Define at workflow for global access
- Define at job for job-specific configuration
- Define at step for temporary values only

---

## Solution 8: Using GitHub Actions

**File:** `.github/workflows/ex8-actions.yml`

```yaml
name: Using GitHub Actions Workflow

on:
  push:
    branches: [ main ]

jobs:
  use-actions:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3
      
      - name: Display Repository Contents
        run: ls -la

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18.x'
          cache: 'npm'

      - name: Display Node Version
        run: |
          echo "Node: $(node --version)"
          echo "npm: $(npm --version)"
      
      - name: Create Package File (if needed)
        run: |
          if [ ! -f package.json ]; then
            npm init -y
            npm install lodash
          fi
      
      - name: Cache Verification
        run: echo "✓ Dependencies cached and ready"
```

**Explanation:**
- `actions/checkout@v3` clones repository to runner
- `actions/setup-node@v3` installs specified Node version
- `cache: 'npm'` automatically caches node_modules
- `with:` provides inputs to actions
- `v3` is semantic version tag (latest patch)
- First run caches dependencies, second run uses cache

**Common Actions:**
- `actions/checkout` - Clone repo
- `actions/setup-node` - Setup Node.js
- `actions/setup-python` - Setup Python
- `actions/cache` - Cache directories
- `actions/upload-artifact` - Store files
- All available at [GitHub Marketplace](https://github.com/marketplace?type=actions)

---

## Solution 9: Advanced Conditionals

**File:** `.github/workflows/ex9-advanced-conditions.yml`

```yaml
name: Advanced Conditionals Workflow

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Always Run
        run: echo "This always runs"
      
      - name: Might Fail Intentionally
        id: maybe-fail
        continue-on-error: true
        run: |
          if [ $RANDOM -gt 16384 ]; then
            echo "Random success"
          else
            exit 1
          fi
      
      - name: Check Status
        run: echo "Previous step status: ${{ steps.maybe-fail.outcome }}"

  on-push-main:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "✓ Running only on push to main"

  on-pull-request:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - run: echo "✓ Running only on pull request"

  on-success:
    needs: test
    if: success()
    runs-on: ubuntu-latest
    steps:
      - run: echo "✓ All previous jobs succeeded"

  on-failure:
    needs: test
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - run: echo "⚠️  A previous job failed"

  always-notify:
    if: always()
    runs-on: ubuntu-latest
    steps:
      - run: echo "Always run for notifications"
```

**Explanation:**
- `github.event_name == 'push'` - Only on push events
- `github.ref == 'refs/heads/main'` - Only main branch
- `success()` - Runs if previous jobs succeeded
- `failure()` - Runs if previous jobs failed
- `always()` - Runs regardless of status
- Can combine with `&&` (and) and `||` (or)
- `continue-on-error: true` doesn't fail job on step failure

**Expression Examples:**
```yaml
if: github.event_name == 'push' && github.ref == 'refs/heads/main'
if: github.actor == 'my-username'
if: contains(github.event.head_commit.message, '[skip ci]')
if: startswith(github.ref, 'refs/tags/v')
```

---

## Solution 10: Complete Multi-Stage Pipeline

**File:** `.github/workflows/ex10-pipeline.yml`

```yaml
name: Multi-Stage Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  # Stage 1: Build
  build:
    runs-on: ubuntu-latest
    outputs:
      build-version: ${{ steps.meta.outputs.version }}
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      
      - name: Setup Environment
        id: meta
        run: |
          VERSION="1.0.${{ github.run_number }}"
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "Version: $VERSION"
      
      - name: Build Application
        run: |
          mkdir -p build
          echo "Build version: 1.0.${{ github.run_number }}" > build/version.txt
          echo "Built at: $(date)" >> build/version.txt
      
      - name: Upload Build
        uses: actions/upload-artifact@v3
        with:
          name: build
          path: build/

  # Stage 2: Test
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download Build
        uses: actions/download-artifact@v3
        with:
          name: build
      
      - name: Run Tests
        run: |
          echo "Testing version:"
          cat version.txt
          echo "✓ Unit tests passed"
          echo "✓ Integration tests passed"
      
      - name: Generate Coverage
        run: |
          mkdir -p coverage
          echo "Coverage: 85%" > coverage/report.txt
      
      - name: Upload Coverage
        uses: actions/upload-artifact@v3
        with:
          name: coverage
          path: coverage/

  # Stage 3: Deploy
  deploy:
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment:
      name: production
    steps:
      - name: Download Build
        uses: actions/download-artifact@v3
        with:
          name: build
      
      - name: Verify Build
        run: |
          echo "Deployment Version:"
          cat version.txt
      
      - name: Deploy
        run: |
          echo "Deploying to production..."
          echo "✓ Deployment successful"
      
      - name: Smoke Tests
        run: |
          echo "✓ Health check passed"
          echo "✓ API responding"

  # Final: Notification
  notify:
    if: always()
    needs: [build, test, deploy]
    runs-on: ubuntu-latest
    steps:
      - name: Pipeline Summary
        run: |
          echo "Pipeline Execution Summary"
          echo "Build: ${{ needs.build.result }}"
          echo "Test: ${{ needs.test.result }}"
          echo "Deploy: ${{ needs.deploy.result }}"
```

**Explanation:**
- **Build stage:** Compiles and creates artifacts
- **Test stage:** Downloads build, runs tests, generates reports
- **Deploy stage:** Only runs on main branch push
- **Notify stage:** Always runs for reporting
- Job outputs pass version through stages
- Artifacts flow between jobs
- Each stage is independent and parallelizable

**Key Features:**
- Sequential but parallel-capable design
- Artifact management between stages
- Conditional deployment
- Status notifications
- Environment protection on production

---

## Quick Debugging Tips

| Problem | Solution |
|---------|----------|
| Job not running | Check `if:` condition and context values |
| Output not accessible | Verify `id:` set and `$GITHUB_OUTPUT` used |
| Variable empty | Check scope (workflow/job/step level) |
| Action not found | Verify action path and version exist |
| Artifacts missing | Ensure upload before download |

---

**Completed:** January 2026 | **Module:** 01-workflow-fundamentals
