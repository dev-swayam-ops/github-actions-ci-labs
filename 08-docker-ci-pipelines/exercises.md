# Module 08 Exercises: Docker and ECR CI/CD Pipelines

## Exercise 1: Basic Docker Image Build

**Objective:** Build a Docker image and run it locally.

**Starter Code:**

```dockerfile
# Create Dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
# TODO: Install dependencies
# TODO: Copy source code
EXPOSE 3000
CMD ["node", "src/index.js"]
```

```json
// package.json
{
  "name": "docker-app",
  "version": "1.0.0",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

```javascript
// src/index.js
const express = require('express');
const app = express();
app.get('/', (req, res) => {
  res.json({ message: 'Hello Docker!' });
});
app.listen(3000, () => console.log('Running on port 3000'));
```

**Acceptance Criteria:**
- [ ] Dockerfile is valid and builds without errors
- [ ] Image is less than 200MB
- [ ] Container runs and listens on port 3000
- [ ] Health endpoint responds with JSON
- [ ] Image can be stopped without hanging

**Hint:** Use alpine base image for small size.

---

## Exercise 2: Docker Multi-stage Build

**Objective:** Optimize Dockerfile with multi-stage builds to reduce image size.

**Starter Code:**

```dockerfile
# Stage 1: Build stage
FROM node:18 AS builder
WORKDIR /build
COPY package*.json ./
# TODO: Install dependencies in builder stage
# TODO: Copy source code

# Stage 2: Runtime stage
# TODO: Use alpine base image
# TODO: Copy only necessary files from builder
# TODO: Configure health check
# TODO: Start application
```

**Requirements:**
- Production dependencies only in final image
- No build tools in final image
- Health check configured
- Image less than 100MB

**Acceptance Criteria:**
- [ ] Multi-stage build configured correctly
- [ ] First stage produces minimal artifacts
- [ ] Final image contains only runtime dependencies
- [ ] Final image smaller than Stage 1 image
- [ ] Health check responds correctly

**Hint:** Use `--only=production` flag for npm install.

---

## Exercise 3: Docker Image Tagging Strategy

**Objective:** Implement comprehensive image tagging in a GitHub Actions workflow.

**Starter Code:**

```yaml
name: Docker Build with Tags

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Generate Tags
        id: tags
        run: |
          # TODO: Create image URI with registry
          # TODO: Generate SHA tag
          # TODO: Generate latest tag
          # TODO: Generate branch tag
          # TODO: Output all tags
          echo "Tags generation needed"
      
      - name: Build Image
        run: |
          # TODO: Build with all tags
          docker build -t myapp:tag1 .
      
      - name: List Images
        run: docker image ls | grep myapp
```

**Requirements:**
- Tag format: `registry/repository:tag`
- Implement tags: commit SHA, latest, branch name
- Display all tags in workflow output
- Only push on non-PR events

**Acceptance Criteria:**
- [ ] Image tagged with commit SHA
- [ ] Latest tag created
- [ ] Branch tag created
- [ ] All tags point to same image
- [ ] `docker image ls` shows all tags

**Hint:** Use `steps.<step-id>.outputs` to pass values between steps.

---

## Exercise 4: AWS ECR Authentication and Push

**Objective:** Authenticate with AWS ECR and push Docker images.

**Starter Code:**

```yaml
name: Push to ECR

on:
  push:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: my-app

jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS
        # TODO: Add aws-actions/configure-aws-credentials
        run: echo "AWS configured"
      
      - name: Login to ECR
        # TODO: Use aws-actions/amazon-ecr-login
        run: echo "ECR login"
      
      - name: Build and Push
        # TODO: Build image with ECR registry URI
        # TODO: Push image to ECR
        run: echo "Push to ECR"
```

**Requirements:**
- AWS credentials from GitHub Secrets
- ECR login using official action
- Build image with full ECR URI
- Push multiple tags
- Success message with image URI

**Acceptance Criteria:**
- [ ] AWS credentials configured successfully
- [ ] ECR login completed without errors
- [ ] Image built with ECR registry URI
- [ ] Multiple tags pushed to ECR
- [ ] Workflow completes successfully

**Hint:** Secrets needed: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`

---

## Exercise 5: Docker .dockerignore Optimization

**Objective:** Create optimal .dockerignore file to reduce build context size.

**Starter Code:**

```
# Create .dockerignore file with appropriate excludes
# Current file (incomplete):

node_modules
# TODO: Add npm debug logs
# TODO: Add git files
# TODO: Add IDE configuration
# TODO: Add test files (optional)
# TODO: Add CI/CD configs
# TODO: Add markdown docs
```

**Requirements:**
- Exclude node_modules
- Exclude npm debug logs
- Exclude .git directory
- Exclude .gitignore and .github
- Exclude IDE config (.vscode, .idea)
- Exclude build artifacts
- Consider including package*.json (needed for build)

**Acceptance Criteria:**
- [ ] .dockerignore file exists
- [ ] Includes standard excludes
- [ ] Build context size reduced
- [ ] Docker build still has required files
- [ ] No unnecessary files in image

**Hint:** Check build context size with `docker build --no-cache .` (watch "Sending build context").

---

## Exercise 6: Multi-architecture Docker Builds

**Objective:** Build Docker images for multiple architectures (AMD64 and ARM64).

**Starter Code:**

```yaml
name: Multi-arch Build

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        # TODO: Use docker/setup-buildx-action
        run: echo "Buildx setup"
      
      - name: Build Multi-arch
        # TODO: Build for linux/amd64 and linux/arm64
        # TODO: Tag appropriately
        run: |
          docker buildx build \
            --platform TODO \
            -t myapp:latest \
            .
```

**Requirements:**
- Setup Buildx action
- Build for amd64 and arm64 platforms
- Use buildx build command with --platform flag
- Tag with appropriate names
- Output image information

**Acceptance Criteria:**
- [ ] Buildx installed and configured
- [ ] Build succeeds for both platforms
- [ ] Output shows both architectures
- [ ] Image manifests created
- [ ] Can build without errors

**Hint:** Multi-arch builds require `docker/setup-buildx-action@v2`

---

## Exercise 7: Docker Image Security Scanning

**Objective:** Scan Docker images for vulnerabilities before pushing.

**Starter Code:**

```yaml
name: Image Security Scan

on:
  push:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build Image
        run: docker build -t myapp:latest .
      
      - name: Save Image
        run: docker save myapp:latest -o image.tar
      
      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          input: 'image.tar'
          format: 'sarif'
          output: 'trivy-results.sarif'
          # TODO: Set severity level
          # TODO: Set exit code on findings
      
      - name: Upload Results
        # TODO: Upload SARIF to GitHub Security tab
        run: echo "Results uploaded"
```

**Requirements:**
- Build Docker image
- Save image to tar file
- Scan with Trivy action
- Configure severity (CRITICAL, HIGH)
- Upload SARIF results
- Fail on critical findings

**Acceptance Criteria:**
- [ ] Image built successfully
- [ ] Trivy scan runs without errors
- [ ] SARIF file generated
- [ ] Results uploaded to GitHub
- [ ] Workflow fails if critical found

**Hint:** Use `codeql-action/upload-sarif@v2` to upload SARIF results.

---

## Exercise 8: Docker Layer Caching Optimization

**Objective:** Implement layer caching to speed up builds.

**Starter Code:**

```dockerfile
# Optimize layer caching by reordering commands
FROM node:18-alpine

WORKDIR /app

# TODO: Copy package files first (cacheable)
# TODO: Install dependencies (cached until package.json changes)
# TODO: Copy application code (most frequently changes)
# TODO: Expose port
# TODO: Start application
```

**Requirements:**
- Order commands from least-changing to most-changing
- Separate package installation from code copy
- Only rebuild changed layers
- Document cache strategy

**Test:**
```bash
# Build once
time docker build -t myapp:v1 .

# Build again (should use cache)
time docker build -t myapp:v1 .

# Modify src/index.js, build again
# Only updated layers rebuilt
time docker build -t myapp:v1 .
```

**Acceptance Criteria:**
- [ ] Package files copied before installation
- [ ] Dependencies installed in separate layer
- [ ] Source code in final layer
- [ ] Cache hits for unchanged layers
- [ ] Build time significantly faster on second run

**Hint:** First build warm up, second build should be near-instant.

---

## Exercise 9: Docker Compose for Local Development

**Objective:** Create Docker Compose file for multi-service local development.

**Starter Code:**

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    # TODO: Add environment variables
    # TODO: Add depends_on for database
    # TODO: Add volume for development
  
  # TODO: Add database service (postgres)
  # TODO: Add redis service for caching
  
  # TODO: Add health checks
  # TODO: Add networks
```

**Requirements:**
- Define app service with build
- Add PostgreSQL service
- Add Redis service
- Configure volumes for development
- Set environment variables
- Configure health checks
- Create custom network

**Acceptance Criteria:**
- [ ] `docker-compose up` starts all services
- [ ] App accessible on localhost:3000
- [ ] Database initialized
- [ ] Redis running
- [ ] Health checks passing
- [ ] Services can communicate

**Hint:** Use `depends_on` for startup ordering, but verify health checks.

---

## Exercise 10: ECR Lifecycle Policy Management

**Objective:** Configure ECR lifecycle policies to manage image retention.

**Starter Code:**

```json
// ecr-lifecycle-policy.json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 images tagged",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 2,
      "description": "Remove untagged images after 7 days",
      # TODO: Configure rule for untagged images
    }
  ]
}
```

**Requirements:**
- Keep last 10 released versions (v* tags)
- Remove untagged images older than 7 days
- Keep latest tag indefinitely
- Archive old branches (optional)

**Apply Policy:**
```bash
aws ecr put-lifecycle-policy \
  --repository-name my-app \
  --lifecycle-policy-text file://ecr-lifecycle-policy.json
```

**Acceptance Criteria:**
- [ ] Lifecycle policy JSON is valid
- [ ] Multiple rules configured
- [ ] Policy applied to ECR repository
- [ ] Old untagged images deleted
- [ ] Latest images retained

**Hint:** Use AWS CLI to put lifecycle policy, or GitHub Secrets + workflow.

---

## Summary

You now understand:
- ✅ Building optimized Docker images
- ✅ Multi-stage Dockerfile patterns
- ✅ Image tagging strategies
- ✅ AWS ECR authentication and push
- ✅ Build context optimization
- ✅ Multi-architecture builds
- ✅ Container security scanning
- ✅ Layer caching strategies
- ✅ Docker Compose for development
- ✅ ECR lifecycle management

Continue mastering containerization and registry management!
