# Module 10 Exercises: Release and Versioning Workflows

## Exercise 1: Semantic Versioning Basics

**Objective:** Understand and apply semantic versioning to a project.

**Starter Code:**

```json
// package.json
{
  "name": "my-app",
  "version": "1.0.0",
  "description": "My application"
}
```

**Requirements:**
- Start with version 1.0.0
- Create versions representing different change types
- Document version meanings

**Scenarios:**
1. Add new feature (backward compatible) → version should be?
2. Fix a bug → version should be?
3. Remove deprecated API → version should be?

**Acceptance Criteria:**
- [ ] Understand MAJOR.MINOR.PATCH
- [ ] Can identify correct version bump
- [ ] Can explain SemVer rules
- [ ] Document version history

**Hint:** MAJOR.MINOR.PATCH where feature=MINOR, bugfix=PATCH, breaking=MAJOR

---

## Exercise 2: Git Tagging Strategy

**Objective:** Implement consistent git tagging for releases.

**Starter Code:**

```bash
# Repository with commits
git log --oneline | head -5

# TODO: Create tags for releases
# TODO: Create annotated tags
# TODO: Push tags to remote
# TODO: List all tags
```

**Requirements:**
- Create lightweight tag: `v1.0.0`
- Create annotated tag: `v1.1.0` with message
- Create release notes
- Push tags to remote

**Acceptance Criteria:**
- [ ] Tag created with correct name format
- [ ] Annotated tags have messages
- [ ] Tags pushed to origin
- [ ] `git tag -l` shows all tags
- [ ] `git show <tag>` displays tag info

**Hint:** Annotated tags: `git tag -a v1.0.0 -m "Release message"`

---

## Exercise 3: Create Release Workflow

**Objective:** Automate release creation on tag push.

**Starter Code:**

```yaml
name: Create Release

on:
  push:
    tags:
      # TODO: Match version tags (v*)
      - ''

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
          # TODO: Extract version from tag
          # TODO: Output version for use in steps
          echo "version=1.0.0" >> $GITHUB_OUTPUT
      
      - name: Create Release
        # TODO: Use action to create GitHub release
        # TODO: Include version in tag_name and body
        # TODO: Use GITHUB_TOKEN
        run: echo "Release created"
```

**Requirements:**
- Trigger on tag push (v*)
- Extract version from tag
- Create GitHub release with version
- Add release notes in body
- Use softprops/action-gh-release or equivalent

**Acceptance Criteria:**
- [ ] Workflow file valid YAML
- [ ] Triggered by version tags
- [ ] Version extracted correctly
- [ ] Release created on GitHub
- [ ] Release visible in releases tab

**Hint:** Use regex pattern `v*` to match version tags.

---

## Exercise 4: Conventional Commits

**Objective:** Implement conventional commit format for automated versioning.

**Starter Code:**

```bash
# Commits following conventional format
git log --oneline | head -10

# Expected format:
# type(scope): description
#
# type: feat, fix, breaking, refactor, docs, etc.
```

**Requirements:**
- Document conventional commit format
- Create 5 commits with proper types
- Identify version bumps from commits
- Create script to detect commit types

**Acceptance Criteria:**
- [ ] At least 2 feat: commits
- [ ] At least 1 fix: commit
- [ ] Commit messages follow format
- [ ] Can extract changes by type
- [ ] Version bump correct for commits

**Hint:** `feat:` = MINOR, `fix:` = PATCH, `breaking:` = MAJOR

---

## Exercise 5: Automated Version Bumping

**Objective:** Automatically bump version based on changes.

**Starter Code:**

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
      
      - name: Configure Git
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
      
      # TODO: Get current version
      # TODO: Bump version using npm version
      # TODO: Update package.json
      # TODO: Commit changes
      # TODO: Push to main
      
      - name: Success
        run: echo "Version bumped successfully"
```

**Requirements:**
- Get current version from package.json
- Bump using npm version command
- Commit changes with message
- Push to main branch
- Support patch, minor, major

**Acceptance Criteria:**
- [ ] Current version extracted
- [ ] npm version bumps correctly
- [ ] package.json updated
- [ ] Changes committed
- [ ] Pushed to remote
- [ ] Correct bump type applied

**Hint:** `npm version patch/minor/major` handles bumping.

---

## Exercise 6: Changelog Generation

**Objective:** Auto-generate changelog from commits.

**Starter Code:**

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
          # TODO: Get previous tag
          # TODO: Get commits since previous tag
          # TODO: Format as markdown
          # TODO: Separate by type (feat, fix, breaking)
          
          PREV_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
          
          if [ -z "$PREV_TAG" ]; then
            CHANGELOG=$(git log --oneline)
          else
            CHANGELOG=$(git log ${PREV_TAG}..HEAD --oneline)
          fi
          
          echo "changelog<<EOF" >> $GITHUB_OUTPUT
          echo "## Changes" >> $GITHUB_OUTPUT
          echo "$CHANGELOG" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
      
      - name: Display Changelog
        run: echo "${{ steps.changelog.outputs.changelog }}"
```

**Requirements:**
- Find previous tag
- Get commits since tag
- Format commits by type
- Output as markdown
- Update CHANGELOG.md file

**Acceptance Criteria:**
- [ ] Previous tag found
- [ ] Commits listed since tag
- [ ] Formatted as markdown
- [ ] Separated by type
- [ ] CHANGELOG.md updated

**Hint:** Use git log with ranges: `PREV_TAG..HEAD`

---

## Exercise 7: Release Notes from PR Labels

**Objective:** Generate release notes from pull request labels.

**Starter Code:**

```yaml
name: Release Notes from PRs

on:
  push:
    tags:
      - 'v*'

jobs:
  generate-notes:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/github-script@v6
      
      - name: Get PRs
        id: prs
        uses: actions/github-script@v6
        with:
          script: |
            // TODO: Get merged PRs since last tag
            // TODO: Group by labels (enhancement, bug, breaking)
            // TODO: Format as markdown
            // TODO: Return release notes
            return "Release notes"
      
      - name: Create Release
        run: |
          echo "${{ steps.prs.outputs.result }}"
```

**Requirements:**
- Query merged PRs since last tag
- Group by label (enhancement, bug, etc.)
- Format as markdown sections
- Include PR numbers and titles
- Use in release notes

**Acceptance Criteria:**
- [ ] PRs queried from GitHub
- [ ] Grouped by label
- [ ] Formatted as markdown
- [ ] PR numbers included
- [ ] Release notes generated

**Hint:** Use github.rest API to query pulls with filters.

---

## Exercise 8: Automated Release on Tag

**Objective:** Fully automate release creation when tag is pushed.

**Starter Code:**

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
      # TODO: Run tests
      - run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      # TODO: Build artifacts
      - run: npm run build
      # TODO: Upload as artifact

  release:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write
    
    steps:
      - uses: actions/checkout@v3
      # TODO: Download artifacts
      # TODO: Create release with artifacts
      # TODO: Generate release notes
      # TODO: Attach files to release
```

**Requirements:**
- Test before release
- Build artifacts
- Create release with assets
- Include generated notes
- Attach build outputs

**Acceptance Criteria:**
- [ ] Test job passes
- [ ] Build job produces artifacts
- [ ] Release created with tag
- [ ] Artifacts attached to release
- [ ] Release notes included

**Hint:** Use needs: to create job dependencies.

---

## Exercise 9: NPM Package Publishing

**Objective:** Automatically publish npm package on release.

**Starter Code:**

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
          # TODO: Configure npm registry
          # TODO: Add authentication
      
      - run: npm ci
      - run: npm test
      - run: npm run build
      
      # TODO: Publish to npm registry
      # TODO: Use NPM_TOKEN from secrets
      # TODO: Add release tag
```

**Requirements:**
- Setup node with npm auth
- Run tests before publish
- Build distribution files
- Publish to npm registry
- Use NPM_TOKEN secret

**Acceptance Criteria:**
- [ ] Node setup includes registry
- [ ] Tests pass before publish
- [ ] Build completes
- [ ] npm publish succeeds
- [ ] Package visible on npm

**Hint:** Use `registry-url` and `NPM_TOKEN` in setup-node.

---

## Exercise 10: Multi-Environment Release

**Objective:** Release to multiple environments (staging, production).

**Starter Code:**

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
    
    steps:
      - name: Determine Environment
        id: env
        run: |
          TAG=${{ github.ref_name }}
          
          if [[ $TAG == v* ]]; then
            # TODO: Production release
            echo "environment=production" >> $GITHUB_OUTPUT
            echo "prerelease=false" >> $GITHUB_OUTPUT
          elif [[ $TAG == rc-* ]]; then
            # TODO: Release candidate
            echo "environment=staging" >> $GITHUB_OUTPUT
            echo "prerelease=true" >> $GITHUB_OUTPUT
          else
            # TODO: Beta release
            echo "environment=beta" >> $GITHUB_OUTPUT
            echo "prerelease=true" >> $GITHUB_OUTPUT
          fi

  deploy:
    needs: determine-env
    runs-on: ubuntu-latest
    environment: ${{ needs.determine-env.outputs.environment }}
    
    steps:
      - uses: actions/checkout@v3
      # TODO: Deploy to environment
      # TODO: Run environment-specific tests
      # TODO: Create release with environment label
      - run: echo "Deploying to ${{ needs.determine-env.outputs.environment }}"
```

**Requirements:**
- Detect tag type (v*, rc-*, beta-*)
- Determine environment (prod, staging, beta)
- Set prerelease flag appropriately
- Deploy to environment
- Create release with label

**Acceptance Criteria:**
- [ ] Tag type detected
- [ ] Correct environment selected
- [ ] Prerelease flag set
- [ ] Deployment triggered
- [ ] Release created with type

**Hint:** Use regex in condition: `if [[ $TAG == rc-* ]]`

---

## Summary

You now understand:
- ✅ Semantic versioning principles
- ✅ Git tagging strategies
- ✅ Release workflow automation
- ✅ Conventional commit format
- ✅ Automated version bumping
- ✅ Changelog generation
- ✅ Release notes from PRs
- ✅ Full release automation
- ✅ NPM package publishing
- ✅ Multi-environment releases

Continue mastering release automation!
