# 📦 Chapter 11: Docker Registry & Image Management

> Understanding how to store, share, and manage Docker images in registries.

---

## 🎯 Learning Objectives

- Understand what a Docker registry is
- Push and pull images to/from Docker Hub
- Use image tagging strategies
- Work with private registries
- Implement image management best practices

---

## 🌍 What is a Docker Registry?

A **Docker Registry** is a storage and distribution system for Docker images.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DOCKER REGISTRY CONCEPT                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Your Machine                    Registry                              │
│   ┌──────────┐                   ┌──────────────────┐                  │
│   │  Docker  │ ── docker push ─> │   Docker Hub     │                  │
│   │  Images  │ <─ docker pull ── │   (or private)   │                  │
│   └──────────┘                   └──────────────────┘                  │
│                                          │                              │
│                                          ▼                              │
│                              ┌──────────────────┐                      │
│                              │  Other Machines  │                      │
│                              │  CI/CD Pipelines │                      │
│                              │  Kubernetes      │                      │
│                              └──────────────────┘                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Real-World Analogy

Think of a registry like **GitHub for code, but for Docker images**:
- GitHub stores your source code
- Docker Registry stores your built images
- Both allow versioning, sharing, and collaboration

---

## 🏢 Types of Registries

| Registry | Type | Use Case |
|----------|------|----------|
| **Docker Hub** | Public/Private | Default, free for public images |
| **Amazon ECR** | Private | AWS-integrated, production |
| **Google GCR/Artifact Registry** | Private | GCP-integrated |
| **Azure ACR** | Private | Azure-integrated |
| **Harbor** | Self-hosted | Enterprise, on-premise |
| **GitHub Container Registry** | Public/Private | GitHub-integrated |

---

## 🔑 Docker Hub Basics

### 1. Create an Account

Go to [hub.docker.com](https://hub.docker.com) and sign up.

### 2. Login from CLI

```bash
# Login to Docker Hub
docker login

# Login with username
docker login -u yourusername

# Login to a different registry
docker login registry.example.com
```

### 3. Image Naming Convention

```
[registry/]username/image-name[:tag]

Examples:
  nginx                          # Official image (no username)
  myuser/myapp                   # Docker Hub (default registry)
  myuser/myapp:v1.0.0           # With version tag
  myuser/myapp:latest           # Latest tag (default)
  
  gcr.io/my-project/myapp:v1    # Google Container Registry
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1  # AWS ECR
```

---

## 📤 Pushing Images to Registry

### Step 1: Build Your Image

```bash
# Build with proper naming
docker build -t myusername/myapp:v1.0.0 .

# Or tag an existing image
docker tag myapp:latest myusername/myapp:v1.0.0
```

### Step 2: Push to Registry

```bash
# Push to Docker Hub
docker push myusername/myapp:v1.0.0

# Push all tags
docker push myusername/myapp --all-tags
```

### Step 3: Verify on Docker Hub

Visit `https://hub.docker.com/r/myusername/myapp`

---

## 📥 Pulling Images from Registry

```bash
# Pull latest (default)
docker pull nginx

# Pull specific version
docker pull nginx:1.25.0

# Pull from private registry
docker pull gcr.io/my-project/myapp:v1.0.0

# Pull and run
docker run myusername/myapp:v1.0.0
```

---

## 🏷️ Image Tagging Strategies

### Strategy 1: Semantic Versioning (Recommended)

```bash
# Major.Minor.Patch
docker tag myapp myuser/myapp:1.0.0
docker tag myapp myuser/myapp:1.0
docker tag myapp myuser/myapp:1
docker tag myapp myuser/myapp:latest

# All point to same image, different granularity
```

| Tag | Meaning | When to Use |
|-----|---------|-------------|
| `1.0.0` | Exact version | Production, pinned |
| `1.0` | Latest patch of 1.0.x | Auto-update patches |
| `1` | Latest of 1.x.x | Auto-update minor |
| `latest` | Most recent build | Development only |

### Strategy 2: Git-Based Tags

```bash
# Using commit SHA
docker tag myapp myuser/myapp:abc123f

# Using branch name
docker tag myapp myuser/myapp:main
docker tag myapp myuser/myapp:feature-login

# Using CI build number
docker tag myapp myuser/myapp:build-456
```

### Strategy 3: Date-Based Tags

```bash
# Using date
docker tag myapp myuser/myapp:2024-01-15
docker tag myapp myuser/myapp:20240115-143052
```

### ⚠️ Avoid Using `latest` in Production

```bash
# BAD - "latest" can change anytime
docker pull myapp:latest

# GOOD - Pinned version
docker pull myapp:1.2.3
```

---

## 🔒 Private Registries

### AWS ECR (Elastic Container Registry)

```bash
# 1. Authenticate
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# 2. Tag image
docker tag myapp:latest \
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0.0

# 3. Push
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0.0
```

### Google Container Registry (GCR)

```bash
# 1. Authenticate
gcloud auth configure-docker

# 2. Tag image
docker tag myapp:latest gcr.io/my-project/myapp:v1.0.0

# 3. Push
docker push gcr.io/my-project/myapp:v1.0.0
```

### Self-Hosted Registry

```bash
# Run a local registry
docker run -d -p 5000:5000 --name registry registry:2

# Tag for local registry
docker tag myapp localhost:5000/myapp:v1.0.0

# Push to local registry
docker push localhost:5000/myapp:v1.0.0

# Pull from local registry
docker pull localhost:5000/myapp:v1.0.0
```

---

## 🔍 Managing Images

### List Images

```bash
# List local images
docker images

# List with filters
docker images --filter "reference=myapp*"

# Show image digests
docker images --digests
```

### Remove Images

```bash
# Remove specific image
docker rmi myapp:v1.0.0

# Remove all unused images
docker image prune

# Remove ALL images (dangerous!)
docker rmi $(docker images -q)
```

### Inspect Images

```bash
# Show image details
docker inspect myapp:v1.0.0

# Show image history (layers)
docker history myapp:v1.0.0

# Check image size
docker images myapp --format "{{.Repository}}:{{.Tag}} - {{.Size}}"
```

---

## 🔐 Image Security in Registries

### 1. Enable Image Scanning

```bash
# Docker Hub - Enable in repository settings

# AWS ECR - Enable scan on push
aws ecr put-image-scanning-configuration \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true

# Scan manually
docker scout cves myapp:v1.0.0
```

### 2. Use Image Digests (Immutable)

```bash
# Pull by digest (cannot be changed)
docker pull myapp@sha256:abc123...

# In Kubernetes (recommended for production)
image: myapp@sha256:abc123...
```

### 3. Sign Images (Docker Content Trust)

```bash
# Enable content trust
export DOCKER_CONTENT_TRUST=1

# Now pushes require signing
docker push myuser/myapp:v1.0.0
```

---

## 🔗 Mapping to Kubernetes

| Docker Registry Concept | Kubernetes Equivalent |
|------------------------|----------------------|
| `docker pull` | Kubelet pulls images |
| Private registry auth | `imagePullSecrets` |
| Image tags | `image:` field in Pod spec |
| `:latest` tag | Avoid! Use pinned versions |
| Registry | Configured per cluster |

### Kubernetes imagePullSecrets Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: myregistry.com/myapp:v1.0.0
  imagePullSecrets:
  - name: my-registry-secret
```

---

## ✅ Summary: Key Commands

| Command | Purpose |
|---------|---------|
| `docker login` | Authenticate to registry |
| `docker tag` | Tag image for registry |
| `docker push` | Upload image to registry |
| `docker pull` | Download image from registry |
| `docker images` | List local images |
| `docker rmi` | Remove local image |
| `docker image prune` | Remove unused images |

### Best Practices

1. ✅ **Use semantic versioning** for production images
2. ✅ **Never use `:latest`** in production
3. ✅ **Use image digests** for maximum security
4. ✅ **Enable image scanning** in your registry
5. ✅ **Clean up old images** regularly
6. ✅ **Use private registries** for proprietary code

---

*Next: Docker Internals - Understanding what happens under the hood*

