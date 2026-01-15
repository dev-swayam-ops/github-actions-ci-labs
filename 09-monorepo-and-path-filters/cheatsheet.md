# Module 09 Cheatsheet: Monorepo and Path Filters

## Path Filters Quick Reference

| Pattern | Matches | Example |
|---------|---------|---------|
| `services/api/**` | All files in api folder | `services/api/src/index.js` |
| `**/*.md` | All markdown files | `README.md`, `docs/guide.md` |
| `package*.json` | Both package files | `package.json`, `package-lock.json` |
| `src/**` | All files under src | `src/app.js`, `src/utils/helper.js` |
| `.github/workflows/**` | All workflow files | `.github/workflows/ci.yml` |
| `!(README.md)` | Negation pattern | Everything except README |

## Path Filter Syntax

**Basic Pattern:**
```yaml
on:
  push:
    paths:
      - 'services/api/**'
      - 'shared/**'
      - 'package.json'
```

**With Ignore:**
```yaml
on:
  push:
    paths:
      - 'services/api/**'
    paths-ignore:
      - '**.md'
      - '.gitignore'
```

**For Pull Requests:**
```yaml
on:
  pull_request:
    paths:
      - 'services/api/**'
      - 'shared/**'
```

## Common Monorepo Patterns

| Pattern | Use Case | Advantage |
|---------|----------|-----------|
| Separate workflows | One per service | Simple, isolated |
| Matrix strategy | Parallel builds | Efficient |
| Change detection | Dynamic triggering | Flexible |
| Workspace install | npm workspaces | Unified deps |

## npm Workspaces Commands

| Command | Purpose |
|---------|---------|
| `npm install` | Install all workspaces |
| `npm ci --workspaces` | Clean install all |
| `npm test --workspaces` | Test all services |
| `npm run build -w @myapp/api` | Build one service |
| `npm list --depth=0` | List workspace packages |

## Git Commands for Monorepos

| Command | Purpose |
|---------|---------|
| `git diff --name-only origin/main..HEAD` | Get changed files |
| `git log --name-status` | Show commits with files |
| `git diff <service>/**` | Diff specific folder |
| `git stash push services/api/` | Stash one service |

## Workflow Conditional Syntax

**Skip job if unchanged:**
```yaml
jobs:
  build-api:
    if: contains(env.CHANGED_SERVICES, 'api')
    runs-on: ubuntu-latest
    steps:
      - run: npm test
```

**Run only on specific paths:**
```yaml
on:
  push:
    paths:
      - 'services/api/**'
      - 'shared/**'
```

**Matrix with condition:**
```yaml
strategy:
  matrix:
    service: [api, web, admin]
steps:
  - if: contains(env.CHANGED, matrix.service)
    run: npm test
```

## Cache Key Strategy

**Per-service caching:**
```yaml
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: npm-${{ matrix.service }}-${{ hashFiles(format('services/{0}/package-lock.json', matrix.service)) }}
```

**Root workspace caching:**
```yaml
- uses: actions/setup-node@v3
  with:
    cache: 'npm'
    cache-dependency-path: package-lock.json
```

## Working Directory Setup

**Global default:**
```yaml
defaults:
  run:
    working-directory: services/api
```

**Per step:**
```yaml
- run: npm test
  working-directory: services/api
```

**Dynamic from matrix:**
```yaml
defaults:
  run:
    working-directory: services/${{ matrix.service }}
```

## Change Detection Script Template

```bash
#!/bin/bash

CHANGED=$(git diff --name-only origin/main..HEAD)

for service in api web admin; do
  if echo "$CHANGED" | grep -q "^services/$service/"; then
    echo "$service=true" >> $GITHUB_OUTPUT
  else
    echo "$service=false" >> $GITHUB_OUTPUT
  fi
done
```

## Monorepo Directory Structure

```
my-monorepo/
├── package.json              # Root workspace config
├── .github/
│   └── workflows/
│       ├── api-ci.yml       # API-specific workflow
│       ├── web-ci.yml       # Web-specific workflow
│       └── validate.yml     # Shared validation
├── services/
│   ├── api/
│   │   ├── package.json     # Service package
│   │   ├── src/
│   │   └── __tests__/
│   └── web/
│       ├── package.json
│       ├── src/
│       └── __tests__/
├── shared/
│   ├── utils/
│   │   └── package.json
│   └── types/
│       └── package.json
└── .gitignore
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Workflow doesn't trigger" | Check path pattern syntax |
| "All services rebuild" | Add change detection |
| "Cache miss" | Verify cache key matches files |
| "Node modules duplicated" | Use `npm ci --workspaces` |
| "Wrong directory" | Set `working-directory` |
| "Shared deps not installed" | Include shared/** in paths |

## Performance Tips

1. **Use path filters** - Skip unchanged services
2. **Cache per service** - Include service name in key
3. **Parallel matrix** - Run services concurrently
4. **Skip on docs** - Ignore .md file changes
5. **Shallow clone** - Use fetch-depth: 1 for push

## Dependency Diagram Example

```
services/api
├── depends on: shared/utils
└── depends on: shared/types

services/web
├── depends on: shared/utils
└── depends on: shared/types

shared/utils
└── depends on: shared/types
```

**Test order:** types → utils → api/web

## Root Workspace Configuration

```json
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": [
    "services/api",
    "services/web",
    "services/admin",
    "shared/utils",
    "shared/types"
  ],
  "scripts": {
    "test": "npm test --workspaces",
    "build": "npm run build --workspaces",
    "lint": "npm run lint --workspaces"
  }
}
```

## Artifact Collection Pattern

```yaml
- name: Upload Service Artifact
  uses: actions/upload-artifact@v3
  with:
    name: ${{ matrix.service }}-build
    path: services/${{ matrix.service }}/dist

# Download in another job
- name: Download All Artifacts
  uses: actions/download-artifact@v3
```

## Export Matrix Outputs

```yaml
jobs:
  build:
    strategy:
      matrix:
        service: [api, web]
    outputs:
      result-${{ matrix.service }}: ${{ steps.build.outputs.result }}
    steps:
      - id: build
        run: echo "result=success" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    steps:
      - run: echo "API: ${{ needs.build.outputs.result-api }}"
        run: echo "Web: ${{ needs.build.outputs.result-web }}"
```

## Common Mistakes to Avoid

- ❌ Path pattern doesn't match files
- ❌ Missing `--workspaces` flag
- ❌ Wrong working-directory context
- ❌ Not including shared/* in paths
- ❌ Hardcoded service names
- ❌ Skipping tests for unchanged services
- ❌ Cache key without service identifier
- ❌ No change detection for matrix jobs

## Best Practices Checklist

- ✅ Path filters on appropriate patterns
- ✅ Change detection for matrix strategies
- ✅ Per-service cache keys
- ✅ Working directory context set
- ✅ Shared dependencies included
- ✅ Parallel execution where possible
- ✅ Documentation of monorepo structure
- ✅ Clear service boundaries
- ✅ Test order respects dependencies
