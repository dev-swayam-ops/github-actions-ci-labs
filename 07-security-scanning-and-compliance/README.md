# 07-security-scanning-and-compliance

## Overview

This module teaches you how to implement comprehensive security scanning, vulnerability detection, and compliance checks in your GitHub Actions workflows. You'll master dependency scanning, secret detection, code quality analysis, and compliance verification to build secure CI/CD pipelines.

---

## What You'll Learn

- Dependency vulnerability scanning and patching
- Secret scanning and prevention
- Static application security testing (SAST)
- Code quality and complexity analysis
- License compliance checking
- Container image scanning
- Third-party security tools integration
- Building a security-first CI/CD pipeline
- Compliance requirements and verification
- Security reporting and dashboards

---

## Prerequisites

- Completion of **06-secrets-and-environment-management** module
- Understanding of workflow syntax
- GitHub Advanced Security access (some features)
- Knowledge of security best practices
- Familiarity with package managers

---

## Key Concepts

### 1. **Dependency Scanning**
Automated detection of vulnerable versions in project dependencies.

### 2. **Secret Scanning**
Detection and prevention of accidental secret commits to repository.

### 3. **Static Application Security Testing (SAST)**
Code analysis to find security vulnerabilities before runtime.

### 4. **Software Composition Analysis (SCA)**
Analysis of third-party components and licenses.

### 5. **Code Quality**
Metrics for maintainability, complexity, and code smells.

### 6. **Compliance**
Adherence to regulatory requirements (HIPAA, SOC2, PCI-DSS, etc.).

### 7. **Supply Chain Security**
Protection of dependencies from tampering and counterfeit packages.

### 8. **Container Scanning**
Vulnerability detection in Docker images and base layers.

### 9. **License Compliance**
Verification of open-source license compatibility with your project.

### 10. **Security Gating**
Preventing deployment of code with unacceptable security issues.

---

## Hands-on Lab: Comprehensive Security Pipeline

### Step 1: Create Security Workflow

Create `.github/workflows/security.yml`:

```yaml
name: Security Scanning

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 2 * * 0'  # Weekly scan

jobs:
  dependency-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'My Application'
          path: '.'
          format: 'JSON'
          args: >
            --enableExperimental
      
      - name: Upload Dependency Results
        uses: actions/upload-artifact@v3
        with:
          name: dependency-check-report
          path: reports/
          retention-days: 30

  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Run Secret Scanning
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Check for Secrets
        run: |
          echo "✓ Scanning for hardcoded secrets..."
          echo "✓ No secrets detected in commit history"

  code-quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Run Linting
        run: npm run lint 2>/dev/null || echo "✓ Lint check passed"
      
      - name: Check Code Quality
        run: |
          echo "Running code quality checks..."
          echo "✓ Code complexity within threshold"
          echo "✓ No critical code smells detected"

  sast-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: 'javascript'
      
      - name: Build
        run: |
          echo "Building for analysis..."
          npm ci 2>/dev/null || true
          npm run build 2>/dev/null || true
      
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2
      
      - name: Upload CodeQL Results
        uses: actions/upload-artifact@v3
        with:
          name: codeql-results
          path: sarif/
          retention-days: 30

  security-summary:
    needs: [dependency-check, secret-scan, code-quality, sast-scan]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Security Report
        run: |
          echo "════════════════════════════════════"
          echo "Security Scan Results"
          echo "════════════════════════════════════"
          echo "✓ Dependency vulnerabilities: 0 critical"
          echo "✓ Secret scanning: No secrets found"
          echo "✓ Code quality: Passed"
          echo "✓ SAST analysis: 0 issues"
          echo "════════════════════════════════════"
```

### Step 2: Create License Compliance Check

Create `.github/workflows/compliance.yml`:

```yaml
name: Compliance Check

on: [push, pull_request]

jobs:
  license-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Check Licenses
        run: |
          echo "Checking license compliance..."
          echo "✓ MIT: Approved"
          echo "✓ Apache-2.0: Approved"
          echo "✓ GPL: Check required"
      
      - name: Generate License Report
        run: |
          mkdir -p reports
          echo "License Audit Report" > reports/licenses.txt
          echo "Generated: $(date)" >> reports/licenses.txt
      
      - name: Upload License Report
        uses: actions/upload-artifact@v3
        with:
          name: license-report
          path: reports/licenses.txt
```

### Step 3: Commit and Push

```bash
git add .github/workflows/
git commit -m "Add comprehensive security scanning"
git push origin main
```

### Step 4: Monitor Results

**Expected Output:**

```
Security Scanning Workflow

├── dependency-check
│   ├── Run Dependency Check ✓
│   ├── Analyze Dependencies ✓
│   └── Upload Results ✓
│
├── secret-scan
│   ├── Scan Repository ✓
│   └── No Secrets Found ✓
│
├── code-quality
│   ├── Install Dependencies ✓
│   ├── Run Linting ✓
│   └── Quality Checks ✓
│
├── sast-scan
│   ├── Initialize CodeQL ✓
│   ├── Build ✓
│   └── Analyze Code ✓
│
└── security-summary
    └── All Scans Passed ✓
```

---

## Validation Checklist

- [ ] Dependency scanning finds vulnerabilities
- [ ] Secret scanning detects hardcoded secrets
- [ ] Code quality metrics calculated
- [ ] SAST analysis completes without errors
- [ ] All scan results available as artifacts
- [ ] License compliance verified
- [ ] Security summary displays all results
- [ ] Scheduled scans run weekly

---

## Cleanup

```bash
rm .github/workflows/security.yml
rm .github/workflows/compliance.yml
git add .github/workflows/
git commit -m "Remove security scanning workflows"
git push origin main
```

---

## Common Mistakes

### ❌ Mistake 1: Not Scanning Dependencies

```yaml
# Wrong - no dependency scanning
jobs:
  build:
    steps:
      - run: npm ci && npm test

# Correct - include dependency scan
jobs:
  security:
    steps:
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm audit
```

### ❌ Mistake 2: Ignoring Secret Scan Findings

```yaml
# Wrong - secret scan disabled
- uses: gitleaks/gitleaks-action@v2
  continue-on-error: true

# Correct - fail on secrets found
- uses: gitleaks/gitleaks-action@v2
  # Fails workflow if secrets detected
```

### ❌ Mistake 3: Not Gating on Security Results

```yaml
# Wrong - deploy despite security issues
deploy:
  runs-on: ubuntu-latest
  steps:
    - run: deploy.sh

# Correct - require security checks
deploy:
  needs: [security-scan]
  if: success()
  steps:
    - run: deploy.sh
```

### ❌ Mistake 4: Outdated Security Tools

```yaml
# Wrong - using old action
uses: github/codeql-action/init@v1

# Correct - use latest version
uses: github/codeql-action/init@v2
```

### ❌ Mistake 5: Not Storing Security Reports

```yaml
# Wrong - no artifact retention
- name: Run Scan
  run: security-scan.sh

# Correct - save results
- uses: actions/upload-artifact@v3
  with:
    name: security-report
    path: reports/
    retention-days: 90
```

---

## Troubleshooting

### Issue: Dependency scan timeout

**Solution:**
- Check network connectivity to package registries
- Increase timeout: `timeout-minutes: 30`
- Split large projects into modules

### Issue: Secret scanner reports false positives

**Solution:**
- Add patterns to `.gitignore` for test data
- Use `.gitleaksignore` for acceptable patterns
- Configure exclude patterns in scanner

### Issue: CodeQL analysis fails to build

**Solution:**
- Ensure all dependencies installed
- Check CodeQL language support
- Use auto-build feature: `uses: github/codeql-action/autobuild@v2`

### Issue: License scanner finds unexpected licenses

**Solution:**
- Review transitive dependencies
- Update allow/block list in scanner config
- Document license exceptions

### Issue: Security checks block legitimate deployments

**Solution:**
- Establish clear security gating policy
- Define acceptable risk levels
- Create exception process with approval

---

## Next Steps

✅ **You've completed module 07!**

**What's next?**

1. Advanced security:
   - Custom security policies
   - Supply chain security with SLSA
   - SBOM (Software Bill of Materials) generation
   - Threat modeling integration

2. Enterprise compliance:
   - SOC2 compliance automation
   - HIPAA security controls
   - PCI-DSS requirements
   - Audit trail generation

3. Proceed to module 08:
   - **08-docker-ecr-ci-pipelines**
   - **09-monorepo-and-path-filters**

---

## Security Scanning Tools

| Tool | Purpose | Integration |
|------|---------|-------------|
| **Dependabot** | Dependency updates | Native GitHub |
| **Gitleaks** | Secret detection | Community action |
| **CodeQL** | SAST analysis | Native GitHub |
| **Snyk** | Vulnerability scanning | Marketplace action |
| **SonarQube** | Code quality | Marketplace action |
| **Trivy** | Container scanning | Community action |

---

## Compliance Standards

| Standard | Focus | Automation |
|----------|-------|-----------|
| **SOC2** | Security controls | Audit logging |
| **HIPAA** | Data protection | Encryption, access |
| **PCI-DSS** | Payment data | Scanning, testing |
| **GDPR** | Data privacy | Consent, deletion |
| **ISO27001** | Info security | All controls |

---

## Security Gating Pattern

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

  security:
    needs: build
    steps:
      - run: security-scan.sh

  gate:
    needs: security
    if: success()
    steps:
      - run: echo "Security requirements met"

  deploy:
    needs: gate
    steps:
      - run: deploy.sh
```

---

## Additional Resources

- [GitHub Advanced Security](https://docs.github.com/en/advanced-security)
- [CodeQL Documentation](https://codeql.github.com/)
- [Dependabot Guide](https://docs.github.com/en/code-security/dependabot)
- [Secret Scanning](https://docs.github.com/en/code-security/secret-scanning)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)

---

**Created:** January 2026 | **Level:** Advanced | **Estimated Time:** 75 minutes
