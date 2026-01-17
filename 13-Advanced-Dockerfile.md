# 🔧 Chapter 13: Advanced Dockerfile Techniques

> Master BuildKit, build arguments, cache optimization, and multi-platform builds.

---

## 🎯 Learning Objectives

- Use BuildKit for faster, more efficient builds
- Pass build-time arguments with ARG
- Optimize layer caching with cache mounts
- Build multi-platform images with Buildx
- Understand ENTRYPOINT vs CMD in depth
- Use .dockerignore effectively

---

## 🚀 BuildKit - Next Generation Builder

**BuildKit** is Docker's modern build engine with significant improvements over the legacy builder.

### Enable BuildKit

```bash
# Option 1: Environment variable (temporary)
DOCKER_BUILDKIT=1 docker build -t myapp .

# Option 2: Docker daemon config (permanent)
# Add to /etc/docker/daemon.json:
{
  "features": {
    "buildkit": true
  }
}

# Option 3: Docker Desktop (enabled by default)
```

### BuildKit Benefits

| Feature | Legacy Builder | BuildKit |
|---------|---------------|----------|
| Parallel builds | ❌ Sequential | ✅ Parallel |
| Cache efficiency | Basic | Advanced |
| Build secrets | ❌ No | ✅ Yes |
| SSH forwarding | ❌ No | ✅ Yes |
| Cache mounts | ❌ No | ✅ Yes |
| Output formats | Image only | Image, local, tar |

---

## 📦 Build Arguments (ARG)

**ARG** allows passing variables at build time (not available at runtime).

### Basic Usage

```dockerfile
# Define build argument with default
ARG NODE_VERSION=18

# Use the argument
FROM node:${NODE_VERSION}-alpine

ARG APP_ENV=production
ARG BUILD_DATE

# Use in RUN commands
RUN echo "Building for ${APP_ENV} on ${BUILD_DATE}"

# ARG values are NOT available after FROM
# Must redeclare if needed in later stages
```

### Pass Arguments During Build

```bash
# Pass single argument
docker build --build-arg NODE_VERSION=20 -t myapp .

# Pass multiple arguments
docker build \
  --build-arg NODE_VERSION=20 \
  --build-arg APP_ENV=staging \
  --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  -t myapp .
```

### ARG vs ENV

| Aspect | ARG | ENV |
|--------|-----|-----|
| Available during | Build only | Build + Runtime |
| Set from CLI | `--build-arg` | `--env` or `-e` |
| Persisted in image | No | Yes |
| Use case | Build-time config | Runtime config |

```dockerfile
# Pattern: Convert ARG to ENV
ARG NODE_ENV=production
ENV NODE_ENV=${NODE_ENV}

# Now NODE_ENV is available at runtime too
```

### Predefined ARGs

Docker provides these automatically:

```dockerfile
# Available without declaring
ARG TARGETPLATFORM    # e.g., linux/amd64
ARG TARGETOS          # e.g., linux
ARG TARGETARCH        # e.g., amd64
ARG BUILDPLATFORM     # Platform of build machine
ARG BUILDOS
ARG BUILDARCH
```

---

## 💾 Cache Mounts (BuildKit)

**Cache mounts** preserve data between builds - huge speedup for package managers!

### Syntax

```dockerfile
# syntax=docker/dockerfile:1

# Cache npm packages
RUN --mount=type=cache,target=/root/.npm \
    npm install

# Cache apt packages
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt \
    apt-get update && apt-get install -y python3

# Cache pip packages
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Cache Go modules
RUN --mount=type=cache,target=/go/pkg/mod \
    go build -o app .
```

### Cache Mount Options

```dockerfile
RUN --mount=type=cache,\
    target=/path/to/cache,\
    sharing=locked,\
    id=my-cache-id \
    command here

# Options:
# target  - Path to cache directory
# sharing - locked|shared|private
# id      - Unique cache identifier
# from    - Use cache from another stage
```

### Real Example: Node.js with Cache

```dockerfile
# syntax=docker/dockerfile:1
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install with npm cache mount
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

COPY . .

CMD ["node", "app.js"]
```

---

## 🔐 Build Secrets (BuildKit)

Pass secrets during build WITHOUT storing them in the image.

### Syntax

```dockerfile
# syntax=docker/dockerfile:1

# Mount a secret file
RUN --mount=type=secret,id=mysecret \
    cat /run/secrets/mysecret

# Use secret in npm install (for private packages)
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install

# Use secret for git clone
RUN --mount=type=secret,id=ssh_key,target=/root/.ssh/id_rsa \
    git clone git@github.com:private/repo.git
```

### Pass Secrets During Build

```bash
# From file
docker build --secret id=mysecret,src=./secret.txt -t myapp .

# From environment variable
docker build --secret id=api_key,env=API_KEY -t myapp .

# NPM token example
docker build --secret id=npmrc,src=$HOME/.npmrc -t myapp .
```

### SSH Forwarding

```dockerfile
# syntax=docker/dockerfile:1

# Forward SSH agent for git clone
RUN --mount=type=ssh \
    git clone git@github.com:private/repo.git
```

```bash
# Build with SSH forwarding
docker build --ssh default -t myapp .
```

---

## 🌍 Multi-Platform Builds (Buildx)

Build images for different architectures (amd64, arm64, etc.)

### Setup Buildx

```bash
# Create a new builder
docker buildx create --name mybuilder --use

# Verify
docker buildx ls

# Inspect (shows supported platforms)
docker buildx inspect --bootstrap
```

### Build for Multiple Platforms

```bash
# Build for multiple architectures
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myuser/myapp:latest \
  --push \
  .

# Build for specific platform
docker buildx build \
  --platform linux/arm64 \
  -t myapp:arm64 \
  --load \
  .
```

### Dockerfile for Multi-Platform

```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM node:18-alpine AS builder

# Use build platform for faster builds
ARG TARGETPLATFORM
ARG BUILDPLATFORM
RUN echo "Building on $BUILDPLATFORM for $TARGETPLATFORM"

WORKDIR /app
COPY . .
RUN npm install && npm run build

# Final image uses target platform
FROM node:18-alpine
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

---

## 🎯 ENTRYPOINT vs CMD Deep Dive

### The Difference

| Aspect | ENTRYPOINT | CMD |
|--------|------------|-----|
| Purpose | Define executable | Default arguments |
| Override | `--entrypoint` flag | Command after image name |
| Combined | ENTRYPOINT + CMD = full command | |

### How They Combine

```dockerfile
ENTRYPOINT ["node"]
CMD ["app.js"]
# Runs: node app.js

# Override CMD at runtime:
docker run myapp server.js
# Runs: node server.js

# Override ENTRYPOINT:
docker run --entrypoint python myapp script.py
# Runs: python script.py
```

### Shell Form vs Exec Form

```dockerfile
# Exec form (recommended) - runs directly
ENTRYPOINT ["node", "app.js"]
CMD ["--port", "3000"]

# Shell form - wraps in /bin/sh -c
ENTRYPOINT node app.js
# PID 1 is shell, not node (signals not forwarded!)
```

### Common Patterns

```dockerfile
# Pattern 1: Fixed command, configurable args
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]

# Pattern 2: Wrapper script
COPY entrypoint.sh /
ENTRYPOINT ["/entrypoint.sh"]
CMD ["start"]

# Pattern 3: Executable container
ENTRYPOINT ["curl"]
CMD ["--help"]
# Usage: docker run mycurl https://example.com
```

### Entrypoint Script Pattern

```bash
#!/bin/sh
# entrypoint.sh

# Run initialization
echo "Starting application..."

# Handle signals
trap 'echo "Shutting down..."; exit 0' SIGTERM SIGINT

# Execute CMD
exec "$@"
```

```dockerfile
COPY entrypoint.sh /
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["node", "app.js"]
```

---

## 🚫 .dockerignore Best Practices

### Comprehensive .dockerignore

```gitignore
# Version control
.git
.gitignore
.gitattributes

# Dependencies (rebuild in container)
node_modules
vendor
__pycache__
*.pyc
.venv
venv

# Build outputs
dist
build
*.egg-info

# IDE and editors
.idea
.vscode
*.swp
*.swo
*~

# OS files
.DS_Store
Thumbs.db

# Docker files (not needed in context)
Dockerfile*
docker-compose*.yml
.dockerignore

# Documentation
README.md
docs
*.md

# Tests
test
tests
__tests__
*.test.js
*.spec.js
coverage
.nyc_output

# CI/CD
.github
.gitlab-ci.yml
.circleci
Jenkinsfile

# Secrets and config
.env
.env.*
*.pem
*.key
secrets

# Logs
logs
*.log
npm-debug.log*

# Temporary files
tmp
temp
*.tmp
```

### Why .dockerignore Matters

```bash
# Without .dockerignore
# Sending build context to Docker daemon  500MB

# With .dockerignore
# Sending build context to Docker daemon  5MB

# Faster builds, smaller context, no secrets leaked!
```

---

## 📊 Layer Caching Optimization

### Order Matters!

```dockerfile
# BAD - Cache busted on every code change
FROM node:18-alpine
COPY . .                    # Code changes = cache bust
RUN npm install             # Reinstalls every time!

# GOOD - Dependencies cached separately
FROM node:18-alpine
COPY package*.json ./       # Only changes when deps change
RUN npm install             # Cached if package.json unchanged
COPY . .                    # Code changes don't affect npm install
```

### Multi-Stage Cache Optimization

```dockerfile
# Stage 1: Dependencies (cached aggressively)
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Build (may change frequently)
FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Stage 3: Production (minimal)
FROM node:18-alpine
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

---

## 🔍 Dockerfile Linting (Hadolint)

### Install Hadolint

```bash
# macOS
brew install hadolint

# Docker
docker run --rm -i hadolint/hadolint < Dockerfile
```

### Run Hadolint

```bash
# Lint Dockerfile
hadolint Dockerfile

# Ignore specific rules
hadolint --ignore DL3008 --ignore DL3009 Dockerfile

# Output as JSON
hadolint -f json Dockerfile
```

### Common Hadolint Rules

| Rule | Issue |
|------|-------|
| DL3008 | Pin apt package versions |
| DL3009 | Delete apt lists after install |
| DL3015 | Use `--no-install-recommends` |
| DL3018 | Pin apk package versions |
| DL3025 | Use JSON for CMD/ENTRYPOINT |
| DL4006 | Set SHELL with pipefail |

### Hadolint in CI

```yaml
# GitHub Actions
- name: Lint Dockerfile
  uses: hadolint/hadolint-action@v3.1.0
  with:
    dockerfile: Dockerfile
```

---

## 🔗 Mapping to Kubernetes

| Docker Build Concept | Kubernetes Relevance |
|---------------------|---------------------|
| Multi-platform builds | Deploy same image to mixed clusters |
| Build args | CI/CD pipeline configuration |
| Build secrets | Use external secret management |
| Buildx | Build in CI, deploy to K8s |
| Layer caching | Faster deployments, CI optimization |

---

## ✅ Summary

### Key Commands

```bash
# BuildKit build
DOCKER_BUILDKIT=1 docker build -t myapp .

# Build with args
docker build --build-arg VERSION=1.0 -t myapp .

# Build with secrets
docker build --secret id=key,src=./key.pem -t myapp .

# Multi-platform build
docker buildx build --platform linux/amd64,linux/arm64 -t myapp --push .

# Lint Dockerfile
hadolint Dockerfile
```

### Best Practices

1. ✅ **Enable BuildKit** for faster builds
2. ✅ **Use cache mounts** for package managers
3. ✅ **Use build secrets** instead of copying secret files
4. ✅ **Order layers** for optimal caching
5. ✅ **Use exec form** for ENTRYPOINT/CMD
6. ✅ **Lint Dockerfiles** with Hadolint
7. ✅ **Use .dockerignore** to exclude unnecessary files

---

*Next: Docker Logging & Monitoring*

