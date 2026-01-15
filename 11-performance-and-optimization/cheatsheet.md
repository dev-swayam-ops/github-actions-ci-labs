# Module 11 Cheatsheet: Performance and Optimization

## Quick Reference Tables

### Performance Optimization Checklist

| Technique | Impact | Effort | When to Use |
|-----------|--------|--------|------------|
| **Caching** | 50-95% faster | Low | Always (npm, docker, etc.) |
| **Parallelization** | 40-80% faster | Medium | Multiple independent jobs |
| **Job Dependencies** | 20-50% faster | Low | Proper job ordering |
| **Matrix Optimization** | 20-40% savings | Low | Reduce unnecessary combinations |
| **Artifact Minimization** | 30-90% less storage | Low | Only upload built files |
| **Service Containers** | Network optimization | Medium | Integration tests |
| **Conditional Logic** | 10-30% faster | Low | Skip unnecessary jobs |
| **Environment Variables** | 5-20% faster | Low | Use CI=true, etc. |
| **Shallow Checkout** | 50-80% faster | Low | Add fetch-depth: 1 |
| **Runners** | 20-40% faster | High | Use self-hosted runners |

---

### Cache Configuration Quick Reference

#### npm Dependencies

```yaml
- uses: actions/setup-node@v3
  with:
    node-version: '18'
    cache: 'npm'
```

**What it caches:**
- node_modules/ directory
- package-lock.json checksum
- npm cache

**Cache key:**
```
{OS}-node-{version}-npm-{hashOf(package-lock.json)}
```

**Hit rate:** 80-95% (unless dependencies change)

---

#### Docker Layers

```yaml
- uses: docker/build-push-action@v4
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**What it caches:**
- Each Docker layer
- Build context

**Hit rate:** 70-90% (depends on code changes)

---

#### GitHub Cache API

```yaml
- uses: actions/cache@v3
  with:
    path: |
      node_modules/
      .next/cache/
    key: ${{ runner.os }}-nextjs-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-nextjs-
```

**Custom cache paths:**
- Build outputs
- Package caches
- Dependency folders

---

### Parallelization Patterns

#### Pattern 1: Independent Jobs (No Dependencies)

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  # Both run in parallel (no `needs`)
```

**Execution time:** max(lint, test) not sum

---

#### Pattern 2: Sequential Dependencies

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

  test:
    needs: build  # Wait for build
    steps:
      - run: npm test

  deploy:
    needs: test   # Wait for test
    steps:
      - run: npm run deploy
```

**Execution time:** build + test + deploy

---

#### Pattern 3: Mixed Dependencies

```yaml
jobs:
  lint:    # No deps, runs immediately
  test:    # No deps, runs immediately
  build:
    needs: [lint, test]  # Wait for both
  deploy:
    needs: build        # Then deploy
```

**Timeline:**
```
Time 0: lint → ...  ┐
        test → ...  ├─ build → deploy
```

---

#### Pattern 4: Matrix Parallelization

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: ['16', '18', '20']
    # Creates 6 jobs running in parallel
```

**Job count:** 2 OS × 3 Node = 6 jobs

**Execution:** All 6 run simultaneously

---

### Common Performance Anti-Patterns

| ❌ Anti-Pattern | Impact | ✅ Solution |
|---|---|---|
| **No caching** | Reinstall deps every run (30-40s waste) | Add `cache: 'npm'` |
| **Sequential jobs** | Slow total time | Remove `needs:` if independent |
| **Large artifacts** | Slow transfer, high storage | Upload only built files |
| **Test duplication** | Redundant work | Use matrix for platforms |
| **No conditionals** | Unnecessary deployments | Add `if:` conditions |
| **Fetch entire history** | Slow checkout | Use `fetch-depth: 1` |
| **No health checks** | Unknown deployment status | Add health validation |
| **Retry without backoff** | Fast failure, no recovery | Add sleep between retries |

---

### Workflow Profiling

#### Finding Bottlenecks

```bash
# In Actions logs, look for:
- Setup time: 10-20s (normal)
- Checkout: 5-15s (normal, use fetch-depth: 1 to reduce)
- npm install: 20-40s (optimize with cache!)
- npm test: varies
- npm build: 10-30s
```

#### Benchmark Template

```yaml
- name: Time each step
  run: |
    START=$(date +%s)
    npm ci
    END=$(date +%s)
    echo "npm ci: $((END - START))s"
    
    START=$(date +%s)
    npm run build
    END=$(date +%s)
    echo "npm build: $((END - START))s"
```

---

### Cache Hit Rate Monitoring

#### Check Cache Status

```yaml
- uses: actions/setup-node@v3
  id: node
  with:
    cache: 'npm'

- name: Cache hit?
  run: echo "Cache hit: ${{ steps.node.outputs.cache-hit }}"
```

#### Expected Hit Rates

```
First run: MISS (0% hit rate)
Subsequent runs: HIT (80-95% if code unchanged)
After dependency update: MISS (new cache needed)
On different branch: HIT (same dependencies)
On different Node version: MISS (different cache key)
```

#### Monitor Trend

```bash
# Track over time
"2024-01-15,HIT"
"2024-01-16,HIT"
"2024-01-17,MISS"  # Dependency change
"2024-01-18,HIT"

# Calculate
Hit Rate = (HIT count / Total runs) × 100
Target: >80%
```

---

## Command Reference

### GitHub Actions Specific

```bash
# Check if on main branch
if [ "${{ github.ref }}" = "refs/heads/main" ]; then
  echo "On main branch"
fi

# Get commit SHA
echo "${{ github.sha }}"

# Get branch name
echo "${{ github.ref_name }}"

# Get actor (user who triggered)
echo "${{ github.actor }}"

# Get run URL
echo "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
```

### Performance Monitoring

```bash
# Time a command
time npm ci
time npm test
time npm run build

# Measure with dates
START=$(date +%s%N)
npm ci
END=$(date +%s%N)
DURATION=$(((END - START) / 1000000))
echo "Took ${DURATION}ms"

# Monitor resource usage (Linux)
/usr/bin/time -v npm test
```

### Docker Build Optimization

```bash
# Build with cache
docker buildx build \
  --cache-from type=local,src=/tmp/.buildx-cache \
  --cache-to type=local,dest=/tmp/.buildx-cache \
  .

# List layers
docker image history myimage

# Check image size
docker images myimage
```

---

## Quick Wins

### Top 5 Immediate Performance Gains

1. **Enable npm caching** (50-80% faster installs)
   ```yaml
   cache: 'npm'
   ```

2. **Parallelize independent jobs** (30-50% total time reduction)
   ```yaml
   # Remove `needs:` from independent jobs
   ```

3. **Minimize artifacts** (50-80% faster transfers)
   ```yaml
   path: dist/  # Not .
   ```

4. **Use shallow checkout** (50-80% faster)
   ```yaml
   fetch-depth: 1
   ```

5. **Conditional deployments** (20-40% fewer jobs)
   ```yaml
   if: github.ref == 'refs/heads/main'
   ```

---

## Optimization Template Workflow

```yaml
name: Optimized Workflow

on: [push]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 1  # ✓ Shallow checkout
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'    # ✓ Enable caching

      - run: npm ci
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 1
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci
      - run: npm test

  build:
    needs: [lint, test]  # ✓ Parallel dependencies
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 1
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci
      - run: npm run build
      
      - uses: actions/upload-artifact@v3
        with:
          name: dist
          path: dist/      # ✓ Minimal artifacts
          retention-days: 1  # ✓ Cleanup

  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'  # ✓ Conditional
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: dist
      - run: npm run deploy
```

---

## Performance Metrics

### Typical Execution Times

**Without optimization:**
- Sequential execution: 60-90 seconds
- High artifact storage
- Cache misses every run

**With optimization:**
- Parallel execution: 20-40 seconds
- Minimal artifacts
- Cache hits 80%+
- **Total improvement: 50-75% faster**

### Cost Impact (GitHub-hosted runners)

**Monthly savings example:**
- 100 workflow runs/month
- 40 second improvement per run
- 100 × 40 sec = 66 minutes saved
- Cost: ~$0.01 USD per minute
- **Monthly savings: ~$0.66**

**For teams with 1000 runs/month:**
- 1000 × 40 sec = 666 minutes
- **Monthly savings: ~$6.66**

---

## Troubleshooting

### Cache Not Working

```
❌ Problem: Cache never hits
✅ Solutions:
  - Check cache key matches
  - Verify file hash changes
  - Check cache-key in logs
  - Cache has 5GB limit, 7-day expiry
```

### Jobs Take Too Long

```
❌ Problem: Workflow slow
✅ Solutions:
  1. Add npm caching
  2. Parallelize jobs (remove needs)
  3. Use shallow checkout
  4. Check for sleeping processes
  5. Use faster runners
```

### Inconsistent Performance

```
❌ Problem: Sometimes fast, sometimes slow
✅ Solutions:
  - Cache invalidation (code changes)
  - Network fluctuations
  - Runner availability
  - Check cache hit rate in logs
```

---

## Performance Scoring

### Rate Your Workflow

```
Caching enabled:           [Yes=5] [No=0]
Parallel jobs:             [Yes=5] [No=0]
Shallow checkout:          [Yes=5] [No=0]
Minimal artifacts:         [Yes=5] [No=0]
Conditional logic:         [Yes=5] [No=0]
Health checks:             [Yes=5] [No=0]
Monitor cache hits:        [Yes=5] [No=0]
Matrix optimized:          [Yes=5] [No=0]
Service container cleanup: [Yes=5] [No=0]
Total runs < 5 minutes:    [Yes=5] [No=0]

Score out of 50
50-45: Excellent ⭐⭐⭐⭐⭐
44-35: Good ⭐⭐⭐⭐
34-25: Fair ⭐⭐⭐
24-0:  Needs improvement ⭐⭐
```

---

## Resources

- [GitHub Actions Performance Docs](https://docs.github.com/en/actions)
- [Caching Dependencies](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows)
- [Workflow Syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)

