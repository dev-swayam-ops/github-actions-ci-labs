# Module 10 Cheatsheet: Release and Versioning Workflows

## Semantic Versioning Quick Reference

| Change | Type | Example | New Version |
|--------|------|---------|-------------|
| Breaking API | MAJOR | Remove endpoint | 1.0.0 → 2.0.0 |
| New feature | MINOR | Add auth | 1.0.0 → 1.1.0 |
| Bug fix | PATCH | Fix timeout | 1.0.0 → 1.0.1 |

**Format:** `MAJOR.MINOR.PATCH` (e.g., `2.3.1`)

## Git Tagging Commands

| Command | Purpose |
|---------|---------|
| `git tag v1.0.0` | Create lightweight tag |
| `git tag -a v1.0.0 -m "message"` | Create annotated tag |
| `git tag -l` | List all tags |
| `git tag -l v1*` | List specific tags |
| `git show v1.0.0` | Show tag details |
| `git push origin v1.0.0` | Push single tag |
| `git push origin --tags` | Push all tags |
| `git tag -d v1.0.0` | Delete local tag |
| `git push origin --delete v1.0.0` | Delete remote tag |

## Conventional Commits

**Format:** `type(scope): description`

| Type | Bump | Example |
|------|------|---------|
| `feat:` | MINOR | `feat(api): add user endpoint` |
| `fix:` | PATCH | `fix(db): resolve timeout` |
| `breaking:` | MAJOR | `breaking(api): remove v1 endpoints` |
| `docs:` | None | `docs(readme): update` |
| `refactor:` | None | `refactor(core): simplify logic` |

## npm Version Bumping

| Command | Effect | Example |
|---------|--------|---------|
| `npm version patch` | 1.0.0 → 1.0.1 | Bug fixes |
| `npm version minor` | 1.0.0 → 1.1.0 | New features |
| `npm version major` | 1.0.0 → 2.0.0 | Breaking changes |
| `npm version 2.5.0` | Set specific | Direct version |
| `npm version --no-git-tag-version` | No tag | Script only |

## Release Workflow Triggers

```yaml
# On tag push
on:
  push:
    tags:
      - 'v*'

# Manual workflow dispatch
on:
  workflow_dispatch:
    inputs:
      version:
        type: string
        required: true

# On schedule
on:
  schedule:
    - cron: '0 0 1 * *'  # Monthly
```

## GitHub Actions Secrets Required

| Secret | Purpose | Where From |
|--------|---------|-----------|
| `GITHUB_TOKEN` | Automatic | Built-in |
| `NPM_TOKEN` | npm registry | https://npmjs.com/settings/tokens |
| `DOCKER_PASSWORD` | Docker Hub | https://hub.docker.com |
| `PYPI_TOKEN` | Python registry | https://pypi.org |

## Get Version from Tag

```bash
# Remove v prefix
VERSION=${GITHUB_REF#refs/tags/v}

# Or from package.json
VERSION=$(node -p "require('./package.json').version")

# Output for next steps
echo "version=${VERSION}" >> $GITHUB_OUTPUT
```

## Changelog Generation

**From commits since tag:**
```bash
PREV_TAG=$(git describe --tags --abbrev=0)
git log ${PREV_TAG}..HEAD --oneline --pretty=format:"- %s (%h)"
```

**From commit types:**
```bash
# Features
git log --oneline | grep "^.*feat"

# Bug fixes
git log --oneline | grep "^.*fix"
```

## Release Creation Actions

**softprops/action-gh-release:**
```yaml
- uses: softprops/action-gh-release@v1
  with:
    tag_name: v${{ version }}
    body: |
      ## Changes
      - New features
      - Bug fixes
    draft: false
    prerelease: false
    files: dist/*
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Actions/create-release (deprecated):**
```yaml
- uses: actions/create-release@v1
  with:
    tag_name: v${{ version }}
    release_name: Release ${{ version }}
    body: Release notes
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Publishing to Registries

**npm:**
```yaml
- uses: actions/setup-node@v3
  with:
    registry-url: https://registry.npmjs.org
- run: npm publish
  env:
    NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

**Docker Hub:**
```yaml
- uses: docker/build-push-action@v4
  with:
    push: true
    tags: myregistry/myapp:${{ version }}
    username: ${{ secrets.DOCKER_USER }}
    password: ${{ secrets.DOCKER_PASSWORD }}
```

**Python (PyPI):**
```yaml
- uses: pypa/gh-action-pypi-publish@release/v1
  with:
    password: ${{ secrets.PYPI_TOKEN }}
```

## Common Version Patterns

| Pattern | Use Case | Example |
|---------|----------|---------|
| `v1.2.3` | Production release | `v1.0.0`, `v2.5.1` |
| `v1.2.3-rc.1` | Release candidate | `v2.0.0-rc.1` |
| `v1.2.3-beta.1` | Beta release | `v1.5.0-beta.2` |
| `v1.2.3-alpha.1` | Alpha release | `v2.0.0-alpha.1` |

## Release Workflow Structure

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  build:
    needs: test
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v3

  release:
    needs: build
    permissions:
      contents: write
    steps:
      - uses: actions/download-artifact@v3
      - uses: softprops/action-gh-release@v1
```

## Conditional Release Deployment

```yaml
on:
  push:
    tags:
      - 'v*'
      - 'rc-*'
      - 'beta-*'

jobs:
  detect-env:
    outputs:
      environment: ${{ steps.env.outputs.env }}
    steps:
      - run: |
          if [[ ${{ github.ref }} == refs/tags/v* ]]; then
            echo "env=production" >> $GITHUB_OUTPUT
          elif [[ ${{ github.ref }} == refs/tags/rc-* ]]; then
            echo "env=staging" >> $GITHUB_OUTPUT
          fi

  deploy:
    needs: detect-env
    environment: ${{ needs.detect-env.outputs.environment }}
```

## Package.json Version Script

```json
{
  "scripts": {
    "version": "npm run build && npm run docs && git add .",
    "postversion": "git push && git push --tags"
  }
}
```

## GitHub Script Release Notes

```yaml
- uses: actions/github-script@v6
  with:
    script: |
      const pulls = await github.rest.pulls.list({
        owner: context.repo.owner,
        repo: context.repo.repo,
        state: 'closed',
        sort: 'updated'
      });
      
      const features = pulls.data
        .filter(p => p.labels.some(l => l.name === 'enhancement'))
        .map(p => `- ${p.title}`);
```

## Release Asset Upload

```yaml
- uses: actions/upload-release-asset@v1
  with:
    upload_url: ${{ steps.create_release.outputs.upload_url }}
    asset_path: ./dist/myapp.zip
    asset_name: myapp-${{ version }}.zip
    asset_content_type: application/zip
```

## CHANGELOG.md Template

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com)
and this project adheres to [Semantic Versioning](https://semver.org).

## [Unreleased]

## [2.0.0] - 2024-01-15

### Added
- New feature X
- New feature Y

### Changed
- Modified behavior of Z

### Fixed
- Fixed bug A
- Fixed bug B

### Removed
- Removed deprecated endpoint

## [1.0.0] - 2024-01-01

### Added
- Initial release
```

## Permissions Required

```yaml
permissions:
  contents: read           # Read code
  contents: write          # Write releases
  packages: write          # Publish packages
  pull-requests: read      # Read PR info
```

## Version Validation

```bash
# Check format
if [[ ! $VERSION =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
  echo "Invalid version format"
  exit 1
fi
```

## Environment Setup for Publishing

```yaml
- uses: actions/setup-node@v3
  with:
    node-version: '18'
    registry-url: 'https://registry.npmjs.org'
    scope: '@myorg'
```

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| "Tag not created" | No write permissions | Add `permissions: { contents: write }` |
| "Release not visible" | Draft mode enabled | Set `draft: false` |
| "npm publish fails" | No auth token | Add `NPM_TOKEN` secret |
| "Version parsing fails" | Non-semver format | Validate format first |
| "Artifact not found" | Wrong path | Verify upload artifact path |
| "Tag regex not matching" | Pattern error | Test with git commands |

## Quick Start Template

```yaml
name: Release

on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm test
      - run: npm run build
      - uses: softprops/action-gh-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Best Practices Checklist

- ✅ Use semantic versioning consistently
- ✅ Tag releases with version number
- ✅ Write descriptive release notes
- ✅ Test before publishing
- ✅ Use conventional commits
- ✅ Auto-generate changelogs
- ✅ Version one source of truth
- ✅ Verify published packages
- ✅ Archive old releases
- ✅ Document release process
