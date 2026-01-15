# Module 10: Release and Versioning Workflows

## Overview

Automated release and versioning workflows ensure consistent, reliable software releases. This module teaches you how to automate version bumping, create releases, generate changelogs, and publish artifacts to registries using GitHub Actions.

### What You'll Learn

1. **Semantic Versioning** - Understanding version numbering schemes
2. **Automated Version Bumping** - Incrementing versions based on commits
3. **Release Creation** - Automating GitHub releases
4. **Changelog Generation** - Auto-generating release notes
5. **Git Tagging** - Marking release versions in Git
6. **Publishing Artifacts** - Deploying to registries (npm, Docker Hub, etc.)
7. **Release Triggers** - Manual and automatic release workflows
8. **Asset Management** - Handling release artifacts

### Prerequisites

- Basic GitHub Actions knowledge (Modules 01-02)
- Git tagging fundamentals
- Understanding of semantic versioning
- Familiarity with package managers (npm, etc.)

---

## Key Concepts

### Semantic Versioning (SemVer)

**Format:** MAJOR.MINOR.PATCH (e.g., 1.2.3)

**Rules:**
- MAJOR: Breaking changes (1.0.0 → 2.0.0)
- MINOR: New features, backward compatible (1.0.0 → 1.1.0)
- PATCH: Bug fixes (1.0.0 → 1.0.1)

**Examples:**
- v1.0.0: Initial release
- v1.1.0: Added feature X (backward compatible)
- v1.1.1: Fixed bug Y
- v2.0.0: Breaking change

### Conventional Commits

**Standard format for automated versioning:**

```
type(scope): description

type:
- feat: New feature (MINOR bump)
- fix: Bug fix (PATCH bump)
- breaking: Breaking change (MAJOR bump)
```

**Examples:**
```
feat(api): add user authentication
fix(database): resolve connection timeout
breaking(api): remove deprecated endpoints
```

### Git Tagging

**Create version tag:**
```bash
git tag v1.2.3
git push origin v1.2.3
```

**Annotated tag (recommended):**
```bash
git tag -a v1.2.3 -m "Release version 1.2.3"
git push origin v1.2.3
```

### Release Workflow Patterns

**Pattern 1: Manual Release**
- Manually trigger release workflow
- Specify version number
- Create release with assets

**Pattern 2: Automated Release**
- Trigger on version tag push
- Auto-detect version from tag
- Generate release notes
- Publish artifacts

**Pattern 3: Scheduled Release**
- Weekly/monthly release cycles
- Auto-bump version
- Create release with accumulated changes

### Changelog Generation

**Automated from commits:**
```markdown
## v1.2.0 (2024-01-15)

### Features
- Add user authentication
- Improve API performance

### Bug Fixes
- Fix database timeout
- Resolve cache invalidation

### Breaking Changes
- Remove deprecated endpoints
```

**Tools:**
- conventional-changelog
- auto-changelog
- gh CLI (generate-notes)

---

## Hands-on Lab: Automated Release Workflow

### Part 1: Setup Package

**Create package.json:**
```json
{
  "name": "release-demo",
  "version": "0.1.0",
  "description": "Automated release demo",
  "main": "src/index.js",
  "scripts": {
    "test": "echo 'Tests passed'",
    "build": "echo 'Build complete'",
    "version": "npm run build && git add ."
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/yourname/release-demo"
  },
  "keywords": ["release", "automation"],
  "author": "Your Name",
  "license": "MIT"
}
```

**Create source file:**
```bash
mkdir src
echo "module.exports = { hello: 'world' };" > src/index.js
```

**Create CHANGELOG.md:**
```markdown
# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

## [0.1.0] - 2024-01-15

### Added
- Initial release
- Basic functionality
```

### Part 2: Create Release Workflow

`.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to release'
        required: true
        type: string

jobs:
  create-release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write
    
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Get Version
        id: version
        run: |
          if [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
            VERSION=${{ github.event.inputs.version }}
          else
            VERSION=${GITHUB_REF#refs/tags/v}
          fi
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
      
      - name: Verify Tag Format
        run: |
          if ! [[ "${{ steps.version.outputs.version }}" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
            echo "Invalid version format"
            exit 1
          fi
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - run: npm ci
      - run: npm run test
      - run: npm run build
      
      - name: Generate Changelog
        id: changelog
        run: |
          PREV_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
          
          if [ -z "$PREV_TAG" ]; then
            CHANGELOG=$(git log --oneline --decorate)
          else
            CHANGELOG=$(git log ${PREV_TAG}..HEAD --oneline)
          fi
          
          echo "changelog<<EOF" >> $GITHUB_OUTPUT
          echo "$CHANGELOG" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
      
      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          tag_name: v${{ steps.version.outputs.version }}
          body: |
            ## Release v${{ steps.version.outputs.version }}
            
            ### Changes
            ${{ steps.changelog.outputs.changelog }}
          draft: false
          prerelease: false
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Upload Artifacts
        uses: actions/upload-artifact@v3
        with:
          name: release-${{ steps.version.outputs.version }}
          path: |
            src/
            package.json
          retention-days: 30
```

### Part 3: Version Bumping Workflow

`.github/workflows/bump-version.yml`:

```yaml
name: Bump Version

on:
  workflow_dispatch:
    inputs:
      bump-type:
        description: 'Version bump type'
        required: true
        type: choice
        options:
          - patch
          - minor
          - major

jobs:
  bump:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Configure Git
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
      
      - name: Get Current Version
        id: current
        run: |
          VERSION=$(node -p "require('./package.json').version")
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
      
      - name: Bump Version
        id: bump
        run: |
          npm version ${{ github.event.inputs.bump-type }} --no-git-tag-version
          NEW_VERSION=$(node -p "require('./package.json').version")
          echo "version=${NEW_VERSION}" >> $GITHUB_OUTPUT
      
      - name: Update Changelog
        run: |
          sed -i '0,/## \[Unreleased\]/s//## [Unreleased]\n\n## [${{ steps.bump.outputs.version }}] - '$(date +%Y-%m-%d)'/' CHANGELOG.md
      
      - name: Commit Changes
        run: |
          git add package.json package-lock.json CHANGELOG.md
          git commit -m "chore: bump version to ${{ steps.bump.outputs.version }}"
          git push origin main
      
      - name: Create Tag
        run: |
          git tag -a v${{ steps.bump.outputs.version }} -m "Release version ${{ steps.bump.outputs.version }}"
          git push origin v${{ steps.bump.outputs.version }}
```

### Part 4: Validate

**Test release workflow:**
```bash
# Option 1: Push tag (triggers workflow)
git tag v0.2.0
git push origin v0.2.0

# Option 2: Manual trigger (from Actions tab)
# Select "Bump Version" workflow
# Choose patch/minor/major
# Run workflow
```

**Expected Results:**
- ✅ Version updated in package.json
- ✅ Tag created and pushed
- ✅ Release created on GitHub
- ✅ Changelog generated
- ✅ Artifacts uploaded

**Success Criteria:**
- [ ] Release workflow creates GitHub release
- [ ] Version bumped correctly (patch/minor/major)
- [ ] Changelog updated with changes
- [ ] Git tag created with version
- [ ] Artifacts available for download

---

## Common Mistakes

### 1. Incorrect Version Format
```yaml
# ✗ Bad: Missing v prefix inconsistency
git tag 1.2.3

# ✓ Good: Consistent v prefix
git tag v1.2.3
```

### 2. Missing Permissions
```yaml
# ✗ Bad: No write permissions
jobs:
  release:
    runs-on: ubuntu-latest

# ✓ Good: Explicit permissions
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write
```

### 3. No Changelog Tracking
```yaml
# ✗ Bad: Release with no history
- run: echo "Release v1.0.0"

# ✓ Good: Include changelog
- name: Generate Changelog
  run: git log <prev-tag>..HEAD --oneline
```

### 4. Publishing Without Testing
```yaml
# ✗ Bad: Publish directly
- run: npm publish

# ✓ Good: Test first
- run: npm test
- run: npm build
- run: npm publish
```

### 5. Hardcoded Version Numbers
```yaml
# ✗ Bad: Version duplicated
version: "1.0.0"
git tag v1.0.0

# ✓ Good: Single source of truth
version: $(jq -r .version package.json)
```

---

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| "Tag not created" | Missing write permissions | Add `permissions: { contents: write }` |
| "Release not published" | Invalid GitHub token | Verify `GITHUB_TOKEN` available |
| "Version mismatch" | Multiple version sources | Use single source (package.json) |
| "Changelog empty" | Incorrect git range | Check prev tag: `git describe --tags` |
| "npm publish fails" | Not authenticated | Add `NPM_TOKEN` secret |
| "Breaking version parsing" | Non-semver format | Validate format before bumping |
| "Assets not attached" | Wrong artifact path | Verify path exists before release |

---

## Summary Checklist

- ✅ Semantic versioning implemented
- ✅ Conventional commits documented
- ✅ Release workflow automated
- ✅ Version bumping scripted
- ✅ Changelog auto-generated
- ✅ Git tags created and pushed
- ✅ Permissions properly configured
- ✅ Testing before release
- ✅ Artifacts uploaded with release
- ✅ Multiple trigger options (tag/manual/scheduled)

---

## Resources

- Semantic Versioning: https://semver.org
- Conventional Commits: https://www.conventionalcommits.org
- GitHub Releases: https://docs.github.com/en/repositories/releasing-projects-on-github
- conventional-changelog: https://github.com/conventional-changelog/conventional-changelog
- softprops/action-gh-release: https://github.com/softprops/action-gh-release
