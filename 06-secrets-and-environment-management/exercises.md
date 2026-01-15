# Module 06 Exercises: Secrets and Environment Management

## Exercise 1: Create and Access Repository Secrets

**Objective:** Create repository-level secrets and access them in workflows

**Acceptance Criteria:**
- 3 repository secrets created (NAME, API_KEY, DB_PASSWORD)
- All secrets accessible in workflow via `${{ secrets.SECRET_NAME }}`
- Secret values not visible in job logs
- Correct environment variable syntax used

**Starter Code:**
```yaml
name: Access Secrets

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Display Secrets
        env:
          # Add secret references
          MY_NAME: 
          API_KEY: 
          DB_PASSWORD: 
        run: |
          echo "Name: $MY_NAME"
          echo "API Key: $API_KEY"
          echo "DB Password: $DB_PASSWORD"
```

**Setup Instructions:**
1. Go to Settings → Secrets and variables → Actions
2. Create: `NAME=MyApp`, `API_KEY=sk_live_xyz`, `DB_PASSWORD=SecurePass123`
3. Verify secrets are masked in logs

---

## Exercise 2: Environment-Specific Secrets

**Objective:** Use different secrets for development and production environments

**Acceptance Criteria:**
- Development environment has dev-specific secrets
- Production environment has prod-specific secrets
- Jobs specify correct environment
- Correct secrets accessible per environment
- Different secrets used based on environment

**Starter Code:**
```yaml
name: Environment Secrets

on: [push]

jobs:
  deploy-dev:
    runs-on: ubuntu-latest
    environment: development
    steps:
      - name: Deploy to Dev
        env:
          # Reference development secret
          API_ENDPOINT: 
        run: echo "Deploying to dev: $API_ENDPOINT"

  deploy-prod:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy to Prod
        env:
          # Reference production secret
          API_ENDPOINT: 
        run: echo "Deploying to prod: $API_ENDPOINT"
```

**Setup Instructions:**
1. Create environment "development" in Settings → Environments
2. Create environment "production" in Settings → Environments
3. Add DEV_API_ENDPOINT to development environment
4. Add PROD_API_ENDPOINT to production environment

---

## Exercise 3: Masking Sensitive Output

**Objective:** Prevent sensitive values from appearing in logs

**Acceptance Criteria:**
- Custom secret value masked in logs
- Uses `::add-mask::` command
- Secret not visible even if echoed
- Masking happens before output

**Starter Code:**
```yaml
name: Mask Sensitive Output

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Generate and Mask Secret
        run: |
          SECRET_VALUE="super-secret-123"
          # Add masking command
          echo "Original secret: $SECRET_VALUE"
```

**Expected Output:**
```
Original secret: ***
```

---

## Exercise 4: Organization Secrets Access

**Objective:** Use organization-level secrets in repository workflow

**Acceptance Criteria:**
- Organization secret created
- Secret accessible in repository workflow
- Repository has permission to access org secret
- Scope correctly limited

**Starter Code:**
```yaml
name: Org Secrets

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Use Org Secret
        env:
          # Reference organization secret
          ORG_API_KEY: 
        run: echo "Using org API key"
```

**Note:** Requires organization account. If not available, skip this exercise.

---

## Exercise 5: Context Variables and Inputs

**Objective:** Access and use context variables in workflows

**Acceptance Criteria:**
- Accesses github context (ref, event_name, run_id)
- Uses env context for custom variables
- Uses matrix context in matrix jobs
- Variables correctly interpolated

**Starter Code:**
```yaml
name: Context Variables

on: [push, pull_request]

env:
  GLOBAL_VAR: "global-value"

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [16, 18, 20]
    steps:
      - name: Display Contexts
        env:
          JOB_VAR: "job-value"
        run: |
          echo "GitHub Ref: ${{ github.ref }}"
          echo "Event Name: ${{ github.event_name }}"
          echo "Run ID: ${{ github.run_id }}"
          echo "Node Version: ${{ matrix.node }}"
          echo "Global: ${{ env.GLOBAL_VAR }}"
          echo "Job: ${{ env.JOB_VAR }}"
```

**Expected Output:**
All context variables printed without errors

---

## Exercise 6: Conditional Deployment with Environment

**Objective:** Deploy only to production on main branch with appropriate approvals

**Acceptance Criteria:**
- Dev deployment automatic on all branches
- Production deployment conditional (main + push only)
- Production requires approval
- Different secrets per environment
- Correct branch checks

**Starter Code:**
```yaml
name: Conditional Deploy

on: [push, pull_request]

jobs:
  deploy-dev:
    runs-on: ubuntu-latest
    environment: development
    steps:
      - run: echo "Deploying to development"

  deploy-prod:
    runs-on: ubuntu-latest
    environment: production
    # Add condition: only main branch, push events
    if: 
    steps:
      - run: echo "Deploying to production"
```

**Condition:** `github.ref == 'refs/heads/main' && github.event_name == 'push'`

---

## Exercise 7: Build Matrix with Environment Variables

**Objective:** Use environment variables in matrix builds

**Acceptance Criteria:**
- Matrix strategy with multiple dimensions
- Environment variables defined per matrix
- Variables accessible in steps
- Different configurations per matrix combination

**Starter Code:**
```yaml
name: Matrix with Env

on: [push]

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: [16, 18, 20]
    env:
      NODE_VERSION: ${{ matrix.node }}
      OS_TYPE: ${{ matrix.os }}
    steps:
      - name: Build
        run: echo "Building on $OS_TYPE with Node $NODE_VERSION"
```

**Expected:** 6 jobs with different environment configurations

---

## Exercise 8: Secret Rotation Strategy

**Objective:** Plan and implement secret rotation

**Acceptance Criteria:**
- Document rotation schedule
- Identify secrets needing rotation
- Create plan for updating without downtime
- Verify old and new secrets work
- Remove old secret after transition

**Documentation Template:**
```markdown
# Secret Rotation Plan

## Secrets to Rotate
- NPM_TOKEN (monthly)
- DEPLOY_TOKEN (quarterly)
- DATABASE_PASSWORD (annually)

## Rotation Process
1. Generate new secret
2. Create new workflow with new secret
3. Test new secret in staging
4. Update production
5. Remove old secret
6. Document rotation date
```

---

## Exercise 9: Secure Artifact Handling

**Objective:** Manage artifacts containing sensitive information

**Acceptance Criteria:**
- Build artifacts uploaded with encryption
- Retention period set appropriately
- Secrets not included in artifacts
- Access controls enforced
- Cleanup policy documented

**Starter Code:**
```yaml
name: Secure Artifacts

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm ci && npm run build
      - name: Upload Artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-secure
          path: dist/
          # Add retention period (keep short)
          retention-days: 
          # Note: Never include secrets in artifacts
```

**Expected:** Artifacts encrypted at rest, retention set to 3-7 days

---

## Exercise 10: Environment Protection and Approvals

**Objective:** Configure environment protection rules and approvals

**Acceptance Criteria:**
- Environment created with protection rules
- Required reviewers configured (2+ for production)
- Approval workflow tested
- Only authorized users can approve
- Deployment blocked without approval

**Setup Instructions:**
1. Go to Settings → Environments → production
2. Enable "Deployment branches and environments"
3. Set "Selected branches" to main
4. Enable "Required reviewers" with 2+ reviewers
5. Add users to reviewers list

**Test Workflow:**
```yaml
name: Test Protection

on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - run: echo "This requires approval to run"
```

**Test:** Push to main and verify approval prompt appears in workflow

---

## Summary

**Key Learning Points:**
- ✅ Creating and accessing repository secrets
- ✅ Environment-specific secrets and configurations
- ✅ Secret masking and log protection
- ✅ Organization-level secret management
- ✅ Context variables and their scopes
- ✅ Conditional deployments with approvals
- ✅ Environment protection rules
- ✅ Secret rotation strategies
- ✅ Secure artifact handling
- ✅ Access control and governance

**Next Steps:**
- Implement secret rotation automation
- Use OIDC for keyless authentication
- Enable advanced security scanning
- Configure compliance logging

**Continue to Exercise Solutions for complete implementations!**
