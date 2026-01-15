# Module 08 Solutions: Docker and ECR CI/CD Pipelines

## Exercise 1: Basic Docker Image Build

**Solution:**

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY src ./src
EXPOSE 3000
CMD ["node", "src/index.js"]
```

```bash
# Build the image
docker build -t docker-app:latest .

# Run the container
docker run -p 3000:3000 docker-app:latest

# Test the endpoint
curl http://localhost:3000

# Stop the container (Ctrl+C or)
# docker stop <container-id>
```

**Explanation:**
- Alpine base image is small (~45MB)
- `npm ci` installs exact versions from package-lock.json
- Source code copied after dependencies (faster rebuilds)
- EXPOSE documents port usage
- CMD sets default entrypoint

---

## Exercise 2: Docker Multi-stage Build

**Solution:**

```dockerfile
# Stage 1: Build stage
FROM node:18 AS builder
WORKDIR /build
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Runtime stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /build/node_modules ./node_modules
COPY src ./src
COPY package.json .

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

CMD ["node", "src/index.js"]
```

**Explanation:**
- Builder stage uses full node image (needed for build tools)
- Runtime stage uses alpine (much smaller)
- Only node_modules copied from builder (no build artifacts)
- Final image ~80MB vs 900MB+ with single stage
- Health check verifies application responsiveness

**Test:**
```bash
docker build -t docker-app:optimized .
docker image ls | grep docker-app
# Compare sizes - optimized should be much smaller
```

---

## Exercise 3: Docker Image Tagging Strategy

**Solution:**

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
          REGISTRY="myregistry.com"
          REPOSITORY="my-app"
          SHA_SHORT=$(echo ${GITHUB_SHA} | cut -c1-7)
          BRANCH=$(echo ${GITHUB_REF#refs/heads/} | sed 's/\//-/g')
          
          TAG_SHA="${REGISTRY}/${REPOSITORY}:${SHA_SHORT}"
          TAG_LATEST="${REGISTRY}/${REPOSITORY}:latest"
          TAG_BRANCH="${REGISTRY}/${REPOSITORY}:${BRANCH}"
          
          echo "tag_sha=${TAG_SHA}" >> $GITHUB_OUTPUT
          echo "tag_latest=${TAG_LATEST}" >> $GITHUB_OUTPUT
          echo "tag_branch=${TAG_BRANCH}" >> $GITHUB_OUTPUT
          
          echo "Tags:"
          echo "  SHA:    ${TAG_SHA}"
          echo "  Latest: ${TAG_LATEST}"
          echo "  Branch: ${TAG_BRANCH}"
      
      - name: Build Image
        run: |
          docker build \
            -t ${{ steps.tags.outputs.tag_sha }} \
            -t ${{ steps.tags.outputs.tag_latest }} \
            -t ${{ steps.tags.outputs.tag_branch }} \
            .
      
      - name: List Images
        run: docker image ls | grep my-app
```

**Explanation:**
- `GITHUB_SHA` provides full commit hash
- `cut -c1-7` creates short commit SHA
- Branch tag sanitized (slashes to dashes)
- Multiple tags point to same image ID
- All tags listed at end for verification

---

## Exercise 4: AWS ECR Authentication and Push

**Solution:**

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
    permissions:
      id-token: write
      contents: read
    
    outputs:
      image-uri: ${{ steps.image.outputs.uri }}
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Generate Image URI
        id: image
        run: |
          REGISTRY=${{ steps.login-ecr.outputs.registry }}
          IMAGE_URI="${REGISTRY}/${ECR_REPOSITORY}:${GITHUB_SHA}"
          echo "uri=${IMAGE_URI}" >> $GITHUB_OUTPUT
          echo "Image URI: ${IMAGE_URI}"
      
      - name: Build and Push
        run: |
          docker build -t ${{ steps.image.outputs.uri }} .
          docker push ${{ steps.image.outputs.uri }}
          docker tag ${{ steps.image.outputs.uri }} \
            ${REGISTRY}/${ECR_REPOSITORY}:latest
          docker push ${REGISTRY}/${ECR_REPOSITORY}:latest
        env:
          REGISTRY: ${{ steps.login-ecr.outputs.registry }}
      
      - name: Success
        run: |
          echo "Image pushed: ${{ steps.image.outputs.uri }}"
```

**Explanation:**
- `aws-actions/amazon-ecr-login` handles ECR authentication
- Automatically generates and refreshes credentials
- Image pushed with commit SHA tag
- Latest tag also updated
- Image URI exported for downstream jobs

---

## Exercise 5: Docker .dockerignore Optimization

**Solution:**

```
# Dependencies
node_modules
npm-debug.log
yarn-error.log

# Git
.git
.gitignore
.gitattributes

# IDE
.vscode
.idea
*.swp
*.swo
*~
.DS_Store

# CI/CD
.github
.gitlab-ci.yml
.travis.yml
Jenkinsfile

# Docs and tests
README.md
CHANGELOG.md
*.test.js
*.spec.js
tests/
__tests__/
coverage/

# Build artifacts
dist/
build/
.next/
.nuxt/
out/

# Environment
.env
.env.local
.env.*.local

# Others
.dockerignore
docker-compose*.yml
Dockerfile*
```

**Benefits:**
- Reduces build context from 200MB to 20MB (typical)
- Faster docker build command
- Smaller data transfer to Docker daemon
- Fewer unnecessary layer invalidations

**Verify:**
```bash
# Before: Check build context size
docker build --no-cache . 2>&1 | grep "Sending build context"

# Should show reduced size after optimization
```

---

## Exercise 6: Multi-architecture Docker Builds

**Solution:**

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
        uses: docker/setup-buildx-action@v2
      
      - name: Build Multi-arch Image
        uses: docker/build-push-action@v4
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          tags: |
            myregistry.com/my-app:latest
            myregistry.com/my-app:${{ github.sha }}
          push: false
          outputs: type=oci,dest=./image
      
      - name: Show Image Info
        run: |
          echo "Multi-arch image built for:"
          echo "  - linux/amd64"
          echo "  - linux/arm64"
          docker buildx ls
```

**Or with native Docker buildx:**

```bash
# Setup once
docker buildx create --name builder
docker buildx use builder

# Build multi-arch
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t my-app:latest \
  --push \
  .
```

**Explanation:**
- Buildx supports multiple platforms natively
- Creates manifest list for different architectures
- Can build for ARM64 from Intel/AMD runner (emulated)
- Significantly slower due to cross-platform emulation
- For production, use native runners for each platform

---

## Exercise 7: Docker Image Security Scanning

**Solution:**

```yaml
name: Image Security Scan

on:
  push:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Build Image
        run: docker build -t my-app:latest .
      
      - name: Save Image
        run: docker save my-app:latest -o image.tar
      
      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          input: 'image.tar'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
      
      - name: Upload Results to GitHub
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
          category: 'trivy'
      
      - name: Generate Report
        if: always()
        run: |
          echo "## Security Scan Results" >> $GITHUB_STEP_SUMMARY
          if [ $? -eq 0 ]; then
            echo "✅ No critical vulnerabilities found" >> $GITHUB_STEP_SUMMARY
          else
            echo "⚠️ Critical vulnerabilities detected" >> $GITHUB_STEP_SUMMARY
            echo "See Security tab for details" >> $GITHUB_STEP_SUMMARY
          fi
```

**Explanation:**
- Trivy scans saved image tar file
- SARIF format integrates with GitHub Security
- `exit-code: '1'` fails workflow on findings
- Results appear in Security → Code scanning
- SARIF upload uses official GitHub action

---

## Exercise 8: Docker Layer Caching Optimization

**Solution:**

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files first (cacheable, rarely changes)
COPY package*.json ./

# Install dependencies (cached until package*.json changes)
RUN npm ci --only=production

# Copy application code (most frequently changes)
COPY src ./src

# Copy additional config
COPY config ./config

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

# Start application
CMD ["node", "src/index.js"]
```

**Layer Optimization Order:**
1. Base image (FROM) - rarely changes
2. System updates (RUN apt-get) - rarely changes
3. Dependencies (package files + npm install) - sometimes changes
4. Application code (COPY src) - frequently changes
5. Entry point (CMD) - rarely changes

**Benchmark:**
```bash
# First build (cold cache)
time docker build -t app:v1 .
# Output: ~15s

# Second build (no changes)
time docker build -t app:v1 .
# Output: ~1s (nearly instant due to cache)

# Modify src/index.js only
time docker build -t app:v1 .
# Output: ~3s (rebuilds only src layer and below)
```

---

## Exercise 9: Docker Compose for Local Development

**Solution:**

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: my-app
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://user:password@db:5432/app_dev
      REDIS_URL: redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - .:/app
      - /app/node_modules
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  db:
    image: postgres:15-alpine
    container_name: my-app-db
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: app_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: my-app-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

**Usage:**
```bash
# Start all services
docker-compose up

# Run with detached mode
docker-compose up -d

# View logs
docker-compose logs -f app

# Stop services
docker-compose down

# Rebuild after Dockerfile changes
docker-compose up --build

# View service status
docker-compose ps
```

---

## Exercise 10: ECR Lifecycle Policy Management

**Solution:**

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 released versions with v* tag",
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
      "description": "Keep latest tag indefinitely",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["latest"],
        "countType": "imageCountMoreThan",
        "countNumber": 0
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 3,
      "description": "Remove untagged images after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 4,
      "description": "Remove old branch tags after 30 days",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["feature-", "bugfix-"],
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 30
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 5,
      "description": "Keep last 50 develop builds",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["develop-"],
        "countType": "imageCountMoreThan",
        "countNumber": 50
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

**Apply Policy via AWS CLI:**

```bash
# Save as ecr-lifecycle-policy.json
aws ecr put-lifecycle-policy \
  --repository-name my-app \
  --lifecycle-policy-text file://ecr-lifecycle-policy.json \
  --region us-east-1

# Verify policy
aws ecr get-lifecycle-policy \
  --repository-name my-app \
  --region us-east-1
```

**Or via GitHub Actions:**

```yaml
- name: Configure AWS
  uses: aws-actions/configure-aws-credentials@v2
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1

- name: Set ECR Lifecycle Policy
  run: |
    aws ecr put-lifecycle-policy \
      --repository-name my-app \
      --lifecycle-policy-text file://ecr-lifecycle-policy.json
```

**Explanation:**
- rulePriority: Lower number executes first
- Keep latest version indefinitely
- Remove untagged images (build artifacts) after 7 days
- Remove feature branches after 30 days
- Keep last 50 develop builds for debugging
- Automatically reduces storage costs

---

## Summary

You now understand:
- ✅ Building optimized Docker images
- ✅ Multi-stage Dockerfile patterns
- ✅ Comprehensive image tagging strategies
- ✅ AWS ECR authentication and workflows
- ✅ Build context optimization with .dockerignore
- ✅ Multi-platform Docker builds
- ✅ Container security scanning with Trivy
- ✅ Layer caching for faster builds
- ✅ Docker Compose for full-stack development
- ✅ ECR lifecycle management for cost optimization

Continue mastering containerization!
