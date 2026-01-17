# 🔄 Chapter 15: Docker in CI/CD

> Building, testing, and deploying Docker images in continuous integration and deployment pipelines.

---

## 🎯 Learning Objectives

- Understand Docker's role in CI/CD pipelines
- Build and push images in GitHub Actions, GitLab CI, Jenkins
- Implement image tagging strategies
- Scan images for vulnerabilities in CI
- Best practices for production deployments

---

## 🔄 Docker in CI/CD Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DOCKER CI/CD PIPELINE                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Developer                                                               │
│      │                                                                   │
│      ▼                                                                   │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐         │
│  │   Code   │───>│  Build   │───>│   Test   │───>│   Push   │         │
│  │  Commit  │    │  Image   │    │  Image   │    │ Registry │         │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘         │
│                                                        │                 │
│                                                        ▼                 │
│                                               ┌──────────────┐          │
│                                               │    Deploy    │          │
│                                               │  to Staging  │          │
│                                               └──────────────┘          │
│                                                        │                 │
│                                                        ▼                 │
│                                               ┌──────────────┐          │
│                                               │   Deploy to  │          │
│                                               │  Production  │          │
│                                               └──────────────┘          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🐙 GitHub Actions

### Basic Build and Push

```yaml
# .github/workflows/docker.yml
name: Docker Build and Push

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=sha

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### With Security Scanning

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy Scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy Results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Fail on Critical Vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          exit-code: '1'
          severity: 'CRITICAL'
```

### Multi-Platform Build

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and Push Multi-Platform
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: myuser/myapp:latest
```

---

## 🦊 GitLab CI

### Basic Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - scan
  - push
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  DOCKER_LATEST: $CI_REGISTRY_IMAGE:latest

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t $DOCKER_IMAGE .
    - docker save $DOCKER_IMAGE > image.tar
  artifacts:
    paths:
      - image.tar
    expire_in: 1 hour

test:
  stage: test
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker load < image.tar
    - docker run --rm $DOCKER_IMAGE npm test

scan:
  stage: scan
  image: 
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $DOCKER_IMAGE
  allow_failure: true

push:
  stage: push
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker load < image.tar
    - docker push $DOCKER_IMAGE
    - docker tag $DOCKER_IMAGE $DOCKER_LATEST
    - docker push $DOCKER_LATEST
  only:
    - main

deploy:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/myapp myapp=$DOCKER_IMAGE
  only:
    - main
  environment:
    name: production
```

---

## 🔧 Jenkins

### Jenkinsfile (Declarative)

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    environment {
        REGISTRY = 'docker.io'
        IMAGE_NAME = 'myuser/myapp'
        DOCKER_CREDENTIALS = credentials('docker-hub-credentials')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}:${env.BUILD_NUMBER}")
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    docker.image("${IMAGE_NAME}:${env.BUILD_NUMBER}").inside {
                        sh 'npm test'
                    }
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                sh """
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image --exit-code 0 --severity HIGH,CRITICAL \
                        ${IMAGE_NAME}:${env.BUILD_NUMBER}
                """
            }
        }
        
        stage('Push') {
            when {
                branch 'main'
            }
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'docker-hub-credentials') {
                        docker.image("${IMAGE_NAME}:${env.BUILD_NUMBER}").push()
                        docker.image("${IMAGE_NAME}:${env.BUILD_NUMBER}").push('latest')
                    }
                }
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    kubectl set image deployment/myapp \
                        myapp=${REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER}
                """
            }
        }
    }
    
    post {
        always {
            sh 'docker rmi ${IMAGE_NAME}:${env.BUILD_NUMBER} || true'
        }
    }
}
```

---

## 🏷️ Image Tagging Strategies in CI/CD

### Recommended Tagging

```bash
# Git SHA (immutable, traceable)
myapp:abc123f

# Semantic version (for releases)
myapp:1.2.3
myapp:1.2
myapp:1

# Branch-based (for development)
myapp:main
myapp:feature-login

# Build number (for tracking)
myapp:build-456

# Date-based
myapp:2024-01-15
myapp:20240115-143052
```

### GitHub Actions Metadata

```yaml
- uses: docker/metadata-action@v5
  with:
    images: myregistry/myapp
    tags: |
      # Git SHA short
      type=sha,prefix=
      
      # Semantic versioning from git tag
      type=semver,pattern={{version}}
      type=semver,pattern={{major}}.{{minor}}
      type=semver,pattern={{major}}
      
      # Branch name
      type=ref,event=branch
      
      # PR number
      type=ref,event=pr
      
      # Latest on default branch
      type=raw,value=latest,enable={{is_default_branch}}
```

---

## 🔒 Security Scanning in CI

### Trivy

```yaml
# GitHub Actions
- name: Run Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:latest
    format: 'table'
    exit-code: '1'
    ignore-unfixed: true
    vuln-type: 'os,library'
    severity: 'CRITICAL,HIGH'
```

### Docker Scout

```yaml
# GitHub Actions
- name: Docker Scout
  uses: docker/scout-action@v1
  with:
    command: cves
    image: myapp:latest
    only-severities: critical,high
    exit-code: true
```

### Snyk

```yaml
# GitHub Actions
- name: Snyk Container Scan
  uses: snyk/actions/docker@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    image: myapp:latest
    args: --severity-threshold=high
```

---

## 🚀 Deployment Strategies

### Rolling Update

```yaml
# Kubernetes deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: myapp
        image: myregistry/myapp:1.2.3
```

### Blue-Green Deployment

```bash
# Deploy new version
kubectl apply -f deployment-green.yaml

# Test new version
curl http://green-service.internal/health

# Switch traffic
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

# Rollback if needed
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'
```

### Canary Deployment

```yaml
# Deploy canary with 10% traffic
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: stable
      weight: 90
    - destination:
        host: myapp
        subset: canary
      weight: 10
```

---

## 🏗️ Build Optimization for CI

### Layer Caching

```yaml
# GitHub Actions with cache
- uses: docker/build-push-action@v5
  with:
    context: .
    cache-from: type=gha
    cache-to: type=gha,mode=max
    
# Or with registry cache
- uses: docker/build-push-action@v5
  with:
    context: .
    cache-from: type=registry,ref=myregistry/myapp:cache
    cache-to: type=registry,ref=myregistry/myapp:cache,mode=max
```

### Parallel Builds

```yaml
# Build multiple services in parallel
jobs:
  build:
    strategy:
      matrix:
        service: [api, worker, frontend]
    steps:
      - uses: docker/build-push-action@v5
        with:
          context: ./${{ matrix.service }}
          tags: myregistry/${{ matrix.service }}:${{ github.sha }}
```

---

## 📋 CI/CD Best Practices

### Do's ✅

```yaml
# 1. Use specific base image versions
FROM node:18.19.0-alpine3.19

# 2. Pin action versions
uses: docker/build-push-action@v5.1.0

# 3. Use image digests for security
image: myapp@sha256:abc123...

# 4. Scan images before pushing
- run: trivy image --exit-code 1 myapp:${{ github.sha }}

# 5. Use build cache
cache-from: type=gha
cache-to: type=gha,mode=max

# 6. Multi-stage builds for smaller images
FROM node:18 AS builder
# ...build...
FROM node:18-alpine AS production
```

### Don'ts ❌

```yaml
# 1. Don't use :latest in production
image: myapp:latest  # BAD

# 2. Don't store secrets in images
COPY .env .  # BAD

# 3. Don't run as root
USER root  # BAD

# 4. Don't skip security scanning
# No scanning = vulnerabilities in production

# 5. Don't build without cache
--no-cache  # Slow builds
```

---

## 🔗 Mapping to Kubernetes

| CI/CD Concept | Kubernetes Integration |
|---------------|----------------------|
| Image push | Stored in registry, pulled by kubelet |
| Image tag | Referenced in Pod spec |
| Image scan | Admission controllers can block |
| Deployment | kubectl, Helm, ArgoCD |
| Rollback | `kubectl rollout undo` |
| Secrets | K8s Secrets, external-secrets |

---

## ✅ Summary

### Key CI/CD Stages

| Stage | Docker Commands |
|-------|----------------|
| Build | `docker build -t image:tag .` |
| Test | `docker run image:tag npm test` |
| Scan | `trivy image image:tag` |
| Push | `docker push image:tag` |
| Deploy | `kubectl set image deployment/app app=image:tag` |

### Best Practices

1. ✅ **Use immutable tags** (SHA, version) not `:latest`
2. ✅ **Scan images** for vulnerabilities before push
3. ✅ **Use build cache** for faster builds
4. ✅ **Multi-stage builds** for smaller images
5. ✅ **Sign images** for supply chain security
6. ✅ **Automate everything** - no manual docker push

---

*Next: Docker Production Readiness Checklist*

