# Module 11 Exercises: Performance and Optimization

## Exercise 1: Identify Workflow Bottleneck

**Objective:** Profile a workflow and identify the slowest step.

**Starter Code:**

```yaml
name: Analyze Performance

on: [push]

jobs:
  workflow:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install dependencies
        run: npm install
      
      - name: Lint code
        run: npm run lint
      
      - name: Run tests
        run: npm test
      
      - name: Build project
        run: npm run build
      
      - name: Check size
        run: du -sh dist/
```

**Requirements:**
- Run workflow
- Record each step's duration
- Identify slowest step
- Document results

**Acceptance Criteria:**
- [ ] Workflow completes successfully
- [ ] Each step duration recorded
- [ ] Slowest step identified
- [ ] Total execution time noted
- [ ] Plan for optimization created

**Hint:** Each step shows duration in Actions tab.

---

## Exercise 2: Enable Caching for Dependencies

**Objective:** Add caching to speed up dependency installation.

**Starter Code:**

```yaml
name: With Caching

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          # TODO: Enable npm caching
      
      - run: npm ci              # Use ci instead of install
      - run: npm run lint
      - run: npm test
```

**Requirements:**
- Enable npm cache in setup-node
- Use `npm ci` for consistency
- Run twice and compare times
- Document cache hits

**Acceptance Criteria:**
- [ ] cache: 'npm' configured
- [ ] npm ci used instead of install
- [ ] First run builds cache
- [ ] Second run is faster
- [ ] Cache hit visible in logs

**Hint:** Add `cache: 'npm'` to setup-node parameters.

---

## Exercise 3: Parallelize Job Execution

**Objective:** Run independent jobs in parallel instead of sequentially.

**Starter Code:**

```yaml
name: Parallel Jobs

on: [push]

jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci

  lint:
    # TODO: Run in parallel with test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          cache: 'npm'
      - run: npm run lint

  test:
    # TODO: Run in parallel with lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          cache: 'npm'
      - run: npm test

  build:
    # TODO: Run after lint and test complete
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          cache: 'npm'
      - run: npm run build
```

**Requirements:**
- Make lint and test parallel
- Build depends on both completing
- Share dependencies via artifacts
- Document execution time improvement

**Acceptance Criteria:**
- [ ] Lint and test run simultaneously
- [ ] Build waits for both
- [ ] No duplicate dependency installation
- [ ] Total time less than sequential
- [ ] Dependencies shared between jobs

**Hint:** Use `needs:` to create dependencies, not steps.

---

## Exercise 4: Optimize Matrix Strategy

**Objective:** Reduce matrix job count while maintaining coverage.

**Starter Code:**

```yaml
name: Optimized Matrix

on: [push]

jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: ['16', '18', '20']
        # Creates 9 jobs - too many!
    
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      - run: npm ci
      - run: npm test
```

**Requirements:**
- Reduce to 4-5 critical combinations
- Keep coverage of key platforms
- Use include/exclude
- Document savings

**Acceptance Criteria:**
- [ ] Matrix job count reduced
- [ ] Critical combos included
- [ ] Coverage maintained
- [ ] Fewer parallel jobs
- [ ] Total time reduced

**Hint:** Only test primary combinations on each platform.

---

## Exercise 5: Minimize Artifact Size

**Objective:** Reduce artifact storage by uploading only necessary files.

**Starter Code:**

```yaml
name: Artifact Optimization

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci
      - run: npm run build
      
      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build
          # TODO: Only upload dist/ folder, not entire repo
          # TODO: Exclude node_modules, src, etc.
          # TODO: Set retention period
          path: .
```

**Requirements:**
- Upload only dist/ folder (built files)
- Exclude source code and dependencies
- Set 1-day retention
- Document size reduction

**Acceptance Criteria:**
- [ ] Only dist/ uploaded
- [ ] node_modules excluded
- [ ] Retention set to 1 day
- [ ] Artifact size <100MB
- [ ] Download still includes needed files

**Hint:** Use `path: dist/` and retention-days: 1.

---

## Exercise 6: Share Cache Between Jobs

**Objective:** Use cache to avoid reinstalling dependencies across jobs.

**Starter Code:**

```yaml
name: Shared Cache

on: [push]

jobs:
  job1:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci

  job2:
    # TODO: Use same cache as job1
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          # TODO: Cache configured to match job1
      - run: npm ci  # Should be fast (cache hit)
```

**Requirements:**
- Both jobs use same cache
- Cache key matches in both
- job2 installs much faster
- Document time savings

**Acceptance Criteria:**
- [ ] Same cache key in both jobs
- [ ] First job builds cache
- [ ] Second job hits cache
- [ ] Second npm ci much faster
- [ ] Cache persists between jobs

**Hint:** Same setup-node caching creates same key.

---

## Exercise 7: Implement Conditional Deployment

**Objective:** Deploy only on successful builds to main branch.

**Starter Code:**

```yaml
name: Conditional Deployment

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: npm test
      - run: npm run build

  deploy:
    # TODO: Only run if build succeeded
    # TODO: Only run if on main branch
    # TODO: Include deployment step
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying..."
```

**Requirements:**
- Deploy only on success
- Deploy only on main branch
- Include condition check
- Log deployment attempt

**Acceptance Criteria:**
- [ ] Deploy job conditional on build success
- [ ] Deploy only on main branch
- [ ] Feature branches skip deploy
- [ ] Pull requests skip deploy
- [ ] logs show correct behavior

**Hint:** Use `if:` with `success()` and branch check.

---

## Exercise 8: Parallel Test Execution

**Objective:** Split tests across multiple jobs to run in parallel.

**Starter Code:**

```yaml
name: Parallel Tests

on: [push]

jobs:
  test-unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:unit

  test-integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:integration

  test-e2e:
    # TODO: Create E2E test job
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:e2e
```

**Requirements:**
- All test jobs run in parallel
- Each has own cache
- Combined results
- Document total time

**Acceptance Criteria:**
- [ ] All test types run parallel
- [ ] All use caching
- [ ] Total time < sequential
- [ ] All results visible
- [ ] Single failure blocks build

**Hint:** No `needs:` means parallel execution.

---

## Exercise 9: Monitor Cache Hit Rate

**Objective:** Track and analyze cache effectiveness.

**Starter Code:**

```yaml
name: Cache Monitoring

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        id: node-cache
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Report Cache Status
        run: |
          # TODO: Log cache hit/miss
          # TODO: Extract from logs
          # TODO: Track over time
          echo "Cache key: ${{ steps.node-cache.outputs.cache-primary-key }}"
          echo "Cache hit: ${{ steps.node-cache.outputs.cache-hit }}"
      
      - run: npm ci
      - run: npm test
```

**Requirements:**
- Report cache hit/miss
- Track over multiple runs
- Document results
- Analyze trends

**Acceptance Criteria:**
- [ ] Cache hit status logged
- [ ] Multiple runs tracked
- [ ] Hit rate calculated
- [ ] Results documented
- [ ] Target: >80% hit rate

**Hint:** setup-node outputs `cache-hit` variable.

---

## Exercise 10: End-to-End Performance Optimization

**Objective:** Apply all optimization techniques to real workflow.

**Starter Code:**

```yaml
name: Fully Optimized Pipeline

on: [push]

env:
  # TODO: Add environment optimization

jobs:
  # TODO: Implement all optimizations:
  # 1. Caching enabled
  # 2. Parallel jobs
  # 3. Conditional deployment
  # 4. Matrix optimization
  # 5. Artifact minimization
  # 6. Shared dependencies
  
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm test
      - run: npm run build
```

**Requirements:**
- Apply all 10 modules' concepts
- Optimize for speed
- Minimize artifacts
- Parallelize where possible
- Add deployment gate

**Acceptance Criteria:**
- [ ] All optimization techniques used
- [ ] Build time <30 seconds
- [ ] Cache hit rate >80%
- [ ] Parallel jobs visible
- [ ] Conditional deployment works

**Hint:** Combine exercises 1-9 into single workflow.

---

## Summary

You now understand:
- ✅ Workflow profiling and analysis
- ✅ Caching strategies and hit rates
- ✅ Job parallelization patterns
- ✅ Matrix optimization
- ✅ Artifact size management
- ✅ Cache sharing between jobs
- ✅ Conditional job execution
- ✅ Parallel test execution
- ✅ Performance monitoring
- ✅ End-to-end optimization

Continue optimizing your workflows!
