# 06-secrets-and-environment-management

## Overview

This module teaches you how to securely manage secrets, API keys, and environment variables in GitHub Actions workflows. You'll master secret encryption, environment management, and security best practices to protect sensitive data.

---

## What You'll Learn

- Understanding GitHub secrets and their encryption
- Creating and managing repository secrets
- Environment-level secrets and variables
- Using organization-level secrets
- Masking sensitive output
- Secret rotation strategies
- Environment configuration patterns
- Context variables and their scope
- Security best practices for CI/CD
- Detecting and preventing secret leaks

---

## Prerequisites

- Completion of **05-reusable-workflows-and-composite-actions** module
- Understanding of workflow syntax
- Knowledge of environment variables
- GitHub repository with admin access
- Familiarity with encryption concepts

---

## Key Concepts

### 1. **Repository Secrets**
Encrypted key-value pairs stored at repository level, accessible to all workflows.

### 2. **Organization Secrets**
Shared secrets available across multiple repositories in an organization.

### 3. **Environment Secrets**
Secrets scoped to specific environments (dev, staging, production).

### 4. **Secret Encryption**
GitHub encrypts secrets at rest using Libsodium public-key cryptography.

### 5. **Secret Masking**
GitHub automatically masks secret values in logs to prevent accidental exposure.

### 6. **Context Variables**
Dynamic values available during workflow execution (github, env, secrets contexts).

### 7. **Environment Variables**
Unencrypted configuration values (API endpoints, feature flags, versions).

### 8. **Masked Output**
Preventing secrets from appearing in workflow logs using `::add-mask::`

### 9. **Protection Rules**
Requirements like approval or status checks before deployment to environment.

### 10. **Secret Rotation**
Strategy for regularly updating secrets to maintain security.

---

## Hands-on Lab: Secure Build and Deploy Pipeline

### Step 1: Create Repository Secrets

Go to Settings → Secrets and variables → Actions, add these secrets:

- Name: `NPM_TOKEN` | Value: `npm_xxxxxxxxxxxxxxxxxxxxx`
- Name: `DEPLOY_TOKEN` | Value: `ghp_xxxxxxxxxxxxxxxxxxxxx`
- Name: `DATABASE_PASSWORD` | Value: `SecurePassword123!`

### Step 2: Create Workflow with Secrets

Create `.github/workflows/secure-pipeline.yml`:

```yaml
name: Secure Build and Deploy

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    environment: development
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install Dependencies
        env:
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
        run: |
          npm ci
      
      - name: Build Application
        run: npm run build
      
      - name: Upload Artifact
        uses: actions/upload-artifact@v3
        with:
          name: build-${{ github.run_id }}
          path: dist/

  test:
    needs: build
    runs-on: ubuntu-latest
    environment: development
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci && npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Production
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
          DB_PASSWORD: ${{ secrets.DATABASE_PASSWORD }}
        run: |
          echo "Deploying with secured credentials"
          # Deployment command would use $DEPLOY_TOKEN and $DB_PASSWORD
      
      - name: Notify Deployment
        if: success()
        run: echo "✓ Deployment successful"
```

### Step 3: Create Environment Configuration

Create `.github/workflows/env-config.yml`:

```yaml
name: Environment Configuration

on: [push]

env:
  API_ENDPOINT: https://api.example.com
  LOG_LEVEL: info

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - name: Display Environment Info
        env:
          DB_HOST: localhost
          DB_PORT: 5432
        run: |
          echo "API Endpoint: ${{ env.API_ENDPOINT }}"
          echo "Log Level: ${{ env.LOG_LEVEL }}"
          echo "Database: ${{ env.DB_HOST }}:${{ env.DB_PORT }}"
```

### Step 4: Commit and Push

```bash
git add .github/workflows/
git commit -m "Add secure pipeline with secrets management"
git push origin main
```

### Step 5: Monitor Execution

**Expected Output:**

```
Workflow: Secure Build and Deploy

├── build (environment: development)
│   ├── Checkout ✓
│   ├── Setup Node ✓
│   ├── Install Dependencies ✓
│   │   └── NPM_TOKEN masked in logs
│   ├── Build Application ✓
│   └── Upload Artifact ✓
│
├── test (environment: development)
│   ├── Checkout ✓
│   ├── Setup Node ✓
│   └── Run Tests ✓
│
└── deploy (environment: production)
    ├── Requires approval ✓
    ├── Deploy to Production ✓
    │   └── DEPLOY_TOKEN & DB_PASSWORD masked
    └── Notify Deployment ✓
```

**Log Output:**
```
Installing Dependencies
npm ci
✓ Installed successfully
(NPM_TOKEN not visible in logs - masked)

Deploy to Production
Deploying with secured credentials
✓ Deployment successful
(DEPLOY_TOKEN and DB_PASSWORD not visible - masked)
```

---

## Validation Checklist

- [ ] Repository secrets created successfully
- [ ] Secrets accessible in workflows via ${{ secrets.SECRET_NAME }}
- [ ] Secret values masked in job logs
- [ ] Environments configured correctly
- [ ] Environment-specific secrets available only in that environment
- [ ] Deployment requires appropriate approvals
- [ ] Environment variables separate from secrets
- [ ] No secrets printed or logged accidentally

---

## Cleanup

```bash
# Delete secrets from Settings (UI only)
# Remove workflow files
rm .github/workflows/secure-pipeline.yml
rm .github/workflows/env-config.yml

# Commit cleanup
git add .github/workflows/
git commit -m "Remove secure pipeline workflows"
git push origin main
```

---

## Common Mistakes

### ❌ Mistake 1: Hardcoding Secrets in Workflows

```yaml
# Wrong - secret in plain text
env:
  NPM_TOKEN: npm_xxxxxxxxxxxxx

# Correct - use secrets
env:
  NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### ❌ Mistake 2: Printing Secrets to Logs

```yaml
# Wrong - secrets visible in logs
- run: echo "Token: ${{ secrets.NPM_TOKEN }}"

# Correct - use masked output
- run: npm ci
  env:
    NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### ❌ Mistake 3: Incorrect Secret Scope

```yaml
# Wrong - using repo secret in different org
jobs:
  build:
    steps:
      - run: echo ${{ secrets.NPM_TOKEN }}

# Correct - use org-level secrets
jobs:
  build:
    steps:
      - run: echo ${{ secrets.ORG_NPM_TOKEN }}
```

### ❌ Mistake 4: Missing Environment Configuration

```yaml
# Wrong - no environment specified
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: deploy.sh

# Correct - specify environment
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: deploy.sh
```

### ❌ Mistake 5: Not Rotating Secrets

```yaml
# Wrong - same secret used for years
NPM_TOKEN: same_token_since_2020

# Correct - rotate regularly
# Update secret every 90 days
# Revoke old token before updating
```

---

## Troubleshooting

### Issue: Secret not available in workflow

**Solution:**
- Verify secret name matches exactly (case-sensitive)
- Check secret exists in correct scope (repo, org, environment)
- Ensure workflow has access to secrets
- Verify repository is not public (some restrictions apply)

### Issue: Secret visible in logs

**Solution:**
- Check if secret is being echoed directly
- Verify secret is passed via `env:` not `run:`
- Use `jobs.<job_id>.outputs[*].value` for sensitive outputs
- Enable log masking: `::add-mask::`

### Issue: Deployment approval not triggering

**Solution:**
- Verify environment is configured in Settings
- Check protection rules are enabled
- Ensure required reviewers are team members
- Verify job uses `environment: name` correctly

### Issue: Environment secrets not accessible

**Solution:**
- Verify secret is set in environment (not just repo)
- Check environment-level protection rules
- Ensure job specifies correct environment name
- Environment secrets override repository secrets

### Issue: Organization secret not working

**Solution:**
- Verify secret is created at organization level
- Check repository has access (not all repos excluded)
- Verify secret name matches in workflow
- Organization admins only can create org secrets

---

## Next Steps

✅ **You've completed module 06!**

**Congratulations! You've mastered secure secrets management.**

**What's next?**

1. Advanced security:
   - OIDC for keyless authentication
   - Secret scanning with GitHub Advanced Security
   - Dependency vulnerability scanning
   - Code security analysis

2. Enterprise security:
   - Organization-wide policies
   - Secret rotation automation
   - Compliance logging
   - Access control improvements

3. Proceed to module 07:
   - **07-security-scanning-and-compliance**
   - **08-docker-ci-pipelines**

---

## Secrets Management Reference

### Repository Secrets

```yaml
# Access in workflow
env:
  MY_SECRET: ${{ secrets.MY_SECRET }}

# Used in:
- run: command
  env:
    SECRET: ${{ secrets.MY_SECRET }}
```

### Environment Secrets

```yaml
jobs:
  deploy:
    environment: production
    steps:
      - run: deploy
        env:
          SECRET: ${{ secrets.DEPLOY_SECRET }}
```

### Organization Secrets

```yaml
# Available to all member repositories
env:
  ORG_SECRET: ${{ secrets.ORG_SECRET }}
```

---

## Context Variables Reference

| Context | Variable | Example |
|---------|----------|---------|
| **github** | github.ref | `refs/heads/main` |
| **github** | github.event_name | `push`, `pull_request` |
| **github** | github.run_id | `1234567890` |
| **env** | env.VAR_NAME | Custom variables |
| **secrets** | secrets.SECRET | Encrypted values |
| **matrix** | matrix.os | `ubuntu-latest` |
| **needs** | needs.job.outputs | Job outputs |

---

## Environment Configuration

| Level | Scope | Use Case | Example |
|-------|-------|----------|---------|
| Workflow | Single workflow file | Shared across jobs | `env: API_URL` |
| Job | Single job | Job-specific vars | `jobs.build.env` |
| Step | Single step | Step-specific use | `steps[0].env` |
| Repository | All workflows | Global defaults | Settings → Variables |
| Organization | All member repos | Shared values | Org settings |

---

## Secret Masking

### Automatic Masking

```yaml
# Automatically masked by GitHub
- run: npm ci
  env:
    NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
# Output shows: *** for token value
```

### Manual Masking

```yaml
- run: |
    SECRET_VALUE="my-secret-data"
    echo "::add-mask::${SECRET_VALUE}"
    echo "Secret: ${SECRET_VALUE}"
    # Output: Secret: ***
```

---

## Best Practices

| Practice | Reason | Implementation |
|----------|--------|-----------------|
| Minimal permissions | Least privilege | Scope secrets to environments |
| Rotation schedule | Security | Update secrets every 90 days |
| Audit trail | Compliance | Enable audit logging |
| Masked outputs | Prevent leaks | Use `::add-mask::` |
| Environment parity | Consistency | Same secrets in test/prod |
| Document access | Governance | Record who needs which secrets |

---

## Protection Rules

```yaml
jobs:
  deploy:
    environment:
      name: production
      url: https://example.com
    # Requires:
    # - Approval from 2 reviewers
    # - Passing branch protection rules
    # - Specific branch (main only)
    steps:
      - run: deploy.sh
```

---

## Additional Resources

- [GitHub Secrets Documentation](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [Environment Variables Guide](https://docs.github.com/en/actions/learn-github-actions/environment-variables)
- [OIDC Token Provider](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/using-openid-connect-with-reusable-workflows)
- [Secret Scanning Documentation](https://docs.github.com/en/code-security/secret-scanning)

---

**Created:** January 2026 | **Level:** Intermediate-Advanced | **Estimated Time:** 60 minutes
