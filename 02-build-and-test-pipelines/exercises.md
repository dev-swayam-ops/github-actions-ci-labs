# Exercises: Build and Test Pipelines

Master CI/CD pipeline creation with real-world build and testing scenarios.

---

## Exercise 1: Basic Build Pipeline (Easy)

**Objective:** Create a simple build pipeline for a Node.js project.

**Instructions:**
1. Create a simple Node.js project structure
2. Create `.github/workflows/ex1-build.yml`
3. Implement workflow with:
   - Checkout code
   - Setup Node.js 18.x
   - Install dependencies with `npm ci`
   - Run build with `npm run build`
4. Add output artifact upload
5. Verify build completes successfully

**Expected Workflow:**
```
Checkout → Setup Node → Install → Build → Upload Artifacts
```

**Acceptance Criteria:**
- Workflow completes in under 1 minute
- Artifacts are uploaded and downloadable
- No errors or warnings

---

## Exercise 2: Multi-Language Testing (Medium)

**Objective:** Test application across multiple Node.js versions.

**Instructions:**
1. Create `.github/workflows/ex2-multiversion.yml`
2. Use matrix strategy with Node versions: [16, 18, 20]
3. For each version:
   - Setup Node version
   - Install dependencies
   - Run tests: `npm test`
   - Show Node version in output
4. Push and verify 3 jobs are created
5. Ensure all versions pass tests

**Hint:** Use matrix with `node: [16, 18, 20]`

**Acceptance Criteria:**
- 3 separate jobs created
- Each tests with correct Node version
- All jobs pass successfully

---

## Exercise 3: Test Coverage Reporting (Medium)

**Objective:** Add code coverage tracking to tests.

**Instructions:**
1. Create `.github/workflows/ex3-coverage.yml`
2. Setup test project with coverage capability
3. Run tests with coverage: `npm test -- --coverage`
4. Create coverage report file
5. Upload coverage as artifact
6. Display coverage percentage in logs

**Sample Coverage Report:**
```
Statements   : 85% (17/20)
Branches     : 80% (8/10)
Functions    : 90% (9/10)
Lines        : 85% (17/20)
```

**Acceptance Criteria:**
- Coverage report generates
- Percentage displayed in logs
- Report available as artifact
- Meets 80%+ coverage target

---

## Exercise 4: Linting and Code Quality (Medium)

**Objective:** Add linting step to catch code quality issues.

**Instructions:**
1. Create `.github/workflows/ex4-lint.yml`
2. Setup Node.js with npm dependencies
3. Add linting step (ESLint or similar)
4. Fail workflow if linting fails
5. Add formatting check (Prettier)
6. Document linting rules used

**Sample Commands:**
```bash
npm run lint      # Run ESLint
npm run format    # Check formatting
```

**Acceptance Criteria:**
- Linting runs without errors
- Formatting validation passes
- Workflow fails on lint errors
- Clear error messages in logs

---

## Exercise 5: Docker Build and Push (Medium)

**Objective:** Build Docker image and push to registry.

**Instructions:**
1. Create `.github/workflows/ex5-docker.yml`
2. Create Dockerfile for simple app
3. Setup Docker Buildx
4. Build Docker image with:
   - Semantic versioning tag
   - Latest tag
   - Git SHA tag
5. Push to Docker registry (use test registry)
6. Log build time and image size

**Tag Strategy:**
```
ghcr.io/owner/repo:v1.0.0
ghcr.io/owner/repo:latest
ghcr.io/owner/repo:abc123def
```

**Acceptance Criteria:**
- Docker image builds successfully
- Multiple tags are applied
- Build time is logged
- Workflow completes in under 3 minutes

---

## Exercise 6: Build Artifact Versioning (Medium)

**Objective:** Version and manage build artifacts.

**Instructions:**
1. Create `.github/workflows/ex6-versioning.yml`
2. Generate semantic version number
3. Create build with version in filename: `app-v1.0.0.tar.gz`
4. Include version metadata (build date, commit SHA)
5. Upload versioned artifact
6. Keep only last 5 versions (retention policy)

**Version Format:**
```
major.minor.patch-buildnumber
Example: 1.0.0-42
```

**Acceptance Criteria:**
- Version format is consistent
- Build artifacts include version
- Metadata file created with build info
- Retention policy is effective

---

## Exercise 7: Integration Tests (Medium)

**Objective:** Run integration tests after successful build.

**Instructions:**
1. Create `.github/workflows/ex7-integration.yml`
2. Build application first
3. Start mock services (echo mock responses)
4. Run integration tests against services
5. Cleanup services
6. Report test results
7. Test depends on build job

**Test Sequence:**
```
Build → Start Services → Run Tests → Cleanup → Report
```

**Acceptance Criteria:**
- Build completes first
- Services start successfully
- Tests run against services
- Results are reported
- Cleanup executes

---

## Exercise 8: Caching Strategy (Medium)

**Objective:** Implement effective dependency caching.

**Instructions:**
1. Create `.github/workflows/ex8-caching.yml`
2. First run: Install dependencies, measure time
3. Use cache with key based on lock file
4. Second run: Verify cache hit
5. Show time difference in logs
6. Document cache strategy

**Cache Configuration:**
```yaml
key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
restore-keys: |
  ${{ runner.os }}-npm-
```

**Acceptance Criteria:**
- First run installs dependencies
- Second run uses cache (cache hit)
- Time savings documented (50%+ faster)
- Cache invalidates on lock file change

---

## Exercise 9: Security Scanning (Medium)

**Objective:** Add security vulnerability scanning.

**Instructions:**
1. Create `.github/workflows/ex9-security.yml`
2. Run dependency audit: `npm audit`
3. Scan for known vulnerabilities
4. Generate security report
5. Fail workflow on high-severity issues
6. Document findings

**Security Checks:**
```bash
npm audit                      # Dependency vulnerabilities
npm audit --production         # Production deps only
npm audit fix                  # Auto-fix if possible
```

**Acceptance Criteria:**
- Security scan completes
- Vulnerabilities identified
- Report is generated
- Workflow fails on critical issues
- Audit results are logged

---

## Exercise 10: Complete Production Pipeline (Medium)

**Objective:** Build full CI/CD pipeline with all features.

**Instructions:**
1. Create `.github/workflows/ex10-complete.yml`
2. Implement 5 stages:
   - **Lint:** Code quality check
   - **Build:** Compile application
   - **Test:** Run unit and integration tests
   - **Security:** Scan for vulnerabilities
   - **Deploy:** Deploy to staging (main branch only)
3. Use job dependencies: Lint → Build → Test → Security → Deploy
4. Add artifacts between stages
5. Include full logging and notifications
6. Matrix tests across 2 Node versions

**Pipeline Flow:**
```
Lint ──→ Build ──→ Test ──→ Security ──→ Deploy (main only)
         │        │        │
         └────────→ Artifacts flow
```

**Acceptance Criteria:**
- All 5 stages execute in correct order
- Each stage uploads/downloads artifacts
- Tests run on multiple versions
- Deploy only on main branch
- Execution time under 5 minutes
- All logs are comprehensive

---

## Self-Assessment

After completing these exercises, you should be able to:

- [ ] Create basic build pipelines for Node.js
- [ ] Implement multi-version testing with matrix strategy
- [ ] Generate and report code coverage
- [ ] Add linting and code quality checks
- [ ] Build and push Docker images
- [ ] Version and manage build artifacts
- [ ] Run integration tests
- [ ] Implement caching strategies
- [ ] Perform security scanning
- [ ] Build complete production pipelines

**Next:** Review the [solutions.md](solutions.md) file for reference implementations.

---

**Difficulty:** Medium | **Time:** 4-5 hours | **Prerequisites:** Modules 00-01 completion
