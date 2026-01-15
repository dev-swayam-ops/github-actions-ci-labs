# Cheatsheet: Build and Test Pipelines

Quick reference for building and testing CI/CD pipelines with GitHub Actions.

---

## Build Tools Setup

### Node.js Setup with Cache

```yaml
- uses: actions/setup-node@v3
  with:
    node-version: '18.x'
    cache: 'npm'
```

### Python Setup with Cache

```yaml
- uses: actions/setup-python@v4
  with:
    python-version: '3.11'
    cache: 'pip'
```

### Java Setup

```yaml
- uses: actions/setup-java@v3
  with:
    java-version: '17'
    distribution: 'temurin'
    cache: 'maven'
```

### Go Setup

```yaml
- uses: actions/setup-go@v4
  with:
    go-version: '1.20'
    cache: true
```

---

## Build Commands Reference

| Language | Tool | Install | Build | Test |
|----------|------|---------|-------|------|
| Node.js | npm | `npm ci` | `npm run build` | `npm test` |
| Node.js | yarn | `yarn install --frozen-lockfile` | `yarn build` | `yarn test` |
| Python | pip | `pip install -r requirements.txt` | `python -m build` | `pytest` |
| Java | Maven | `mvn dependency:resolve` | `mvn package` | `mvn test` |
| Go | go mod | `go mod download` | `go build` | `go test ./...` |
| Rust | cargo | `cargo fetch` | `cargo build --release` | `cargo test` |

---

## Testing Frameworks

### JavaScript/Node.js

```bash
# Jest
npm test                    # Run tests
npm test -- --coverage      # With coverage
npm test -- --watch         # Watch mode

# Mocha
npm test                    # Basic test
npm test -- --reporter json # JSON output

# Vitest
npm test                    # Run tests
npm test -- --coverage      # Coverage report
```

### Python

```bash
# pytest
pytest                      # Run all tests
pytest --cov               # With coverage
pytest -v                  # Verbose
pytest path/to/tests       # Specific tests

# unittest
python -m unittest discover # Run all tests
python -m unittest TestClass # Run specific class
```

### Java

```bash
# JUnit with Maven
mvn test                   # Run tests
mvn test -Dtest=TestClass # Specific test
mvn jacoco:report         # Coverage report
```

---

## Code Coverage

### JavaScript (Jest)

```bash
npm test -- --coverage

# Output:
# Statements   : 85% ( 17/20 )
# Branches     : 80% (  8/10 )
# Functions    : 90% (  9/10 )
# Lines        : 85% ( 17/20 )
```

### Python (pytest)

```bash
pip install pytest-cov
pytest --cov=src --cov-report=html
```

### Java (JaCoCo)

```xml
<!-- pom.xml -->
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
</plugin>
```

---

## Docker in CI/CD

### Setup Docker Buildx

```yaml
- uses: docker/setup-buildx-action@v2
```

### Build and Push

```yaml
- uses: docker/build-push-action@v4
  with:
    context: .
    push: true
    tags: ghcr.io/owner/repo:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Login to Registry

```yaml
- uses: docker/login-action@v2
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

---

## Linting and Code Quality

### JavaScript

```bash
# ESLint
npm run lint              # Run linter
npm run lint -- --fix     # Auto-fix

# Prettier
npm run format            # Format code
npm run format:check      # Check formatting
```

### Python

```bash
# pylint
pylint src/

# flake8
flake8 src/

# black
black --check src/
```

### General Checks

```yaml
- name: Lint Check
  run: npm run lint

- name: Format Check
  run: npm run format:check

- name: Security Scan
  run: npm audit
```

---

## Artifact Management

### Upload Artifacts

```yaml
- uses: actions/upload-artifact@v3
  with:
    name: build-output
    path: dist/
    retention-days: 5
```

### Download Artifacts

```yaml
- uses: actions/download-artifact@v3
  with:
    name: build-output
    path: ./download-dir/
```

### Multiple Artifacts

```yaml
# Upload multiple
- uses: actions/upload-artifact@v3
  with:
    name: all-artifacts
    path: |
      dist/
      build/
      coverage/

# Download all
- uses: actions/download-artifact@v3
  with:
    path: artifacts/
```

---

## Caching Dependencies

### Automatic Cache (Recommended)

```yaml
- uses: actions/setup-node@v3
  with:
    node-version: '18'
    cache: 'npm'       # Also: pip, maven, gradle, cargo
```

### Manual Cache

```yaml
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
      ${{ runner.os }}-
```

### Cache Multiple Paths

```yaml
path: |
  ~/.npm
  node_modules
  dist/
```

---

## Job Dependencies

### Sequential Execution

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
  
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: npm test
  
  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy"
```

### Share Data Between Jobs

```yaml
jobs:
  build:
    outputs:
      version: ${{ steps.meta.outputs.version }}
    steps:
      - id: meta
        run: echo "version=1.0.0" >> $GITHUB_OUTPUT
  
  deploy:
    needs: build
    steps:
      - run: echo ${{ needs.build.outputs.version }}
```

---

## Matrix Strategy

### Basic Matrix

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [16, 18, 20]
    # Creates 6 jobs: 2 OS × 3 node versions
```

### Include/Exclude

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [16, 18, 20]
    include:
      - os: macos-latest
        node: 18
    exclude:
      - os: windows-latest
        node: 16
```

### Dynamic Values

```yaml
steps:
  - run: |
      echo "OS: ${{ runner.os }}"
      echo "Node: ${{ matrix.node }}"
      echo "Arch: ${{ runner.arch }}"
```

---

## Conditional Execution

### Run on Specific Conditions

```yaml
steps:
  - if: github.ref == 'refs/heads/main'
    run: echo "Running on main"
  
  - if: github.event_name == 'push'
    run: echo "Running on push"
  
  - if: success()
    run: echo "Previous step succeeded"
  
  - if: failure()
    run: echo "Previous step failed"
  
  - if: always()
    run: echo "Always run"
```

### Job-Level Conditions

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy"
```

---

## Security Scanning

### Dependency Audit

```bash
npm audit                    # List vulnerabilities
npm audit fix               # Auto-fix
npm audit --production      # Production deps only
npm audit --json           # JSON format
```

### Container Scanning

```yaml
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.IMAGE_NAME }}
    format: 'sarif'
    output: 'trivy-results.sarif'
```

### SAST Scanning

```yaml
- uses: github/super-linter@v4
  with:
    DEFAULT_BRANCH: main
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Deployment Patterns

### Deploy to Main Branch Only

```yaml
deploy:
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  needs: [test, security]
  environment: production
  runs-on: ubuntu-latest
  steps:
    - run: echo "Deploy to production"
```

### Approve Deployment

```yaml
deploy:
  environment:
    name: production
    deployment_branch_policy:
      protected_branches: true
```

### Conditional Files

```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'package.json'
      - '.github/workflows/**'
```

---

## Performance Tips

| Optimization | Impact | Method |
|--------------|--------|--------|
| Use cache | 50-70% faster | `cache: 'npm'` in setup |
| Parallel jobs | Linear scaling | Matrix strategy |
| Container cache | 30-40% faster | `cache-from: type=gha` |
| Slim images | Smaller build | Alpine base image |
| Selective paths | Skip runs | `paths:` filter |

---

## Workflow Template

```yaml
name: Build and Deploy

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '18'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci && npm run lint

  build:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci && npm run build
      - uses: actions/upload-artifact@v3
        with:
          name: build
          path: dist/

  test:
    needs: build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: ['16', '18', '20']
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      - uses: actions/download-artifact@v3
        with:
          name: build
      - run: npm ci && npm test

  security:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci && npm audit

  deploy:
    needs: security
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: build
      - run: echo "Deploying..."
```

---

## Troubleshooting Guide

| Problem | Solution |
|---------|----------|
| Cache not working | Verify correct path, ensure files exist |
| Build timeout | Increase `timeout-minutes`, optimize build |
| Artifact not found | Verify upload before download, check name |
| Tests fail intermittently | Add timeouts, check for race conditions |
| Docker push fails | Verify registry login, check token permissions |
| Caching artifact issues | Use unique artifact names per job/matrix |
| Conditional job not running | Check `if:` condition syntax and values |

---

## Links and Resources

| Resource | URL |
|----------|-----|
| Setup Actions | https://github.com/actions/setup-* |
| Testing Tools | https://docs.github.com/en/actions/automating-builds-and-tests |
| Docker Docs | https://docs.docker.com/ |
| Coverage Tools | https://codecov.io/ |

---

**Cheatsheet Version:** 1.0 | **Updated:** January 2026 | **Module:** 02-build-and-test-pipelines
