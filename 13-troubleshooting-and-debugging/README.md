# Module 13: Troubleshooting and Debugging

## Overview

This module teaches you how to diagnose and fix issues in GitHub Actions workflows. You'll learn to read logs, identify errors, debug problems systematically, and resolve common issues that prevent workflows from succeeding.

## What You'll Learn

### Core Debugging Skills

1. **Reading GitHub Actions Logs** - Navigate and interpret workflow logs
2. **Common Error Types** - Identify error categories and causes
3. **Syntax Error Detection** - Find and fix YAML and script errors
4. **Environment Variable Debugging** - Troubleshoot variable issues
5. **Secret Management Problems** - Fix secret exposure and access issues
6. **Permission Errors** - Resolve authentication and authorization failures
7. **Conditional Logic Issues** - Debug `if:` conditions and job dependencies
8. **Docker Debugging** - Troubleshoot container-related problems
9. **Network Problems** - Debug connectivity and timeout issues
10. **Performance Issues** - Identify and fix slow workflows

## Prerequisites

- Completed Modules 0-12
- Basic understanding of GitHub Actions workflows
- Familiarity with YAML syntax
- Basic shell scripting knowledge

## Key Concepts

### Error Categories

```
┌─ Syntax Errors (workflow won't start)
│  ├─ Invalid YAML
│  ├─ Missing required fields
│  └─ Indentation issues
│
├─ Runtime Errors (workflow starts but fails)
│  ├─ Command execution failures
│  ├─ Missing files
│  └─ Permission denied
│
├─ Configuration Errors
│  ├─ Wrong environment variables
│  ├─ Secret not found
│  └─ Invalid secret reference
│
└─ Logical Errors
   ├─ Wrong conditional
   ├─ Dependency issues
   └─ Unexpected behavior
```

### Debugging Workflow

```
1. Check Workflow Status
   └─ Green checkmark? Pass
   └─ Red X? Check logs

2. Identify Failed Job
   └─ Click on failed job
   └─ Find which step failed

3. Read Error Message
   └─ Look at error output
   └─ Identify error type

4. Add Debug Output
   └─ Echo variables
   └─ Print step status

5. Verify Assumptions
   └─ Check conditions
   └─ Verify inputs

6. Test Fix
   └─ Update workflow
   └─ Re-run workflow

7. Validate
   └─ Confirm success
   └─ Check output
```

## Hands-On Lab: Debugging a Broken Workflow

### Scenario

You have a workflow with multiple errors. Your task is to identify, debug, and fix all issues.

### Step 1: Create Broken Workflow

Create a file `.github/workflows/debug-lab.yml`:

```yaml
name: Debug Lab

on: push

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Test variables
        run: |
          echo "HOME: $HOME"
          echo "UNDEFINED_VAR: $UNDEFINED_VARIABLE"
          # Missing variable will show empty
      
      - name: Test conditions
        if: github.ref == 'refs/heads/main' && success()
        run: echo "Condition met"
      
      - name: Test secret (WRONG)
        run: echo ${{ secrets.NONEXISTENT_SECRET }}
        # This won't print (good)
      
      - name: Test with error
        run: |
          echo "Starting test..."
          invalid_command_here
          echo "This won't run"
```

### Step 2: Push and Observe

```bash
git add .github/workflows/debug-lab.yml
git commit -m "Add debug lab workflow"
git push
```

**Expected Output:**
- Workflow starts running
- First step succeeds
- Second step might be skipped (depends on condition)
- Step "Test with error" fails with command not found

### Step 3: Examine Logs

**Path:** Actions tab → Latest run → Click job → Find failed step

```
Run invalid_command_here
  /bin/bash: line 2: invalid_command_here: command not found
  Error: Process completed with exit code 127.
```

**What you learn:**
- Exit code 127 = command not found
- Full path to error (line 2 of step)
- Step immediately stops after error

### Step 4: Add Debug Statements

Update workflow:

```yaml
- name: Debug test
  run: |
    set -x  # Enable debug mode
    echo "Step starting..."
    echo "Current directory: $(pwd)"
    echo "User: $(whoami)"
    echo "Path: $PATH"
    set +x  # Disable debug mode
    echo "Debug complete"
```

**Expected Output:**
```
+ echo 'Step starting...'
Step starting...
+ echo 'Current directory: /home/runner/work/myrepo/myrepo'
Current directory: /home/runner/work/myrepo/myrepo
+ echo 'User: runner'
User: runner
+ echo 'Path: /usr/local/sbin:/usr/local/bin:...'
Path: /usr/local/sbin:/usr/local/bin:...
+ set +x
Debug complete
```

### Step 5: Check Environment Variables

```yaml
- name: Show environment
  run: |
    echo "=== GitHub Context ==="
    echo "Repository: ${{ github.repository }}"
    echo "Ref: ${{ github.ref }}"
    echo "SHA: ${{ github.sha }}"
    echo "Actor: ${{ github.actor }}"
    
    echo "=== Available Variables ==="
    env | sort
```

**Expected Output:**
```
=== GitHub Context ===
Repository: username/myrepo
Ref: refs/heads/main
SHA: a1b2c3d4e5f6...
Actor: john.dev

=== Available Variables ===
CI=true
GITHUB_ACTION=run
GITHUB_ACTOR=john.dev
...
```

### Step 6: Test Conditionals

```yaml
- name: Test if conditions
  run: |
    echo "Branch: ${{ github.ref_name }}"
    echo "Is main? ${{ github.ref_name == 'main' }}"
    echo "Is pull request? ${{ github.event_name == 'pull_request' }}"
    echo "Success? ${{ job.status == 'success' }}"
```

### Step 7: Validate Fixes

After adding debug steps:

```bash
git add .
git commit -m "Add debug steps"
git push
```

**Success Indicators:**
- ✓ Workflow runs to completion
- ✓ All steps execute (or skip correctly)
- ✓ Debug output shows expected values
- ✓ No permission errors
- ✓ Conditionals evaluate correctly

### Step 8: Cleanup

Remove debug statements:

```yaml
# Delete test steps from workflow
rm .github/workflows/debug-lab.yml
git commit -am "Remove debug workflow"
```

## Common Debugging Techniques

### Technique 1: Echo Variables

```bash
echo "Variable value: $MY_VAR"
echo "Secret available: ${{ secrets.MY_SECRET != '' }}"
echo "Current path: $(pwd)"
echo "Git status: $(git status --short)"
```

### Technique 2: Enable Bash Debug Mode

```bash
set -x      # Enable debug (show each command)
# commands here
set +x      # Disable debug
```

**Output includes:**
```
+ command_that_runs
output from command
+ next_command
```

### Technique 3: Check Conditions

```yaml
- name: Debug conditional
  run: echo "Condition result: ${{ github.ref == 'refs/heads/main' }}"
```

### Technique 4: List Files

```bash
echo "=== Repository Contents ==="
find . -type f -name "*.yml" -o -name "*.yaml"

echo "=== Artifacts ==="
ls -la dist/ || echo "dist/ not found"
```

### Technique 5: Test Commands

```bash
# Test if command exists
which npm && echo "npm found" || echo "npm not found"

# Test if file exists
[ -f package.json ] && echo "package.json exists" || echo "not found"

# Test if directory writable
[ -w /tmp ] && echo "writable" || echo "not writable"
```

## Common Issues and Solutions

### Issue 1: Secret Not Available

**Error:**
```
Run echo ${{ secrets.API_KEY }}

```

**Cause:** Secret not defined in repository

**Solution:**
1. Go to Settings → Secrets → Actions
2. Add new secret: `API_KEY`
3. Set value
4. Re-run workflow

---

### Issue 2: Environment Variable Empty

**Problem:**
```bash
echo $UNDEFINED_VAR  # Prints nothing
```

**Cause:** Variable not set

**Solution:**
```yaml
env:
  MY_VAR: value

steps:
  - run: echo $MY_VAR
```

---

### Issue 3: Syntax Error in YAML

**Error:**
```
YAML parse error on line 15: mapping values are not allowed here
```

**Common causes:**
- Missing quotes: `key: value with spaces`
- Wrong indentation
- Using tabs instead of spaces
- Colons in unquoted strings

**Solution:**
```yaml
# ✓ Correct
env:
  MESSAGE: "Hello: World"
  PATH: "/usr/bin"

# ✗ Wrong
env:
  MESSAGE: Hello: World  # Ambiguous
```

---

### Issue 4: Command Not Found

**Error:**
```
/bin/bash: line 1: npm: command not found
```

**Cause:** Tool not installed or not in PATH

**Solution:**
```yaml
steps:
  - uses: actions/setup-node@v3
    with:
      node-version: '18'
  - run: npm --version
```

---

### Issue 5: Permission Denied

**Error:**
```
Permission denied: ./deploy.sh
```

**Cause:** Script not executable

**Solution:**
```bash
chmod +x ./deploy.sh
./deploy.sh
```

Or in workflow:

```yaml
- run: bash ./deploy.sh
```

---

### Issue 6: Job Dependency Mismatch

**Error:**
```
Job 'deploy' depends on non-existent job 'test'
```

**Cause:** Referenced job name doesn't exist

**Solution:**
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "test"

  deploy:
    needs: test  # ✓ Matches job name above
    steps:
      - run: echo "deploy"
```

---

### Issue 7: Conditional Never Runs

**Problem:**
```yaml
if: github.ref == 'refs/heads/main'
# Step never runs even on main branch
```

**Cause:** Wrong comparison or branch context

**Solution:**
```yaml
# ✓ Correct: full ref
if: github.ref == 'refs/heads/main'

# ✓ Or use ref_name
if: github.ref_name == 'main'

# Debug to verify
- run: echo "Ref: ${{ github.ref }}, Ref name: ${{ github.ref_name }}"
```

---

### Issue 8: Secret Exposed in Logs

**Problem:** Secret value printed in workflow output

**Cause:** Accidentally echoing secret

```yaml
- run: echo "Password: ${{ secrets.PASSWORD }}"  # BAD!
```

**Solution:**
```yaml
# GitHub automatically masks secrets
# But don't print them anyway

# ✓ Use in environment variable
env:
  API_KEY: ${{ secrets.API_KEY }}

# ✓ Pass to command without echoing
- run: deploy --key=${{ secrets.API_KEY }}
```

---

### Issue 9: Artifact Not Found

**Error:**
```
Error: failed to download artifact: Artifact not found
```

**Cause:** Artifact not uploaded or wrong name

**Solution:**
```yaml
jobs:
  build:
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v3
        with:
          name: dist  # ✓ Name the artifact
          path: dist/

  deploy:
    needs: build
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: dist  # ✓ Match the name
          path: dist/
```

---

### Issue 10: Cache Not Working

**Problem:** Cache never hits, always does full install

**Cause:** Cache key doesn't match or dependencies changed

**Solution:**
```yaml
- uses: actions/setup-node@v3
  with:
    node-version: '18'
    cache: 'npm'  # ✓ Enable caching

# Cache automatically created from package-lock.json
# Key: Linux-node-18-npm-<hash>
```

To debug cache:

```yaml
- uses: actions/setup-node@v3
  id: cache
  with:
    node-version: '18'
    cache: 'npm'

- run: echo "Cache hit: ${{ steps.cache.outputs.cache-hit }}"
```

## Validation Checklist

Before declaring workflow "fixed":

- [ ] Workflow syntax valid (no red warnings)
- [ ] All required jobs execute
- [ ] No unexpected skipped steps
- [ ] Error messages resolved
- [ ] Logs show expected output
- [ ] Secrets not exposed
- [ ] Conditional logic correct
- [ ] Artifacts created if expected
- [ ] Health checks pass
- [ ] No timeout errors

## Troubleshooting Checklist

When workflow fails:

- [ ] Check last step output for error message
- [ ] Identify error type (syntax/runtime/config)
- [ ] Search error message in GitHub docs
- [ ] Add debug echo statements
- [ ] Verify environment variables set
- [ ] Check secrets exist and are named correctly
- [ ] Validate YAML syntax
- [ ] Check job dependencies
- [ ] Review conditional logic
- [ ] Test in simpler workflow first

## Next Steps

1. **Practice debugging** - Use exercises 1-5
2. **Learn advanced techniques** - Use exercises 6-7
3. **Master problem-solving** - Use exercises 8-10
4. **Apply to real workflows** - Debug your own workflows
5. **Build debugging library** - Document solutions

## Key Takeaways

✓ Master GitHub Actions logs
✓ Identify common error patterns
✓ Use systematic debugging approach
✓ Add effective debug statements
✓ Verify assumptions before fixing
✓ Understand error categories
✓ Know where to find help
✓ Document solutions
✓ Prevent future issues
✓ Debug confidently

## Exercises & Resources

📝 **exercises.md** - 10 debugging challenges
✅ **solutions.md** - Complete solutions with explanations
📋 **cheatsheet.md** - Quick debugging command reference

---

**Duration:** 2-3 hours for all exercises
**Difficulty:** Intermediate
**Prerequisites:** Modules 0-12

Happy debugging! 🐛🔍
