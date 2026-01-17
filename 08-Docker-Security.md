# Chapter 8: Docker Security Best Practices

> Building and running secure containers

---

## 🎯 Learning Objectives

- Run containers as non-root users
- Scan images for vulnerabilities
- Manage secrets properly
- Use read-only filesystems
- Apply security best practices

---

## 🔐 Security Risks Overview

| Risk | Impact | Solution |
|------|--------|----------|
| Running as root | Container escape, full control | Use non-root USER |
| Vulnerable images | Exploitable CVEs | Scan images regularly |
| Secrets in images | Credential exposure | Runtime secrets |
| Excessive permissions | Privilege escalation | Drop capabilities |
| Writable filesystem | Malware installation | Read-only mount |

---

## 1️⃣ Don't Run as Root

### Create Non-Root User

```dockerfile
FROM node:18-alpine

# Create non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

WORKDIR /app
COPY --chown=appuser:appgroup . .

# Switch to non-root
USER appuser

CMD ["node", "app.js"]
```

### Verify

```bash
docker run --rm myapp whoami
# Output: appuser

docker run --rm myapp id
# Output: uid=1001(appuser) gid=1001(appgroup)
```

---

## 2️⃣ Scan Images for Vulnerabilities

### Docker Scout

```bash
docker scout cves nginx:latest
docker scout quickview nginx:latest
```

### Trivy (Popular Open Source)

```bash
# Install
brew install trivy

# Basic scan
trivy image nginx:latest

# Only HIGH and CRITICAL
trivy image --severity HIGH,CRITICAL nginx:latest

# Fail on critical (for CI/CD)
trivy image --exit-code 1 --severity CRITICAL myapp
```

---

## 3️⃣ Secrets Management

### ❌ BAD - Secrets in Dockerfile

```dockerfile
# NEVER DO THIS!
ENV API_KEY=sk-secret-key
COPY .env /app/.env
```

### ✅ GOOD - Runtime Secrets

```bash
# Pass via environment
docker run -e API_KEY=secret myapp

# Use env file
docker run --env-file .env.local myapp
```

### ✅ BETTER - Docker Secrets

```yaml
# docker-compose.yml
services:
  api:
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

```javascript
// In app - secrets as files
const password = fs.readFileSync('/run/secrets/db_password', 'utf8');
```

---

## 4️⃣ Read-Only Filesystem

```bash
# Run read-only
docker run --read-only myapp

# With writable temp directories
docker run --read-only \
  --tmpfs /tmp \
  --tmpfs /var/run \
  myapp
```

### Docker Compose

```yaml
services:
  api:
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
```

---

## 5️⃣ Drop Capabilities

```bash
# Drop all, add only needed
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
```

### Common Capabilities

| Capability | Purpose |
|------------|---------|
| NET_BIND_SERVICE | Bind to ports < 1024 |
| CHOWN | Change file ownership |
| SETUID | Change user ID |

### Docker Compose

```yaml
services:
  api:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

---

## 6️⃣ Use Minimal Base Images

| Image Type | Size | Attack Surface |
|------------|------|----------------|
| ubuntu | ~77 MB | High |
| debian-slim | ~50 MB | Medium |
| alpine | ~5 MB | Low |
| distroless | ~20 MB | Very Low |
| scratch | 0 MB | None |

### Distroless Example

```dockerfile
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM gcr.io/distroless/nodejs18-debian11
COPY --from=builder /app/dist ./dist
CMD ["dist/server.js"]
```

### Go with Scratch

```dockerfile
FROM golang:1.21 AS builder
COPY . .
RUN CGO_ENABLED=0 go build -o server

FROM scratch
COPY --from=builder /app/server /
ENTRYPOINT ["/server"]
```

---

## 📋 Security Checklist

### Image Building

- [ ] Use specific image tags, not `:latest`
- [ ] Use minimal base images (Alpine, distroless)
- [ ] Create and use non-root user
- [ ] Don't store secrets in image
- [ ] Use multi-stage builds
- [ ] Scan images for vulnerabilities
- [ ] Use `.dockerignore`

### Container Runtime

- [ ] Run as non-root (USER instruction)
- [ ] Use read-only filesystem when possible
- [ ] Drop unnecessary capabilities
- [ ] Never run with `--privileged`
- [ ] Limit resources (CPU, memory)
- [ ] Use secrets management

### Networking

- [ ] Don't expose unnecessary ports
- [ ] Use internal networks for internal services
- [ ] Use TLS for external communication

---

## 📝 Secure Dockerfile Template

```dockerfile
FROM node:18.20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

FROM node:18.20-alpine

# Non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 -G nodejs

WORKDIR /app
COPY --from=builder --chown=nodejs:nodejs /app .

# Switch to non-root
USER nodejs

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s \
    CMD wget --spider http://localhost:3000/health || exit 1

CMD ["node", "app.js"]
```

---

## 🔧 Secure Docker Run

```bash
docker run -d \
  --name secure-app \
  --user 1001:1001 \
  --read-only \
  --tmpfs /tmp \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt=no-new-privileges:true \
  -p 3000:3000 \
  myapp
```

---

## 📝 Key Takeaways

1. **Non-root** - Always use USER instruction
2. **Scan images** - Before deployment
3. **No secrets in images** - Pass at runtime
4. **Minimal base** - Alpine or distroless
5. **Read-only** - When possible
6. **Drop capabilities** - Least privilege

---

## 🔗 Navigation

[← Chapter 7: Multi-Stage Builds](./07-Multi-Stage-Builds.md) | [Chapter 9: Resource Management →](./09-Resource-Management.md)

