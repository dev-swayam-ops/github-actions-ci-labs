# 00-setup-and-basics

## Overview

This module introduces you to GitHub Actions CI/CD fundamentals. You'll learn how to set up your first GitHub Actions workflow and understand the basic building blocks of automation.

---

## What You'll Learn

- How to create and configure a GitHub repository for CI/CD
- Basic YAML syntax used in GitHub Actions workflows
- Understanding the `.github/workflows/` directory structure
- Creating your first "Hello World" workflow
- Understanding workflow triggers (events)
- Basic job and step structure
- Viewing workflow results and logs

---

## Prerequisites

- GitHub account (free tier is sufficient)
- Basic understanding of Git and repositories
- Text editor or VS Code
- Command line familiarity (Windows PowerShell, macOS/Linux bash)
- Basic YAML knowledge (we'll cover this!)

---

## Key Concepts

### 1. **GitHub Actions**
GitHub Actions is a CI/CD platform that lets you automate tasks when events happen in your repository.

### 2. **Workflow**
A workflow is an automated process defined in YAML format that runs jobs when triggered by an event.

### 3. **Trigger (Event)**
Events that cause a workflow to run: `push`, `pull_request`, `schedule`, manual trigger, etc.

### 4. **Job**
A set of steps that run in a workflow. Jobs run in parallel by default.

### 5. **Step**
An individual task within a job. Steps run sequentially.

### 6. **Action**
A reusable unit of code. Can be created or used from the GitHub Actions marketplace.

### 7. **YAML Format**
GitHub Actions workflows use YAML syntax. Key points:
- Indentation matters (use 2 spaces)
- Keys and values separated by colons
- Lists start with hyphens

---

## Hands-on Lab: Create Your First Workflow

### Step 1: Prepare Your Repository

```bash
# Clone or create a repository
git clone https://github.com/YOUR-USERNAME/YOUR-REPO.git
cd YOUR-REPO

# Create the workflows directory
mkdir -p .github/workflows

# Navigate to the directory
cd .github/workflows
```

### Step 2: Create a Simple Workflow File

Create a file named `hello-world.yml`:

```yaml
name: Hello World Workflow

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  hello-job:
    runs-on: ubuntu-latest
    steps:
      - name: Print Hello Message
        run: echo "Hello from GitHub Actions!"
      
      - name: Print Current Date
        run: date
      
      - name: Print Working Directory
        run: pwd
```

### Step 3: Commit and Push

```bash
git add .github/workflows/hello-world.yml
git commit -m "Add hello world workflow"
git push origin main
```

### Step 4: Monitor Execution

**Expected Output:**

Go to your repository on GitHub → `Actions` tab → Click the workflow run

You should see:

```
Hello from GitHub Actions!
Wed Jan 15 10:30:45 UTC 2026
/home/runner/work/YOUR-REPO/YOUR-REPO
```

The workflow will show:
- ✅ Completed (green checkmark)
- Execution time
- Logs for each step

---

## Validation Checklist

- [ ] Repository has `.github/workflows/` directory
- [ ] `hello-world.yml` file exists and is valid YAML
- [ ] Workflow triggered on push and pull_request
- [ ] All steps executed successfully
- [ ] You can view logs in the Actions tab
- [ ] Workflow completes in under 30 seconds

---

## Cleanup

To remove the workflow from your repository:

```bash
rm .github/workflows/hello-world.yml
git add -A
git commit -m "Remove hello world workflow"
git push origin main
```

---

## Common Mistakes

### ❌ Mistake 1: Incorrect YAML Indentation

```yaml
# Wrong - 4 spaces
jobs:
    hello-job:
        runs-on: ubuntu-latest
```

```yaml
# Correct - 2 spaces
jobs:
  hello-job:
    runs-on: ubuntu-latest
```

### ❌ Mistake 2: Wrong Directory Path

Workflow files must be in `.github/workflows/` (not `.github/workflow/`)

### ❌ Mistake 3: Invalid Branch Name

```yaml
# Wrong - if main branch doesn't exist
on:
  push:
    branches: [ master ]
```

```yaml
# Correct - verify your default branch name
on:
  push:
    branches: [ main ]
```

### ❌ Mistake 4: Using Tabs Instead of Spaces

YAML requires spaces. Tabs will cause parsing errors.

### ❌ Mistake 5: Missing Runner Specification

Every job needs a runner:

```yaml
jobs:
  my-job:
    # Wrong - no runs-on specified
    steps:
      - run: echo "Hello"
```

```yaml
jobs:
  my-job:
    runs-on: ubuntu-latest  # Correct
    steps:
      - run: echo "Hello"
```

---

## Troubleshooting

### Issue: Workflow file not executing

**Solution:**
1. Check the file is in `.github/workflows/`
2. Verify the filename ends with `.yml` or `.yaml`
3. Check YAML syntax at [yamllint.com](https://www.yamllint.com)
4. Ensure the trigger event matches (check branch names)

### Issue: "No workflows found"

**Solution:**
- Push the `.github/workflows/` directory to your repository
- Workflows only execute on the remote repository, not locally

### Issue: YAML parsing error

**Solution:**
- Use 2 spaces for indentation, never tabs
- Use a YAML validator to check syntax
- Check for special characters that need quotes

### Issue: Workflow shows as "Queued" for too long

**Solution:**
- This can happen if GitHub Actions is busy
- Workflows usually start within 1-2 minutes
- Check if you have any workflow concurrency limits

---

## Next Steps

✅ **You've completed module 00!**

**What's next?**

1. Proceed to **01-workflow-fundamentals** to learn:
   - Different trigger types (schedule, manual, webhooks)
   - Understanding workflow syntax in detail
   - Working with artifacts and logs
   
2. Practice creating variations of workflows:
   - Trigger on different events
   - Add more steps and jobs
   - Use environment variables

3. Explore the [GitHub Actions Marketplace](https://github.com/marketplace?type=actions) for reusable actions

---

## Quick Reference

| Term | Definition |
|------|-----------|
| Workflow | YAML file that defines automation |
| Trigger | Event that starts a workflow |
| Job | Group of steps |
| Step | Individual task |
| Action | Reusable code unit |
| Runner | Machine that executes workflows |

---

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [YAML Syntax Basics](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Workflow Triggers Reference](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)

---

**Created:** January 2026 | **Level:** Beginner | **Estimated Time:** 30 minutes
