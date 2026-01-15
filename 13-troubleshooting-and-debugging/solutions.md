# Module 13 Solutions: Troubleshooting and Debugging

## Exercise 1 Solution: Find and Fix YAML Syntax Error

**Issues Found:**

1. **Line 7:** Missing colon after `test`
2. **Line 12:** Unclosed quote in `echo "More test`
3. **Line 15:** Missing colon after step name

**Fixed Workflow:**

```yaml
name: Broken Syntax

on: push

jobs:
  test:                    # ✓ Added colon
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Test step
        run: |
          echo "Testing"
          echo "More test"  # ✓ Closed quote
      
      - name: Second step  # ✓ Added colon
        run: echo "Second"
```

**Validation:**
```
✓ No syntax errors in Actions tab
✓ Workflow runs successfully
✓ Both steps execute
```

**Learning:** YAML requires colons after keys, proper quoting, and consistent indentation.

---

## Exercise 2 Solution: Debug Missing Environment Variable

**Complete Solution:**

```yaml
name: Debug Variables

on: push

jobs:
  test:
    runs-on: ubuntu-latest
    env:              # ✓ Add environment variables
      API_URL: https://api.example.com
      API_KEY: ${{ secrets.API_KEY }}  # ✓ Use secret
      DB_HOST: localhost
    
    steps:
      - name: Print variables
        run: |
          echo "=== Checking Variables ==="
          echo "API_URL: $API_URL"
          echo "API_KEY: ${API_KEY:-(not set)}"  # Safe print
          echo "Database: $DB_HOST"
          
          # Verify they're set before using
          [ -z "$API_URL" ] && echo "ERROR: API_URL not set" && exit 1
          [ -z "$API_KEY" ] && echo "ERROR: API_KEY not set" && exit 1
          
          # Now safe to use
          curl -H "Authorization: Bearer $API_KEY" \
            "$API_URL/health" || true
```

**Setup Secret:**

1. Go to Settings → Secrets and Variables → Actions
2. New repository secret:
   - Name: `API_KEY`
   - Value: `test-key-12345`
3. Re-run workflow

**Output:**
```
=== Checking Variables ===
API_URL: https://api.example.com
API_KEY: *** (masked by GitHub)
Database: localhost
```

**Key Learning:** 
- Use `env:` block to define variables
- Use `${{ secrets.NAME }}` for sensitive data
- Validate variables exist before using them

---

## Exercise 3 Solution: Fix Permission Denied Error

**Step 1: Create the Script**

Create `scripts/deploy.sh`:

```bash
#!/bin/bash
# Simple deploy script

echo "Starting deployment..."
echo "Current directory: $(pwd)"
echo "User: $(whoami)"
echo "✓ Deployment complete"
```

**Step 2: Make Executable**

```bash
chmod +x scripts/deploy.sh
git add scripts/deploy.sh
git commit -m "Add deploy script"
```

**Step 3: Fixed Workflow**

**Option A: Execute with bash**

```yaml
name: Deploy

on: push

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run deploy script
        run: bash ./scripts/deploy.sh  # ✓ Use bash prefix
```

**Option B: Make script executable in workflow**

```yaml
- name: Run deploy script
  run: |
    chmod +x ./scripts/deploy.sh
    ./scripts/deploy.sh  # ✓ Now executable
```

**Output:**
```
Starting deployment...
Current directory: /home/runner/work/myrepo/myrepo
User: runner
✓ Deployment complete
```

**Key Learning:** Scripts need execute permission or be run with bash prefix.

---

## Exercise 4 Solution: Debug Conditional Logic

**Issues Fixed:**

1. **Line 10:** Single `=` changed to `==`
2. **Line 13:** Syntax correct but needs debugging

**Fixed Workflow:**

```yaml
name: Conditionals

on: push

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Debug conditions
        run: |
          echo "=== Current Context ==="
          echo "Branch: ${{ github.ref }}"
          echo "Branch name: ${{ github.ref_name }}"
          echo "Event: ${{ github.event_name }}"
          echo ""
          echo "=== Condition Evaluations ==="
          echo "Is main? ${{ github.ref == 'refs/heads/main' }}"
          echo "Is push? ${{ github.event_name == 'push' }}"
      
      - name: Should only run on main  # ✓ Fixed ==
        if: github.ref == 'refs/heads/main'
        run: echo "Running on main branch"
      
      - name: Should run on any branch
        if: github.event_name == 'push' || github.event_name == 'pull_request'
        run: echo "Running on push or PR"
      
      - name: Only on develop
        if: github.ref == 'refs/heads/develop'
        run: echo "Running on develop"
      
      - name: Skip on PRs
        if: github.event_name != 'pull_request'
        run: echo "Not running on PR"
```

**Sample Output (on main branch):**
```
=== Current Context ===
Branch: refs/heads/main
Branch name: main
Event: push

=== Condition Evaluations ===
Is main? true
Is push? true

Running on main branch
Running on push or PR
Not running on PR
```

**Debugging Tips:**
- Always echo context variables first
- Use `==` not `=` for comparison
- Use `!=` for inequality
- Use `||` for OR, `&&` for AND
- Test with different branch names

**Key Learning:** Use `echo` to verify condition values before relying on them.

---

## Exercise 5 Solution: Resolve Secret Access Issue

**Setup Secret:**

```bash
# In GitHub Settings → Secrets and Variables → Actions
# New repository secret:
NAME: API_TOKEN
VALUE: sk_test_abc123xyz789
```

**Fixed Workflow:**

```yaml
name: Secret Test

on: push

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Verify secret exists
        run: |
          if [ -z "${{ secrets.API_TOKEN }}" ]; then
            echo "ERROR: API_TOKEN secret not found"
            exit 1
          fi
          echo "✓ API_TOKEN secret is available"
      
      - name: Use secret safely
        env:
          TOKEN: ${{ secrets.API_TOKEN }}
        run: |
          # Use in environment variable (not printed)
          echo "✓ Using token in secure way"
          
          # Simulate API call
          curl -H "Authorization: Bearer ${TOKEN}" \
            https://api.example.com/status || true
      
      - name: Deploy with secret
        run: |
          # Pass secret as argument (won't print due to GitHub masking)
          ./deploy.sh "${{ secrets.API_TOKEN }}"
      
      - name: ❌ Don't do this
        run: echo "Secret is: ${{ secrets.API_TOKEN }}"  # Bad - but GitHub masks it
```

**GitHub Masking:**
Even if you accidentally print a secret, GitHub automatically masks it:
```
Deployed with: ***
```

**Output:**
```
✓ API_TOKEN secret is available
✓ Using token in secure way
[GitHub masks secret value in logs]
```

**Key Learning:**
- Secrets stored in Settings, not in code
- Access with `${{ secrets.NAME }}`
- GitHub masks secrets in logs automatically
- Use secrets in environment variables or command arguments

---

## Exercise 6 Solution: Fix Service Container Connection Error

**Corrected Job Order:**

```yaml
name: Database Test

on: push

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_PASSWORD: testpass   # ✓ Required
          POSTGRES_DB: testdb           # ✓ Create test DB
          POSTGRES_USER: testuser       # ✓ Create user
        
        # ✓ Critical: Health check
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Wait for database
        run: |
          echo "Waiting for PostgreSQL..."
          max_attempts=10
          attempt=1
          
          while [ $attempt -le $max_attempts ]; do
            if pg_isready -h postgres -p 5432 > /dev/null 2>&1; then
              echo "✓ PostgreSQL is ready"
              break
            fi
            echo "  Attempt $attempt/$max_attempts..."
            sleep 1
            attempt=$((attempt + 1))
          done
      
      - name: Connect and query
        run: |
          psql -h postgres \
            -U testuser \
            -d testdb \
            -c "SELECT 1 as connection_test;"
        env:
          PGPASSWORD: testpass
      
      - name: Create test data
        run: |
          psql -h postgres \
            -U testuser \
            -d testdb \
            -c "CREATE TABLE IF NOT EXISTS users (id INT, name VARCHAR(50));"
          
          psql -h postgres \
            -U testuser \
            -d testdb \
            -c "INSERT INTO users VALUES (1, 'Test User');"
      
      - name: Query test data
        run: |
          psql -h postgres \
            -U testuser \
            -d testdb \
            -c "SELECT * FROM users;"
        env:
          PGPASSWORD: testpass
```

**Output:**
```
Waiting for PostgreSQL...
✓ PostgreSQL is ready
 connection_test
─────────────────
              1
(1 row)

CREATE TABLE
INSERT 0 1

 id |   name
────┼───────────
  1 | Test User
(1 row)
```

**Key Issues Fixed:**
1. ✓ Added environment variables (POSTGRES_PASSWORD, POSTGRES_DB)
2. ✓ Added health check to wait for service
3. ✓ Service accessible via hostname `postgres`
4. ✓ Create database before querying
5. ✓ Pass password via PGPASSWORD

**Key Learning:** Services need health checks to ensure they're ready before tests run.

---

## Exercise 7 Solution: Debug Docker Build Failure

**Fixed Dockerfile:**

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files first (leverages layer caching)
COPY package*.json ./

# Install production dependencies
RUN npm ci --production

# Copy source code
COPY src ./src

# Build the application (must be after npm ci)
RUN npm run build

# Expose port
EXPOSE 3000

# Set startup command (JSON format is correct)
CMD ["node", "dist/index.js"]
```

**Working Workflow:**

```yaml
name: Build Image

on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Build Docker image
        run: |
          docker build -t myapp:latest .
          echo "✓ Image built successfully"
      
      - name: Verify image
        run: |
          docker images myapp:latest
          docker inspect myapp:latest | jq .[0].ContainerConfig.Cmd
      
      - name: Test image (optional)
        run: |
          docker run --rm myapp:latest --version || echo "✓ Image runnable"
```

**Output:**
```
[+] Building 25.3s (10/10) FINISHED
 => [1/10] FROM node:18-alpine@sha256:...
 => [2/10] WORKDIR /app
 => [3/10] COPY package*.json ./
 => [4/10] RUN npm ci --production
 => [5/10] COPY src ./src
 => [6/10] RUN npm run build
 => [7/10] EXPOSE 3000
 => [8/10] CMD ["node", "dist/index.js"]
 => exporting to docker image format
 => => writing image sha256:abc123...

✓ Image built successfully
```

**Issues Fixed:**
1. **Order matters:** Copy files → install deps → build → expose → command
2. **CMD syntax:** Use JSON format `["node", "dist/index.js"]`
3. **No ENTRYPOINT:** Only need CMD (unless overriding)
4. **Build before expose:** Build step needs to happen before EXPOSE

**Key Learning:** Dockerfile layer order affects build speed and cache effectiveness.

---

## Exercise 8 Solution: Troubleshoot Timeout and Performance Issue

**Optimized Workflow:**

```yaml
name: Optimized Pipeline

on: push

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 5  # ✓ Reasonable timeout
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'  # ✓ Enable caching
      
      - name: Install dependencies
        run: npm ci  # ✓ Use ci instead of install
      
      - name: Run tests
        run: npm test  # Faster with caching
      
      - name: Build
        run: npm run build
      
      - name: Upload minimal artifact
        uses: actions/upload-artifact@v3
        with:
          name: dist
          path: dist/        # ✓ Only dist, not entire repo
          retention-days: 7  # ✓ Auto cleanup
```

**Performance Improvements:**

| Before | After | Savings |
|--------|-------|---------|
| npm install | npm ci | 5-10 sec |
| No cache | With cache | 20-25 sec (2nd run) |
| 10 min timeout | 5 min timeout | More aggressive |
| Upload entire repo | Upload dist only | 80-90% smaller |
| No retention | 7-day retention | Auto cleanup |

**Expected Times:**
- First run: 3-4 minutes
- Second run: 1-2 minutes (cache hit)
- Total workflow: <5 minutes

**Output:**
```
Install dependencies (npm ci)
  ✓ restored npm cache from key: Linux-node-18-npm-a1b2c3d
  up to date, audited 156 packages in 2.1s

Run tests
  Ran 23 tests in 15s

Build
  ✓ Built successfully

Upload minimal artifact
  ✓ Uploaded 5.2 MB (dist only)
  └─ 97% smaller than full repo
```

**Key Learning:** Caching and minimizing artifact uploads save 50%+ time.

---

## Exercise 9 Solution: Complete Debugging Challenge

**All Issues Fixed:**

```yaml
name: Complex Workflow  # ✓ Fixed: missing colon

on: push  # ✓ Fixed: missing colon

jobs:
  quality:  # ✓ Fixed: missing colon
    runs-on: ubuntu-latest
    
    outputs:
      result: ${{ job.status }}  # ✓ Fixed: = to :
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3  # ✓ Added @v3
        with:
          node-version: '18'
          cache: 'npm'  # ✓ Fixed: removed quotes
      
      - name: Lint  # ✓ Added colon
        run: npm run lint

  test:
    needs: quality
    if: github.ref == 'refs/heads/main'  # ✓ Closed quote
    runs-on: ubuntu-latest  # ✓ Added missing property
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install and lint
        run: |
          npm ci
          npm run lint
      
      - name: Run tests
        run: npm test
        env:
          # ✓ Define MISSING_VARIABLE
          MISSING_VARIABLE: test_value

  deploy:
    needs: test
    if: success()
    environment: production
    runs-on: ubuntu-latest  # ✓ Added missing property
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy
        run: bash ./deploy.sh  # ✓ Use bash prefix
        env:
          # ✓ Define API_URL
          API_URL: https://api.example.com
      
      - name: Health check
        run: curl -f "$API_URL/health" || echo "Health check completed"
        env:
          API_URL: https://api.example.com
```

**Issues Fixed Summary:**

| Issue | Line | Fix | Type |
|-------|------|-----|------|
| Missing colon after `name` | 1 | Added `:` | Syntax |
| Missing colon after `on` | 3 | Added `:` | Syntax |
| Missing colon after `quality` | 5 | Added `:` | Syntax |
| Wrong operator `=` | 10 | Changed to `:` | Syntax |
| Unclosed quote | 21 | Added `'` | Syntax |
| Missing `runs-on` | 26 | Added property | Configuration |
| Missing `runs-on` | 41 | Added property | Configuration |
| Script not executable | 54 | Used `bash` prefix | Runtime |
| Undefined variable | 18 | Added `env:` section | Runtime |
| Undefined variable | 59 | Added `env:` section | Runtime |

**Output:**
```
✓ Workflow syntax valid
✓ All jobs execute in order
✓ Quality job passes
✓ Test job passes
✓ Deploy job executes
✓ Health check completes
```

**Key Learning:** Fix syntax errors first, then runtime, then logic.

---

## Quick Debugging Checklist

**When workflow fails:**

1. ✓ Check Actions tab for syntax errors (red X)
2. ✓ Click failed job to see error message
3. ✓ Find the exact step that failed
4. ✓ Read error output carefully
5. ✓ Add echo statements to debug
6. ✓ Check environment variables exist
7. ✓ Verify job dependencies
8. ✓ Test conditions locally
9. ✓ Re-run workflow after fix
10. ✓ Confirm success before moving on

---

## Summary

You've learned:
✅ Fix YAML syntax errors
✅ Debug missing variables
✅ Resolve permission issues
✅ Fix conditionals
✅ Access secrets properly
✅ Fix job dependencies
✅ Troubleshoot services
✅ Debug Docker builds
✅ Optimize performance
✅ Systematic debugging approach

Keep debugging! 🐛🔍
