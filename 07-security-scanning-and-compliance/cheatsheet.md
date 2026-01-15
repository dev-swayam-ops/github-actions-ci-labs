# Module 07 Cheatsheet: Security Scanning and Compliance

## Quick Reference Tables

### Dependency Scanning Tools

| Tool | Language | Command | Output | Time |
|------|----------|---------|--------|------|
| npm audit | Node.js | `npm audit` | JSON/text | <5s |
| pip audit | Python | `pip-audit` | JSON/text | <5s |
| gradle dependencyCheck | Java | `gradle dependencyCheck` | HTML/JSON | 10-30s |
| Snyk | Multi | `snyk test` | HTML/JSON | 5-15s |
| OWASP Dependency-Check | Multi | `dependency-check --project X` | HTML/JSON | 30-60s |

### Secret Detection Tools

| Tool | Method | Detection | Config | Use Case |
|------|--------|-----------|--------|----------|
| Gitleaks | Pattern matching | Git history | `.gitleaksignore` | Pre-commit/CI |
| TruffleHog | Entropy + pattern | Secrets in code | Custom regex | GitHub scan |
| git-secrets | Pre-commit hook | Known patterns | `.git/hooks/` | Local dev |
| detect-secrets | ML + pattern | Baseline diffs | `.secretsrc` | Large codebase |

### SAST/Code Quality Tools

| Tool | Languages | Type | Cost | Integration |
|------|-----------|------|------|-------------|
| CodeQL | Multi | Semantic SAST | Free in GH | Native GitHub |
| SonarQube | Multi | Quality gates | Free/Paid | REST API |
| ESLint | JavaScript | Linting | Free | npm script |
| Pylint | Python | Linting | Free | pip install |
| Checkstyle | Java | Code style | Free | Maven/Gradle |

### Container Scanning Tools

| Tool | Format | Speed | Accuracy | Best For |
|------|--------|-------|----------|----------|
| Trivy | SARIF/JSON | Very Fast | High | CI pipelines |
| Snyk Container | JSON | Fast | High | Docker images |
| Clair | JSON | Medium | Medium | Registry scanning |
| Anchore | JSON | Slow | Very High | Deep analysis |
| Grype | SARIF/JSON | Fast | High | OS vulnerabilities |

### Compliance Standards

| Standard | Focus | Domains | Frequency | Auditor |
|----------|-------|---------|-----------|---------|
| SOC 2 Type II | Operational controls | Security, availability | Annual | External |
| HIPAA | Healthcare data | Encryption, access | Continuous | Internal |
| PCI-DSS | Payment data | Encryption, network | Annual | QSA |
| GDPR | Data privacy | Consent, deletion | Continuous | Internal |
| ISO 27001 | Information security | All aspects | Annual | Auditor |

---

## Common Security Commands

### Dependency Scanning

```bash
# npm audit
npm audit                           # Basic audit
npm audit fix                       # Auto-fix vulnerabilities
npm audit --json > audit.json       # JSON output
npm audit --audit-level=moderate    # Set minimum severity

# pip-audit (Python)
pip-audit                           # Scan packages
pip-audit --skip-editable          # Skip editable installs
pip-audit --format json            # JSON output

# Maven (Java)
mvn dependency-check:check         # Run OWASP check
mvn verify                         # Include in build
```

### Secret Scanning

```bash
# Gitleaks
gitleaks detect --source . --verbose          # Scan directory
gitleaks detect --source . --report-path .    # Generate report
gitleaks protect --pre-commit                 # Setup pre-commit hook

# TruffleHog
truffleHog github --repo <user/repo> --json  # Scan GitHub repo
truffleHog filesystem . --json               # Scan filesystem

# git-secrets
git secrets --install                        # Setup hooks
git secrets --add <pattern>                  # Add pattern
```

### SAST/Code Quality

```bash
# ESLint
npx eslint . --format json > report.json     # Lint with report
npx eslint . --fix                          # Auto-fix issues

# Pylint
pylint src/ --output-format=json            # Python linting
pylint src/ --disable=C0111                 # Skip doc warnings

# CodeQL
codeql database create mydb --language=javascript  # Create DB
codeql database analyze mydb --format=sarif       # Analyze
```

### Container Scanning

```bash
# Trivy
trivy image myapp:latest                   # Scan image
trivy image --severity HIGH,CRITICAL myapp:latest
trivy image -f sarif > results.sarif       # SARIF output
trivy image --exit-code 1 myapp:latest     # Fail on findings

# Snyk Container
snyk container test myapp:latest           # Quick test
snyk container test --severity=high myapp  # By severity
snyk container test --json > report.json   # JSON output
```

### License Compliance

```bash
# npm
npm list --depth=0 --json                  # List licenses
npm audit --prod                           # Production deps only

# Python
pip-licenses --format=json                 # List all licenses
pip-licenses --with-urls                   # Include URLs

# FOSSA
fossa test                                 # Check compliance
fossa report --format json                 # Detailed report
```

---

## GitHub Actions Snippets

### Minimal Dependency Scan

```yaml
- uses: actions/setup-node@v3
  with:
    node-version: '18'
    cache: 'npm'
- run: npm ci
- run: npm audit --audit-level=moderate
```

### Minimal Secret Scan

```yaml
- uses: actions/checkout@v3
  with:
    fetch-depth: 0
- uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Minimal CodeQL SAST

```yaml
- uses: actions/checkout@v3
- uses: github/codeql-action/init@v2
  with:
    languages: 'javascript'
- uses: github/codeql-action/autobuild@v2
- uses: github/codeql-action/analyze@v2
```

### Minimal Container Scan

```yaml
- run: docker build -t myapp:latest .
- run: docker save myapp:latest -o image.tar
- uses: aquasecurity/trivy-action@master
  with:
    input: 'image.tar'
    severity: 'CRITICAL,HIGH'
```

---

## Security Gating Pattern

```yaml
jobs:
  security-checks:
    # Run all security scans
    
  gate:
    needs: security-checks
    if: success()
    runs-on: ubuntu-latest
    steps:
      - run: echo "Security gate passed"
  
  deploy:
    needs: gate
    if: success()
    # Deploy to production
```

**Key Concept:** The `needs` and `if` keywords create sequential gates.

---

## Environment Variables for Security

```bash
# Secrets to add in GitHub Settings → Secrets
GITHUB_TOKEN           # Automatic (available)
SNYK_TOKEN            # From snyk.io
SONARQUBE_HOST        # From SonarQube instance
SONARQUBE_TOKEN       # SonarQube authentication
REGISTRY_USERNAME     # Private registry access
REGISTRY_PASSWORD     # Private registry access

# Set in workflow
env:
  AUDIT_LEVEL: moderate
  SCAN_TIMEOUT: 600
  FAIL_ON_WARNING: true
```

---

## Permissions Required

```yaml
permissions:
  contents: read        # Read repo contents
  security-events: write # Upload SARIF results
  pull-requests: write  # Comment on PRs
  issues: write         # Create security issues
```

---

## SARIF Format (Security Results)

```json
{
  "version": "2.1.0",
  "runs": [
    {
      "tool": {
        "driver": {
          "name": "CodeQL",
          "version": "2.9.0"
        }
      },
      "results": [
        {
          "ruleId": "js/sql-injection",
          "message": {
            "text": "SQL injection vulnerability"
          },
          "locations": [
            {
              "physicalLocation": {
                "artifactLocation": {
                  "uri": "src/api.js"
                },
                "region": {
                  "startLine": 45
                }
              }
            }
          ],
          "level": "warning"
        }
      ]
    }
  ]
}
```

---

## Common Issues & Fixes

| Issue | Cause | Solution |
|-------|-------|----------|
| "npm audit fix not working" | Peer dependency conflict | Use `--force` or `--legacy-peer-deps` |
| "Gitleaks false positives" | Pattern too broad | Add patterns to `.gitleaksignore` |
| "CodeQL timeout" | Large codebase | Reduce `autobuild` scope |
| "Trivy failing on base image" | Vulnerable base | Update Dockerfile `FROM` image |
| "License check too strict" | Approved license missing | Add to approved list |
| "SARIF upload failing" | Format error | Validate with SARIF validator |
| "Secret scan missing secrets" | Entropy too high | Adjust detector sensitivity |
| "Container scan slow" | Large layers | Use multi-stage Dockerfile |

---

## Pro Tips

### 1. Cache Scan Results
```yaml
- uses: actions/cache@v3
  with:
    path: ~/.cache/trivy
    key: trivy-${{ github.run_id }}
```

### 2. Skip Specific Vulnerabilities
```yaml
# .trivyignore
AVD-AWS-0001  # Your approved exception
```

### 3. Only Scan Changed Files
```bash
# Get changed files
git diff --name-only origin/main..HEAD > changed.txt
# Pass to scanner
npm audit --json > report.json
```

### 4. Use Service Container for Scans
```yaml
services:
  sonarqube:
    image: sonarqube:latest
    options: >-
      --health-cmd "curl http://localhost:9000"
```

### 5. Create Security Issue on Failure
```yaml
- uses: actions/github-script@v6
  if: failure()
  with:
    script: |
      github.rest.issues.create({
        owner: context.repo.owner,
        repo: context.repo.repo,
        title: 'Security scan failed',
        body: 'Check workflow for details'
      })
```

---

## Security Checklist

- [ ] Dependencies scanned for vulnerabilities
- [ ] No secrets in repository history
- [ ] SAST analysis configured
- [ ] Code quality gates enforced
- [ ] Container images scanned before push
- [ ] License compliance verified
- [ ] Security gates block deployments
- [ ] SBOM generated for releases
- [ ] Scheduled scans configured
- [ ] Security dashboard created
- [ ] Compliance standards documented
- [ ] Incident response plan defined

---

## Resource Links

**Tools:**
- npm audit: https://docs.npmjs.com/cli/audit
- Gitleaks: https://github.com/gitleaks/gitleaks
- CodeQL: https://codeql.github.com
- Trivy: https://github.com/aquasecurity/trivy
- SonarQube: https://www.sonarqube.org

**Standards:**
- OWASP Top 10: https://owasp.org/Top10
- CWE: https://cwe.mitre.org
- CVE: https://cve.mitre.org
- CVSS: https://www.first.org/cvss

**GitHub Docs:**
- Code scanning: https://docs.github.com/en/code-security/code-scanning
- Secret scanning: https://docs.github.com/en/code-security/secret-scanning
- Dependabot: https://docs.github.com/en/code-security/dependabot
