# Module 13 Exercises: Troubleshooting and Debugging

## Exercise 1: Find and Fix YAML Syntax Error

**Objective:** Identify and fix YAML syntax problems that prevent workflow from starting.

**Broken Workflow:**

```yaml
name: Broken Syntax

on: push

jobs:
  test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Test step
        run: |
          echo "Testing"
          echo "More test
      
      - name Second step
        run: echo "Second"
```

**Problems to Find:**
1. Missing colon after job name
2. Unclosed quoted string
3. Missing colon in step name

**Requirements:**
- [ ] Identify all syntax errors
- [ ] Fix all errors
- [ ] Workflow should run without syntax warnings
- [ ] Both steps should execute

**Hint:** Copy into `.github/workflows/test.yml` and check Actions tab for syntax error message.

---

## Exercise 2: Debug Missing Environment Variable

**Objective:** Find and fix undefined variable that causes script failure.

**Starter Code:**

```yaml
name: Debug Variables

on: push

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Print variables
        run: |
          echo "API_URL: $API_URL"
          echo "API_KEY: $API_KEY"
          echo "Database: $DB_HOST"
          
          curl -H "Authorization: $API_KEY" $API_URL/health
```

**Requirements:**
- [ ] Add missing environment variables
- [ ] Set correct values
- [ ] Print variables before using them
- [ ] No "undefined variable" errors

**Hint:** Add `env:` section to job or step.

---

## Exercise 3: Fix Permission Denied Error

**Objective:** Resolve permission error when running script.

**Scenario:**

You have a deploy script `scripts/deploy.sh` but get error:
```
Permission denied: ./scripts/deploy.sh
```

**Task:**
- [ ] Create simple script that won't run
- [ ] Add workflow to execute it
- [ ] Fix permission error
- [ ] Script should execute successfully

**Starter Code:**

```yaml
name: Deploy

on: push

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run deploy script
        run: ./scripts/deploy.sh
```

**Requirements:**
- [ ] Identify permission issue
- [ ] Fix with `chmod` or `bash` prefix
- [ ] Script executes without error

**Hint:** Either make script executable or use `bash ./scripts/deploy.sh`

---

## Exercise 4: Debug Conditional Logic

**Objective:** Fix conditional that should run but doesn't (or vice versa).

**Broken Workflow:**

```yaml
name: Conditionals

on: push

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Should only run on main
        if: github.ref = 'refs/heads/main'
        run: echo "Running on main"
      
      - name: Should run on any branch
        if: github.event_name == 'push' || github.event_name == 'pull_request'
        run: echo "Running on push or PR"
```

**Problems:**
- [ ] Single `=` instead of `==`
- [ ] Verify conditionals work correctly

**Requirements:**
- [ ] Fix all conditionals
- [ ] Add debug output to show condition values
- [ ] Verify correct steps run on your branch

**Hint:** Use `echo "${{ github.ref }}"` to see actual value.

---

## Exercise 5: Resolve Secret Access Issue

**Objective:** Fix workflow that can't access secrets.

**Scenario:**

Workflow tries to use secret but fails silently (no output).

**Starter Code:**

```yaml
name: Secret Test

on: push

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Use secret
        run: |
          echo "Secret value: $SECRET_KEY"
          echo "Deploying with: ${{ secrets.API_TOKEN }}"
```

**Requirements:**
- [ ] Add secret in GitHub Settings
- [ ] Update workflow to use secret correctly
- [ ] Secret should not print in logs
- [ ] Use secret in command without exposing

**Validation:**
- [ ] Secret exists in Settings → Secrets → Actions
- [ ] Workflow accesses it without error
- [ ] Logs don't show actual secret value (GitHub masks it)

**Hint:** Secrets accessed as `${{ secrets.NAME }}` not `$SECRET_NAME`

---

## Exercise 6: Debug Failed Job Dependency

**Objective:** Fix job dependency that causes workflow failure.

**Broken Workflow:**

```yaml
name: Dependencies

on: push

jobs:
  build:
    needs: test  # ✗ test job doesn't exist!
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building..."

  test:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Testing..."
      
  deploy:
    needs: [build, compile]  # ✗ compile doesn't exist!
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying..."
```

**Requirements:**
- [ ] Identify non-existent job references
- [ ] Fix all job names
- [ ] Proper dependency order: test → build → deploy
- [ ] All jobs execute in correct order

**Validation:**
- [ ] No "depends on non-existent job" errors
- [ ] Jobs run in dependency order
- [ ] Correct jobs wait for previous ones

**Hint:** Job names in `needs:` must match exactly.

---

## Exercise 7: Fix Service Container Connection Error

**Objective:** Debug workflow that can't connect to database service.

**Broken Code:**

```yaml
name: Database Test

on: push

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        # Missing health check!
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Connect to database
        run: |
          psql -h localhost \
            -U postgres \
            -d testdb \
            -c "SELECT 1"
```

**Problems:**
- [ ] Missing environment variables for postgres
- [ ] No health check - connection attempts too early
- [ ] Missing database initialization

**Requirements:**
- [ ] Add POSTGRES_PASSWORD env var
- [ ] Add health check with retries
- [ ] Create test database
- [ ] Connection should succeed

**Validation:**
- [ ] No "connection refused" errors
- [ ] Database responds to query
- [ ] Step completes successfully

**Hint:** Service needs health check and proper env vars.

---

## Exercise 8: Debug Docker Build Failure

**Objective:** Find and fix Docker image build error.

**Broken Dockerfile:**

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
COPY src ./src

RUN npm ci --production

EXPOSE 3000

# ✗ Wrong command syntax
CMD "npm" "start"

# ✗ Missing build:
RUN npm run build

ENTRYPOINT ["node", "dist/index.js"]
```

**Issues:**
- [ ] Build step in wrong order
- [ ] CMD syntax error (should use JSON format or shell)
- [ ] Multiple ENTRYPOINT/CMD issues

**Requirements:**
- [ ] Create valid Dockerfile
- [ ] Build should complete without errors
- [ ] Image should run without container errors

**Workflow:**

```yaml
name: Build Image

on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build Docker image
        run: docker build -t myapp:latest .
```

**Validation:**
- [ ] Docker build succeeds
- [ ] No layers fail
- [ ] Image size reasonable
- [ ] All commands execute

**Hint:** Order matters: copy files, install deps, build, then specify command.

---

## Exercise 9: Troubleshoot Timeout and Performance Issue

**Objective:** Identify slow step and optimize it.

**Slow Workflow:**

```yaml
name: Slow Pipeline

on: push

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 10  # ✗ Too long!
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          # ✗ No caching
      
      - name: Install dependencies
        run: npm install  # ✗ Slow, should be ci
      
      - name: Run all tests
        run: npm test  # Takes 5+ minutes
      
      - name: Build large artifact
        run: npm run build
      
      - name: Upload everything
        uses: actions/upload-artifact@v3
        with:
          path: .  # ✗ Uploads entire repo!
```

**Requirements:**
- [ ] Add caching for npm dependencies
- [ ] Change npm install to npm ci
- [ ] Reduce artifact upload (only dist/)
- [ ] Set reasonable timeout (5 min)
- [ ] Total time should be <3 minutes

**Validation:**
- [ ] Workflow completes in 3 minutes
- [ ] Cache hits on second run
- [ ] Artifact size <100MB
- [ ] All steps complete before timeout

**Hint:** Check each step's duration in logs.

---

## Exercise 10: Complete Debugging Challenge

**Objective:** Fix a workflow with multiple issues.

**Broken Workflow:**

```yaml
name Complex Workflow

on push

jobs:
  quality
    runs-on: ubuntu-latest
    outputs:
      result=${{ job.status }}
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node
        with:
          node-version: 18
          cache npm
      
      - name Lint
        run: npm run lint

  test:
    needs quality
    if: github.ref == 'refs/heads/main  # ✗ Unclosed quote
    steps:
      - uses: actions/checkout@v3
      - run: echo $MISSING_VARIABLE | grep test
      - run: python3 nonexistent.py  # ✗ Will fail

  deploy:
    needs test
    if: success()
    environment: production
    steps:
      - name: Deploy
        run: ./deploy.sh  # ✗ Not executable
      
      - name: Health check
        run: curl $API_URL/health  # ✗ Undefined var
```

**Find and Fix:**
- [ ] YAML syntax errors (5 issues)
- [ ] Missing environment variables (2 issues)
- [ ] Permission issues (1 issue)
- [ ] Conditional logic issues (1 issue)
- [ ] Non-existent commands (1 issue)

**Complete Fixes Required:**
- All syntax errors fixed
- All required environment variables set
- All permissions correct
- All conditionals functional
- Workflow runs to completion

**Validation:**
- [ ] No YAML errors
- [ ] All jobs execute (or skip correctly)
- [ ] No "command not found" errors
- [ ] No "undefined variable" errors
- [ ] Output shows proper execution

**Hint:** Start with syntax errors, then runtime errors, then logic.

---

## Summary

You've practiced:
- ✅ Finding YAML syntax errors
- ✅ Debugging environment variables
- ✅ Fixing permission issues
- ✅ Understanding conditionals
- ✅ Working with secrets
- ✅ Managing job dependencies
- ✅ Troubleshooting services
- ✅ Debugging Docker builds
- ✅ Optimizing performance
- ✅ Systematic problem-solving

Continue debugging your workflows!
