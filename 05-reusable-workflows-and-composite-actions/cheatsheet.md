# Module 05 Cheatsheet: Reusable Workflows and Composite Actions

## Reusable Workflow Syntax

### Define Reusable Workflow

```yaml
name: My Reusable Workflow

on:
  workflow_call:
    inputs:
      node-version:
        type: string
        required: false
        default: '18'
        description: 'Node.js version'
    outputs:
      artifact-name:
        description: 'Name of build artifact'
        value: ${{ jobs.build.outputs.artifact-name }}
    secrets:
      npm-token:
        required: true
        description: 'NPM registry token'

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.build.outputs.artifact-name }}
    steps:
      - id: build
        run: echo "artifact-name=build-123" >> $GITHUB_OUTPUT
```

### Call Reusable Workflow

```yaml
jobs:
  build:
    uses: ./.github/workflows/build.yml
    with:
      node-version: '18'
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}

  test:
    needs: build
    uses: owner/repo/.github/workflows/test.yml@v1
    with:
      node-version: ${{ matrix.node }}
    secrets: inherit
```

---

## Input Types and Syntax

| Type | Example | Usage |
|------|---------|-------|
| **string** | `'18'` or `'main'` | `${{ inputs.param }}` |
| **boolean** | `true` or `false` | `if: inputs.flag` |
| **choice** | `'option1'` or `'option2'` | Selection dropdown |
| **environment** | `'prod'` or `'staging'` | Auto-formatted environment |

**Input Definition:**

```yaml
inputs:
  node-version:
    type: string
    required: true
    description: 'Node.js version'
    default: '18'

  use-cache:
    type: boolean
    required: false
    default: true

  environment:
    type: choice
    options:
      - dev
      - staging
      - prod
```

---

## Outputs and Dependencies

### Define Job Output

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.build.outputs.artifact-name }}
    steps:
      - id: build
        run: echo "artifact-name=build-${{ github.run_id }}" >> $GITHUB_OUTPUT
```

### Define Workflow Output

```yaml
on:
  workflow_call:
    outputs:
      artifact-name:
        description: 'Name of artifact'
        value: ${{ jobs.build.outputs.artifact-name }}
```

### Access Outputs

```yaml
jobs:
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Artifact: ${{ needs.build.outputs.artifact-name }}"
```

---

## Secrets Handling

### Pass Secrets Explicitly

```yaml
jobs:
  deploy:
    uses: ./.github/workflows/deploy.yml
    secrets:
      api-key: ${{ secrets.API_KEY }}
      db-password: ${{ secrets.DB_PASSWORD }}
```

### Inherit All Secrets

```yaml
jobs:
  build:
    uses: ./.github/workflows/build.yml
    secrets: inherit
```

### Define Secret in Workflow

```yaml
on:
  workflow_call:
    secrets:
      npm-token:
        required: true
        description: 'NPM registry token'

jobs:
  build:
    steps:
      - run: npm ci
        env:
          NPM_TOKEN: ${{ secrets.npm-token }}
```

---

## Composite Action Reference

### Create action.yml

```yaml
name: 'My Composite Action'
description: 'What the action does'
author: 'Author Name'

inputs:
  param1:
    description: 'Parameter 1'
    required: true
    default: 'default-value'

outputs:
  result:
    description: 'Output description'
    value: ${{ steps.main.outputs.result }}

runs:
  using: 'composite'
  steps:
    - id: main
      shell: bash
      run: |
        echo "Running composite action"
        echo "result=success" >> $GITHUB_OUTPUT
```

### Use Composite Action

```yaml
steps:
  - uses: ./.github/actions/my-action
    id: my-action
    with:
      param1: 'value'
  
  - run: echo "Result: ${{ steps.my-action.outputs.result }}"
```

---

## Composite Action Shells

| Shell | Example | Use Case |
|-------|---------|----------|
| bash | `shell: bash` | Linux, macOS |
| pwsh | `shell: pwsh` | Windows PowerShell |
| cmd | `shell: cmd` | Windows batch |
| python | `shell: python` | Python scripts |

```yaml
runs:
  using: 'composite'
  steps:
    - shell: bash
      run: echo "Linux/macOS"
    
    - shell: pwsh
      run: Write-Host "Windows"
    
    - shell: python
      run: print("Python script")
```

---

## Common Patterns

### Build Once, Test Multiple

```yaml
jobs:
  build:
    uses: ./.github/workflows/build.yml

  test-unit:
    needs: build
    uses: ./.github/workflows/test.yml
    with:
      test-type: 'unit'

  test-e2e:
    needs: build
    uses: ./.github/workflows/test.yml
    with:
      test-type: 'e2e'
```

### Matrix with Reusable

```yaml
jobs:
  test:
    strategy:
      matrix:
        node: [16, 18, 20]
    uses: ./.github/workflows/test.yml
    with:
      node-version: ${{ matrix.node }}
```

### Conditional Reusable

```yaml
jobs:
  deploy:
    if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/')
    uses: ./.github/workflows/deploy.yml
    secrets: inherit
```

### Sequential Reusable Workflows

```yaml
jobs:
  build:
    uses: ./.github/workflows/build.yml

  test:
    needs: build
    uses: ./.github/workflows/test.yml

  deploy:
    needs: test
    uses: ./.github/workflows/deploy.yml
```

---

## Workflow File Paths

| Path | Type | Usage |
|------|------|-------|
| `./.github/workflows/name.yml` | Local reusable | `uses: ./.github/workflows/name.yml` |
| `./.github/actions/name/action.yml` | Local composite | `uses: ./.github/actions/name` |
| `owner/repo/.github/workflows/name.yml@v1` | Remote reusable | Full path + ref |
| `owner/repo/path/to/action@v1` | Remote composite | Full path + ref |

---

## Reference Syntax

### Available Contexts

```yaml
# Inputs
${{ inputs.input-name }}

# Secrets
${{ secrets.secret-name }}

# Outputs from jobs
${{ jobs.job-id.outputs.output-name }}

# Outputs from steps
${{ steps.step-id.outputs.output-name }}

# Outputs from called workflow
${{ needs.job-id.outputs.output-name }}

# GitHub context
${{ github.ref }}
${{ github.event_name }}
${{ github.run_id }}
```

---

## Troubleshooting Commands

### Check Workflow Syntax

```bash
# Validate YAML syntax
yamllint .github/workflows/

# Check for common issues
grep -r "workflow_call" .github/workflows/
```

### Debug Outputs

```yaml
- name: Debug Outputs
  run: |
    echo "Inputs: ${{ inputs.node-version }}"
    echo "Secrets: ${{ secrets.npm-token }}"
    echo "Job outputs: ${{ jobs.build.outputs.artifact-name }}"
```

### Verify Workflow Dispatch

```yaml
on:
  workflow_dispatch:
    inputs:
      debug:
        type: boolean
        default: false

jobs:
  test:
    if: inputs.debug == true
    runs-on: ubuntu-latest
    steps:
      - run: echo "Debug mode enabled"
```

---

## Best Practices

| Practice | Reason | Example |
|----------|--------|---------|
| Clear names | Easy to understand | `build-artifact` not `a` |
| Type safety | Prevent errors | Always specify input `type:` |
| Document well | Clarity for users | Good `description:` fields |
| Version actions | Reproducibility | Use @v1 not @main |
| Small focused | Better reuse | One concern per workflow |
| DRY principle | Less duplication | Extract to reusables |
| Explicit secrets | Security | Never use `secrets: inherit` lightly |

---

## Performance Tips

| Optimization | Impact | Implementation |
|--------------|--------|-----------------|
| Parallel workflows | Linear speedup | Multiple independent jobs |
| Cache in reusables | 80% faster deps | `cache: 'npm'` in setup |
| Conditional calls | Skip unnecessary | `if:` conditions |
| Artifact pass-through | Avoid rebuilds | Download instead of rebuild |

---

## Limitations

| Feature | Limit | Workaround |
|---------|-------|-----------|
| Max inputs | ~30 (practical) | Use fewer, larger inputs |
| Max outputs | ~40 (practical) | Consolidate outputs |
| Secret length | 64KB | Use file-based secrets |
| Nesting depth | 2 levels | Design flat structure |

---

## Pro Tips

**1. Use consistent naming:**
```yaml
# Good
inputs:
  node-version
  build-command
outputs:
  artifact-name

# Bad
inputs:
  n
  cmd
outputs:
  a
```

**2. Set sensible defaults:**
```yaml
inputs:
  node-version:
    default: '18'  # Most common version
  cache:
    default: true  # Usually want caching
```

**3. Validate inputs:**
```yaml
steps:
  - run: |
      if [[ ! "${{ inputs.node-version }}" =~ ^[0-9]+$ ]]; then
        echo "Invalid Node version"
        exit 1
      fi
```

**4. Clear output examples:**
```yaml
outputs:
  artifact-url:
    description: 'Full URL to artifact (e.g., https://github.com/org/repo/actions/runs/123)'
    value: ${{ steps.upload.outputs.artifact-url }}
```

**5. Use action.yml for discovery:**
```yaml
# First thing developers see
description: 'Comprehensive description of what action does'
author: 'Your Name'
```

---

## Quick Reference: Step-by-Step

### Create Reusable Workflow

```bash
# 1. Create file
vim .github/workflows/reusable.yml

# 2. Add trigger
on:
  workflow_call:

# 3. Define inputs/outputs/secrets
# 4. Add jobs and steps
# 5. Test with main workflow
uses: ./.github/workflows/reusable.yml
```

### Create Composite Action

```bash
# 1. Create directory
mkdir -p .github/actions/my-action

# 2. Create action.yml
vim .github/actions/my-action/action.yml

# 3. Define inputs/outputs
# 4. Add steps with shell
# 5. Use in workflow
uses: ./.github/actions/my-action
```

---

**Last Updated:** January 2026 | **Module:** 05 | **Level:** Intermediate-Advanced
