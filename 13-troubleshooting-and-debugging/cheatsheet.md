# Module 13 Cheatsheet: Troubleshooting and Debugging

## Quick Command Reference

### Essential Debugging Commands

| Command | Purpose | Example |
|---------|---------|---------|
| `echo $VAR` | Print variable value | `echo $HOME` |
| `echo ${{ env.VAR }}` | Print GitHub Actions variable | `echo ${{ github.ref }}` |
| `set -x` | Enable debug mode (show commands) | `set -x; npm test; set +x` |
| `env \| sort` | List all environment variables | `env \| grep -i github` |
| `which COMMAND` | Check if command exists | `which npm` |
| `[ -f FILE ]` | Test if file exists | `[ -f package.json ] && echo "exists"` |
| `[ -w DIR ]` | Test if directory writable | `[ -w /tmp ] && echo "writable"` |
| `ls -la` | List files with details | `ls -la .github/workflows/` |
| `pwd` | Print working directory | `pwd` |
| `git status` | Check git status | `git status --short` |

---

## GitHub Context Variables Quick Reference

### Basic Variables

```yaml
${{ github.repository }}       # owner/repo
${{ github.ref }}              # refs/heads/main (full)
${{ github.ref_name }}         # main (just branch name)
${{ github.sha }}              # commit hash
${{ github.actor }}            # user who triggered
${{ github.event_name }}       # push, pull_request, etc
```

### Runner Variables

```yaml
${{ runner.os }}               # Linux, Windows, macOS
${{ runner.arch }}             # x64, arm64
```

### Job Variables

```yaml
${{ job.status }}              # success, failure, cancelled
${{ job.container.id }}        # container ID if running in container
```

---

## Common Error Messages and Solutions

### Error: YAML parse error

```
Error: YAML parse error on line 15: mapping values are not allowed here
```

**Causes:**
- Missing colon after key
- Unclosed quote
- Wrong indentation

**Solution:**
```yaml
# ✓ Correct
key: value

# ✗ Wrong
key value  # Missing colon
```

---

### Error: command not found

```
/bin/bash: line 1: npm: command not found
```

**Causes:**
- Tool not installed
- Tool not in PATH
- Typo in command name

**Solution:**
```yaml
steps:
  - uses: actions/setup-node@v3
    with:
      node-version: '18'
  - run: npm --version  # Now works
```

---

### Error: Permission denied

```
Permission denied: ./scripts/deploy.sh
```

**Solution:**
```bash
# Option 1: Make executable
chmod +x ./scripts/deploy.sh
./scripts/deploy.sh

# Option 2: Use bash prefix
bash ./scripts/deploy.sh
```

---

### Error: Secret not found

```
The secret MY_SECRET is not set in your repository
```

**Solution:**
1. Settings → Secrets and Variables → Actions
2. Create new secret: `MY_SECRET`
3. Use in workflow: `${{ secrets.MY_SECRET }}`

---

### Error: Job dependency not found

```
Job 'deploy' depends on non-existent job 'test'
```

**Solution:**
```yaml
jobs:
  test:        # ✓ Define the job first
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  deploy:
    needs: test  # ✓ Reference existing job
```

---

### Error: Connection refused

```
curl: (7) Failed to connect to localhost port 5432
```

**Solution:**
- Add health check to service
- Wait for service to be ready
- Use service hostname (not localhost)

```yaml
services:
  postgres:
    image: postgres:15
    options: >-
      --health-cmd pg_isready
      --health-interval 10s

steps:
  - run: psql -h postgres ...  # Use service name
```

---

## Debug Techniques

### Technique 1: Echo Everything

```bash
echo "=== Environment ==="
echo "Branch: ${{ github.ref_name }}"
echo "Actor: ${{ github.actor }}"
echo "Commit: ${{ github.sha }}"

echo "=== Variables ==="
echo "MY_VAR: $MY_VAR"
echo "NODE_ENV: $NODE_ENV"
```

---

### Technique 2: Enable Bash Debug Mode

```bash
set -x      # Show commands as they run
npm ci
npm test
set +x      # Stop showing commands
```

**Output shows:**
```
+ npm ci
+ npm test
```

---

### Technique 3: Test Before Running

```bash
# Check if command exists
which npm || echo "npm not installed"

# Check if file exists
[ -f package.json ] || echo "package.json missing"

# Check if variable set
[ -z "$API_KEY" ] && echo "API_KEY not set"
```

---

### Technique 4: Validate Conditions

```yaml
- name: Check conditions
  run: |
    echo "Branch: ${{ github.ref_name }}"
    echo "Is main? ${{ github.ref_name == 'main' }}"
    echo "Event: ${{ github.event_name }}"
    echo "Is push? ${{ github.event_name == 'push' }}"
```

---

### Technique 5: List Files and Directories

```bash
echo "=== Current directory ==="
pwd

echo "=== Repository structure ==="
ls -la

echo "=== Find YAML files ==="
find . -name "*.yml" -o -name "*.yaml"

echo "=== Check artifacts ==="
ls -la dist/ || echo "dist/ not found"
```

---

## Conditional Logic Reference

### Basic Conditionals

```yaml
if: github.ref == 'refs/heads/main'          # On main branch
if: github.ref_name == 'main'                 # Simpler syntax
if: github.event_name == 'push'               # On push event
if: github.event_name == 'pull_request'       # On PR
if: startsWith(github.ref, 'refs/tags/')      # On tag push
```

### Combining Conditions

```yaml
if: success()                                 # Previous step ok
if: failure()                                 # Previous step failed
if: always()                                  # Always run
if: cancelled()                               # Workflow cancelled

if: success() && github.ref_name == 'main'    # AND condition
if: github.event_name == 'push' || github.event_name == 'pull_request'  # OR
```

---

## Environment Variables vs Secrets

| Feature | Environment Variables | Secrets |
|---------|----------------------|---------|
| **Set in** | Workflow YAML | Settings → Secrets |
| **Access** | `$VAR` or `${{ env.VAR }}` | `${{ secrets.NAME }}` |
| **Visible** | In logs | Masked in logs |
| **Usage** | Public config | Sensitive data |
| **Example** | `NODE_ENV: production` | `${{ secrets.API_KEY }}` |

---

## File and Directory Commands

### Check File Existence

```bash
# File exists?
test -f filename && echo "exists" || echo "not found"
[ -f filename ] && echo "exists"

# Directory exists?
test -d dirname && echo "exists"
[ -d dirname ] && echo "exists"

# Readable?
[ -r filename ] && echo "readable"

# Writable?
[ -w filename ] && echo "writable"

# Executable?
[ -x filename ] && echo "executable"
```

### List Files

```bash
# List current directory
ls

# List with details
ls -la

# Show directory structure
tree .

# Find specific files
find . -name "*.yml"
find . -type f -name "package.json"
```

---

## Service Container Debugging

### Check Service Status

```yaml
- name: Check PostgreSQL
  run: |
    pg_isready -h postgres -p 5432 || echo "Not ready"
    
- name: Check MySQL
  run: |
    mysqladmin -h mysql -u root ping || echo "Not ready"
    
- name: Check Redis
  run: |
    redis-cli -h redis ping || echo "Not ready"
```

### Wait for Service

```bash
#!/bin/bash
echo "Waiting for service..."
for i in {1..30}; do
  if pg_isready -h postgres -p 5432 > /dev/null; then
    echo "✓ Service ready"
    exit 0
  fi
  sleep 1
done
echo "✗ Service timeout"
exit 1
```

---

## Docker Debugging

### Build Diagnostics

```bash
# Build with verbose output
docker build -t myapp:latest --progress=plain .

# Check image
docker images myapp

# Inspect image
docker inspect myapp:latest

# Check layers
docker image history myapp:latest
```

### Run Diagnostics

```bash
# Run image and check
docker run --rm myapp:latest echo "Working"

# Check exit code
docker run myapp:latest; echo "Exit code: $?"

# View image command
docker inspect myapp | jq .[0].Cmd
```

---

## Git Commands for Debugging

```bash
# Check current branch
git branch --show-current
git symbolic-ref --short HEAD

# Show recent commits
git log --oneline -10

# Check status
git status --short

# Show what changed
git diff HEAD~1

# Check remote
git remote -v
git remote show origin
```

---

## Performance Debugging

### Find Slow Steps

```bash
# Each step shows duration in Actions logs
# Look for steps taking >30 seconds

# Find large files
du -sh * | sort -hr

# Check disk usage
df -h

# Check memory
free -h
```

### Profile Installation

```bash
time npm ci
# Output: real    0m32.456s

time npm install
# Output: real    0m45.123s

# npm ci is faster ✓
```

---

## Secrets Safety Checklist

- ✓ Never hardcode secrets in workflow YAML
- ✓ Never print secrets in logs
- ✓ Use `${{ secrets.NAME }}` syntax
- ✓ GitHub masks secrets automatically
- ✓ Rotate secrets regularly
- ✓ Store in Settings → Secrets
- ✓ Use environment-specific secrets
- ✓ Audit secret usage

---

## Debugging Workflow Template

Use this template when debugging:

```yaml
name: Debug Template

on: [push]

jobs:
  debug:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Show context
        run: |
          echo "=== GitHub Context ==="
          echo "Ref: ${{ github.ref }}"
          echo "Ref Name: ${{ github.ref_name }}"
          echo "Event: ${{ github.event_name }}"
          echo "Actor: ${{ github.actor }}"
          echo "SHA: ${{ github.sha }}"
      
      - name: Show environment
        run: |
          echo "=== Environment ==="
          echo "OS: $(uname -s)"
          echo "User: $(whoami)"
          echo "PWD: $(pwd)"
          echo "PATH: $PATH"
      
      - name: Enable debug mode
        run: |
          set -x
          echo "Commands will be shown"
          echo "This is a test"
          set +x
      
      - name: Check files
        run: |
          echo "=== Files ==="
          ls -la
          echo "=== Finding YAMLs ==="
          find . -name "*.yml" -o -name "*.yaml"
      
      - name: Test conditions
        run: |
          echo "Condition tests:"
          echo "Is main: ${{ github.ref_name == 'main' }}"
          echo "Is push: ${{ github.event_name == 'push' }}"
```

---

## Common Fixes Quick Reference

| Problem | Quick Fix |
|---------|-----------|
| YAML syntax error | Check for missing colons, quotes, indentation |
| Command not found | Install tool with actions, or check PATH |
| Permission denied | Add `bash` prefix or use `chmod +x` |
| Variable empty | Define in `env:` section or as secret |
| Job never runs | Check `if:` condition and job dependencies |
| Service unreachable | Add health check, wait for service |
| Cache missing | Ensure filename matches, check 5GB limit |
| Secret exposed | GitHub masks it, but don't rely on it |
| Artifact not found | Upload with correct name, download matching name |
| Timeout | Reduce duration or add timeout-minutes |

---

## Resources

- [GitHub Actions Documentation](https://docs.github.com/actions)
- [Workflow Syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [Context Variables](https://docs.github.com/en/actions/learn-github-actions/contexts)
- [Debugging Documentation](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows)

---

## Pro Tips

1. **Always add debug output** when creating complex workflows
2. **Test conditionals** with echo before relying on them
3. **Check service health** before running tests
4. **Use meaningful variable names** for easy debugging
5. **Add timestamps** to long-running steps
6. **Document assumptions** in comments
7. **Version your workflows** for easy rollback
8. **Test locally first** if possible
9. **Keep logs clean** by removing debug code once fixed
10. **Share solutions** with your team

Happy Debugging! 🐛🔍
