# Exercises: Setup and Basics

Complete the following exercises to reinforce your understanding of GitHub Actions setup and fundamentals.

---

## Exercise 1: Create a Basic Workflow File (Easy)

**Objective:** Create a simple workflow that prints your repository name.

**Instructions:**
1. Create a file `.github/workflows/ex1-basic.yml`
2. Define a workflow that runs on `push` to the `main` branch
3. Add a single job called `info-job`
4. Add a step that runs `echo "Repository initialized"`
5. Push to your repository and verify execution

**Hint:** Look at the hello-world example in README.md

**Acceptance Criteria:**
- File exists at `.github/workflows/ex1-basic.yml`
- Workflow runs on push events
- Step executes successfully

---

## Exercise 2: YAML Indentation Practice (Easy)

**Objective:** Understand proper YAML indentation.

**Instructions:**
1. Create a file `.github/workflows/ex2-indentation.yml`
2. Identify which of the following is correct YAML:

```yaml
# Option A
name: Test
on: push
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Hello"

# Option B
name: Test
on: push
jobs:
    test:
        runs-on: ubuntu-latest
        steps:
            - run: echo "Hello"
```

3. Create a workflow using the correct indentation
4. Push and verify

**Acceptance Criteria:**
- Workflow passes YAML validation
- Uses 2-space indentation throughout

---

## Exercise 3: Multiple Triggers (Easy)

**Objective:** Create a workflow triggered by multiple events.

**Instructions:**
1. Create a file `.github/workflows/ex3-triggers.yml`
2. Configure the workflow to trigger on:
   - Push to `main` and `develop` branches
   - Pull requests to `main`
3. Add a step that prints the trigger event type
4. Push and create a pull request to test both triggers

**Hint:** Use the `on:` section with multiple event types

**Acceptance Criteria:**
- Workflow runs on push to both branches
- Workflow runs on pull request

---

## Exercise 4: Multiple Steps in a Job (Easy)

**Objective:** Create a workflow with sequential steps.

**Instructions:**
1. Create a file `.github/workflows/ex4-steps.yml`
2. Create a job with at least 5 steps:
   - Print "Starting workflow"
   - Display current date
   - List files in current directory
   - Display OS information
   - Print "Workflow complete"
3. Push and verify all steps execute in order

**Acceptance Criteria:**
- All 5 steps execute sequentially
- Each step produces visible output
- Workflow completes successfully

---

## Exercise 5: YAML Syntax Validation (Medium)

**Objective:** Learn to identify and fix YAML syntax errors.

**Instructions:**
1. Create a file `.github/workflows/ex5-syntax.yml`
2. Intentionally introduce these errors one at a time:
   - Missing colon after a key
   - Inconsistent indentation (4 spaces in one section)
   - Using tabs instead of spaces
   - Unclosed quotes
3. Push each version and observe the error
4. Fix all errors and push the corrected version
5. Verify workflow now runs successfully

**Acceptance Criteria:**
- Document what each error looks like
- Final version has no syntax errors

---

## Exercise 6: Understanding Runners (Medium)

**Objective:** Explore different runner environments.

**Instructions:**
1. Create a file `.github/workflows/ex6-runners.yml`
2. Create three separate jobs, one for each runner:
   - `ubuntu-latest`
   - `macos-latest`
   - `windows-latest`
3. Each job should print the OS information
4. Run the workflow and compare outputs
5. Document differences in the outputs

**Example Command for Each:**
- Ubuntu: `uname -a`
- macOS: `uname -a`
- Windows: `systeminfo`

**Acceptance Criteria:**
- All three jobs execute successfully
- You can see OS differences in the logs

---

## Exercise 7: Conditional Execution with Needs (Medium)

**Objective:** Create jobs that depend on each other.

**Instructions:**
1. Create a file `.github/workflows/ex7-needs.yml`
2. Create two jobs:
   - `setup` job that prints "Setup complete"
   - `test` job that depends on `setup` job
3. The `test` job should only run after `setup` completes
4. Push and verify job execution order in logs

**Hint:** Use the `needs:` keyword in the second job

**Acceptance Criteria:**
- Jobs run in the correct order
- Dependencies are clearly shown in the workflow logs

---

## Exercise 8: Environment Variables (Medium)

**Objective:** Use environment variables in workflows.

**Instructions:**
1. Create a file `.github/workflows/ex8-env.yml`
2. Define environment variables at the workflow level:
   - `ENVIRONMENT: development`
   - `PROJECT_NAME: MyApp`
   - `VERSION: 1.0.0`
3. Create a job that uses these variables in echo statements
4. Also define a job-level variable and a step-level variable
5. Print all three levels to show scope

**Acceptance Criteria:**
- Environment variables are accessible in steps
- All three variable levels print correctly

---

## Exercise 9: Workflow Status and Logs (Medium)

**Objective:** Navigate and understand workflow logs.

**Instructions:**
1. Commit a workflow that has at least 3 steps
2. Go to GitHub Actions tab
3. Find the latest workflow run
4. For each step, identify:
   - Step name
   - Execution time
   - Step status (success/failure)
   - Log messages
5. Document what information you can extract from logs

**Acceptance Criteria:**
- You can navigate to a workflow run
- You can read and understand all log sections

---

## Exercise 10: Repository Metadata in Workflows (Medium)

**Objective:** Access GitHub repository context information.

**Instructions:**
1. Create a file `.github/workflows/ex10-context.yml`
2. Use GitHub's context variables to print:
   - Repository name: `${{ github.repository }}`
   - Repository owner: `${{ github.repository_owner }}`
   - Commit SHA: `${{ github.sha }}`
   - Ref (branch/tag): `${{ github.ref }}`
   - Event name: `${{ github.event_name }}`
3. Push and verify all variables display correctly
4. Create a pull request and run again to see context changes

**Hint:** These are built-in GitHub context variables available in all workflows

**Acceptance Criteria:**
- All context variables display correct values
- Variables change appropriately for different triggers

---

## Self-Assessment

After completing these exercises, you should be able to:

- [ ] Create a basic GitHub Actions workflow file
- [ ] Understand and write proper YAML syntax
- [ ] Configure multiple triggers for workflows
- [ ] Organize steps in logical order
- [ ] Identify and fix common YAML errors
- [ ] Understand different runner environments
- [ ] Use workflow context variables
- [ ] Navigate workflow logs and understand status
- [ ] Define and use environment variables at different scopes
- [ ] Create dependent jobs using `needs:`

**Next:** Review the [solutions.md](solutions.md) file to see example solutions.

---

**Difficulty:** Easy to Medium | **Time:** 2-3 hours | **Prerequisites:** Module 00 completion
