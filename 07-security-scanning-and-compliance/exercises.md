# Module 07 Exercises: Security Scanning and Compliance

## Exercise 1: Dependency Vulnerability Scanning

**Objective:** Scan project dependencies for known vulnerabilities

**Acceptance Criteria:**
- Workflow uses npm audit or dependency scanner
- Scans for vulnerabilities in dependencies
- Reports vulnerabilities with severity levels
- Fails on critical vulnerabilities
- Generates vulnerability report artifact

**Starter Code:**
```yaml
name: Dependency Scan

on: [push, pull_request]

jobs:
  dependency-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - name: Audit Dependencies
        # Add npm audit or dependency check command
        run: 
      - name: Upload Report
        uses: actions/upload-artifact@v3
        with:
          # Add artifact configuration
```

**Expected Output:**
Dependency audit report with vulnerability severity levels

---

## Exercise 2: Secret Scanning

**Objective:** Detect and prevent hardcoded secrets in codebase

**Acceptance Criteria:**
- Workflow scans repository history
- Detects hardcoded secrets (API keys, tokens, passwords)
- Prevents secrets from being committed
- Reports findings with line numbers
- Blocks deployment if secrets found

**Starter Code:**
```yaml
name: Secret Scan

on: [push, pull_request]

jobs:
  secret-detection:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - name: Scan for Secrets
        # Add secret scanning action
        uses: 
      - name: Report Results
        # Add result reporting step
        run: 
```

**Expected:** Gitleaks or equivalent tool scans and reports any found secrets

---

## Exercise 3: Static Application Security Testing (SAST)

**Objective:** Analyze code for security vulnerabilities using SAST tools

**Acceptance Criteria:**
- CodeQL initialized for code analysis
- Code built for analysis
- SAST analysis completes
- Results uploaded as SARIF format
- Vulnerabilities categorized by severity
- Results displayed in Security tab

**Starter Code:**
```yaml
name: SAST Analysis

on: [push, pull_request]

jobs:
  codeql:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Initialize CodeQL
        # Add CodeQL init step
        uses: 
      - name: Build
        # Add build command
        run: 
      - name: Analyze
        # Add CodeQL analyze step
        uses: 
```

**Expected:** CodeQL analysis with vulnerability detection and SARIF upload

---

## Exercise 4: Code Quality Metrics

**Objective:** Measure and enforce code quality standards

**Acceptance Criteria:**
- Code complexity analyzed
- Coverage metrics collected
- Code smells identified
- Quality gate enforced
- Metrics reported in artifacts
- Fails build if quality degrades

**Starter Code:**
```yaml
name: Code Quality

on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - name: Run Linter
        # Add eslint or similar
        run: 
      - name: Measure Coverage
        # Add coverage tool
        run: 
      - name: Quality Gate
        # Add pass/fail logic
        run: 
```

**Expected:** Quality metrics and enforcement of minimum thresholds

---

## Exercise 5: Container Image Scanning

**Objective:** Scan Docker images for vulnerabilities before deployment

**Acceptance Criteria:**
- Docker image built
- Scanned for vulnerabilities
- Base image vulnerabilities detected
- Severity-based reporting
- Critical vulnerabilities block deployment
- Scan results stored as artifact

**Starter Code:**
```yaml
name: Container Scan

on: [push]

jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker Image
        run: docker build -t myapp:latest .
      - name: Scan Image
        # Add container scanning tool
        uses: 
      - name: Report Vulnerabilities
        # Add vulnerability reporting
        run: 
```

**Expected:** Container vulnerability scan with critical issue detection

---

## Exercise 6: License Compliance Checking

**Objective:** Verify open-source licenses comply with policy

**Acceptance Criteria:**
- Dependencies listed with licenses
- License compatibility verified
- Unapproved licenses detected
- License report generated
- Fails on non-compliant licenses
- Exception process documented

**Starter Code:**
```yaml
name: License Check

on: [push, pull_request]

jobs:
  license-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - name: Audit Licenses
        # Add license check command
        run: 
      - name: Generate Report
        # Create license report
        run: 
      - name: Upload Report
        uses: actions/upload-artifact@v3
        with:
          # Add artifact for license report
```

**Expected:** License audit report with compliance status

---

## Exercise 7: Security Gating (Blocking on Issues)

**Objective:** Prevent deployment when security issues exist

**Acceptance Criteria:**
- Security checks run before deployment
- Deployment blocked on critical findings
- Clear gating criteria defined
- Exception process available
- Approval required to override gate
- Audit trail of exceptions

**Starter Code:**
```yaml
name: Security Gate

on: [push]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - run: npm audit
      - run: npm run lint
      - name: Security Checks
        # Security checks here
        run: 

  gate:
    needs: security-scan
    # Add condition to block on failure
    if: 
    runs-on: ubuntu-latest
    steps:
      - run: echo "Security requirements met"

  deploy:
    needs: gate
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying..."
```

**Expected:** Deployment only proceeds if security gate passes

---

## Exercise 8: SBOM Generation

**Objective:** Generate Software Bill of Materials for supply chain security

**Acceptance Criteria:**
- SBOM generated in standard format (SPDX, CycloneDX)
- All dependencies listed with versions
- Dependency relationships captured
- SBOM uploaded as artifact
- SBOM included in release
- Machine-readable format

**Starter Code:**
```yaml
name: SBOM Generation

on: [push, release]

jobs:
  generate-sbom:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Generate SBOM
        # Add SBOM generation tool
        uses: 
      - name: Upload SBOM
        uses: actions/upload-artifact@v3
        with:
          # Add SBOM artifact configuration
```

**Challenge:** Generate valid SPDX or CycloneDX format

---

## Exercise 9: Scheduled Security Scans

**Objective:** Run security scans on schedule for continuous monitoring

**Acceptance Criteria:**
- Workflow scheduled to run weekly
- All security scans execute
- Results compared to baseline
- Alerts sent on new vulnerabilities
- Trends tracked over time
- Historical data retained

**Starter Code:**
```yaml
name: Scheduled Security Scan

on:
  schedule:
    # Add cron schedule for weekly scans
    - cron: 

jobs:
  nightly-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Full Security Scan
        # Run comprehensive security checks
        run: 
      - name: Compare Baseline
        # Compare to previous scan
        run: 
      - name: Alert on Changes
        # Notify if new issues found
        run: 
```

**Expected:** Workflow runs automatically on schedule, reports differences

---

## Exercise 10: Security Dashboard and Reporting

**Objective:** Create security dashboard and aggregated reporting

**Acceptance Criteria:**
- All security scan results aggregated
- Dashboard summary generated
- Trends visualized
- Compliance status tracked
- Executive summary created
- Alert on policy violations

**Starter Code:**
```yaml
name: Security Dashboard

on:
  workflow_run:
    workflows: ['Security Scanning']
    types: [completed]

jobs:
  aggregate-results:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Collect Scan Results
        # Download all scan artifacts
        run: 
      - name: Generate Dashboard
        # Create dashboard/summary
        run: 
      - name: Create Report
        # Generate comprehensive report
        run: 
      - name: Send Notification
        # Alert stakeholders
        run: 
```

**Challenge:** Build comprehensive security dashboard

---

## Summary

**Key Learning Points:**
- ✅ Dependency vulnerability scanning
- ✅ Secret detection and prevention
- ✅ Static application security testing (SAST)
- ✅ Code quality and complexity analysis
- ✅ License compliance checking
- ✅ Container image scanning
- ✅ Security gating and enforcement
- ✅ SBOM generation for supply chain
- ✅ Scheduled continuous scanning
- ✅ Security dashboards and reporting

**Next Steps:**
- Implement all security checks in CI/CD
- Configure escalation for critical issues
- Build exception process for violations
- Integrate with incident response

**Continue to Exercise Solutions!**
