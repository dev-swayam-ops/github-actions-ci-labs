# Cheatsheet: Matrix and Multi-OS CI

Quick reference for matrix strategy and multi-OS testing in GitHub Actions.

---

## Matrix Strategy Basics

### Simple Matrix

```yaml
strategy:
  matrix:
    node: [16, 18, 20]
    # Creates 3 jobs
```

### Multi-Dimensional Matrix

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [16, 18, 20]
    # Creates 2 × 3 = 6 jobs
```

### Matrix Calculation

```
Total Jobs = dimension1 × dimension2 × dimension3 × ...

Example:
- 3 OS × 2 Node versions = 6 jobs
- 2 OS × 3 Node × 2 npm = 12 jobs
```

---

## Matrix Syntax Reference

```yaml
strategy:
  matrix:
    key1: [value1, value2]
    key2: [value1, value2, value3]
  
  fail-fast: false
  max-parallel: 4

# Access in job
runs-on: ${{ matrix.key1 }}
env:
  MY_VAR: ${{ matrix.key2 }}
```

---

## Common Matrix Patterns

### Node.js Versions

```yaml
strategy:
  matrix:
    node: [14, 16, 18, 20]
```

### Multiple OS

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
```

### Full Stack

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [16, 18, 20]
    npm: [8, 9]
```

---

## Exclusion and Inclusion

### Exclude Combinations

```yaml
strategy:
  matrix:
    os: [ubuntu, windows, macos]
    node: [16, 18, 20]
  exclude:
    - os: windows
      node: 16
    - os: macos
      node: 16
```

### Include Special Cases

```yaml
strategy:
  matrix:
    os: [ubuntu, windows]
    node: [18, 20]
  include:
    - os: macos
      node: 18
      arch: arm64
```

---

## Accessing Matrix Values

| Variable | Purpose | Example |
|----------|---------|---------|
| `${{ matrix.key }}` | Get matrix value | `${{ matrix.node }}` |
| `${{ runner.os }}` | Get runner OS | `Linux`, `Windows`, `macOS` |
| `${{ runner.arch }}` | Get architecture | `X64`, `ARM64` |
| `matrix.custom-value` | Custom key | Any defined matrix key |

---

## Fail-Fast Configuration

```yaml
# Default: fail-fast is true
strategy:
  matrix:
    node: [16, 18, 20]
  fail-fast: true   # Cancel others on first failure

# Run all tests regardless
strategy:
  matrix:
    node: [16, 18, 20]
  fail-fast: false  # Continue all jobs
```

**Behavior:**

| Setting | Behavior | Use Case |
|---------|----------|----------|
| `true` | Stops on first failure | Fast CI feedback |
| `false` | Runs all jobs | Complete test coverage |

---

## Job Name with Matrix

```yaml
name: Test on ${{ matrix.os }} / Node ${{ matrix.node }}
```

Shows in Actions UI:
```
├── Test on ubuntu-latest / Node 16
├── Test on ubuntu-latest / Node 18
├── Test on ubuntu-latest / Node 20
└── ...
```

---

## Conditional Execution by OS

```yaml
# Run on Linux only
if: runner.os == 'Linux'

# Run on Windows only
if: runner.os == 'Windows'

# Run on macOS only
if: runner.os == 'macOS'

# Run on Linux or macOS
if: runner.os != 'Windows'
```

---

## OS-Specific Commands

| Command | Linux | Windows |
|---------|-------|---------|
| List files | `ls` | `dir` |
| Print | `echo` | `echo` |
| Set env | `export VAR=X` | `$env:VAR='X'` |
| Delete | `rm file` | `del file` |
| Create dir | `mkdir dir` | `mkdir dir` |
| Concatenate | `cat file` | `type file` |
| Which | `which cmd` | `where cmd` |
| Current dir | `pwd` | `cd` |

---

## Cross-Platform Script

```yaml
- name: Multi-OS Friendly
  run: |
    if [ "$RUNNER_OS" == "Windows" ]; then
      echo "Windows"
      dir
    else
      echo "Linux/macOS"
      ls -la
    fi
  shell: bash
```

Or use `runner.os` context:

```yaml
- if: runner.os == 'Windows'
  run: dir

- if: runner.os != 'Windows'
  run: ls -la
```

---

## Parallel Job Limits

```yaml
strategy:
  matrix:
    os: [ubuntu, windows, macos]
  max-parallel: 2    # Only 2 jobs at once
```

**Default:** All matrix jobs run in parallel

---

## Matrix Outputs

```yaml
jobs:
  test:
    strategy:
      matrix:
        node: [16, 18, 20]
    outputs:
      version: ${{ steps.meta.outputs.version }}
    steps:
      - id: meta
        run: echo "version=${{ matrix.node }}" >> $GITHUB_OUTPUT

  summary:
    needs: test
    steps:
      - run: echo "${{ needs.test.outputs.version }}"
```

---

## Runner Operating Systems

| Name | OS | Shell | Cost | Speed |
|------|----|----|------|-------|
| `ubuntu-latest` | Linux | bash | $ | Fast |
| `ubuntu-20.04` | Linux | bash | $ | Fast |
| `windows-latest` | Windows | pwsh | $$ | Slower |
| `windows-2019` | Windows | pwsh | $$ | Slower |
| `macos-latest` | macOS | bash | $$$ | Medium |
| `macos-12` | Intel macOS | bash | $$ | Medium |

---

## Performance Optimization

| Strategy | Benefit | Method |
|----------|---------|--------|
| Parallel matrix | Linear speedup | Multiple dimensions |
| Fail-fast | Quick feedback | `fail-fast: true` |
| Selective test | Save runners | `exclude:` patterns |
| Cache per version | 60% faster | Include version in key |
| Skip unchanged | Save time | Use `paths:` filter |

---

## Common Mistakes

| Mistake | Wrong | Correct |
|--------|-------|---------|
| Missing colon | `matrix` | `matrix:` |
| Single value | `node: 18` | `node: [18]` |
| Wrong access | `${{ matrix }}` | `${{ matrix.node }}` |
| Typo in OS | `ubuntu` | `ubuntu-latest` |
| Missing dimension | `matrix: {node}` | `matrix: node: []` |

---

## Matrix Job Naming Template

```yaml
name: ${{ matrix.os }} / Node ${{ matrix.node }}

# Or more detailed
name: Build ${{ matrix.os }} (Node ${{ matrix.node }})

# Or with include attribute
name: Test Suite - ${{ matrix.config }}
```

---

## Quick Matrix Examples

### Python Versions

```yaml
strategy:
  matrix:
    python-version: ['3.8', '3.9', '3.10', '3.11']
```

### Java Versions

```yaml
strategy:
  matrix:
    java-version: ['11', '17', '20']
```

### Ruby Versions

```yaml
strategy:
  matrix:
    ruby-version: ['2.7', '3.0', '3.1', '3.2']
```

### Go Versions

```yaml
strategy:
  matrix:
    go-version: ['1.19', '1.20', '1.21']
```

---

## Matrix with Conditional Inclusion

```yaml
strategy:
  matrix:
    include:
      - os: ubuntu-latest
        node: 18
        experimental: false
      - os: macos-latest
        node: 20
        experimental: true
```

Then use in steps:

```yaml
- if: matrix.experimental == true
  run: echo "Running experimental tests"
```

---

## Debugging Matrix Issues

```yaml
- name: Debug Matrix
  run: |
    echo "OS: ${{ matrix.os }}"
    echo "Node: ${{ matrix.node }}"
    echo "Runner: ${{ runner.os }}"
    echo "Arch: ${{ runner.arch }}"
```

---

## Matrix with Docker

```yaml
strategy:
  matrix:
    container: ['node:16', 'node:18', 'node:20']

container:
  image: ${{ matrix.container }}
```

---

## Cost Optimization

**Reduce jobs:**
- Use `exclude:` for unnecessary combinations
- Reduce matrix dimensions
- Use selective runners

**Speed up:**
- Enable caching per version
- Use `fail-fast: true` for quick feedback
- Run expensive tests on single OS
- Cache node_modules

---

## Links and Resources

| Resource | URL |
|----------|-----|
| Matrix Docs | https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs |
| Runners | https://docs.github.com/en/actions/using-github-hosted-runners |
| Contexts | https://docs.github.com/en/actions/learn-github-actions/contexts |
| Expressions | https://docs.github.com/en/actions/learn-github-actions/expressions |

---

**Cheatsheet Version:** 1.0 | **Updated:** January 2026 | **Module:** 03-matrix-and-multi-os-ci
