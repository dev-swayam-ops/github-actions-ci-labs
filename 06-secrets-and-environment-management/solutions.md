# Module 06 Solutions: Secrets and Environment Management

## Exercise 1: Create and Access Repository Secrets

**Solution:**

**GitHub UI Setup:**
1. Navigate to Settings → Secrets and variables → Actions
2. Click "New repository secret"
3. Create these secrets:
   - Name: `NAME`, Value: `MyApplication`
   - Name: `API_KEY`, Value: `sk_live_abc123xyz789`
   - Name: `DB_PASSWORD`, Value: `SecureP@ssw0rd123!`

**Workflow:**
```yaml
name: Access Secrets

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Display Secrets
        env:
          MY_NAME: ${{ secrets.NAME }}
          API_KEY: ${{ secrets.API_KEY }}
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
        run: |
          echo "Name: $MY_NAME"
          echo "API Key: $API_KEY"
          echo "DB Password: $DB_PASSWORD"
```

**Explanation:**
- Secrets created in Settings UI
- Referenced via `${{ secrets.SECRET_NAME }}` syntax
- Passed to steps via `env:` section
- GitHub automatically masks values in logs (shows ***)
- Secret names are case-sensitive
- Only available when explicitly passed to env

**Log Output (safe):**
```
Name: MyApplication
API Key: ***
DB Password: ***
```

---

## Exercise 2: Environment-Specific Secrets

**Solution:**

**Step 1: Create Environments in UI**

Settings → Environments:
1. Click "New environment"
2. Create "development" environment
3. Create "production" environment with protection rules

**Step 2: Add Secrets to Environments**

For "development" environment:
- Create secret `API_ENDPOINT = https://api-dev.example.com`

For "production" environment:
- Create secret `API_ENDPOINT = https://api-prod.example.com`
- Enable "Required reviewers" (2+)
- Set "Deployment branches" to main only

**Workflow:**
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
          API_ENDPOINT: ${{ secrets.API_ENDPOINT }}
        run: |
          echo "Deploying to dev"
          echo "API: $API_ENDPOINT"

  deploy-prod:
    runs-on: ubuntu-latest
    environment: production
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to Prod
        env:
          API_ENDPOINT: ${{ secrets.API_ENDPOINT }}
        run: |
          echo "Deploying to prod"
          echo "API: $API_ENDPOINT"
```

**Explanation:**
- `environment: development` limits job to that environment's secrets
- Same secret name, different values per environment
- Production environment has protection rules
- Workflow pauses for approval before production deployment
- Environment secrets override repository secrets
- Useful for dev/staging/prod patterns

**Access Control:**
- Development: Available to all users
- Production: Requires approval from 2+ reviewers
- Main branch only: Other branches cannot deploy to prod

---

## Exercise 3: Masking Sensitive Output

**Solution:**

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
          echo "::add-mask::${SECRET_VALUE}"
          echo "Original secret: $SECRET_VALUE"
          echo "API key: ${SECRET_VALUE}"
```

**Explanation:**
- `::add-mask::VALUE` tells GitHub to mask that value
- Once masked, value appears as *** in all logs
- Must mask before using value in any output
- Helps prevent accidental secret exposure
- Can mask multiple values by calling multiple times

**Alternative with Secrets:**
```yaml
- name: Mask Secret Values
  run: |
    # GitHub automatically masks secret values
    echo "::add-mask::${{ secrets.API_KEY }}"
    echo "Deploying with API key"
    # API key appears as ***
```

**Common Masking Pattern:**
```yaml
steps:
  - name: Generate Tokens
    id: tokens
    run: |
      TOKEN=$(openssl rand -base64 32)
      echo "::add-mask::${TOKEN}"
      echo "token=${TOKEN}" >> $GITHUB_OUTPUT
      
  - name: Use Token
    run: |
      # Token is masked in output
      echo "Token: ${{ steps.tokens.outputs.token }}"
      # Output: Token: ***
```

---

## Exercise 4: Organization Secrets Access

**Solution:**

**Organizational Setup** (admin-only):
1. Go to Organization Settings → Secrets and variables → Actions
2. Click "New organization secret"
3. Create secret: `ORG_API_KEY = org_xyz_secret`
4. Under "Repository access", select target repositories
5. Choose "Selected repositories" and add your repo

**Workflow:**
```yaml
name: Org Secrets

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Use Org Secret
        env:
          ORG_API_KEY: ${{ secrets.ORG_API_KEY }}
        run: |
          echo "Org API Key available: $ORG_API_KEY"
          # Use for shared services across organization
```

**Explanation:**
- Organization secrets created at org level
- Available to selected member repositories
- Useful for shared services (npm registry, deploy tokens)
- Fewer secrets to manage across repos
- Organization admins control access

**Visibility Hierarchy:**
```
Organization Secrets (all orgs/selected repos)
├── Used in this repo
└── Also available to team-a/repo1, team-b/repo2

Repository Secrets (this repo only)
└── Only available in this repository

Environment Secrets (specific environment)
└── Only in this environment of this repository
```

---

## Exercise 5: Context Variables and Inputs

**Solution:**

```yaml
name: Context Variables

on: [push, pull_request]

env:
  GLOBAL_VAR: "global-value"
  LOG_LEVEL: "debug"

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [16, 18, 20]
        os: [ubuntu-latest, windows-latest]
    steps:
      - name: Display Contexts
        env:
          JOB_VAR: "job-value"
          NODE_ENV: "ci"
        run: |
          echo "=== GitHub Context ==="
          echo "Ref: ${{ github.ref }}"
          echo "Event: ${{ github.event_name }}"
          echo "Run ID: ${{ github.run_id }}"
          echo "Repository: ${{ github.repository }}"
          echo "Branch: ${{ github.ref_name }}"
          
          echo "=== Matrix Context ==="
          echo "Node: ${{ matrix.node }}"
          echo "OS: ${{ matrix.os }}"
          
          echo "=== Environment Variables ==="
          echo "Global: ${{ env.GLOBAL_VAR }}"
          echo "Job: ${{ env.JOB_VAR }}"
          echo "Log Level: ${{ env.LOG_LEVEL }}"
```

**Explanation:**
- `github.*` context: Workflow metadata (event, branch, run ID)
- `env.*` context: Environment variables
- `matrix.*` context: Current matrix combination values
- `secrets.*` context: Encrypted secrets
- Each context has different scope and visibility

**Common Variables:**
```yaml
${{ github.ref }}              # Full ref: refs/heads/main
${{ github.ref_name }}        # Branch name: main
${{ github.event_name }}      # Event type: push, pull_request
${{ github.run_id }}          # Unique run ID
${{ github.run_number }}      # Run number (shorter)
${{ github.actor }}           # User who triggered
${{ github.repository }}      # owner/repo format
${{ github.sha }}             # Commit SHA
```

---

## Exercise 6: Conditional Deployment with Environment

**Solution:**

```yaml
name: Conditional Deploy

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm ci && npm run build

  deploy-dev:
    needs: build
    runs-on: ubuntu-latest
    environment: development
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to Dev
        env:
          API_ENDPOINT: ${{ secrets.DEV_API_ENDPOINT }}
        run: echo "Deploying to development environment"

  deploy-prod:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to Production
        env:
          API_ENDPOINT: ${{ secrets.PROD_API_ENDPOINT }}
          DEPLOY_KEY: ${{ secrets.PROD_DEPLOY_KEY }}
        run: echo "Deploying to production environment"
```

**Explanation:**
- Dev deployment runs on all push events (all branches)
- Prod deployment conditional: `if: github.ref == 'refs/heads/main' && github.event_name == 'push'`
- Production environment configured with:
  - Required reviewers (2+)
  - Branch protection (main only)
  - URL for reference
- Different secrets per environment
- Approval required before prod job executes

**Conditional Logic:**
```yaml
# Main branch and push event only
if: github.ref == 'refs/heads/main' && github.event_name == 'push'

# Release tags only
if: startsWith(github.ref, 'refs/tags/')

# Pull requests only
if: github.event_name == 'pull_request'

# Skip on draft PR
if: !github.event.pull_request.draft

# Custom branch patterns
if: startsWith(github.ref, 'refs/heads/release/')
```

---

## Exercise 7: Build Matrix with Environment Variables

**Solution:**

```yaml
name: Matrix with Env

on: [push]

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [16, 18, 20]
    env:
      NODE_VERSION: ${{ matrix.node }}
      OS_TYPE: ${{ matrix.os }}
      BUILD_DIR: ./build
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      
      - name: Display Matrix Config
        run: |
          echo "Building on: $OS_TYPE"
          echo "Node version: $NODE_VERSION"
          echo "Build directory: $BUILD_DIR"
      
      - name: Build
        run: npm ci && npm run build
      
      - name: Test
        run: npm test
      
      - name: Upload Artifact
        uses: actions/upload-artifact@v3
        with:
          name: build-${{ matrix.os }}-${{ matrix.node }}
          path: ${{ env.BUILD_DIR }}/
```

**Explanation:**
- Matrix creates 9 jobs (3 OS × 3 Node versions)
- Each job gets unique matrix values
- Environment variables set based on matrix
- Useful for testing across configurations
- Artifact name includes matrix values for clarity

**Matrix Output:**
```
Total jobs created: 9
├── ubuntu-16
├── ubuntu-18
├── ubuntu-20
├── windows-16
├── windows-18
├── windows-20
├── macos-16
├── macos-18
└── macos-20
```

---

## Exercise 8: Secret Rotation Strategy

**Solution:**

**Documentation:**
```markdown
# Secret Rotation Strategy

## Secrets and Rotation Schedule

| Secret | Current | Rotation | Last Rotated |
|--------|---------|----------|--------------|
| NPM_TOKEN | npm_xxx | Monthly | 2026-01-01 |
| DEPLOY_KEY | ghp_xxx | Quarterly | 2025-10-01 |
| DB_PASSWORD | SecureXxx | Annually | 2025-01-01 |
| API_KEY | sk_live_xxx | Monthly | 2026-01-01 |

## Rotation Process

### 1. Generate New Secret
- Create new token in service (npm, GitHub, AWS, etc.)
- Verify new token works with test call

### 2. Create Staging Test
- Add new secret as temp in workflow
- Test with both old and new secret
- Verify no downtime

### 3. Update Production
- Add new secret to GitHub repository
- Update production environment if needed
- Deploy with new secret

### 4. Validation
- Monitor logs for any errors
- Verify functionality with new secret
- Check all systems still working

### 5. Cleanup
- Remove old secret from GitHub
- Revoke old token in service
- Update rotation log

## Calendar

January: NPM_TOKEN, API_KEY
February: NPM_TOKEN, API_KEY
...
April: DEPLOY_KEY (quarterly)
January: DB_PASSWORD (annually)
```

**Rotation Workflow:**
```yaml
name: Secret Rotation Check

on:
  schedule:
    # Run on 1st of each month at 9am UTC
    - cron: '0 9 1 * *'
  workflow_dispatch:

jobs:
  rotation-reminder:
    runs-on: ubuntu-latest
    steps:
      - name: Check Monthly Rotations
        run: |
          echo "Secrets due for rotation:"
          echo "- NPM_TOKEN (monthly)"
          echo "- API_KEY (monthly)"
          
          # In April, also remind about quarterly
          if [ $(date +%m) -eq 4 ]; then
            echo "- DEPLOY_KEY (quarterly)"
          fi
```

---

## Exercise 9: Secure Artifact Handling

**Solution:**

```yaml
name: Secure Artifacts

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Build Application
        run: npm ci && npm run build
      
      - name: Remove Secrets from Build
        run: |
          # Ensure no .env files in dist/
          find dist/ -name ".env*" -delete
          find dist/ -name "*.log" -delete
          echo "Cleaned sensitive files"
      
      - name: Upload Build Artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-${{ github.run_id }}
          path: dist/
          retention-days: 3
          # Short retention for security
      
      - name: Upload to Secure Storage
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
        run: |
          echo "Uploading to secure storage"
          # Token masked in logs automatically
```

**Explanation:**
- Artifacts encrypted at rest by GitHub
- Retention set to 3 days (short lifecycle)
- Sensitive files excluded from artifacts
- Secrets never included in build output
- Access controlled via repository permissions

**Artifact Security Checklist:**
```
✓ No secrets in artifact contents
✓ Retention set appropriately (3-7 days typical)
✓ Archive includes only necessary files
✓ Access limited to repo members
✓ Automatic cleanup after retention
✓ Version control excludes .env files
```

---

## Exercise 10: Environment Protection and Approvals

**Solution:**

**GitHub UI Configuration:**

1. Go to Settings → Environments
2. Click "New environment"
3. Name: "production"
4. Configure protection rules:
   - ✓ Required reviewers: Enable
   - Number of reviewers: 2
   - Restrict who can approve:
     - Add specific users/teams
   - Deployment branches:
     - ✓ Selected branches
     - Branch: main

**Test Workflow:**
```yaml
name: Test Protection

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm ci && npm run build

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/checkout@v3
      
      - name: Pre-deployment Checks
        run: |
          echo "Running pre-deployment checks..."
          echo "✓ Build verified"
          echo "✓ Tests passed"
      
      - name: Deploy to Production
        env:
          DEPLOY_TOKEN: ${{ secrets.PROD_DEPLOY_TOKEN }}
        run: |
          echo "Deploying to production"
          # Deployment commands here
      
      - name: Post-deployment Verification
        run: |
          echo "Verifying deployment..."
          echo "✓ Health checks passed"
          echo "✓ Smoke tests passed"
```

**Approval Workflow:**
1. Push to main triggers workflow
2. Build job completes successfully
3. Deploy job **pauses** awaiting approval
4. In workflow UI, "Review deployments" appears
5. 2+ approvers must review and approve
6. Deployment proceeds automatically

**Protection Benefits:**
- Prevents accidental production deployments
- Ensures code review before production
- Creates audit trail of approvals
- Can restrict approvers to specific users
- Works with branch protections

---

## Summary

**Mastery Checkpoint:**

You now understand:
- ✅ Repository, organization, and environment secrets
- ✅ Secret access and scoping
- ✅ Automatic masking and manual masking
- ✅ Context variables and their scopes
- ✅ Conditional deployments
- ✅ Environment protection rules
- ✅ Approval workflows
- ✅ Secret rotation strategies
- ✅ Secure artifact handling
- ✅ Access control and governance

**Advanced Topics:**
- OIDC token provider for keyless auth
- GitHub Advanced Security scanning
- Custom masking patterns
- Compliance logging and auditing

Continue practicing and secure your pipelines!
