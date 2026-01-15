# Exercises: Matrix and Multi-OS CI

Master the matrix strategy for testing across multiple operating systems and language versions.

---

## Exercise 1: Basic Matrix Setup (Easy)

**Objective:** Create your first matrix strategy workflow.

**Instructions:**
1. Create `.github/workflows/ex1-basic-matrix.yml`
2. Define a simple matrix with:
   - `node: [16, 18, 20]`
3. Create a single job that uses the matrix
4. Job should display Node version from matrix
5. Verify 3 jobs are created

**Hint:** Use `${{ matrix.node }}` to access matrix value

**Acceptance Criteria:**
- Workflow creates exactly 3 jobs
- Each job uses correct Node version
- All jobs complete successfully

---

## Exercise 2: Multi-Dimensional Matrix (Easy)

**Objective:** Create matrix with multiple dimensions.

**Instructions:**
1. Create `.github/workflows/ex2-multidim-matrix.yml`
2. Define matrix with:
   - `os: [ubuntu-latest, windows-latest]`
   - `node: [16, 18, 20]`
3. Display OS and Node in output
4. Verify 6 jobs are created (2 OS × 3 Node)

**Math Check:** 2 × 3 = 6 jobs

**Acceptance Criteria:**
- 6 job variations created
- Each combines correct OS and Node version
- Matrix combinations are all valid

---

## Exercise 3: Triple-Dimension Matrix (Medium)

**Objective:** Add a third dimension to matrix.

**Instructions:**
1. Create `.github/workflows/ex3-triple-matrix.yml`
2. Define matrix with three dimensions:
   - `os: [ubuntu-latest, macos-latest]`
   - `node: [18, 20]`
   - `npm-version: [8, 9]`
3. Display all three values in output
4. Verify 8 jobs are created (2 × 2 × 2)

**Acceptance Criteria:**
- 8 job variations created
- Each displays correct OS, Node, npm combination
- Jobs execute in parallel

---

## Exercise 4: Matrix Exclusion (Medium)

**Objective:** Use exclude pattern to skip specific combinations.

**Instructions:**
1. Create `.github/workflows/ex4-exclude.yml`
2. Create matrix with:
   - `os: [ubuntu-latest, windows-latest, macos-latest]`
   - `node: [16, 18, 20]`
3. Exclude combinations:
   - Windows with Node 16
   - macOS with Node 16
4. Verify 7 jobs created (9 - 2 excluded)

**Hint:** Use `exclude:` section with OS and Node conditions

**Acceptance Criteria:**
- 7 jobs created (not 9)
- Excluded combinations don't appear
- Valid combinations all present

---

## Exercise 5: Matrix Inclusion (Medium)

**Objective:** Add special cases beyond standard matrix.

**Instructions:**
1. Create `.github/workflows/ex5-include.yml`
2. Create basic matrix:
   - `os: [ubuntu-latest, windows-latest]`
   - `node: [18, 20]`
3. Add inclusion for macOS with Node 18 only
4. Verify 5 jobs (4 base + 1 included)
5. macOS should only have Node 18

**Acceptance Criteria:**
- 5 jobs created
- macOS job appears once (only Node 18)
- All other combinations present

---

## Exercise 6: OS-Specific Commands (Medium)

**Objective:** Run different commands based on OS.

**Instructions:**
1. Create `.github/workflows/ex6-os-commands.yml`
2. Create matrix with 2 OS:
   - `os: [ubuntu-latest, windows-latest]`
3. Add conditional steps:
   - Linux: Run `ls -la`
   - Windows: Run `dir`
4. Both should work without errors
5. Test both job runs

**Hint:** Use `if: runner.os == 'Linux'`

**Acceptance Criteria:**
- Linux job executes `ls -la`
- Windows job executes `dir`
- No errors in either job

---

## Exercise 7: Fail-Fast Configuration (Medium)

**Objective:** Understand and control fail-fast behavior.

**Instructions:**
1. Create `.github/workflows/ex7-fail-fast.yml`
2. Create two workflows:
   - `fail-fast-true`: Stops on first failure
   - `fail-fast-false`: Continues all jobs
3. Create a matrix with 4 jobs
4. Add intentional failure in first job
5. Compare execution times and behavior

**Acceptance Criteria:**
- `fail-fast: true` stops remaining jobs quickly
- `fail-fast: false` runs all jobs
- Time difference is clear in logs

---

## Exercise 8: Cache Per Matrix Item (Medium)

**Objective:** Implement matrix-aware caching.

**Instructions:**
1. Create `.github/workflows/ex8-matrix-cache.yml`
2. Create matrix with Node versions: [16, 18, 20]
3. Setup Node with cache enabled
4. Cache key should include matrix values
5. Run twice to verify cache hit
6. Observe different cache keys per version

**Cache Key Pattern:** `${{ runner.os }}-npm-${{ matrix.node }}`

**Acceptance Criteria:**
- Each Node version has separate cache
- Second run uses cached dependencies
- Different cache keys per version

---

## Exercise 9: Matrix Outputs (Medium)

**Objective:** Share data from matrix jobs to dependent job.

**Instructions:**
1. Create `.github/workflows/ex9-matrix-outputs.yml`
2. Matrix job outputs Node version
3. Dependent job (no matrix) collects outputs
4. Dependent job displays all versions tested
5. Use `needs: job-name` to depend on matrix job

**Hint:** `needs` returns array of outputs from matrix

**Acceptance Criteria:**
- Matrix job outputs its Node version
- Dependent job receives all outputs
- All versions displayed in summary

---

## Exercise 10: Complete Multi-Platform Pipeline (Medium)

**Objective:** Build realistic multi-platform testing pipeline.

**Instructions:**
1. Create `.github/workflows/ex10-complete-platform.yml`
2. Implement three jobs:
   - `lint`: Single run on Ubuntu
   - `test`: Matrix across 3 OS × 2 Node versions
   - `build`: Runs after all tests
3. Test matrix combinations:
   - OS: [ubuntu-latest, windows-latest, macos-latest]
   - Node: [18, 20]
4. Include OS-specific cleanup steps
5. Build only runs if all tests pass

**Acceptance Criteria:**
- 6 test jobs created (3 OS × 2 Node)
- All jobs execute in correct order
- Total execution under 5 minutes
- Build waits for all test completion

---

## Self-Assessment

After completing these exercises, you should be able to:

- [ ] Create basic matrix strategies
- [ ] Use multiple matrix dimensions
- [ ] Exclude specific combinations
- [ ] Include special cases
- [ ] Run OS-specific commands
- [ ] Control fail-fast behavior
- [ ] Implement matrix-aware caching
- [ ] Share data from matrix jobs
- [ ] Design multi-platform pipelines
- [ ] Optimize matrix performance

**Next:** Review the [solutions.md](solutions.md) file for reference implementations.

---

**Difficulty:** Easy to Medium | **Time:** 2-3 hours | **Prerequisites:** Module 02 completion
