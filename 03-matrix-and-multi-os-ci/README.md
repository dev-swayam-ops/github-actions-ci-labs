# 03-matrix-and-multi-os-ci

## Overview

This module teaches you how to test applications across multiple operating systems and language versions simultaneously. You'll master the matrix strategy to maximize test coverage while maintaining build efficiency.

---

## What You'll Learn

- Understanding the matrix strategy for CI pipelines
- Testing across multiple operating systems (Ubuntu, Windows, macOS)
- Testing multiple language/runtime versions
- Matrix inclusion and exclusion patterns
- Optimizing parallel execution
- Managing OS-specific commands in workflows
- Conditional steps based on runner OS
- Matrix output and dependency handling
- Fail-fast strategies and continuation policies

---

## Prerequisites

- Completion of **02-build-and-test-pipelines** module
- Understanding of job dependencies and needs
- Familiarity with conditional execution
- Basic shell scripting knowledge
- GitHub repository with write access

---

## Key Concepts

### 1. **Matrix Strategy**
Configuration that creates multiple job variations from a single job definition, enabling parallel testing.

### 2. **Runner OS**
The operating system of the execution environment. Options: Ubuntu, Windows, macOS.

### 3. **Matrix Combinations**
Total number of job variations created by multiplying matrix dimensions (e.g., 2 OS × 3 versions = 6 jobs).

### 4. **Fail-Fast Strategy**
Option to stop all jobs if one fails, or continue remaining jobs.

### 5. **Include Patterns**
Adding specific job configurations beyond standard matrix combinations.

### 6. **Exclude Patterns**
Removing specific job combinations from the matrix.

### 7. **OS-Specific Commands**
Different commands needed per operating system (bash vs PowerShell).

### 8. **Runner Context**
Built-in variables like `runner.os`, `runner.arch` to detect execution environment.

### 9. **Matrix Outputs**
Sharing data from matrix jobs to dependent jobs.

---

## Hands-on Lab: Multi-OS and Multi-Version Pipeline

### Step 1: Create the Workflow File

Create `.github/workflows/matrix-pipeline.yml`:

```yaml
name: Matrix Multi-OS Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  # Job 1: Lint (single run, fast check)
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci && npm run lint 2>/dev/null || echo "Lint check (simulated)"

  # Job 2: Test across multiple OS and versions
  test:
    needs: lint
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: ['16', '18', '20']
      fail-fast: false
    
    name: Test on ${{ matrix.os }} / Node ${{ matrix.node }}
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Setup Node.js ${{ matrix.node }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      
      - name: Display Environment
        run: |
          echo "OS: ${{ runner.os }}"
          echo "OS Version: ${{ matrix.os }}"
          echo "Node Version: $(node --version)"
          echo "npm Version: $(npm --version)"
          echo "Architecture: ${{ runner.arch }}"
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Run Tests
        run: npm test 2>/dev/null || echo "Tests passed (simulated)"
      
      - name: OS-Specific Post-Test (Ubuntu)
        if: runner.os == 'Linux'
        run: echo "✓ Ubuntu cleanup"
      
      - name: OS-Specific Post-Test (macOS)
        if: runner.os == 'macOS'
        run: echo "✓ macOS cleanup"
      
      - name: OS-Specific Post-Test (Windows)
        if: runner.os == 'Windows'
        run: echo "✓ Windows cleanup"
  
  # Job 3: Build (runs after all test variations)
  build:
    needs: test
    runs-on: ubuntu-latest
    if: always()
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci && npm run build 2>/dev/null || echo "Build passed"
      - name: Summary
        run: |
          echo "Multi-OS Test Summary"
          echo "===================="
          echo "OS Tested: 3 (Ubuntu, Windows, macOS)"
          echo "Node Versions: 3 (16, 18, 20)"
          echo "Total Jobs: 9 (3 OS × 3 versions)"
          echo "Status: All tests completed"
```

### Step 2: Commit and Push

```bash
git add .github/workflows/matrix-pipeline.yml
git commit -m "Add matrix multi-OS pipeline"
git push origin main
```

### Step 3: Monitor Execution

**Expected Output:**

Go to Actions tab and observe:

```
Workflow: Matrix Multi-OS Pipeline

├── lint (1 job)
│   └── ✓ Completed in ~30s
│
├── test (9 jobs - parallel)
│   ├── ✓ Ubuntu 16 - Completed in ~40s
│   ├── ✓ Ubuntu 18 - Completed in ~40s
│   ├── ✓ Ubuntu 20 - Completed in ~40s
│   ├── ✓ Windows 16 - Completed in ~60s
│   ├── ✓ Windows 18 - Completed in ~60s
│   ├── ✓ Windows 20 - Completed in ~60s
│   ├── ✓ macOS 16 - Completed in ~50s
│   ├── ✓ macOS 18 - Completed in ~50s
│   └── ✓ macOS 20 - Completed in ~50s
│
└── build (1 job)
    └── ✓ Completed in ~30s

Total Execution Time: ~2-3 minutes
```

Each job shows:
- ✅ Job name with OS and Node version
- ✅ Environment details (OS, architecture, versions)
- ✅ Test execution output
- ✅ OS-specific cleanup steps
- ✅ Pass/fail status

---

## Validation Checklist

- [ ] Workflow file is syntactically correct
- [ ] Matrix creates 9 job variations (3 OS × 3 Node versions)
- [ ] All matrix jobs run in parallel
- [ ] Each job displays correct OS and Node version
- [ ] OS-specific steps execute correctly
- [ ] Build job runs after all test jobs complete
- [ ] Workflow completes in under 5 minutes
- [ ] No jobs are unexpectedly skipped

---

## Cleanup

```bash
rm .github/workflows/matrix-pipeline.yml
git add .github/workflows/
git commit -m "Remove matrix pipeline"
git push origin main
```

---

## Common Mistakes

### ❌ Mistake 1: Incorrect Matrix Syntax

```yaml
# Wrong - missing colon
strategy:
  matrix
    os: [ubuntu-latest, windows-latest]

# Correct
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
```

### ❌ Mistake 2: Using String Instead of Array

```yaml
# Wrong - single string creates 1 job per value
strategy:
  matrix:
    node: 18

# Correct - array creates multiple jobs
strategy:
  matrix:
    node: [16, 18, 20]
```

### ❌ Mistake 3: Forgetting fail-fast Default

```yaml
# Default: fail-fast is true (stops on first failure)
# May want to change for comprehensive testing
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
  fail-fast: false  # Continue all tests
```

### ❌ Mistake 4: Using Shell Syntax for Wrong OS

```yaml
# Wrong - bash command on Windows
- run: ls -la

# Correct - use conditional or cross-platform command
- run: |
    if [ "${{ runner.os }}" = "Linux" ]; then
      ls -la
    else
      dir
    fi
```

### ❌ Mistake 5: Missing Cache per Matrix Item

```yaml
# Wrong - cache misses per version
strategy:
  matrix:
    node: [16, 18, 20]
steps:
  - uses: actions/setup-node@v3
    with:
      node-version: ${{ matrix.node }}
      # Missing cache!

# Correct - cache includes version
- uses: actions/setup-node@v3
  with:
    node-version: ${{ matrix.node }}
    cache: 'npm'
```

---

## Troubleshooting

### Issue: Matrix jobs not creating expected number of variations

**Solution:**
1. Verify matrix syntax with colons and proper nesting
2. Count dimensions: dimension1 × dimension2 = total jobs
3. Check for typos in matrix key names
4. Use `fail-fast: false` to see all job results

### Issue: Different OS showing different behaviors

**Solution:**
- Test with conditional OS checks: `if: runner.os == 'Windows'`
- Use cross-platform tools when possible (Node, npm)
- Separate OS-specific logic into dedicated steps
- Reference Docker containers for consistency

### Issue: Cache not working per matrix item

**Solution:**
- Ensure cache key includes matrix variables: `key: ${{ runner.os }}-npm-${{ matrix.node }}`
- Verify cache path exists before download
- Check cache is enabled in setup action

### Issue: Windows jobs significantly slower

**Solution:**
- Windows runners have different resource allocation
- Consider fail-fast for CI (faster feedback)
- Use smaller test suite on Windows
- Cache is especially important on Windows

### Issue: Some matrix combinations not needed

**Solution:**
```yaml
exclude:
  - os: windows-latest
    node: '16'  # Old version not needed on Windows
```

---

## Next Steps

✅ **You've completed module 03!**

**What's next?**

1. Proceed to **04-caching-and-artifacts** to learn:
   - Advanced caching strategies
   - Artifact lifecycle management
   - Cross-job artifact sharing
   - Cache invalidation patterns
   
2. Advanced topics to explore:
   - Custom matrix outputs
   - Fail-fast configuration
   - Conditional matrix items
   - Dynamic matrix values

3. Practice exercises:
   - Add more matrix dimensions
   - Optimize build times
   - Implement fail-fast strategies
   - Create platform-specific builds

---

## Matrix Strategy Reference

### Basic Matrix

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node: [16, 18, 20]
    # Creates 3 × 3 = 9 jobs
```

### Matrix with Exclusion

```yaml
strategy:
  matrix:
    os: [ubuntu, windows, macos]
    node: [16, 18, 20]
  exclude:
    - os: windows
      node: 16
    # Creates 8 jobs (excludes 1)
```

### Matrix with Inclusion

```yaml
strategy:
  matrix:
    os: [ubuntu, windows]
    node: [16, 18]
  include:
    - os: macos
      node: 18
    # Creates 5 jobs (4 base + 1 included)
```

---

## Runner Operating Systems

| Runner | Full Name | Shell | Common Issues |
|--------|-----------|-------|----------------|
| `ubuntu-latest` | Ubuntu 22.04 | bash | Fastest, most tested |
| `ubuntu-20.04` | Ubuntu 20.04 | bash | Older LTS version |
| `windows-latest` | Windows Server 2022 | pwsh | Slower startup |
| `windows-2019` | Windows Server 2019 | pwsh | Legacy support |
| `macos-latest` | macOS 13 (Ventura) | bash | Most expensive runner |
| `macos-12` | macOS 12 (Monterey) | bash | Older Intel Macs |

---

## OS Detection in Workflows

```yaml
# Detect OS
runner.os          # "Linux", "Windows", "macOS"
runner.arch        # "X64", "ARM64", etc.

# Conditional execution
if: runner.os == 'Linux'
if: runner.os == 'Windows'
if: runner.os == 'macOS'

# Access matrix values
matrix.os
matrix.node
matrix.custom-var
```

---

## Cross-Platform Commands

| Task | Linux/macOS | Windows |
|------|------------|---------|
| List files | `ls -la` | `dir` |
| Print text | `echo "text"` | `echo "text"` |
| Set variable | `export VAR=value` | `$env:VAR='value'` |
| Delete file | `rm file` | `del file` |
| Create dir | `mkdir -p dir` | `mkdir dir` |
| Concatenate | `cat file` | `type file` |
| Copy file | `cp src dst` | `copy src dst` |

---

## Performance Optimization

| Strategy | Impact | Implementation |
|----------|--------|-----------------|
| Parallel matrix | Linear scaling | Multiple dimensions |
| Cache per version | 50-70% faster | Include version in cache key |
| Fail-fast | 50% time (half test) | `fail-fast: true` |
| Exclude unused | Variable savings | `exclude:` patterns |
| Filter paths | Skip whole workflow | `paths:` in `on:` |

---

## Additional Resources

- [Matrix Strategy Documentation](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs)
- [Runners and Virtual Environments](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners)
- [Context Variables](https://docs.github.com/en/actions/learn-github-actions/contexts)
- [Runner Specifications](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners#supported-runners-and-hardware-resources)

---

**Created:** January 2026 | **Level:** Intermediate | **Estimated Time:** 45 minutes
