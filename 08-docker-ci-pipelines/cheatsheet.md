# Module 08 Cheatsheet: Docker and ECR CI/CD Pipelines

## Quick Reference

### Docker Commands

| Task | Command |
|------|---------|
| Build image | `docker build -t myapp:latest .` |
| Run container | `docker run -p 3000:3000 myapp:latest` |
| List images | `docker image ls` |
| Remove image | `docker image rm myapp:latest` |
| Push to registry | `docker push registry/myapp:latest` |
| Pull from registry | `docker pull registry/myapp:latest` |
| Save image to tar | `docker save myapp:latest -o image.tar` |
| Load image from tar | `docker load -i image.tar` |
| View image layers | `docker history myapp:latest` |
| Inspect image | `docker image inspect myapp:latest` |
| Tag image | `docker tag myapp:latest myapp:v1.0` |
| Stop container | `docker stop <container-id>` |
| Remove container | `docker rm <container-id>` |
| View logs | `docker logs <container-id>` |

### Docker Build Flags

| Flag | Purpose | Example |
|------|---------|---------|
| `-t` | Tag image | `docker build -t myapp:1.0 .` |
| `-f` | Specify Dockerfile | `docker build -f Dockerfile.prod .` |
| `--build-arg` | Build argument | `docker build --build-arg VERSION=1.0 .` |
| `--no-cache` | Skip cache | `docker build --no-cache .` |
| `--platform` | Target platform | `docker build --platform linux/arm64 .` |
| `--progress` | Show progress | `docker build --progress=plain .` |

### Dockerfile Instructions

| Instruction | Purpose | Example |
|-------------|---------|---------|
| FROM | Base image | `FROM node:18-alpine` |
| WORKDIR | Set working directory | `WORKDIR /app` |
| COPY | Copy files | `COPY src ./src` |
| ADD | Copy/extract files | `ADD archive.tar.gz .` |
| RUN | Execute command | `RUN npm ci` |
| ENV | Set environment variable | `ENV NODE_ENV=production` |
| EXPOSE | Document port | `EXPOSE 3000` |
| CMD | Default command | `CMD ["node", "index.js"]` |
| ENTRYPOINT | Entrypoint script | `ENTRYPOINT ["./start.sh"]` |
| ARG | Build-time variable | `ARG VERSION=1.0` |
| HEALTHCHECK | Health probe | `HEALTHCHECK --interval=30s CMD curl localhost:3000` |
| USER | Run as user | `USER appuser` |
| VOLUME | Mount point | `VOLUME /data` |

### Common Dockerfile Patterns

**Minimal Node.js Image:**
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY src ./src
EXPOSE 3000
CMD ["node", "src/index.js"]
```

**Multi-stage Build:**
```dockerfile
FROM node:18 AS builder
WORKDIR /build
COPY package*.json ./
RUN npm ci

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /build/node_modules ./node_modules
COPY src ./src
CMD ["node", "src/index.js"]
```

**With Health Check:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

**With Build Args:**
```dockerfile
ARG VERSION=latest
ARG BUILD_DATE
RUN echo "Building version ${VERSION} on ${BUILD_DATE}"
```

---

## AWS ECR Commands

| Task | Command |
|------|---------|
| Create repository | `aws ecr create-repository --repository-name myapp` |
| List repositories | `aws ecr describe-repositories` |
| List images | `aws ecr list-images --repository-name myapp` |
| Get login token | `aws ecr get-authorization-token` |
| Push image | `aws ecr batch-upload-layer-part` |
| Pull image | `aws ecr batch-get-image` |
| Delete image | `aws ecr batch-delete-image --repository-name myapp --image-ids imageTag=v1.0` |
| Scan image | `aws ecr start-image-scan --repository-name myapp --image-id imageTag=latest` |
| Get scan results | `aws ecr describe-image-scan-findings --repository-name myapp --image-id imageTag=latest` |
| Set lifecycle policy | `aws ecr put-lifecycle-policy --repository-name myapp --lifecycle-policy-text file://policy.json` |
| Get lifecycle policy | `aws ecr get-lifecycle-policy --repository-name myapp` |

---

## GitHub Actions for Docker

### Docker Build Action

```yaml
- uses: docker/setup-buildx-action@v2

- uses: docker/build-push-action@v4
  with:
    context: .
    file: ./Dockerfile
    push: true
    tags: |
      myregistry/myapp:latest
      myregistry/myapp:${{ github.sha }}
    platforms: linux/amd64,linux/arm64
```

### ECR Login

```yaml
- uses: aws-actions/configure-aws-credentials@v2
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1

- id: login-ecr
  uses: aws-actions/amazon-ecr-login@v1

- run: |
    docker build -t ${{ steps.login-ecr.outputs.registry }}/myapp:latest .
    docker push ${{ steps.login-ecr.outputs.registry }}/myapp:latest
```

### Trivy Scanning

```yaml
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:latest
    format: sarif
    output: trivy-results.sarif
    severity: CRITICAL,HIGH

- uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: trivy-results.sarif
```

---

## Image Tagging Strategies

| Strategy | Format | Use Case |
|----------|--------|----------|
| Semantic Versioning | `v1.0.0`, `v1.0.1` | Release versions |
| Commit SHA | `abc123def` | Build traceability |
| Branch | `main`, `develop` | Branch-specific builds |
| Date | `2024-01-15`, `20240115` | Build history |
| Build Number | `build-123` | CI system tracking |
| Combined | `v1.0.0-main-abc123` | Production tracking |

**Recommended Multi-tag Approach:**
```bash
docker tag myapp:${COMMIT_SHA} myapp:latest
docker tag myapp:${COMMIT_SHA} myapp:${VERSION}
docker tag myapp:${COMMIT_SHA} myapp:${BRANCH}
```

---

## Docker Compose

### Basic Commands

| Command | Purpose |
|---------|---------|
| `docker-compose up` | Start services |
| `docker-compose up -d` | Start detached |
| `docker-compose down` | Stop and remove |
| `docker-compose ps` | List services |
| `docker-compose logs` | View logs |
| `docker-compose exec app sh` | Execute in service |
| `docker-compose build` | Rebuild images |
| `docker-compose pull` | Pull latest images |

### Basic Structure

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
    depends_on:
      - db
    networks:
      - app-net

  db:
    image: postgres:15-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - app-net

volumes:
  db_data:

networks:
  app-net:
```

---

## Security Scanning Tools

| Tool | Speed | Accuracy | Language |
|------|-------|----------|----------|
| Trivy | Very Fast | Medium | Go |
| Snyk Container | Fast | High | JavaScript |
| Grype | Fast | High | Go |
| Clair | Slow | Medium | Go |
| Anchore | Very Slow | Very High | Python |

### Trivy Usage

```bash
# Scan image
trivy image myapp:latest

# Show only high/critical
trivy image --severity HIGH,CRITICAL myapp:latest

# JSON output
trivy image --format json myapp:latest > report.json

# SARIF output
trivy image --format sarif myapp:latest > report.sarif

# Exit with code 1 on findings
trivy image --exit-code 1 myapp:latest
```

---

## Multi-architecture Build

### With buildx

```bash
# Setup
docker buildx create --name builder
docker buildx use builder

# Build multi-arch
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myapp:latest \
  --push \
  .

# View platforms
docker buildx ls
```

### In GitHub Actions

```yaml
- uses: docker/setup-buildx-action@v2

- uses: docker/build-push-action@v4
  with:
    platforms: linux/amd64,linux/arm64
    push: true
    tags: myregistry/myapp:latest
```

---

## Layer Caching Tips

| Practice | Impact | Example |
|----------|--------|---------|
| Order by change frequency | +50% faster | Dependencies before code |
| Use specific base versions | +20% faster | `node:18.10.0` vs `node:latest` |
| Separate concerns | +30% faster | Separate system, dependency, code layers |
| Use .dockerignore | +40% faster | Exclude node_modules, .git |
| Minimal base images | +60% smaller | alpine vs ubuntu |

---

## Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| "repository not found" | Repository doesn't exist | Create with `aws ecr create-repository` |
| "Access denied" | IAM permissions | Add ECR push permissions to policy |
| "Login failed" | Invalid credentials | Verify AWS_ACCESS_KEY_ID and SECRET |
| "Image too large" | Unoptimized Dockerfile | Use multi-stage, alpine, .dockerignore |
| "Out of space" | Old images accumulating | Set ECR lifecycle policy |
| "Build timeout" | Large dependency install | Use caching, smaller base image |
| "Cross-arch build slow" | Emulation overhead | Use native runners when possible |
| "Port already in use" | Local conflict | Change host port: `-p 3001:3000` |
| "Health check failing" | Service not ready | Increase start_period in health check |
| "Network unreachable" | Container isolation | Use `docker network create` and link |

---

## .dockerignore Template

```
# Dependencies
node_modules
npm-debug.log
yarn-error.log
pnpm-error.log

# Git
.git
.gitignore
.gitattributes

# IDE
.vscode
.idea
.sublime-*
*.swp
*.swo

# CI/CD
.github
.gitlab-ci.yml
.travis.yml
Jenkinsfile

# Docs
README.md
CHANGELOG.md
docs/

# Tests
*.test.js
*.spec.js
__tests__/
coverage/

# Build
dist/
build/
.next/
.nuxt/

# Environment
.env
.env.local

# Other
.dockerignore
Dockerfile*
docker-compose*.yml
```

---

## ECR Lifecycle Policy Template

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 released versions",
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
      "description": "Remove untagged after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

---

## Environment Variables

**In Dockerfile:**
```dockerfile
ENV NODE_ENV=production
ENV PORT=3000
ENV LOG_LEVEL=info
```

**At build time:**
```bash
docker build --build-arg VERSION=1.0 .
```

**At runtime:**
```bash
docker run -e NODE_ENV=development myapp
```

**In docker-compose:**
```yaml
environment:
  NODE_ENV: production
  DATABASE_URL: postgresql://...
```

---

## Health Checks

```dockerfile
# HTTP endpoint
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Command execution
HEALTHCHECK --interval=30s --timeout=3s \
  CMD node -e "require('http').get('http://localhost:3000')"

# Script
HEALTHCHECK CMD ./healthcheck.sh
```

---

## Pro Tips

1. **Use specific base image versions**
   ```dockerfile
   FROM node:18.10.0-alpine    # ✓ Good
   FROM node:latest             # ✗ Bad
   ```

2. **Separate concerns**
   ```dockerfile
   RUN apt-get update && apt-get install -y curl  # Single layer
   COPY package*.json ./
   RUN npm ci
   COPY src ./
   ```

3. **Run as non-root**
   ```dockerfile
   USER appuser                 # Don't use root
   ```

4. **Use COPY instead of ADD**
   ```dockerfile
   COPY src ./src              # ✓ Clear intent
   ```

5. **Order Dockerfile from least to most changing**
   ```dockerfile
   FROM ...                    # Changes rarely
   RUN apt-get ...             # Changes rarely
   COPY package*.json ./       # Changes sometimes
   RUN npm ci                  # Changes sometimes
   COPY src ./                 # Changes often
   CMD [...]                   # Changes rarely
   ```

---

## Validation Checklist

- [ ] Dockerfile optimized with multi-stage build
- [ ] Base image appropriate and versioned
- [ ] Dependencies installed before code
- [ ] .dockerignore configured
- [ ] Health check implemented
- [ ] Non-root user used
- [ ] Image size reasonable (<200MB typical)
- [ ] Secrets not in image
- [ ] Layer caching working (build fast on second run)
- [ ] ECR repository created
- [ ] IAM user has push permissions
- [ ] GitHub Secrets configured
- [ ] Image scanned for vulnerabilities
- [ ] Lifecycle policy manages old images
- [ ] Multiple tags (sha, latest, branch)

---

## Resource Links

**Docker:**
- Docs: https://docs.docker.com
- Hub: https://hub.docker.com
- Best Practices: https://docs.docker.com/develop/dev-best-practices

**AWS ECR:**
- Documentation: https://docs.aws.amazon.com/ecr
- Registry URL: https://console.aws.amazon.com/ecr

**Security:**
- Trivy: https://github.com/aquasecurity/trivy
- Snyk: https://snyk.io
- Grype: https://github.com/anchore/grype

**GitHub Actions:**
- Docker Build Action: https://github.com/docker/build-push-action
- AWS ECR Action: https://github.com/aws-actions/amazon-ecr-login
- Setup Buildx: https://github.com/docker/setup-buildx-action
