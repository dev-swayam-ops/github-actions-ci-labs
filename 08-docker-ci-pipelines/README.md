# Module 08: Docker and ECR CI/CD Pipelines

## Overview

Docker containerization and Amazon Elastic Container Registry (ECR) integration are essential for modern CI/CD pipelines. This module teaches you how to automate Docker image building, scanning, and deployment to container registries using GitHub Actions.

### What You'll Learn

1. **Docker Image Building** - Automated image creation and optimization
2. **Multi-stage Builds** - Reducing image size with build optimization
3. **Image Tagging Strategies** - Semantic versioning and registry management
4. **ECR Integration** - AWS authentication and image pushing
5. **Container Security** - Image scanning and vulnerability detection
6. **Layer Caching** - Optimizing build performance
7. **Multi-architecture Builds** - Building for ARM64 and x86_64
8. **Image Cleanup** - Managing registry storage and versions

### Prerequisites

- Basic Docker knowledge
- AWS account with ECR repository
- GitHub Actions fundamentals (Modules 01-02)
- AWS credentials configured in GitHub Secrets

---

## Key Concepts

### Docker Images

**Definition:** A lightweight, standalone, executable package containing application code, dependencies, runtime, and system tools.

**Structure:**
- Base image (FROM)
- Dependencies (RUN)
- Application code (COPY)
- Configuration (ENV)
- Entry point (CMD/ENTRYPOINT)

**Advantages:**
- Consistency across environments
- Portability across platforms
- Isolation from host system
- Version control via tags

### Dockerfile Best Practices

```dockerfile
# ✓ GOOD: Multi-stage with small final image
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY src ./src
EXPOSE 3000
CMD ["node", "src/index.js"]

# ✗ BAD: Large base image, no optimization
FROM ubuntu:22.04
COPY . /app
RUN apt-get install -y node npm
RUN cd /app && npm install
```

### ECR (Elastic Container Registry)

**Features:**
- AWS-managed container registry
- Built-in image vulnerability scanning
- Lifecycle policies for cleanup
- Cross-region replication
- Integration with ECS/EKS

**Workflow:**
1. Authenticate Docker with ECR
2. Build image with appropriate tags
3. Push to ECR repository
4. Trigger deployment (ECS/EKS)

### Image Tagging Strategy

```bash
# Common patterns:
myapp:latest              # Latest version
myapp:1.0.0              # Semantic versioning
myapp:1.0.0-alpine       # Variant
myapp:main-abc123        # Branch + commit
myapp:build-2024-01-15   # Build date
```

### Layer Caching

Docker builds cache layers by hash:
```dockerfile
# These layers cached independently
FROM node:18                          # Layer 1: Base image
RUN apt-get update && apt-get upgrade # Layer 2: System updates
COPY package*.json ./                 # Layer 3: Package files (cache busts on change)
RUN npm ci                            # Layer 4: Dependencies (reinstalled if Layer 3 changes)
COPY src ./                           # Layer 5: Source code (cache busts on change)
```

**Optimization:** Order commands from least-changing to most-changing.

---

## Hands-on Lab: Complete Docker-ECR Pipeline

### Part 1: Setup

**Step 1: Create AWS ECR Repository**

```bash
# AWS CLI
aws ecr create-repository \
  --repository-name github-actions-app \
  --region us-east-1

# Output: Copy the repository URI
# Example: 123456789.dkr.ecr.us-east-1.amazonaws.com/github-actions-app
```

**Step 2: Create IAM User for GitHub Actions**

```bash
# Create user
aws iam create-user --user-name github-actions-ecr

# Create policy
cat > policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "arn:aws:ecr:us-east-1:123456789:repository/github-actions-app"
    }
  ]
}
EOF

aws iam put-user-policy \
  --user-name github-actions-ecr \
  --policy-name ECRAccess \
  --policy-document file://policy.json

# Create access key
aws iam create-access-key --user-name github-actions-ecr
```

**Step 3: Add Secrets to GitHub**

In GitHub Settings → Secrets and Variables → Actions:
```
AWS_ACCESS_KEY_ID=<from step above>
AWS_SECRET_ACCESS_KEY=<from step above>
AWS_REGION=us-east-1
ECR_REPOSITORY=github-actions-app
```

**Step 4: Create Simple Node.js Application**

```bash
# Repository structure
.
├── Dockerfile
├── .dockerignore
├── .github/workflows/docker-ecr.yml
├── package.json
└── src/
    └── index.js
```

`Dockerfile`:
```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY src ./src

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

# Start application
CMD ["node", "src/index.js"]
```

`.dockerignore`:
```
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.DS_Store
```

`package.json`:
```json
{
  "name": "github-actions-app",
  "version": "1.0.0",
  "description": "GitHub Actions Docker Demo",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "test": "echo 'Tests passed'",
    "lint": "echo 'No lint issues'"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

`src/index.js`:
```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.json({ message: 'Hello from GitHub Actions Docker!' });
});

app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### Part 2: Docker-ECR Workflow

`.github/workflows/docker-ecr.yml`:

```yaml
name: Docker Build and Push to ECR

on:
  push:
    branches: [main, develop]
    paths:
      - 'src/**'
      - 'Dockerfile'
      - '.dockerignore'
      - 'package*.json'
      - '.github/workflows/docker-ecr.yml'
  pull_request:
    branches: [main]
  workflow_dispatch:

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: github-actions-app

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    
    outputs:
      image-uri: ${{ steps.image.outputs.uri }}
      image-tag: ${{ steps.image.outputs.tag }}
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Generate Image Tag
        id: image
        run: |
          REGISTRY=${{ steps.login-ecr.outputs.registry }}
          REPOSITORY=${{ env.ECR_REPOSITORY }}
          
          # Create multiple tags
          IMAGE_TAG=${{ github.sha }}
          LATEST_TAG=latest
          BRANCH_TAG=${GITHUB_REF#refs/heads/}
          
          IMAGE_URI="$REGISTRY/$REPOSITORY:$IMAGE_TAG"
          LATEST_URI="$REGISTRY/$REPOSITORY:$LATEST_TAG"
          BRANCH_URI="$REGISTRY/$REPOSITORY:$BRANCH_TAG"
          
          echo "uri=$IMAGE_URI" >> $GITHUB_OUTPUT
          echo "tag=$IMAGE_TAG" >> $GITHUB_OUTPUT
          echo "registry=$REGISTRY" >> $GITHUB_OUTPUT
          echo "repository=$REPOSITORY" >> $GITHUB_OUTPUT
          
          # Display
          echo "Image URI: $IMAGE_URI"
          echo "Latest: $LATEST_URI"
          echo "Branch: $BRANCH_URI"
      
      - name: Build Docker Image
        run: |
          docker build \
            -t ${{ steps.image.outputs.uri }} \
            -t ${{ steps.image.outputs.registry }}/${{ steps.image.outputs.repository }}:latest \
            -t ${{ steps.image.outputs.registry }}/${{ steps.image.outputs.repository }}:${{ github.ref_name }} \
            --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
            --build-arg VCS_REF=${{ github.sha }} \
            --build-arg VERSION=${{ github.ref_name }} \
            .
      
      - name: Inspect Image
        run: |
          docker image inspect ${{ steps.image.outputs.uri }}
          docker image ls | grep ${{ env.ECR_REPOSITORY }}
      
      - name: Push Image to ECR
        if: github.event_name != 'pull_request'
        run: |
          docker push ${{ steps.image.outputs.uri }}
          docker push ${{ steps.image.outputs.registry }}/${{ steps.image.outputs.repository }}:latest
          docker push ${{ steps.image.outputs.registry }}/${{ steps.image.outputs.repository }}:${{ github.ref_name }}
          echo "Image pushed successfully"
      
      - name: Print Summary
        run: |
          echo "## Docker Build Summary" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "- **Image URI**: ${{ steps.image.outputs.uri }}" >> $GITHUB_STEP_SUMMARY
          echo "- **Registry**: ${{ steps.image.outputs.registry }}" >> $GITHUB_STEP_SUMMARY
          echo "- **Repository**: ${{ steps.image.outputs.repository }}" >> $GITHUB_STEP_SUMMARY
          echo "- **Tags**: ${{ github.sha }}, latest, ${{ github.ref_name }}" >> $GITHUB_STEP_SUMMARY
  
  scan-image:
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.event_name != 'pull_request'
    
    steps:
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Scan Image with ECR
        run: |
          aws ecr start-image-scan \
            --repository-name ${{ env.ECR_REPOSITORY }} \
            --image-id imageTag=${{ needs.build-and-push.outputs.image-tag }} \
            --region ${{ env.AWS_REGION }}
      
      - name: Wait for Scan Results
        run: |
          aws ecr wait image-scan-complete \
            --repository-name ${{ env.ECR_REPOSITORY }} \
            --image-id imageTag=${{ needs.build-and-push.outputs.image-tag }} \
            --region ${{ env.AWS_REGION }}
      
      - name: Get Scan Results
        run: |
          aws ecr describe-image-scan-findings \
            --repository-name ${{ env.ECR_REPOSITORY }} \
            --image-id imageTag=${{ needs.build-and-push.outputs.image-tag }} \
            --region ${{ env.AWS_REGION }}
```

### Part 3: Validate

**After pushing to main branch:**

```bash
# Check workflow status
git push origin main

# View in GitHub Actions tab
# Verify in ECR console:
aws ecr list-images --repository-name github-actions-app

# Pull and test locally
docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/github-actions-app:latest
docker run -p 3000:3000 123456789.dkr.ecr.us-east-1.amazonaws.com/github-actions-app:latest
curl http://localhost:3000
```

**Success Criteria:**
- ✅ Docker image built successfully
- ✅ Image pushed to ECR
- ✅ Multiple tags created (commit, latest, branch)
- ✅ Image scan completed
- ✅ No critical vulnerabilities
- ✅ Workflow summary generated

---

## Common Mistakes

### 1. Pushing on Every Commit
**Problem:** Unnecessary builds for non-code changes
```yaml
# ✗ Bad: Builds on every push
on: [push]

# ✓ Good: Only build on code changes
on:
  push:
    paths:
      - 'src/**'
      - 'Dockerfile'
      - 'package*.json'
```

### 2. Large Base Images
**Problem:** 2GB image from ubuntu:22.04
```dockerfile
# ✗ Bad: 500MB image
FROM ubuntu:22.04
RUN apt-get install nodejs npm

# ✓ Good: 30MB image
FROM node:18-alpine
```

### 3. Not Using Layer Cache
**Problem:** Full rebuild every time
```dockerfile
# ✗ Bad: Changes in COPY bust all layers
COPY . .
RUN npm ci

# ✓ Good: Only rebuild on package changes
COPY package*.json .
RUN npm ci
COPY src src
```

### 4. Forgetting Image Scan
**Problem:** Unknown vulnerabilities in production
```yaml
# ✓ Always scan before deploying
- name: Scan Image
  run: trivy image myapp:latest --severity HIGH
```

### 5. No Resource Limits
**Problem:** Runaway containers
```dockerfile
# ✓ Set reasonable limits
# In deployment config (ECS/K8s), not Dockerfile
# But plan for them
HEALTHCHECK --interval=30s --timeout=3s --retries=3
```

---

## Troubleshooting Guide

| Problem | Cause | Solution |
|---------|-------|----------|
| "Login Failed" | Invalid AWS credentials | Verify secrets in GitHub, check IAM user permissions |
| "Repository not found" | ECR repo doesn't exist | Run `aws ecr create-repository` |
| "Permission Denied" | IAM policy missing | Add ECR push permissions to policy |
| "Image too large" | Non-optimized layers | Use `.dockerignore`, multi-stage builds |
| "Build timeout" | Large dependencies | Use layer caching, reduce install scope |
| "Scan failed" | Image not properly pushed | Verify push step completed successfully |
| "Registry login expired" | ECR token expiration | AWS credential rotation (automatic in action) |
| "Out of space" | Old images accumulating | Set ECR lifecycle policies |

---

## Advanced Topics

### Multi-Architecture Builds

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v2

- name: Build Multi-arch Image
  run: |
    docker buildx build \
      --platform linux/amd64,linux/arm64 \
      -t ${{ steps.image.outputs.uri }} \
      --push \
      .
```

### Caching Strategy

```yaml
- uses: actions/cache@v3
  with:
    path: |
      ~/.npm
      ~/.cache/pip
    key: ${{ runner.os }}-dependencies-${{ hashFiles('**/package-lock.json') }}
```

### Private Base Images

```bash
# In Dockerfile
FROM private-registry.com/mybase:latest

# In workflow
docker login -u ${{ secrets.DOCKER_USER }} -p ${{ secrets.DOCKER_TOKEN }} private-registry.com
```

---

## Summary Checklist

- ✅ Docker image optimized with multi-stage build
- ✅ `.dockerignore` configured to exclude unnecessary files
- ✅ Proper layer caching implemented (dependencies before code)
- ✅ ECR repository created with appropriate permissions
- ✅ GitHub secrets configured (AWS credentials)
- ✅ Docker build workflow automated
- ✅ Multiple image tags (commit, latest, branch)
- ✅ Image scanning enabled
- ✅ Health checks configured
- ✅ Workflow runs on code changes only
- ✅ Image URI exported for downstream workflows

---

## Resources

- Docker Docs: https://docs.docker.com
- Docker Best Practices: https://docs.docker.com/develop/dev-best-practices
- Amazon ECR: https://docs.aws.amazon.com/ecr
- GitHub Docker Action: https://github.com/docker/build-push-action
- Multi-arch builds: https://docs.docker.com/build/building/multi-platform
