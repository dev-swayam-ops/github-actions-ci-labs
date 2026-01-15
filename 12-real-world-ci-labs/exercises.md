# Module 12 Exercises: Real-World CI/CD Labs

## Exercise 1: Static Website Deployment

**Objective:** Deploy a static website (HTML/CSS/JS) to GitHub Pages.

**Starter Code:**

```yaml
name: Deploy Static Site

on:
  push:
    branches: [main]
    paths:
      - 'docs/**'
      - '.github/workflows/deploy-static.yml'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build static site
        run: |
          # TODO: Build/optimize HTML, CSS, JS
          mkdir -p build
          cp docs/index.html build/
          cp docs/styles.css build/
          cp docs/script.js build/
      
      - name: Upload artifacts
        # TODO: Upload build to artifact storage
        uses: actions/upload-artifact@v3
        with:
          name: site
          path: build/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Download artifacts
        uses: actions/download-artifact@v3
        with:
          name: site
          path: build/
      
      - name: Deploy to GitHub Pages
        # TODO: Deploy static site to GitHub Pages
        run: |
          echo "Deploying static site..."
          # In real scenario: push to gh-pages branch or use actions
```

**Requirements:**
- Build static assets
- Create artifact
- Deploy to hosting
- Verify deployment

**Acceptance Criteria:**
- [ ] Workflow runs on main branch
- [ ] Build completes successfully
- [ ] Artifacts created
- [ ] Deployment step included
- [ ] Site accessible (or simulated)

**Hint:** Use actions/deploy-pages for GitHub Pages.

---

## Exercise 2: Node.js API with Database Testing

**Objective:** Test Node.js API against real database service.

**Starter Code:**

```yaml
name: API Integration Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        # TODO: Setup PostgreSQL service container
        image: postgres:15-alpine
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - run: npm ci
      - run: npm run migrate  # Run database migrations
      
      - name: Run API tests
        run: npm run test:api
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb
```

**Requirements:**
- PostgreSQL service running
- Database migrations
- API tests against live DB
- Proper environment variables

**Acceptance Criteria:**
- [ ] Service container starts
- [ ] Migrations run successfully
- [ ] API tests connect to DB
- [ ] Tests pass or fail appropriately
- [ ] Logs show database operations

**Hint:** Services are accessible at service name (postgres) in job.

---

## Exercise 3: Docker Image Build and Push

**Objective:** Build Docker image and push to registry.

**Starter Code:**

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/myapp

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Log in to Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          # TODO: Add authentication
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and Push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          # TODO: Add tags for version and latest
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Requirements:**
- Docker image builds
- Pushes to registry
- Multiple tags applied
- Caching enabled

**Acceptance Criteria:**
- [ ] Image builds successfully
- [ ] Authenticates with registry
- [ ] Pushes with correct tags
- [ ] Latest and SHA tags present
- [ ] Build cache working

**Hint:** docker/build-push-action handles entire process.

---

## Exercise 4: Multi-Environment Deployment

**Objective:** Deploy to dev, staging, and production environments.

**Starter Code:**

```yaml
name: Multi-Env Deploy

on:
  push:
    branches: [main, develop]

jobs:
  deploy-dev:
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: development
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to Dev
        run: |
          echo "Deploying to development..."
          # TODO: Deploy to dev environment

  deploy-staging:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to Staging
        run: |
          echo "Deploying to staging..."
          # TODO: Deploy to staging environment

  deploy-prod:
    if: github.ref == 'refs/heads/main'
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      # TODO: Add production URL
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to Production
        run: |
          echo "Deploying to production..."
          # TODO: Deploy to production environment
```

**Requirements:**
- Deploy to different environments
- Based on branch
- Staging before production
- Manual approval for production

**Acceptance Criteria:**
- [ ] Dev deploys on develop branch
- [ ] Staging auto-deploys on main
- [ ] Production requires approval
- [ ] Environments defined in GitHub
- [ ] Conditional logic correct

**Hint:** Use `environment:` block for approval gates.

---

## Exercise 5: Health Checks and Validation

**Objective:** Validate deployments with health checks.

**Starter Code:**

```yaml
name: Deploy with Health Checks

on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy Application
        run: |
          echo "Deploying..."
          sleep 2
          echo "✓ Deployed"
      
      - name: Wait for service startup
        run: sleep 5
      
      - name: Health Check
        run: |
          # TODO: Check service health
          MAX_ATTEMPTS=10
          ATTEMPT=1
          
          while [ $ATTEMPT -le $MAX_ATTEMPTS ]; do
            if curl -f http://localhost:3000/health 2>/dev/null; then
              echo "✓ Service healthy"
              exit 0
            fi
            ATTEMPT=$((ATTEMPT + 1))
            sleep 2
          done
          
          echo "✗ Service failed health check"
          exit 1
      
      - name: Smoke Tests
        run: |
          # TODO: Run basic API tests
          curl -f http://localhost:3000/api/status || exit 1
          echo "✓ Smoke tests passed"
      
      - name: Rollback on Failure
        if: failure()
        run: |
          echo "Deploying previous version..."
          # TODO: Rollback logic
```

**Requirements:**
- Deploy application
- Health check endpoint
- Retry logic
- Rollback on failure

**Acceptance Criteria:**
- [ ] Deployment executes
- [ ] Health check validates service
- [ ] Retries on failure
- [ ] Rollback triggered on health fail
- [ ] Logs show status

**Hint:** `if: failure()` triggers on previous step failure.

---

## Exercise 6: Notification on Deployment

**Objective:** Send notifications for deployment status.

**Starter Code:**

```yaml
name: Deploy with Notifications

on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy
        id: deploy
        run: |
          echo "Deploying..."
          echo "deployment_url=https://myapp.com" >> $GITHUB_OUTPUT
      
      - name: Notify on Success
        if: success()
        run: |
          # TODO: Send success notification
          echo "✓ Deployment successful"
          # Could be: Slack, email, webhook
      
      - name: Notify on Failure
        if: failure()
        run: |
          # TODO: Send failure notification
          echo "✗ Deployment failed"
          # Alert team immediately
      
      - name: Post to Slack
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployment: ${{ job.status }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "Deployment Status: *${{ job.status }}*\nCommit: ${{ github.sha }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

**Requirements:**
- Notifications on success
- Notifications on failure
- Include deployment details
- Multiple notification channels

**Acceptance Criteria:**
- [ ] Success notification sent
- [ ] Failure notification sent
- [ ] Includes relevant info
- [ ] Slack webhook configured
- [ ] Notifications helpful

**Hint:** Use `if: success()` and `if: failure()` conditions.

---

## Exercise 7: Canary Deployment

**Objective:** Gradually roll out new version to subset of users.

**Starter Code:**

```yaml
name: Canary Deployment

on:
  push:
    tags: ['v*']

jobs:
  canary:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to 5% of Users
        run: |
          echo "Deploying new version to 5% canary group..."
          # TODO: Route 5% traffic to new version
      
      - name: Monitor Metrics
        run: |
          echo "Monitoring error rates, latency..."
          sleep 5
          # TODO: Check metrics
          # If bad metrics detected, rollback
      
      - name: Proceed if Healthy
        run: |
          echo "Canary metrics healthy, proceeding..."
      
      - name: Deploy to 50% of Users
        run: |
          echo "Deploying to 50%..."
          # TODO: Increase traffic
      
      - name: Monitor Again
        run: |
          echo "Monitoring 50% deployment..."
          sleep 5
      
      - name: Full Rollout
        run: |
          echo "All users now on new version..."
          # TODO: 100% traffic to new version
```

**Requirements:**
- Canary deployment steps
- Traffic routing
- Monitoring between steps
- Rollback capability

**Acceptance Criteria:**
- [ ] Deploy to small percentage
- [ ] Monitor metrics
- [ ] Gradually increase traffic
- [ ] Full rollout on success
- [ ] Rollback on failure

**Hint:** Actual implementation requires load balancer/service mesh support.

---

## Exercise 8: Blue-Green Deployment

**Objective:** Maintain two production environments for instant rollback.

**Starter Code:**

```yaml
name: Blue-Green Deploy

on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Get Current Environment
        id: current
        run: |
          # Check which is live (blue or green)
          CURRENT=$(curl -s http://myapp.com/version | jq -r .env)
          if [ "$CURRENT" = "blue" ]; then
            echo "target=green" >> $GITHUB_OUTPUT
          else
            echo "target=blue" >> $GITHUB_OUTPUT
          fi
      
      - name: Deploy to Inactive Environment
        run: |
          echo "Deploying to ${{ steps.current.outputs.target }}..."
          # TODO: Deploy new version to inactive env
      
      - name: Test New Environment
        run: |
          echo "Testing new deployment..."
          # TODO: Run smoke tests on new env
      
      - name: Switch Traffic
        run: |
          echo "Switching traffic to ${{ steps.current.outputs.target }}..."
          # TODO: Update load balancer
          # Old version still running, instant rollback possible
      
      - name: Keep Old Version Ready
        run: |
          echo "Old version ready for quick rollback"
          # Both versions running, quick switch if needed
```

**Requirements:**
- Two environments (blue/green)
- Deploy to inactive
- Test before switching
- Instant rollback capable

**Acceptance Criteria:**
- [ ] Two environments defined
- [ ] Deploy to inactive
- [ ] Testing before switch
- [ ] Traffic routed to new
- [ ] Old version ready

**Hint:** Requires infrastructure supporting multiple versions.

---

## Exercise 9: Scheduled Maintenance Tasks

**Objective:** Run maintenance jobs on schedule (backups, cleanup, etc).

**Starter Code:**

```yaml
name: Scheduled Maintenance

on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM UTC
  workflow_dispatch:     # Allow manual trigger

jobs:
  backup:
    runs-on: ubuntu-latest
    steps:
      - name: Backup Database
        run: |
          echo "Creating database backup..."
          # TODO: Execute backup command
          echo "✓ Backup complete"
      
      - name: Upload Backup
        uses: actions/upload-artifact@v3
        with:
          name: db-backup
          path: backup/
          retention-days: 30

  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Clean Old Logs
        run: |
          echo "Removing logs older than 90 days..."
          # TODO: Cleanup old logs
      
      - name: Clean Temp Files
        run: |
          echo "Cleaning temporary files..."
          # TODO: Cleanup temp storage

  health-check:
    runs-on: ubuntu-latest
    steps:
      - name: Check All Systems
        run: |
          echo "Checking system health..."
          # TODO: Comprehensive health checks
```

**Requirements:**
- Scheduled execution
- Multiple maintenance tasks
- Backup storage
- Cleanup operations
- Health validation

**Acceptance Criteria:**
- [ ] Schedule configured
- [ ] Runs at specified time
- [ ] Manual trigger available
- [ ] All jobs execute
- [ ] Results logged

**Hint:** cron format: minute hour day month day-of-week

---

## Exercise 10: Complete Enterprise CI/CD Pipeline

**Objective:** Combine all concepts into full enterprise pipeline.

**Starter Code:**

```yaml
name: Enterprise CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # TODO: Implement complete pipeline:
  # 1. Code quality checks
  # 2. Security scanning
  # 3. Unit tests
  # 4. Integration tests
  # 5. Build artifacts
  # 6. Deploy to dev
  # 7. Deploy to staging (auto on main)
  # 8. Deploy to production (manual approval)
  # 9. Health checks
  # 10. Notifications
  
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm run lint
      - run: npm test

  build:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm run build

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to staging..."

  deploy-prod:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: echo "Deploying to production..."
```

**Requirements:**
- Quality checks
- Build step
- Multi-environment deploy
- Approval gates
- Health validation
- Notifications

**Acceptance Criteria:**
- [ ] All modules' concepts included
- [ ] Proper job ordering
- [ ] Conditional deployments
- [ ] Manual approvals work
- [ ] Complete workflow functional

**Hint:** Combine previous modules and exercises.

---

## Summary

You now understand:
- ✅ Static site deployment
- ✅ Service testing with dependencies
- ✅ Docker image building
- ✅ Multi-environment deployments
- ✅ Health checks and validation
- ✅ Deployment notifications
- ✅ Canary deployment patterns
- ✅ Blue-green deployments
- ✅ Scheduled maintenance
- ✅ Complete enterprise pipelines

Continue building production-ready CI/CD systems!
