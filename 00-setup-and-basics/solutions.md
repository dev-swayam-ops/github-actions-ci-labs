# Solutions: Setup and Basics

Complete solutions for all exercises in the Setup and Basics module.

---

## Solution 1: Create a Basic Workflow File

**File:** `.github/workflows/ex1-basic.yml`

```yaml
name: Basic Workflow

on:
  push:
    branches: [ main ]

jobs:
  info-job:
    runs-on: ubuntu-latest
    steps:
      - name: Print Welcome Message
        run: echo "Repository initialized"
```

**Explanation:**
- `name:` defines the workflow display name
- `on: push:` triggers on push events
- `branches: [ main ]` restricts to main branch only
- `jobs:` defines workflow jobs
- `runs-on:` specifies the runner environment
- `steps:` contains sequential tasks
- `run:` executes a shell command

**Verification:**
Push to `main` branch and check Actions tab for successful execution.

---

## Solution 2: YAML Indentation Practice

**Correct Option:** Option A

```yaml
name: Test
on: push
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Hello"
```

**Explanation:**
- GitHub Actions workflows require 2-space indentation
- Option B uses 4-space indentation, which causes parsing errors
- Consistency in spacing is critical for YAML validity
- Tools like VSCode with YAML extensions can highlight indentation issues

**Why Option B Fails:**
- YAML parsers expecting 2-space indentation will misinterpret 4-space blocks
- The nested structure becomes ambiguous

---

## Solution 3: Multiple Triggers

**File:** `.github/workflows/ex3-triggers.yml`

```yaml
name: Multiple Triggers

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  trigger-job:
    runs-on: ubuntu-latest
    steps:
      - name: Print Trigger Event
        run: echo "Triggered by ${{ github.event_name }}"
      
      - name: Print Branch
        run: echo "Branch/Ref: ${{ github.ref }}"
```

**Explanation:**
- `on:` section can specify multiple event types
- `push:` and `pull_request:` are separate event types
- Can restrict to specific branches using `branches:`
- `${{ github.event_name }}` is a context variable showing the trigger event
- `${{ github.ref }}` shows the current branch or tag

**Testing:**
1. Push to `main` → workflow runs
2. Push to `develop` → workflow runs
3. Create PR to `main` → workflow runs
4. Create PR to other branches → workflow does NOT run

---

## Solution 4: Multiple Steps in a Job

**File:** `.github/workflows/ex4-steps.yml`

```yaml
name: Multiple Steps Workflow

on:
  push:
    branches: [ main ]

jobs:
  multi-step-job:
    runs-on: ubuntu-latest
    steps:
      - name: Step 1 - Start
        run: echo "Starting workflow"
      
      - name: Step 2 - Current Date
        run: date
      
      - name: Step 3 - List Files
        run: ls -la
      
      - name: Step 4 - OS Info
        run: uname -a
      
      - name: Step 5 - Complete
        run: echo "Workflow complete"
```

**Explanation:**
- Steps execute sequentially (one after another)
- Each step has a `name:` for identification in logs
- `run:` executes shell commands
- Output from each step is visible in workflow logs
- If any step fails, subsequent steps may not execute (by default)

**Expected Output Order:**
```
Starting workflow
[date and time]
[list of files]
[operating system info]
Workflow complete
```

---

## Solution 5: YAML Syntax Validation

**File:** `.github/workflows/ex5-syntax.yml` (Corrected)

```yaml
name: YAML Syntax Validation

on:
  push:
    branches: [ main ]

jobs:
  syntax-check:
    runs-on: ubuntu-latest
    steps:
      - name: Validate Configuration
        run: echo "YAML syntax is valid"
      
      - name: Proper Indentation
        run: echo "Using 2 spaces consistently"
      
      - name: Proper Quotes
        run: echo "Using proper quotation marks"
```

**Common Errors Found:**

| Error | Example | Fix |
|-------|---------|-----|
| Missing colon | `name: Test` (missing) | `name: Test` |
| Bad indentation | 4 spaces mixed with 2 | Use 2 spaces consistently |
| Tabs instead of spaces | Tab character present | Replace tabs with spaces |
| Unclosed quotes | `run: echo "test` | `run: echo "test"` |
| Invalid nesting | Wrong indentation level | Proper hierarchy: workflow → job → step |

**Validation Tools:**
- [yamllint.com](https://www.yamllint.com)
- VSCode YAML extension
- GitHub's own syntax checker in Actions tab

---

## Solution 6: Understanding Runners

**File:** `.github/workflows/ex6-runners.yml`

```yaml
name: Multi-Runner Workflow

on:
  push:
    branches: [ main ]

jobs:
  ubuntu-job:
    runs-on: ubuntu-latest
    steps:
      - name: Get Ubuntu Info
        run: uname -a

  macos-job:
    runs-on: macos-latest
    steps:
      - name: Get macOS Info
        run: uname -a

  windows-job:
    runs-on: windows-latest
    steps:
      - name: Get Windows Info
        run: systeminfo
```

**Expected Output Differences:**

- **Ubuntu:** `Linux runner-xyz 5.15.0-xxx #1 SMP ... GNU/Linux`
- **macOS:** `Darwin runner-xyz 12.6.1 ... arm64 Apple`
- **Windows:** Multiple lines with system info

**Key Observations:**
- Different operating systems have different utilities
- Command syntax varies (Windows uses `systeminfo` instead of `uname`)
- Execution time may differ
- Available tools and languages differ by runner

**Note:** Windows runners may take longer to start.

---

## Solution 7: Conditional Execution with Needs

**File:** `.github/workflows/ex7-needs.yml`

```yaml
name: Dependent Jobs

on:
  push:
    branches: [ main ]

jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - name: Setup Phase
        run: echo "Setup complete"

  test:
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - name: Run Tests
        run: echo "Running tests after setup"
      
      - name: Test Summary
        run: echo "All tests passed"
```

**Explanation:**
- `needs: setup` means the `test` job waits for `setup` to complete
- If `setup` fails, `test` will not execute
- Multiple dependencies: `needs: [setup, build]` (array syntax)
- Useful for sequential workflows: setup → build → test → deploy

**Workflow Execution:**
1. `setup` job runs first
2. Waits for completion
3. `test` job runs only if `setup` succeeds
4. Both jobs run on the same or different runners

---

## Solution 8: Environment Variables

**File:** `.github/workflows/ex8-env.yml`

```yaml
name: Environment Variables

on:
  push:
    branches: [ main ]

env:
  ENVIRONMENT: development
  PROJECT_NAME: MyApp
  VERSION: 1.0.0

jobs:
  var-test:
    runs-on: ubuntu-latest
    env:
      JOB_ENV: job-level
    
    steps:
      - name: Workflow-Level Variables
        run: |
          echo "Environment: $ENVIRONMENT"
          echo "Project: $PROJECT_NAME"
          echo "Version: $VERSION"
      
      - name: Job-Level Variable
        run: echo "Job Env: $JOB_ENV"
      
      - name: Step-Level Variable
        env:
          STEP_VAR: step-value
        run: echo "Step Var: $STEP_VAR"
      
      - name: All Variables
        env:
          STEP_VAR: step-value
        run: |
          echo "Workflow: $ENVIRONMENT"
          echo "Job: $JOB_ENV"
          echo "Step: $STEP_VAR"
```

**Variable Scope:**
- **Workflow-level:** Available to all jobs and steps
- **Job-level:** Available only to steps in that job
- **Step-level:** Available only to that step
- **Precedence:** Step > Job > Workflow (step level overrides others)

**Expected Output:**
```
Environment: development
Project: MyApp
Version: 1.0.0
Job Env: job-level
Step Var: step-value
[Final output shows all three]
```

---

## Solution 9: Workflow Status and Logs

**Navigation Steps:**

1. Go to your repository on GitHub
2. Click **Actions** tab
3. Click the workflow name
4. Click a workflow run
5. Expand each job/step to see details

**Log Information to Identify:**

```
Name:         [Step name from YAML]
Status:       ✅ Success or ❌ Failure
Duration:     Time taken (e.g., 2s)
Output:       Command output and results
Timestamp:    When the step executed
```

**Document Template:**

| Step | Duration | Status | Key Output |
|------|----------|--------|-----------|
| Step 1 - ... | 1s | ✅ | [output] |
| Step 2 - ... | 2s | ✅ | [output] |
| Step 3 - ... | 1s | ✅ | [output] |

**Key Insights:**
- Failed steps show red X and stop execution
- Logs can be searched and filtered
- Timing helps identify performance issues
- Full transcript available for debugging

---

## Solution 10: Repository Metadata in Workflows

**File:** `.github/workflows/ex10-context.yml`

```yaml
name: GitHub Context Variables

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  context-job:
    runs-on: ubuntu-latest
    steps:
      - name: Repository Context
        run: |
          echo "Repository: ${{ github.repository }}"
          echo "Owner: ${{ github.repository_owner }}"
          echo "Commit SHA: ${{ github.sha }}"
          echo "Ref: ${{ github.ref }}"
          echo "Event: ${{ github.event_name }}"
      
      - name: Branch-specific Info
        if: github.ref == 'refs/heads/main'
        run: echo "Running on main branch!"
      
      - name: Pull Request Info
        if: github.event_name == 'pull_request'
        run: echo "This is a pull request workflow"
```

**Sample Output (On Push):**
```
Repository: your-username/your-repo
Owner: your-username
Commit SHA: abc123def456...
Ref: refs/heads/main
Event: push
Running on main branch!
```

**Sample Output (On PR):**
```
Repository: your-username/your-repo
Owner: your-username
Commit SHA: [PR commit SHA]
Ref: refs/pull/1/merge
Event: pull_request
This is a pull request workflow
```

**Key Context Variables:**

| Variable | Value | Use Case |
|----------|-------|----------|
| `github.repository` | owner/repo | Logging, notifications |
| `github.sha` | commit hash | Version tagging, artifacts |
| `github.ref` | branch/tag | Conditional execution |
| `github.event_name` | push, pull_request, etc. | Different logic per event |
| `github.actor` | username | Attribution, permissions |

---

## Quick Tips for Success

1. **Always use 2 spaces** for YAML indentation
2. **Validate YAML** before pushing to catch syntax errors
3. **Use meaningful step names** for easy log reading
4. **Test workflows locally** using yamllint before GitHub
5. **Check context variables** in the Actions documentation
6. **Keep workflows simple** - start small, then add complexity
7. **Use conditional execution** to skip unnecessary steps
8. **Monitor execution time** and optimize if needed

---

## Troubleshooting Summary

| Problem | Solution |
|---------|----------|
| Workflow won't trigger | Check branch name matches `on:` configuration |
| YAML error | Use yamllint tool or VSCode YAML extension |
| Variables not working | Check syntax: `${{ variable }}` with proper spacing |
| Steps running in wrong order | Check you're using sequential execution (default) |
| Permission denied errors | Ensure runner has proper permissions for commands |

---

**Completed:** January 2026 | **Module:** 00-setup-and-basics
