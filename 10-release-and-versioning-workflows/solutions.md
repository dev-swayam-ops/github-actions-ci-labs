# Module 10 Solutions: Release and Versioning Workflows

## Exercise 1: Semantic Versioning Basics

**Solution:**

**Current version:** 1.0.0 (MAJOR.MINOR.PATCH)

**Scenario Responses:**

1. **Add new feature (backward compatible)**
   - Version: 1.1.0 (MINOR bump)
   - Reason: New functionality, backward compatible

2. **Fix a bug**
   - Version: 1.0.1 (PATCH bump)
   - Reason: Bug fix only, no new features

3. **Remove deprecated API (breaking change)**
   - Version: 2.0.0 (MAJOR bump)
   - Reason: Breaking change, incompatible

**SemVer Rules:**
- MAJOR (X.0.0): Incompatible API changes
- MINOR (0.X.0): Backward-compatible new features
- PATCH (0.0.X): Backward-compatible bug fixes

**Version History Example:**
```
v1.0.0 → Initial release
v1.1.0 → Added authentication
v1.1.1 → Fixed database bug
v1.2.0 → Added logging feature
v2.0.0 → Removed deprecated endpoints
```

---

## Exercise 2: Git Tagging Strategy

**Solution:**

```bash
# Create lightweight tag
git tag v1.0.0
git push origin v1.0.0

# Create annotated tag (recommended)
git tag -a v1.1.0 -m "Release version 1.1.0: Added user authentication"
git push origin v1.1.0

# Create another annotated tag
git tag -a v1.2.0 -m "Release version 1.2.0: Improved API performance"
git push origin v1.2.0

# List all tags
git tag -l

# Show tag details
git show v1.1.0

# Delete tag (if needed)
git tag -d v1.0.0
git push origin --delete v1.0.0
```

**Expected Output:**
```
v1.0.0
v1.1.0
v1.2.0
```

**Recommended:** Always use annotated tags in production.

---

## Exercise 3: Create Release Workflow

**Solution:**

```yaml
name: Create Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Get Version
        id: version
        run: |
          VERSION=${GITHUB_REF#refs/tags/v}
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
      
      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          tag_name: v${{ steps.version.outputs.version }}
          body: |
            ## Release v${{ steps.version.outputs.version }}
            
            See commit history for details.
          draft: false
          prerelease: false
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**How it works:**
- Triggers on tag push (v*)
- Extracts version from tag name
- Creates GitHub release automatically
- Uses softprops/action-gh-release action
- Release visible in "Releases" tab

**Test:**
```bash
git tag v0.2.0
git push origin v0.2.0
# Workflow triggers automatically
```

---

## Exercise 4: Conventional Commits

**Solution:**

**Commit Examples:**

```bash
# Feature commits (MINOR bump)
git commit -m "feat(api): add user authentication system"
git commit -m "feat(auth): implement JWT token generation"

# Bug fix commits (PATCH bump)
git commit -m "fix(api): resolve database timeout issue"
git commit -m "fix(auth): fix token expiration calculation"

# Breaking change (MAJOR bump)
git commit -m "breaking(api): remove deprecated /v1 endpoints"

# Documentation
git commit -m "docs(readme): update installation instructions"

# Refactor (no version bump)
git commit -m "refactor(api): simplify authentication logic"
```

**View commits:**
```bash
git log --oneline --grep="^feat" --grep="^fix" --grep="^breaking" -E
```

**Version bump determination:**
```
- feat: commits → MINOR bump (1.0.0 → 1.1.0)
- fix: commits → PATCH bump (1.0.0 → 1.0.1)
- breaking: commits → MAJOR bump (1.0.0 → 2.0.0)
```

---

## Exercise 5: Automated Version Bumping

**Solution:**

```yaml
name: Bump Version

on:
  workflow_dispatch:
    inputs:
      bump-type:
        description: 'patch, minor, or major'
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
      
      - name: Commit Changes
        run: |
          git add package.json package-lock.json
          git commit -m "chore: bump version to ${{ steps.bump.outputs.version }}"
          git push origin main
      
      - name: Create Tag
        run: |
          git tag -a v${{ steps.bump.outputs.version }} -m "Release version ${{ steps.bump.outputs.version }}"
          git push origin v${{ steps.bump.outputs.version }}
      
      - name: Summary
        run: |
          echo "Version bumped from ${{ steps.current.outputs.version }} to ${{ steps.bump.outputs.version }}"
```

**Usage:**
- Go to Actions tab
- Select "Bump Version"
- Choose patch/minor/major
- Run workflow
- Automatically creates tag and commits

---

## Exercise 6: Changelog Generation

**Solution:**

```yaml
name: Generate Changelog

on:
  workflow_dispatch:

jobs:
  changelog:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Generate Changelog
        id: changelog
        run: |
          # Get previous tag
          PREV_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
          
          echo "Previous tag: $PREV_TAG"
          
          if [ -z "$PREV_TAG" ]; then
            # No previous tag, get all commits
            CHANGELOG=$(git log --oneline --pretty=format:"- %s (%h)")
          else
            # Get commits since previous tag
            CHANGELOG=$(git log ${PREV_TAG}..HEAD --oneline --pretty=format:"- %s (%h)")
          fi
          
          # Output to variable
          echo "changelog<<EOF" >> $GITHUB_OUTPUT
          echo "## Changes" >> $GITHUB_OUTPUT
          echo "" >> $GITHUB_OUTPUT
          echo "### Features" >> $GITHUB_OUTPUT
          echo "$CHANGELOG" | grep "feat:" >> $GITHUB_OUTPUT || echo "None"
          echo "" >> $GITHUB_OUTPUT
          echo "### Bug Fixes" >> $GITHUB_OUTPUT
          echo "$CHANGELOG" | grep "fix:" >> $GITHUB_OUTPUT || echo "None"
          echo "EOF" >> $GITHUB_OUTPUT
      
      - name: Update CHANGELOG.md
        run: |
          VERSION=$(node -p "require('./package.json').version" 2>/dev/null || echo "1.0.0")
          DATE=$(date +%Y-%m-%d)
          
          {
            echo "# Changelog"
            echo ""
            echo "## [Unreleased]"
            echo ""
            echo "## [$VERSION] - $DATE"
            echo ""
            echo "${{ steps.changelog.outputs.changelog }}"
            echo ""
            cat CHANGELOG.md 2>/dev/null || echo ""
          } > CHANGELOG.md.new
          
          mv CHANGELOG.md.new CHANGELOG.md
          
          git add CHANGELOG.md
          git commit -m "docs: update changelog for $VERSION" || true
          git push origin main || true
```

---

## Exercise 7: Release Notes from PR Labels

**Solution:**

```yaml
name: Release Notes from PRs

on:
  push:
    tags:
      - 'v*'

jobs:
  generate-notes:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
    
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - uses: actions/github-script@v6
        id: prs
        with:
          script: |
            const tag = context.ref.replace('refs/tags/v', '');
            
            // Get previous tag
            let prevTag;
            try {
              const tags = await exec.getExecOutput('git tag -l --sort=-version:refname');
              const tagList = tags.stdout.trim().split('\n');
              prevTag = tagList[1] || '';
            } catch {
              prevTag = '';
            }
            
            // Get merged PRs
            const { data: pulls } = await github.rest.pulls.list({
              owner: context.repo.owner,
              repo: context.repo.repo,
              state: 'closed',
              sort: 'updated',
              direction: 'desc',
              per_page: 100
            });
            
            // Group by label
            const features = [];
            const bugfixes = [];
            const breaking = [];
            
            for (const pr of pulls) {
              if (!pr.merged_at) continue;
              
              const labels = pr.labels.map(l => l.name);
              
              if (labels.includes('breaking')) {
                breaking.push(`- ${pr.title} (#${pr.number})`);
              } else if (labels.includes('enhancement')) {
                features.push(`- ${pr.title} (#${pr.number})`);
              } else if (labels.includes('bug')) {
                bugfixes.push(`- ${pr.title} (#${pr.number})`);
              }
            }
            
            // Format output
            let output = `## Release v${tag}\n\n`;
            
            if (breaking.length > 0) {
              output += `### ⚠️ Breaking Changes\n${breaking.join('\n')}\n\n`;
            }
            if (features.length > 0) {
              output += `### ✨ Features\n${features.join('\n')}\n\n`;
            }
            if (bugfixes.length > 0) {
              output += `### 🐛 Bug Fixes\n${bugfixes.join('\n')}\n\n`;
            }
            
            return output;
      
      - name: Create Release with Notes
        uses: softprops/action-gh-release@v1
        with:
          body: ${{ steps.prs.outputs.result }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Exercise 8: Automated Release on Tag

**Solution:**

```yaml
name: Automated Release

on:
  push:
    tags:
      - 'v*'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v3
        with:
          name: build-artifacts
          path: dist/

  release:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write
    
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Download Artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-artifacts
          path: dist/
      
      - name: Get Version
        id: version
        run: |
          VERSION=${GITHUB_REF#refs/tags/v}
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
      
      - name: Generate Release Notes
        id: notes
        run: |
          PREV_TAG=$(git describe --tags --abbrev=0 2>/dev/null | head -1 || echo "")
          
          if [ -z "$PREV_TAG" ]; then
            NOTES=$(git log --oneline)
          else
            NOTES=$(git log ${PREV_TAG}..HEAD --oneline)
          fi
          
          echo "notes<<EOF" >> $GITHUB_OUTPUT
          echo "$NOTES" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
      
      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          tag_name: v${{ steps.version.outputs.version }}
          body: |
            ## Release v${{ steps.version.outputs.version }}
            
            ### Changes
            ${{ steps.notes.outputs.notes }}
          files: dist/*
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Exercise 9: NPM Package Publishing

**Solution:**

```yaml
name: Publish to NPM

on:
  push:
    tags:
      - 'v*'

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          registry-url: https://registry.npmjs.org
      
      - run: npm ci
      - run: npm test
      - run: npm run build
      
      - name: Get Version
        id: version
        run: |
          VERSION=$(node -p "require('./package.json').version")
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
      
      - name: Publish
        run: npm publish --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
      
      - name: Verify Published
        run: npm view @myapp/package@${{ steps.version.outputs.version }}
```

**Setup:**
1. Create npm token: https://www.npmjs.com/settings/tokens
2. Add as GitHub Secret: `NPM_TOKEN`
3. Ensure package.json has correct name and version

---

## Exercise 10: Multi-Environment Release

**Solution:**

```yaml
name: Multi-Env Release

on:
  push:
    tags:
      - 'v*'
      - 'rc-*'
      - 'beta-*'

jobs:
  determine-env:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.env.outputs.environment }}
      prerelease: ${{ steps.env.outputs.prerelease }}
      version: ${{ steps.env.outputs.version }}
    
    steps:
      - name: Determine Environment
        id: env
        run: |
          TAG=${{ github.ref_name }}
          
          if [[ $TAG == v* ]]; then
            # Production release
            VERSION=${TAG:1}  # Remove 'v' prefix
            echo "environment=production" >> $GITHUB_OUTPUT
            echo "prerelease=false" >> $GITHUB_OUTPUT
          elif [[ $TAG == rc-* ]]; then
            # Release candidate
            VERSION=${TAG:3}  # Remove 'rc-' prefix
            echo "environment=staging" >> $GITHUB_OUTPUT
            echo "prerelease=true" >> $GITHUB_OUTPUT
          else
            # Beta release
            VERSION=${TAG:5}  # Remove 'beta-' prefix
            echo "environment=beta" >> $GITHUB_OUTPUT
            echo "prerelease=true" >> $GITHUB_OUTPUT
          fi
          
          echo "version=${VERSION}" >> $GITHUB_OUTPUT

  deploy:
    needs: determine-env
    runs-on: ubuntu-latest
    environment: ${{ needs.determine-env.outputs.environment }}
    permissions:
      contents: write
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci
      - run: npm test
      - run: npm run build
      
      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          tag_name: ${{ github.ref_name }}
          body: |
            ## Release ${{ github.ref_name }}
            
            **Environment:** ${{ needs.determine-env.outputs.environment }}
            **Prerelease:** ${{ needs.determine-env.outputs.prerelease }}
            **Version:** ${{ needs.determine-env.outputs.version }}
          prerelease: ${{ needs.determine-env.outputs.prerelease == 'true' }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Deploy
        run: |
          echo "Deploying to ${{ needs.determine-env.outputs.environment }}"
          echo "Version: ${{ needs.determine-env.outputs.version }}"
          echo "Prerelease: ${{ needs.determine-env.outputs.prerelease }}"
```

**Usage:**
```bash
# Production release
git tag v1.0.0
git push origin v1.0.0

# Release candidate
git tag rc-1.0.0
git push origin rc-1.0.0

# Beta release
git tag beta-1.0.0
git push origin beta-1.0.0
```

---

## Summary

You now understand:
- ✅ Semantic versioning application
- ✅ Git tagging best practices
- ✅ Automated release creation
- ✅ Conventional commit usage
- ✅ Automated version bumping
- ✅ Changelog generation from commits
- ✅ Release notes from PR labels
- ✅ Full release automation pipeline
- ✅ NPM package publishing
- ✅ Multi-environment release strategies

Continue mastering release automation!
