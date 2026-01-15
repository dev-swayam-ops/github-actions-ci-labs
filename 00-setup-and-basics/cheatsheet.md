# Cheatsheet: Setup and Basics

Quick reference guide for GitHub Actions setup and fundamental concepts.

---

## Workflow Structure Template

```yaml
name: Workflow Name

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  VARIABLE_NAME: value

jobs:
  job-name:
    runs-on: ubuntu-latest
    env:
      JOB_VAR: job-value
    steps:
      - name: Step Name
        run: command
```

---

## Common Commands and Purpose

| Command | Purpose | Example |
|---------|---------|---------|
| `echo "text"` | Print message to output | `echo "Hello GitHub Actions"` |
| `date` | Display current date/time | `date` |
| `pwd` | Print working directory | `pwd` |
| `ls` or `dir` | List files | `ls -la` |
| `mkdir` | Create directory | `mkdir -p .github/workflows` |
| `cat` | Display file contents | `cat package.json` |
| `grep` | Search in text | `grep "version" package.json` |
| `uname -a` | System information (Linux/Mac) | `uname -a` |
| `systeminfo` | System information (Windows) | `systeminfo` |
| `whoami` | Current user | `whoami` |

---

## Trigger Events (on:)

| Event | When It Triggers | Syntax |
|-------|-----------------|--------|
| `push` | Commit pushed to repository | `on: push:` |
| `pull_request` | PR created/updated | `on: pull_request:` |
| `schedule` | On cron schedule | `on: schedule: - cron: '0 0 * * *'` |
| `manual` | Manual trigger | `on: workflow_dispatch:` |
| `release` | Release published | `on: release:` |
| `issues` | Issue opened/edited | `on: issues:` |

---

## Runner Environments

| Runner | Operating System | Use Case |
|--------|------------------|----------|
| `ubuntu-latest` | Linux | Most common, free, good default |
| `macos-latest` | macOS | iOS/macOS development |
| `windows-latest` | Windows | .NET development |
| `ubuntu-20.04` | Ubuntu 20.04 | Specific version |
| `macos-11` | macOS Monterey | Specific version |

---

## Job Syntax Reference

```yaml
jobs:
  job-name:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
    env:
      VARIABLE: value
    steps:
      - name: Step Name
        run: echo "Hello"
```

| Key | Purpose | Example |
|-----|---------|---------|
| `runs-on` | Runner environment | `ubuntu-latest` |
| `timeout-minutes` | Max execution time | `30` |
| `needs` | Depends on another job | `needs: setup` |
| `if` | Conditional execution | `if: github.ref == 'refs/heads/main'` |
| `env` | Job-level variables | `MY_VAR: value` |

---

## Step Syntax Reference

```yaml
steps:
  - name: Step Name
    run: command
    working-directory: path
    shell: bash
    env:
      VAR: value
    if: condition
```

| Key | Purpose | Example |
|-----|---------|---------|
| `name` | Step identifier | `"Print Hello"` |
| `run` | Shell command | `echo "test"` |
| `uses` | GitHub Action | `actions/checkout@v2` |
| `with` | Action parameters | `key: value` |
| `shell` | Shell type | `bash`, `pwsh`, `sh` |
| `working-directory` | Working path | `./src` |
| `env` | Step variables | `MY_VAR: value` |
| `if` | Conditional | `success()`, `failure()` |

---

## Context Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `${{ github.repository }}` | Owner/Repo | `user/my-repo` |
| `${{ github.repository_owner }}` | Repository owner | `user` |
| `${{ github.sha }}` | Commit SHA | `abc123def456` |
| `${{ github.ref }}` | Branch/Tag reference | `refs/heads/main` |
| `${{ github.event_name }}` | Event type | `push`, `pull_request` |
| `${{ github.actor }}` | Username triggering action | `octocat` |
| `${{ github.workspace }}` | Working directory | `/home/runner/work/repo/repo` |
| `${{ runner.os }}` | Runner OS | `Linux`, `Windows`, `macOS` |
| `${{ job.status }}` | Job status | `success`, `failure` |

---

## Multi-line Commands

```yaml
- name: Run Multiple Commands
  run: |
    echo "Line 1"
    echo "Line 2"
    command-one
    command-two
```

Use pipe `|` to write multiple commands in one step.

---

## Conditional Execution

```yaml
steps:
  - name: Only on Main
    if: github.ref == 'refs/heads/main'
    run: echo "Running on main"

  - name: Only on Push
    if: github.event_name == 'push'
    run: echo "This is a push"

  - name: Only on Failure
    if: failure()
    run: echo "Previous step failed"

  - name: Only on Success
    if: success()
    run: echo "Previous step succeeded"
```

---

## YAML Syntax Quick Guide

```yaml
# Comments start with #

# Strings
simple: Hello
quoted: "Hello World"
multiline: |
  Line 1
  Line 2

# Numbers
integer: 42
decimal: 3.14

# Boolean
enabled: true
disabled: false

# Lists
items:
  - item1
  - item2
  - item3

# Or inline
colors: [ red, green, blue ]

# Dictionaries
person:
  name: John
  age: 30
```

**Key Rules:**
- Use 2 spaces (not tabs)
- Colons separate keys and values
- Dash starts list items
- Indentation shows hierarchy

---

## Directory Structure

```
your-repo/
├── .github/
│   └── workflows/
│       ├── hello.yml
│       ├── test.yml
│       └── deploy.yml
├── src/
├── README.md
└── package.json
```

Workflows MUST be in `.github/workflows/` directory.

---

## Common Workflow Patterns

### Simple Echo Pattern
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Hello"
```

### Checkout + Commands Pattern
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - run: |
          npm install
          npm test
```

### Conditional Step Pattern
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - if: github.ref == 'refs/heads/main'
        run: echo "Deploying to production"
```

### Job Dependencies Pattern
```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Setup"
  
  test:
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - run: echo "Test"
```

---

## Debugging Tips

| Issue | Debug Command |
|-------|---------------|
| Variable not set | `echo "Value: $VARIABLE"` |
| File not found | `ls -la` or `dir /s` |
| Wrong directory | `pwd` (Linux/Mac) or `cd` (Windows) |
| Environment info | `uname -a` or `systeminfo` |
| Path variable | `echo $PATH` or `echo %PATH%` |

---

## Quick Reference: File Operations

```yaml
# Create file
- run: echo "content" > filename.txt

# Append to file
- run: echo "content" >> filename.txt

# Copy file
- run: cp source.txt destination.txt

# Remove file
- run: rm filename.txt

# Create directory
- run: mkdir -p path/to/dir
```

---

## Useful Links

| Resource | URL |
|----------|-----|
| GitHub Actions Docs | https://docs.github.com/en/actions |
| Workflow Syntax | https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions |
| Trigger Events | https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows |
| Marketplace | https://github.com/marketplace?type=actions |
| YAML Validator | https://www.yamllint.com |

---

## Keyboard Shortcuts (GitHub UI)

| Action | Shortcut |
|--------|----------|
| Go to Actions tab | `g` then `a` |
| Refresh page | `F5` |
| Search | `Ctrl+F` or `Cmd+F` |

---

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Unexpected value` | YAML syntax error | Check indentation, use yamllint |
| `No workflows found` | File not in `.github/workflows/` | Move file to correct directory |
| `Branch not found` | Branch doesn't exist | Verify branch name in `on:` section |
| `Variable undefined` | Wrong syntax | Use `${{ variable }}` format |
| `Command not found` | Wrong runner/OS | Use appropriate commands for runner OS |

---

## Workflow File Naming

- Use lowercase letters
- Use hyphens for word separation
- Example: `build-and-test.yml`, `deploy-production.yml`
- Must end in `.yml` or `.yaml`

---

**Quick Start Command:**

```bash
mkdir -p .github/workflows
echo 'name: Hello World
on: push
jobs:
  hello:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Hello"' > .github/workflows/hello.yml
git add .github/workflows/hello.yml
git commit -m "Add hello workflow"
git push
```

---

**Cheatsheet Version:** 1.0 | **Updated:** January 2026 | **Module:** 00-setup-and-basics
