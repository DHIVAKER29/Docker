# ✅ Chapter 16: Docker Production Readiness Checklist

> A comprehensive checklist to ensure your Docker containers are production-ready.

---

## 🎯 Purpose

This chapter consolidates everything we've learned into actionable checklists for:
- Dockerfile best practices
- Container runtime configuration
- Security hardening
- Monitoring and observability
- CI/CD integration

---

## 📋 Dockerfile Checklist

### Base Image

```
□ Use official images when possible
□ Use specific version tags (not :latest)
□ Use minimal base images (alpine, distroless, scratch)
□ Regularly update base images for security patches
```

```dockerfile
# ✅ Good
FROM node:18.19.0-alpine3.19

# ❌ Bad
FROM node
FROM node:latest
```

### Build Optimization

```
□ Order layers from least to most frequently changing
□ Combine RUN commands to reduce layers
□ Use multi-stage builds
□ Use .dockerignore to exclude unnecessary files
□ Use BuildKit for faster builds
□ Leverage build cache effectively
```

```dockerfile
# ✅ Optimized layer order
COPY package*.json ./
RUN npm ci --only=production
COPY . .
```

### Security

```
□ Run as non-root user
□ Don't install unnecessary packages
□ Remove package manager cache after install
□ Don't store secrets in the image
□ Use COPY instead of ADD (unless extracting archives)
□ Set proper file permissions
```

```dockerfile
# ✅ Security-focused
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

USER appuser
```

### Configuration

```
□ Use ENTRYPOINT for fixed commands
□ Use CMD for default arguments
□ Use exec form (JSON array) not shell form
□ Define EXPOSE for documentation
□ Use ENV for runtime configuration
□ Use ARG for build-time configuration
```

```dockerfile
# ✅ Proper exec form
ENTRYPOINT ["node"]
CMD ["app.js"]
```

### Health & Signals

```
□ Add HEALTHCHECK instruction
□ Handle SIGTERM for graceful shutdown
□ Set appropriate timeouts
□ Use init process if needed (--init or tini)
```

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

---

## 🏃 Runtime Configuration Checklist

### Resource Limits

```
□ Set memory limits (-m, --memory)
□ Set CPU limits (--cpus)
□ Set restart policy (--restart)
□ Consider OOM behavior (--oom-kill-disable with caution)
```

```bash
docker run -d \
  --memory=512m \
  --cpus=0.5 \
  --restart=unless-stopped \
  myapp
```

### Networking

```
□ Use user-defined networks (not default bridge)
□ Don't expose unnecessary ports
□ Use internal networks when possible
□ Bind to 0.0.0.0 inside container
```

```bash
docker network create myapp-network
docker run --network myapp-network myapp
```

### Storage

```
□ Use named volumes for persistent data
□ Mount volumes as read-only when possible (:ro)
□ Don't store data in container layer
□ Plan volume backup strategy
```

```bash
docker run -v mydata:/app/data:ro myapp
```

### Security Runtime

```
□ Don't run as --privileged
□ Use --read-only when possible
□ Drop unnecessary capabilities (--cap-drop ALL)
□ Add only required capabilities (--cap-add)
□ Use seccomp profiles
□ Don't mount Docker socket into containers
```

```bash
docker run \
  --read-only \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  myapp
```

---

## 🔒 Security Checklist

### Image Security

```
□ Scan images for vulnerabilities (Trivy, Scout, Snyk)
□ Use minimal base images
□ Keep images updated
□ Sign images (Docker Content Trust)
□ Use image digests for production
□ Verify image provenance
```

### Secrets Management

```
□ Never store secrets in images
□ Never store secrets in environment variables (visible in inspect)
□ Use Docker secrets or external secret management
□ Rotate secrets regularly
□ Use build secrets (BuildKit) for build-time secrets
```

### Access Control

```
□ Limit Docker daemon access
□ Use TLS for remote Docker daemon
□ Implement RBAC in orchestrators
□ Audit container actions
□ Use read-only root filesystem
```

### Network Security

```
□ Use network segmentation
□ Don't expose management ports
□ Use TLS for service communication
□ Implement network policies
□ Disable inter-container communication when not needed
```

---

## 📊 Monitoring Checklist

### Logging

```
□ Log to stdout/stderr (not files)
□ Use structured logging (JSON)
□ Configure log rotation
□ Set up centralized logging
□ Include correlation IDs
□ Don't log sensitive data
```

### Health Checks

```
□ Implement health check endpoints
□ Configure Docker HEALTHCHECK
□ Monitor health status
□ Set appropriate intervals and timeouts
□ Have dependency health checks
```

### Metrics

```
□ Expose application metrics
□ Monitor resource usage (CPU, memory, I/O)
□ Set up alerting thresholds
□ Track container restarts
□ Monitor image vulnerabilities over time
```

---

## 🔄 CI/CD Checklist

### Build Pipeline

```
□ Automated builds on code push
□ Lint Dockerfiles (Hadolint)
□ Run unit tests in containers
□ Security scan before push
□ Use build cache for speed
□ Multi-platform builds if needed
```

### Image Management

```
□ Consistent tagging strategy
□ Use immutable tags (SHA, version)
□ Push to private registry
□ Clean up old images
□ Document image contents
```

### Deployment

```
□ Never deploy :latest to production
□ Use rolling updates
□ Implement health checks for rollback
□ Have rollback strategy
□ Use GitOps for deployments
□ Test in staging before production
```

---

## 📝 Quick Reference: Production Docker Run

```bash
docker run -d \
  --name myapp \
  --hostname myapp \
  --restart unless-stopped \
  --memory 512m \
  --cpus 0.5 \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  --user 1001:1001 \
  --network myapp-network \
  --health-cmd "curl -f http://localhost:8080/health || exit 1" \
  --health-interval 30s \
  --health-timeout 10s \
  --health-retries 3 \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  --env NODE_ENV=production \
  -v myapp-data:/app/data:rw \
  -v /app/config/config.json:/app/config.json:ro \
  -p 127.0.0.1:8080:8080 \
  myregistry/myapp:1.2.3@sha256:abc123...
```

---

## 📝 Quick Reference: Production Docker Compose

```yaml
version: '3.8'

x-common: &common
  restart: unless-stopped
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"
  security_opt:
    - no-new-privileges:true
  read_only: true
  tmpfs:
    - /tmp:rw,noexec,nosuid

services:
  app:
    <<: *common
    image: myregistry/myapp:1.2.3
    user: "1001:1001"
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    networks:
      - frontend
      - backend
    volumes:
      - app-data:/app/data
    environment:
      NODE_ENV: production
    expose:
      - "8080"

  nginx:
    <<: *common
    image: nginx:1.25-alpine
    ports:
      - "443:443"
    networks:
      - frontend
    depends_on:
      app:
        condition: service_healthy

networks:
  frontend:
  backend:
    internal: true

volumes:
  app-data:
```

---

## 📝 Quick Reference: Production Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

# Build stage
FROM node:18.19.0-alpine3.19 AS builder

WORKDIR /app

# Install dependencies first (cache layer)
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

# Copy source
COPY --chown=node:node . .

# Build if needed
RUN npm run build

# Production stage
FROM node:18.19.0-alpine3.19 AS production

# Security: Add non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

# Security: Install only runtime dependencies
RUN apk add --no-cache dumb-init curl

WORKDIR /app

# Copy only production files
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

# Security: Switch to non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Use dumb-init for proper signal handling
ENTRYPOINT ["/usr/bin/dumb-init", "--"]

CMD ["node", "dist/index.js"]

EXPOSE 8080

# Labels for metadata
LABEL org.opencontainers.image.source="https://github.com/myorg/myapp"
LABEL org.opencontainers.image.description="My Production App"
LABEL org.opencontainers.image.licenses="MIT"
```

---

## 🔗 Mapping to Kubernetes

| Docker Production Config | Kubernetes Equivalent |
|-------------------------|----------------------|
| `--memory`, `--cpus` | `resources.limits` |
| `--restart` | `restartPolicy`, Deployment |
| HEALTHCHECK | `livenessProbe`, `readinessProbe` |
| `--cap-drop` | `securityContext.capabilities` |
| `--read-only` | `securityContext.readOnlyRootFilesystem` |
| `--user` | `securityContext.runAsUser` |
| Named volumes | PersistentVolumeClaim |
| Networks | NetworkPolicy, Services |
| Secrets | Kubernetes Secrets |
| Log driver | Node-level logging |

---

## ✅ Final Checklist Summary

### Before Building

- [ ] Minimal base image selected
- [ ] .dockerignore configured
- [ ] Multi-stage build implemented
- [ ] Non-root user configured
- [ ] Health check defined

### Before Pushing

- [ ] Image scanned for vulnerabilities
- [ ] No secrets in image
- [ ] Proper tagging applied
- [ ] Image signed (if required)

### Before Deploying

- [ ] Resource limits set
- [ ] Security options configured
- [ ] Logging configured
- [ ] Health checks working
- [ ] Rollback plan ready

### In Production

- [ ] Monitoring active
- [ ] Alerting configured
- [ ] Log aggregation working
- [ ] Backup strategy implemented
- [ ] Update process defined

---

*🎉 Congratulations! You've completed the Docker curriculum. You're now ready for Kubernetes!*

