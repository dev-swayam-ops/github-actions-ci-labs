# Module 06 Cheatsheet: Secrets and Environment Management

## Secret Types and Access

### Repository Secrets

```yaml
# Access in workflow
env:
  MY_SECRET: ${{ secrets.MY_SECRET }}

# UI Location
Settings → Secrets and variables → Actions → New repository secret
```

### Environment Secrets

```yaml
jobs:
  deploy:
    environment: production
    steps:
      - run: deploy.sh
        env:
          SECRET: ${{ secrets.PROD_SECRET }}

# UI Location
Settings → Environments → [name] → Secrets
```

### Organization Secrets

```yaml
# Access in any member repository
env:
  ORG_SECRET: ${{ secrets.ORG_SECRET }}

# UI Location (org admins only)
Organization Settings → Secrets and variables → Actions
```

---

## Secret Reference Syntax

```yaml
# In environment variables
env:
  TOKEN: ${{ secrets.MY_TOKEN }}

# In workflow input
with:
  api-key: ${{ secrets.API_KEY }}

# In conditional
if: contains(secrets.MY_SECRET, 'value')
```

**Important:** Secrets are only available through context syntax, never as direct variables.

---

## Automatic Masking

```yaml
# GitHub automatically masks these
- run: echo "Token: ${{ secrets.API_TOKEN }}"
# Output: Token: ***

# Also masked in:
- Step output logs
- Job environment variables
- Workflow commands
```

---

## Manual Masking

```yaml
- name: Mask Custom Secret
  run: |
    SECRET=$(openssl rand -base64 32)
    echo "::add-mask::${SECRET}"
    echo "Generated: $SECRET"
    # Output: Generated: ***
```

---

## Context Variables

| Context | Access | Scope | Example |
|---------|--------|-------|---------|
| **github** | `${{ github.ref }}` | Workflow | refs/heads/main |
| **env** | `${{ env.VAR }}` | Defined scope | Custom value |
| **secrets** | `${{ secrets.TOKEN }}` | Job/step env | Encrypted |
| **matrix** | `${{ matrix.os }}` | Matrix job | ubuntu-latest |
| **needs** | `${{ needs.build.outputs.name }}` | Dependent job | Job outputs |

---

## GitHub Context Variables

```yaml
${{ github.ref }}              # Full ref: refs/heads/main
${{ github.ref_name }}        # Branch/tag name: main
${{ github.event_name }}      # Event type: push, pull_request
${{ github.event_action }}    # Action: opened, synchronized
${{ github.run_id }}          # Unique run ID
${{ github.run_number }}      # Run counter
${{ github.run_attempt }}     # Attempt number
${{ github.actor }}           # User who triggered
${{ github.repository }}      # owner/repo
${{ github.repository_owner }} # owner
${{ github.sha }}             # Full commit SHA
${{ github.base_ref }}        # PR target branch
${{ github.head_ref }}        # PR source branch
${{ github.server_url }}      # https://github.com
${{ github.api_url }}         # API URL
${{ github.workspace }}       # Working directory
```

---

## Environment Configuration

### Workflow-Level Variables

```yaml
name: My Workflow
env:
  GLOBAL_VAR: value

jobs:
  build:
    steps:
      - run: echo ${{ env.GLOBAL_VAR }}
```

### Job-Level Variables

```yaml
jobs:
  build:
    env:
      JOB_VAR: value
    steps:
      - run: echo ${{ env.JOB_VAR }}
```

### Step-Level Variables

```yaml
steps:
  - name: Step
    env:
      STEP_VAR: value
    run: echo ${{ env.STEP_VAR }}
```

---

## Environment Setup

### Create Environment

```yaml
Settings → Environments → New environment
Name: production
```

### Add Secrets to Environment

```
Settings → Environments → [name] → Secrets
Add secret for environment-specific values
```

### Protection Rules

```
Settings → Environments → [name] → Deployment protection rules

- Required reviewers: Enable (2+ recommended)
- Restrict reviewers: Optional
- Branch policy: Select main only
- Wait timer: Optional delay
```

---

## Conditional Deployments

### Common Conditions

```yaml
# Main branch push only
if: github.ref == 'refs/heads/main' && github.event_name == 'push'

# Release tags only
if: startsWith(github.ref, 'refs/tags/')

# Pull request
if: github.event_name == 'pull_request'

# Not draft PR
if: !github.event.pull_request.draft

# Release branch
if: startsWith(github.ref, 'refs/heads/release/')

# Successful previous job
if: success()

# Custom branch pattern
if: contains(github.ref, 'production')
```

---

## Environment Pattern

### Three-Environment Setup

```yaml
jobs:
  deploy-dev:
    environment: development
    if: always()
    
  deploy-staging:
    environment: staging
    if: github.ref == 'refs/heads/develop'
    
  deploy-prod:
    environment: production
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

**Access Control:**
- Development: All pushes
- Staging: Feature branch pushes
- Production: Main branch push only (requires approval)

---

## Masking Strategies

### Automatic (Secrets)

```yaml
env:
  TOKEN: ${{ secrets.API_TOKEN }}
# Automatically masked in logs
```

### Manual (Generated Values)

```yaml
- name: Generate Secret
  run: |
    TOKEN=$(generate_token.sh)
    echo "::add-mask::${TOKEN}"
    echo "Token: ${TOKEN}"  # Shows as ***
```

### Add-Mask Command

```bash
echo "::add-mask::SECRET_VALUE"
# All subsequent occurrences of SECRET_VALUE appear as ***
```

---

## Secret Rotation Checklist

| Step | Action | Frequency |
|------|--------|-----------|
| 1 | Identify expiring secrets | Monthly |
| 2 | Generate new credentials | Per secret |
| 3 | Test with staging environment | Before production |
| 4 | Add new secret to GitHub | Parallel period |
| 5 | Deploy with new secret | After staging test |
| 6 | Revoke old credentials | After verification |
| 7 | Document rotation | Per secret |

---

## Common Secret Types

| Type | Example | Rotation |
|------|---------|----------|
| **npm token** | `npm_xxxxx` | Monthly |
| **GitHub token** | `ghp_xxxxx` | Monthly |
| **API key** | `sk_live_xxxxx` | Monthly |
| **Database password** | `SecureP@ss` | Quarterly |
| **SSH key** | Private key content | Annually |
| **SSL certificate** | Certificate PEM | Annually |
| **OAuth token** | `oauth_xxxxx` | Quarterly |

---

## Troubleshooting

### Secret Not Found

```yaml
# Check 1: Exact name match (case-sensitive)
${{ secrets.MY_SECRET }}  # Must match GitHub exactly

# Check 2: In correct scope
# Repo secret: use in same repo
# Env secret: use in that environment job
# Org secret: use in authorized repos

# Check 3: Available in context
# Only available in job steps (not in on: triggers)

# Check 4: Syntax correct
${{ secrets.NAME }}  # Correct
$secrets.NAME       # Wrong
secrets.NAME        # Wrong
```

### Secret Visible in Logs

```yaml
# Wrong - echoing secret
- run: echo ${{ secrets.API_KEY }}

# Correct - use in command, let GitHub mask
- run: npm ci
  env:
    NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### Environment Not Found

```yaml
# Check 1: Environment exists in Settings
# Check 2: Job references exact name
jobs:
  deploy:
    environment: production  # Must exist in Settings

# Check 3: Permissions correct
# User must have access to see protected environment
```

---

## Security Best Practices

| Practice | Why | Implementation |
|----------|-----|-----------------|
| Minimal scope | Least privilege | Use environment secrets |
| Short retention | Reduce exposure window | 3-7 days artifacts |
| Frequent rotation | Limit damage if compromised | Monthly for API keys |
| Approval required | Human verification | 2+ reviewers for prod |
| Audit logging | Compliance/investigation | Enable audit logs |
| Masking output | Prevent log leaks | Use `::add-mask::` |
| No hardcoding | Prevent commits of secrets | Use ${{ secrets.NAME }} |

---

## API and CLI Commands

### Using GitHub CLI

```bash
# Create secret via CLI
gh secret set MY_SECRET -b "secret value"

# List secrets
gh secret list

# Delete secret
gh secret delete MY_SECRET

# Set environment secret
gh secret set PROD_SECRET -b "value" --env production
```

### Using REST API

```bash
# Get secret (names only, not values)
curl -H "Authorization: token TOKEN" \
  https://api.github.com/repos/OWNER/REPO/actions/secrets

# Create secret
curl -X POST \
  -H "Authorization: token TOKEN" \
  https://api.github.com/repos/OWNER/REPO/actions/secrets/MY_SECRET \
  -d '{"encrypted_value":"...","key_id":"..."}'
```

---

## Protection Rules Configuration

### Required Reviewers

```
Settings → Environments → production

✓ Required reviewers: Enable
Number: 2 or more
Restrict to: Specific users/teams (optional)
```

### Deployment Branches

```
Settings → Environments → production

✓ Deployment branches and environments
Selected branches only:
- Branch: main
```

### Wait Timer

```
Settings → Environments → production

Wait timer: [5] minutes before deployment
```

---

## Output Masking Pattern

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      token: ${{ steps.generate.outputs.token }}
    steps:
      - id: generate
        run: |
          TOKEN=$(generate-token.sh)
          echo "::add-mask::${TOKEN}"
          echo "token=${TOKEN}" >> $GITHUB_OUTPUT
```

---

## Pro Tips

**1. Use meaningful secret names:**
```yaml
# Good
PROD_DATABASE_PASSWORD
STAGING_API_KEY
NPM_REGISTRY_TOKEN

# Bad
SECRET1
PASS
KEY
```

**2. Document secret purposes:**
```markdown
# Secrets Used

| Secret | Purpose | Rotation |
|--------|---------|----------|
| NPM_TOKEN | npm registry | Monthly |
| DEPLOY_KEY | CloudFormation | Quarterly |
```

**3. Set environment-specific values:**
```yaml
# Different secrets per environment
Dev:    DEV_API_KEY
Staging: STAGING_API_KEY
Prod:    PROD_API_KEY
```

**4. Test secret access:**
```yaml
- name: Verify Secret Access
  run: |
    if [ -z "${{ secrets.API_TOKEN }}" ]; then
      echo "Secret not available!"
      exit 1
    fi
```

**5. Use repository variables for non-sensitive data:**
```yaml
# Use secrets for: passwords, tokens, keys
# Use variables for: URLs, versions, feature flags

env:
  API_ENDPOINT: https://api.example.com  # Variable
  API_TOKEN: ${{ secrets.API_TOKEN }}    # Secret
```

---

## Limits

| Item | Limit | Notes |
|------|-------|-------|
| Secret length | 64 KB | Very generous |
| Secrets per repo | Unlimited | Practical limit: ~100 |
| Organization secrets | Unlimited | Practical limit: ~100 |
| Masking values | Unlimited | Per secret |
| Environment count | Unlimited | Practical: 3-5 typical |
| Required reviewers | Unlimited | Recommend 2-3 |

---

**Last Updated:** January 2026 | **Module:** 06 | **Level:** Intermediate-Advanced
