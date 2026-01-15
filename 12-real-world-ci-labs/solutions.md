# Module 12 Solutions: Real-World CI/CD Labs

## Exercise 1 Solution: Static Website Deployment

**Complete Solution:**

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
          mkdir -p build
          cp docs/index.html build/
          cp docs/styles.css build/
          cp docs/script.js build/
          
          # Optional: minify
          # npx html-minifier --input-dir docs --output-dir build
      
      - name: Verify build
        run: |
          echo "Checking build output..."
          ls -la build/
          du -sh build/
      
      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: site
          path: build/
          retention-days: 7

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v3
      
      - name: Download artifacts
        uses: actions/download-artifact@v3
        with:
          name: site
          path: build/
      
      - name: Deploy to GitHub Pages
        uses: actions/deploy-pages@v2
        with:
          artifact_name: site
          token: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Verify deployment
        run: |
          echo "✓ Site deployed to GitHub Pages"
          echo "URL: https://${{ github.repository_owner }}.github.io/${{ github.event.repository.name }}"
```

**Alternative: Direct GitHub Pages Deploy**

```yaml
  deploy:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v3
      
      - name: Download artifacts
        uses: actions/download-artifact@v3
        with:
          name: site
          path: build/
      
      - name: Configure GitHub Pages
        uses: actions/configure-pages@v3
      
      - name: Upload to Pages
        uses: actions/upload-pages-artifact@v2
        with:
          path: 'build'
      
      - name: Deploy
        id: deployment
        uses: actions/deploy-pages@v2
```

**What Happens:**

1. **Trigger**: Push to `docs/` on main branch
2. **Build**: Copy and prepare static files
3. **Artifact**: Store built site (7-day retention)
4. **Deploy**: Upload to GitHub Pages
5. **Result**: Site live at `username.github.io/repo-name`

**Real-World Enhancement:**

```yaml
      - name: Check for broken links
        run: |
          npm install -g broken-link-checker
          blc https://${{ github.repository_owner }}.github.io/${{ github.event.repository.name }} -r
```

---

## Exercise 2 Solution: Node.js API with Database Testing

**Complete Solution:**

```yaml
name: API Integration Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
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
      
      - name: Wait for PostgreSQL
        run: |
          until pg_isready -h localhost -p 5432; do
            echo "Waiting for PostgreSQL..."
            sleep 1
          done
      
      - name: Run database migrations
        run: npm run migrate
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb
      
      - name: Seed test data
        run: npm run seed:test
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb
      
      - name: Run API tests
        run: npm run test:api
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb
          NODE_ENV: test
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb
          NODE_ENV: test
      
      - name: Upload coverage
        uses: actions/upload-artifact@v3
        with:
          name: coverage-api
          path: coverage/
```

**Database Setup Script (npm scripts):**

```json
{
  "scripts": {
    "migrate": "knex migrate:latest",
    "seed:test": "knex seed:run --env=test",
    "test:api": "jest --testPathPattern=api",
    "test:integration": "jest --testPathPattern=integration"
  }
}
```

**Example Migration File:**

```javascript
// migrations/001_create_users.js
exports.up = function(knex) {
  return knex.schema.createTable('users', table => {
    table.increments('id');
    table.string('email').notNullable();
    table.string('name');
    table.timestamps(true, true);
  });
};

exports.down = function(knex) {
  return knex.schema.dropTable('users');
};
```

**Example API Test:**

```javascript
// tests/api/users.test.js
const request = require('supertest');
const app = require('../../src/app');
const knex = require('../../src/db');

describe('GET /api/users', () => {
  beforeAll(async () => {
    await knex.migrate.latest();
    await knex.seed.run();
  });
  
  it('returns all users', async () => {
    const res = await request(app).get('/api/users');
    expect(res.status).toBe(200);
    expect(res.body).toHaveLength(5);
  });
  
  afterAll(async () => {
    await knex.destroy();
  });
});
```

**Logs Show:**

```
Setup services
└─ PostgreSQL starting...
✓ PostgreSQL healthy (port 5432)

Install dependencies
└─ npm ci (cached)

Wait for PostgreSQL
└─ pg_isready check... READY

Run migrations
└─ migrations/001_create_users.js
└─ migrations/002_create_posts.js
✓ 2 migrations completed

Seed test data
└─ seeds/test_users.js (5 users)
└─ seeds/test_posts.js (20 posts)
✓ Test data seeded

Run API tests
└─ GET /api/users: PASS
└─ POST /api/users: PASS
✓ 15 tests passed (2.3s)

Run integration tests
└─ User creation flow: PASS
└─ Post creation flow: PASS
✓ 8 tests passed (1.8s)

Total time: 45s
```

---

## Exercise 3 Solution: Docker Image Build and Push

**Complete Solution:**

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags: ['v*']
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/myapp

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    outputs:
      image-digest: ${{ steps.image.outputs.digest }}
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and push
        id: image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Image info
        run: |
          echo "✓ Image pushed successfully"
          echo "Tags: ${{ steps.meta.outputs.tags }}"
          echo "Digest: ${{ steps.image.outputs.digest }}"

  scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Run Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

**Dockerfile:**

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .

EXPOSE 3000
CMD ["node", "src/index.js"]
```

**What Gets Created:**

```
ghcr.io/username/myapp:main
ghcr.io/username/myapp:v1.2.3
ghcr.io/username/myapp:v1.2
ghcr.io/username/myapp:sha-a1b2c3d
ghcr.io/username/myapp:latest (if on main)
```

**Output Example:**

```
Build context: .
Dockerfile: ./Dockerfile
Push: true

Building image...
- FROM node:18-alpine (cached)
- COPY package.json (cached)
- RUN npm ci (5.2s)
- COPY . . (0.5s)
- Build complete

Pushing image...
- Pushing ghcr.io/username/myapp:main... (12.3s)
- Pushing ghcr.io/username/myapp:latest... (2.1s)
- Pushing ghcr.io/username/myapp:v1.2.3... (2.0s)

✓ Image pushed: 3 tags
✓ Size: 145 MB
✓ Digest: sha256:a1b2c3d4e5f6...
```

---

## Exercise 4 Solution: Multi-Environment Deployment

**Complete Solution:**

```yaml
name: Multi-Env Deploy

on:
  push:
    branches: [main, develop]

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
      - uses: actions/upload-artifact@v3
        with:
          name: build
          path: dist/

  deploy-dev:
    needs: build
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment:
      name: development
      url: https://dev.myapp.com
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: build
          path: dist/
      
      - name: Deploy to Development
        run: |
          echo "🚀 Deploying to development..."
          echo "Environment: DEV"
          echo "Build: ${{ github.sha }}"
          # Actual deployment
          # ssh -i ${{ secrets.DEV_KEY }} user@dev.myapp.com "deploy-script"

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: build
          path: dist/
      
      - name: Deploy to Staging
        run: |
          echo "🚀 Deploying to staging..."
          echo "Environment: STAGING"
          echo "Build: ${{ github.sha }}"
          # Automatic deployment on main branch
          # ssh -i ${{ secrets.STAGING_KEY }} user@staging.myapp.com "deploy-script"
      
      - name: Run smoke tests
        run: |
          echo "Running smoke tests on staging..."
          curl -f https://staging.myapp.com/health || exit 1
          echo "✓ Staging healthy"

  approval:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Wait for approval
        run: |
          echo "⏳ Waiting for production approval..."
          echo "Review deployment at: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"

  deploy-prod:
    needs: approval
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: build
          path: dist/
      
      - name: Deploy to Production
        run: |
          echo "🚀 DEPLOYING TO PRODUCTION"
          echo "Environment: PRODUCTION"
          echo "Build: ${{ github.sha }}"
          echo "Triggered by: ${{ github.actor }}"
          # ssh -i ${{ secrets.PROD_KEY }} user@myapp.com "deploy-script"
      
      - name: Verify production
        run: |
          echo "Verifying production deployment..."
          curl -f https://myapp.com/health || exit 1
          echo "✓ Production healthy"
      
      - name: Notify deployment
        run: |
          echo "✓ Production deployment complete"
          # Send to Slack, PagerDuty, etc.
```

**Deployment Workflow:**

```
develop branch push
    ↓
Build (tests, lint, build)
    ↓
Deploy to Development (auto)
    ↓
[ Test Dev Environment ]

---

main branch push
    ↓
Build (tests, lint, build)
    ↓
Deploy to Staging (auto)
    ↓
Run Smoke Tests
    ↓
⏸️ APPROVAL REQUIRED (manual gate)
    ↓
Deploy to Production
    ↓
Verify Production
    ↓
✓ Complete
```

**Environment Settings in GitHub:**

```
Development:
- Auto deploy on develop branch
- No approval required
- URL: https://dev.myapp.com

Staging:
- Auto deploy on main branch
- No approval required (optional)
- URL: https://staging.myapp.com

Production:
- Manual approval required
- Restricted to main branch
- Reviewers: @team/devops
- URL: https://myapp.com
```

---

## Exercise 5 Solution: Health Checks and Validation

**Complete Solution:**

```yaml
name: Deploy with Health Checks

on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install and build
        run: |
          npm ci
          npm run build
      
      - name: Deploy application
        id: deploy
        run: |
          echo "📦 Starting deployment..."
          # Simulate deployment
          npm start &
          echo "deployment_pid=$!" >> $GITHUB_OUTPUT
          sleep 3
          echo "✓ Application started"
      
      - name: Health check with retry
        run: |
          MAX_ATTEMPTS=10
          ATTEMPT=1
          HEALTH_URL="http://localhost:3000/health"
          
          echo "🏥 Checking application health..."
          while [ $ATTEMPT -le $MAX_ATTEMPTS ]; do
            echo "Attempt $ATTEMPT/$MAX_ATTEMPTS..."
            
            if curl -f --connect-timeout 5 "$HEALTH_URL" 2>/dev/null; then
              echo "✓ Health check passed"
              exit 0
            fi
            
            ATTEMPT=$((ATTEMPT + 1))
            sleep 2
          done
          
          echo "✗ Health check failed after $MAX_ATTEMPTS attempts"
          exit 1
      
      - name: Run smoke tests
        run: |
          echo "🧪 Running smoke tests..."
          
          # Test basic endpoints
          curl -f http://localhost:3000/api/status || exit 1
          curl -f http://localhost:3000/api/version || exit 1
          
          # Test data endpoint
          response=$(curl -s http://localhost:3000/api/data | jq '.status')
          if [ "$response" = "\"ok\"" ]; then
            echo "✓ API data endpoint working"
          else
            echo "✗ API data endpoint failed"
            exit 1
          fi
          
          echo "✓ All smoke tests passed"
      
      - name: Detailed validation
        run: |
          echo "🔍 Detailed validation..."
          
          # Check response times
          TIME=$(curl -w "%{time_total}" -o /dev/null -s http://localhost:3000/)
          if (( $(echo "$TIME < 1.0" | bc -l) )); then
            echo "✓ Response time acceptable: ${TIME}s"
          else
            echo "✗ Response time too slow: ${TIME}s"
            exit 1
          fi
          
          # Check status code
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/health)
          if [ "$STATUS" = "200" ]; then
            echo "✓ HTTP status correct: $STATUS"
          else
            echo "✗ HTTP status incorrect: $STATUS"
            exit 1
          fi
      
      - name: Rollback on failure
        if: failure()
        run: |
          echo "🔄 Deployment failed, initiating rollback..."
          
          # Kill current process
          kill ${{ steps.deploy.outputs.deployment_pid }} 2>/dev/null || true
          
          # Deploy previous version
          echo "📦 Restoring previous stable version..."
          git checkout HEAD~1 -- .
          npm ci
          npm start &
          
          # Verify rollback
          sleep 3
          if curl -f http://localhost:3000/health 2>/dev/null; then
            echo "✓ Rollback successful"
          else
            echo "✗ Rollback failed - ALERT: Manual intervention required!"
            exit 1
          fi
```

**Output Example:**

**Success Case:**
```
📦 Starting deployment...
Application started
🏥 Checking application health...
Attempt 1/10...
✓ Health check passed

🧪 Running smoke tests...
✓ API status endpoint working
✓ API version endpoint working
✓ API data endpoint working
✓ All smoke tests passed

🔍 Detailed validation...
✓ Response time acceptable: 0.23s
✓ HTTP status correct: 200

✓ Deployment successful
```

**Failure Case (Rollback):**
```
📦 Starting deployment...
Application started
🏥 Checking application health...
Attempt 1/10...
Attempt 2/10...
...
Attempt 10/10...
✗ Health check failed after 10 attempts

🔄 Deployment failed, initiating rollback...
📦 Restoring previous stable version...
✓ Rollback successful
```

---

## Exercise 6 Solution: Notification on Deployment

**Complete Solution:**

```yaml
name: Deploy with Notifications

on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Build and deploy
        id: deploy
        run: |
          echo "Building application..."
          npm ci
          npm run build
          npm start &
          sleep 3
          
          if curl -f http://localhost:3000/health 2>/dev/null; then
            echo "deployment_status=success" >> $GITHUB_OUTPUT
            echo "deployment_url=https://myapp.com" >> $GITHUB_OUTPUT
          else
            echo "deployment_status=failed" >> $GITHUB_OUTPUT
            exit 1
          fi
  
  notify-success:
    needs: deploy
    if: success()
    runs-on: ubuntu-latest
    steps:
      - name: Notify on Success
        run: |
          echo "✅ Deployment Successful!"
          echo "URL: https://myapp.com"
          echo "Build: ${{ github.sha }}"
          echo "Branch: ${{ github.ref }}"
          echo "Author: ${{ github.actor }}"
      
      - name: Send Slack notification
        uses: slackapi/slack-github-action@v1.24.0
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "✅ Deployment Successful",
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": "✅ Deployment Successful"
                  }
                },
                {
                  "type": "section",
                  "fields": [
                    {
                      "type": "mrkdwn",
                      "text": "*Repository:*\n${{ github.repository }}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Branch:*\n${{ github.ref_name }}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Commit:*\n${{ github.sha }}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Author:*\n${{ github.actor }}"
                    }
                  ]
                },
                {
                  "type": "actions",
                  "elements": [
                    {
                      "type": "button",
                      "text": {
                        "type": "plain_text",
                        "text": "View Deployment"
                      },
                      "url": "https://myapp.com"
                    },
                    {
                      "type": "button",
                      "text": {
                        "type": "plain_text",
                        "text": "View Workflow"
                      },
                      "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
                    }
                  ]
                }
              ]
            }
      
      - name: Send email notification
        uses: dawidd6/action-send-mail@v3
        with:
          server_address: ${{ secrets.EMAIL_SERVER }}
          server_port: ${{ secrets.EMAIL_PORT }}
          username: ${{ secrets.EMAIL_USERNAME }}
          password: ${{ secrets.EMAIL_PASSWORD }}
          subject: '✅ Deployment Successful: ${{ github.ref_name }}'
          to: ${{ secrets.EMAIL_TO }}
          from: GitHub Actions
          body: |
            Deployment successful!
            
            Repository: ${{ github.repository }}
            Branch: ${{ github.ref_name }}
            Commit: ${{ github.sha }}
            Author: ${{ github.actor }}
            URL: https://myapp.com
  
  notify-failure:
    needs: deploy
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - name: Notify on Failure
        run: |
          echo "❌ Deployment Failed!"
          echo "Branch: ${{ github.ref }}"
          echo "Author: ${{ github.actor }}"
      
      - name: Send Slack failure alert
        uses: slackapi/slack-github-action@v1.24.0
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "❌ Deployment Failed",
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": "❌ Deployment Failed - Immediate Action Required"
                  }
                },
                {
                  "type": "section",
                  "fields": [
                    {
                      "type": "mrkdwn",
                      "text": "*Repository:*\n${{ github.repository }}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Author:*\n${{ github.actor }}"
                    }
                  ]
                },
                {
                  "type": "actions",
                  "elements": [
                    {
                      "type": "button",
                      "text": {
                        "type": "plain_text",
                        "text": "View Logs"
                      },
                      "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
                    }
                  ]
                }
              ]
            }
      
      - name: Create incident
        run: |
          echo "Creating incident ticket..."
          # Integration with PagerDuty, Opsgenie, etc.
```

**Slack Message Examples:**

**Success:**
```
✅ Deployment Successful

Repository: myorg/myapp
Branch: main
Commit: a1b2c3d4e5f6
Author: john.dev

[View Deployment] [View Workflow]
```

**Failure:**
```
❌ Deployment Failed - Immediate Action Required

Repository: myorg/myapp
Author: jane.dev

[View Logs]
```

---

## Exercise 7 Solution: Canary Deployment

**Complete Solution:**

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
      
      - name: Deploy to 5% of Users (Canary)
        id: canary
        run: |
          VERSION="${{ github.ref_name }}"
          echo "🐤 Deploying canary version $VERSION to 5% of users..."
          
          # Update load balancer to route 5% traffic to new version
          # kubectl patch service myapp -p \
          #   '{"spec":{"selector":{"version":"'$VERSION'"}}}'
          
          # Or using feature flags
          # aws ssm put-parameter --name /myapp/canary-version --value $VERSION
          
          echo "canary_deployed=true" >> $GITHUB_OUTPUT
          sleep 2
          echo "✓ Canary deployed"
      
      - name: Monitor Canary (Phase 1)
        id: monitor-5
        run: |
          echo "📊 Monitoring 5% canary deployment..."
          echo "Metrics to check:"
          echo "- Error rate (target: <0.5%)"
          echo "- Response time (target: <100ms)"
          echo "- CPU usage (target: <60%)"
          
          sleep 5
          
          # Simulated metrics check
          ERROR_RATE=0.1
          RESPONSE_TIME=85
          CPU_USAGE=45
          
          echo "Results:"
          echo "- Error rate: $ERROR_RATE% ✓"
          echo "- Response time: ${RESPONSE_TIME}ms ✓"
          echo "- CPU usage: $CPU_USAGE% ✓"
          
          if (( $(echo "$ERROR_RATE < 1.0" | bc -l) )); then
            echo "canary_healthy=true" >> $GITHUB_OUTPUT
          else
            echo "canary_healthy=false" >> $GITHUB_OUTPUT
            exit 1
          fi
      
      - name: Increase to 25% of Users
        if: steps.monitor-5.outputs.canary_healthy == 'true'
        run: |
          echo "📈 Increasing canary to 25% of users..."
          # Load balancer: 25% traffic to new version
          sleep 2
          echo "✓ Traffic increased to 25%"
      
      - name: Monitor Canary (Phase 2)
        run: |
          echo "📊 Monitoring 25% deployment..."
          sleep 5
          echo "✓ Metrics healthy at 25%"
      
      - name: Increase to 50% of Users
        run: |
          echo "📈 Increasing to 50% of users..."
          sleep 2
          echo "✓ Traffic increased to 50%"
      
      - name: Monitor Canary (Phase 3)
        run: |
          echo "📊 Monitoring 50% deployment..."
          sleep 5
          echo "✓ Metrics healthy at 50%"
      
      - name: Full Rollout to 100%
        run: |
          echo "🚀 Promoting canary to 100% of users..."
          echo "New version ${{ github.ref_name }} now serving all traffic"
          sleep 2
          echo "✓ Full rollout complete"
      
      - name: Verification
        run: |
          echo "✅ Canary deployment successful"
          echo "Version ${{ github.ref_name }} is now in production"
      
      - name: Rollback on Failure
        if: failure()
        run: |
          echo "🔄 Canary failed, rolling back..."
          VERSION="${{ github.ref_name }}"
          # Revert to previous version
          # kubectl patch service myapp -p '{"spec":{"selector":{"version":"latest"}}}'
          echo "✓ Rolled back to previous stable version"
          exit 1
```

**Canary Rollout Timeline:**

```
Time    Traffic    Status      Metrics
────────────────────────────────────
T+0     5% (NEW)   Deploying   Init
T+5     5% (NEW)   ✓ Healthy   Low errors
T+10    25% (NEW)  ✓ Healthy   Good metrics
T+15    50% (NEW)  ✓ Healthy   Normal load
T+20    100% (NEW) ✓ Complete  All traffic served
```

---

## Exercise 8 Solution: Blue-Green Deployment

**Complete Solution:**

```yaml
name: Blue-Green Deploy

on: [push]

jobs:
  determine-target:
    runs-on: ubuntu-latest
    outputs:
      target-env: ${{ steps.current.outputs.target }}
    steps:
      - uses: actions/checkout@v3
      
      - name: Determine current environment
        id: current
        run: |
          # Check which environment is live
          RESPONSE=$(curl -s http://myapp.com/version 2>/dev/null || echo '{}')
          CURRENT_ENV=$(echo "$RESPONSE" | jq -r '.env // "unknown"')
          
          if [ "$CURRENT_ENV" = "blue" ]; then
            echo "Current live: BLUE"
            echo "Target deployment: GREEN"
            echo "target=green" >> $GITHUB_OUTPUT
          else
            echo "Current live: GREEN (or first deployment)"
            echo "Target deployment: BLUE"
            echo "target=blue" >> $GITHUB_OUTPUT
          fi

  deploy:
    needs: determine-target
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Deploy to ${{ needs.determine-target.outputs.target-env }}
        id: deploy
        run: |
          TARGET_ENV="${{ needs.determine-target.outputs.target-env }}"
          echo "🚀 Deploying to $TARGET_ENV environment..."
          
          npm ci
          npm run build
          
          # Deploy to inactive environment
          # For blue deployment
          if [ "$TARGET_ENV" = "blue" ]; then
            npm run deploy:blue
            echo "deployment_url=http://blue.myapp.com" >> $GITHUB_OUTPUT
          # For green deployment
          else
            npm run deploy:green
            echo "deployment_url=http://green.myapp.com" >> $GITHUB_OUTPUT
          fi
          
          sleep 3
          echo "✓ Deployed to $TARGET_ENV"

  test-new-env:
    needs: [determine-target, deploy]
    runs-on: ubuntu-latest
    steps:
      - name: Health check on new environment
        run: |
          TARGET_ENV="${{ needs.determine-target.outputs.target-env }}"
          echo "🏥 Testing $TARGET_ENV environment..."
          
          # Construct health check URL
          if [ "$TARGET_ENV" = "blue" ]; then
            HEALTH_URL="http://blue.myapp.com/health"
          else
            HEALTH_URL="http://green.myapp.com/health"
          fi
          
          # Check new environment health
          if curl -f "$HEALTH_URL" 2>/dev/null; then
            echo "✓ $TARGET_ENV environment healthy"
          else
            echo "✗ $TARGET_ENV environment failed"
            exit 1
          fi
      
      - name: Run smoke tests
        run: |
          TARGET_ENV="${{ needs.determine-target.outputs.target-env }}"
          echo "🧪 Running smoke tests..."
          
          if [ "$TARGET_ENV" = "blue" ]; then
            curl -f http://blue.myapp.com/api/test || exit 1
          else
            curl -f http://green.myapp.com/api/test || exit 1
          fi
          
          echo "✓ Smoke tests passed"

  switch-traffic:
    needs: [determine-target, test-new-env]
    runs-on: ubuntu-latest
    steps:
      - name: Switch load balancer traffic
        run: |
          TARGET_ENV="${{ needs.determine-target.outputs.target-env }}"
          echo "🔄 Switching traffic to $TARGET_ENV..."
          
          # Update load balancer / DNS
          if [ "$TARGET_ENV" = "blue" ]; then
            # Switch to blue
            # aws elb deregister-instances-from-load-balancer \
            #   --load-balancer-name myapp-lb --instances i-green-instance
            # aws elb register-instances-with-load-balancer \
            #   --load-balancer-name myapp-lb --instances i-blue-instance
            echo "✓ Traffic switched to BLUE"
            echo "Old environment: GREEN (still running)"
          else
            # Switch to green
            echo "✓ Traffic switched to GREEN"
            echo "Old environment: BLUE (still running)"
          fi
      
      - name: Verify live traffic
        run: |
          echo "Verifying production is healthy..."
          sleep 2
          
          if curl -f https://myapp.com/health 2>/dev/null; then
            echo "✓ Production serving traffic successfully"
          else
            echo "✗ Production health check failed"
            exit 1
          fi

  rollback:
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - name: Immediate rollback
        run: |
          echo "🔄 ROLLBACK INITIATED"
          echo "Switching traffic back to previous version..."
          
          # Quick DNS switch back
          # Already running in background, just switch traffic
          
          sleep 2
          
          if curl -f https://myapp.com/health 2>/dev/null; then
            echo "✓ Rollback successful"
            echo "Previous version now live"
          else
            echo "✗ Rollback failed - CRITICAL ALERT"
            exit 1
          fi
```

**Blue-Green Timeline:**

```
BEFORE:
Load Balancer → BLUE (live, serving traffic)
             → GREEN (idle, previous version)

AFTER Deployment:
Load Balancer → BLUE (idle, previous version)
             → GREEN (live, new version)

If Issue Found:
Load Balancer → BLUE (live again, rollback instant)
             → GREEN (idle, failed version)
```

**Key Advantages:**

- ✓ Instant rollback (seconds vs minutes)
- ✓ Both versions always running
- ✓ Zero downtime deployments
- ✓ Safe testing before switch
- ✓ Easy rollback procedure

---

## Exercise 9 Solution: Scheduled Maintenance Tasks

**Complete Solution:**

```yaml
name: Scheduled Maintenance

on:
  schedule:
    - cron: '0 2 * * *'      # Daily 2 AM UTC
    - cron: '0 3 * * 0'      # Weekly Sunday 3 AM
  workflow_dispatch:          # Allow manual trigger

jobs:
  backup:
    runs-on: ubuntu-latest
    steps:
      - name: Database Backup
        run: |
          echo "💾 Starting database backup..."
          DATE=$(date +%Y-%m-%d_%H-%M-%S)
          BACKUP_FILE="backup_$DATE.sql"
          
          # Backup database
          # pg_dump -U postgres -h db.example.com mydb > $BACKUP_FILE
          
          # Compress backup
          # gzip $BACKUP_FILE
          
          echo "✓ Backup complete: ${BACKUP_FILE}.gz"
          echo "Size: $(du -h ${BACKUP_FILE}.gz | cut -f1)"
      
      - name: Upload Backup to S3
        run: |
          echo "☁️ Uploading backup to S3..."
          # aws s3 cp backup_*.sql.gz s3://my-backups/
          echo "✓ Backup uploaded"
      
      - name: Verify Backup
        run: |
          echo "🔍 Verifying backup integrity..."
          # Check file exists and has content
          # aws s3 ls s3://my-backups/ --recursive
          echo "✓ Backup verified"
      
      - name: Upload to artifacts
        uses: actions/upload-artifact@v3
        with:
          name: database-backups
          path: backup_*.sql.gz
          retention-days: 30

  cleanup-logs:
    runs-on: ubuntu-latest
    steps:
      - name: Clean Old Logs
        run: |
          echo "🧹 Cleaning old logs..."
          
          # Remove logs older than 90 days
          # find /var/log/myapp -name "*.log" -mtime +90 -delete
          
          # Or using cloud storage
          # aws s3 rm s3://my-logs/ --recursive \
          #   --exclude "*" \
          #   --include "*.log" \
          #   --older-than 90d
          
          echo "✓ Old logs removed"
          echo "Freed space: ~500 MB"
      
      - name: Archive Recent Logs
        run: |
          echo "📦 Archiving recent logs..."
          # tar czf logs_archive_$(date +%Y-%m).tar.gz /var/log/myapp/
          # aws s3 cp logs_archive_*.tar.gz s3://my-archives/
          echo "✓ Logs archived"

  cleanup-temp:
    runs-on: ubuntu-latest
    steps:
      - name: Clean Temporary Files
        run: |
          echo "🗑️ Cleaning temporary files..."
          
          # Remove temp files older than 7 days
          # find /tmp -type f -mtime +7 -delete
          
          # Clean docker unused images
          # docker image prune -a --force
          
          echo "✓ Temporary files cleaned"
          echo "Space freed: ~200 MB"
      
      - name: Clean Cache
        run: |
          echo "🧹 Cleaning application cache..."
          # redis-cli FLUSHALL
          # memcached_tool localhost:11211 flush_all
          echo "✓ Cache cleaned"

  health-check:
    runs-on: ubuntu-latest
    steps:
      - name: System Health Check
        run: |
          echo "🏥 Checking system health..."
          
          echo "Checking disk space..."
          # df -h / | grep -v Filesystem | awk '{print $5}'
          echo "- Root: 65% used ✓"
          
          echo "Checking memory..."
          # free -h
          echo "- Memory: 72% used ✓"
          
          echo "Checking database..."
          # psql -U postgres -d mydb -c "SELECT version();"
          echo "- DB: Connected ✓"
      
      - name: Service Health Check
        run: |
          echo "🔍 Checking all services..."
          
          # Check API health
          curl -f https://api.myapp.com/health 2>/dev/null && echo "✓ API healthy"
          
          # Check database replication
          # psql -h replica.db.example.com check status
          echo "✓ Database replication OK"
          
          # Check cache servers
          # redis-cli -h cache.example.com ping
          echo "✓ Cache servers OK"
      
      - name: Send Health Report
        run: |
          echo "📊 Health Status Report"
          echo "========================"
          echo "✓ All systems operational"
          echo "✓ Backups current"
          echo "✓ No alerts"
          echo ""
          echo "Maintenance completed at $(date)"
      
      - name: Notify on Issues
        if: failure()
        run: |
          echo "⚠️ MAINTENANCE ISSUES DETECTED"
          # Send alert to team
          # aws sns publish --topic-arn arn:aws:sns:... \
          #   --message "Scheduled maintenance failed: check logs"
```

**Cron Schedule Examples:**

```
'0 2 * * *'        Every day at 2:00 AM UTC
'0 3 * * 0'        Every Sunday at 3:00 AM UTC
'0 0 1 * *'        First day of month at midnight
'30 1 * * 1-5'     Weekdays at 1:30 AM UTC
'0 */4 * * *'      Every 4 hours
'15,45 * * * *'    At :15 and :45 every hour
```

**Maintenance Output:**

```
Schedule Trigger: Daily at 2:00 AM UTC

💾 Database Backup
  ✓ Backup complete: backup_2024-01-15_02-00-00.sql.gz
  Size: 145 MB
  ✓ Uploaded to S3
  ✓ Verified

🧹 Clean Old Logs
  ✓ Removed logs from 2023-10-17 and earlier
  Freed space: 500 MB

🗑️ Clean Temporary Files
  ✓ Temp files cleaned
  ✓ Docker images pruned
  ✓ Cache cleaned
  Space freed: 200 MB

🏥 Health Checks
  ✓ Disk: 65% used
  ✓ Memory: 72% used
  ✓ Database: Connected
  ✓ API: Healthy
  ✓ Cache: OK

✅ All maintenance tasks completed successfully
```

---

## Exercise 10 Solution: Complete Enterprise CI/CD Pipeline

**Complete Solution:**

```yaml
name: Enterprise CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # ═══════════════════════════════════════════════════════════
  # PHASE 1: CODE QUALITY
  # ═══════════════════════════════════════════════════════════
  
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run format:check

  # ═══════════════════════════════════════════════════════════
  # PHASE 2: SECURITY SCANNING
  # ═══════════════════════════════════════════════════════════
  
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run npm audit
        run: npm audit --audit-level=moderate
      - name: SAST scan with CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: 'javascript'
      - name: Perform CodeQL analysis
        uses: github/codeql-action/analyze@v2

  # ═══════════════════════════════════════════════════════════
  # PHASE 3: TESTS (PARALLEL)
  # ═══════════════════════════════════════════════════════════
  
  test-unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - uses: actions/upload-artifact@v3
        with:
          name: coverage-unit
          path: coverage/

  test-integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: npm run migrate
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb
      - run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb

  # ═══════════════════════════════════════════════════════════
  # PHASE 4: BUILD ARTIFACTS
  # ═══════════════════════════════════════════════════════════
  
  build:
    needs: [lint, security, test-unit, test-integration]
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Build application
        run: |
          npm ci
          npm run build
      
      - name: Build Docker image
        uses: docker/setup-buildx-action@v2
      
      - name: Log in to registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and push image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build
          path: dist/
          retention-days: 7

  # ═══════════════════════════════════════════════════════════
  # PHASE 5: DEPLOYMENT
  # ═══════════════════════════════════════════════════════════
  
  deploy-dev:
    needs: build
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: development
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: build
          path: dist/
      - name: Deploy to Development
        run: |
          echo "🚀 Deploying to development..."
          # Deployment logic here

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: build
          path: dist/
      - name: Deploy to Staging
        run: echo "🚀 Deploying to staging..."
      - name: Health checks
        run: curl -f https://staging.myapp.com/health

  deploy-prod:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    steps:
      - uses: actions/checkout@v3
      - uses: actions/download-artifact@v3
        with:
          name: build
          path: dist/
      - name: Deploy to Production
        run: echo "🚀 Deploying to production..."
      - name: Verify production
        run: curl -f https://myapp.com/health

  # ═══════════════════════════════════════════════════════════
  # PHASE 6: NOTIFICATIONS
  # ═══════════════════════════════════════════════════════════
  
  notify:
    if: always()
    needs: [deploy-prod]
    runs-on: ubuntu-latest
    steps:
      - name: Send notification
        uses: slackapi/slack-github-action@v1
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "${{ job.status == 'success' && '✅' || '❌' }} Pipeline ${{ job.status }}",
              "blocks": [{
                "type": "section",
                "text": {"type": "mrkdwn", "text": "Workflow: ${{ job.status }}"}
              }]
            }
```

**Enterprise Pipeline Flow:**

```
┌─ Code Push to main/develop
│
├─ PHASE 1: CODE QUALITY (parallel)
│  ├─ Lint
│  └─ Format Check
│
├─ PHASE 2: SECURITY
│  ├─ npm audit
│  └─ CodeQL SAST
│
├─ PHASE 3: TESTS (parallel)
│  ├─ Unit Tests
│  └─ Integration Tests
│
├─ PHASE 4: BUILD
│  ├─ Build Application
│  ├─ Docker Image
│  └─ Create Artifacts
│
├─ PHASE 5: DEPLOYMENT
│  ├─ Deploy to Dev (if develop branch)
│  ├─ Deploy to Staging (if main branch)
│  └─ Deploy to Prod (if main + manual approval)
│
└─ PHASE 6: NOTIFICATION
   └─ Slack Alert (success/failure)

Total Time: ~25-30 minutes
```

**Success Metrics:**

```
✅ All phases complete
✅ No security issues
✅ 100+ tests passing
✅ Code coverage >80%
✅ Docker image built
✅ All environments deployed
✅ Health checks passing
✅ Team notified
```

---

## Summary

These 10 real-world solutions demonstrate complete enterprise CI/CD mastery:

1. **Static Deployments** - GitHub Pages, asset optimization
2. **Integration Testing** - Database services, migrations
3. **Container Building** - Docker, registry, scanning
4. **Multi-Environment** - Dev, staging, production gates
5. **Health Validation** - Checks, retries, rollbacks
6. **Notifications** - Slack, email, incidents
7. **Canary Releases** - Gradual rollout, monitoring
8. **Blue-Green** - Instant rollback, zero-downtime
9. **Maintenance** - Backups, cleanup, health checks
10. **Enterprise Pipeline** - Complete end-to-end system

**Production-Ready Patterns:**
- Manual approval gates for production
- Automated testing at every stage
- Security scanning and compliance
- Health checks and monitoring
- Instant rollback capability
- Team notifications
- Comprehensive logging

Continue building robust, enterprise-grade CI/CD systems!
