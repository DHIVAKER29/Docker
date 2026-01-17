# Chapter 7: Multi-Stage Builds & Image Optimization

> Creating production-ready, optimized Docker images

---

## 🎯 Learning Objectives

- Understand why images get bloated
- Use multi-stage builds to reduce size
- Apply security best practices
- Build different targets from one Dockerfile

---

## 🤔 The Problem: Bloated Images

Single-stage builds include everything:

```
Final Image (1.2 GB!)
├── Your compiled app (5 MB)       ✅ NEEDED
├── Source code (50 MB)            ❌ NOT NEEDED
├── Build tools (200 MB)           ❌ NOT NEEDED
├── Dev dependencies (400 MB)      ❌ NOT NEEDED
├── npm/pip cache (300 MB)         ❌ NOT NEEDED
└── Git history (50 MB)            ❌ NOT NEEDED
```

**Problems:**
- Slow push/pull
- Wastes storage
- Security risk (larger attack surface)

---

## ✅ The Solution: Multi-Stage Builds

Use multiple `FROM` statements - copy only what you need!

```
STAGE 1: Build               STAGE 2: Production
(full tools)                  (minimal)
┌─────────────────┐          ┌─────────────────┐
│ Source code     │   COPY   │ ✅ Compiled app │
│ Build tools     │ ───────▶ │ ✅ Runtime deps │
│ All deps        │  (only   │                 │
│ Cache           │   app!)  │ Total: 126 MB   │
│                 │          │                 │
│ Total: 1.14 GB  │          │ (90% smaller!)  │
└─────────────────┘          └─────────────────┘
        │
        └── DISCARDED!
```

---

## 📝 Multi-Stage Dockerfile Syntax

```dockerfile
# ============================================
# STAGE 1: Build
# ============================================
FROM node:18 AS builder
#            ^^^^^^^^^
#            Name this stage

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# ============================================
# STAGE 2: Production
# ============================================
FROM node:18-alpine AS production
#    ^^^^^^^^^^^^^^
#    Smaller base!

WORKDIR /app

# Copy ONLY from builder stage
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

CMD ["node", "dist/server.js"]
```

---

## 🔑 Key Syntax

### Naming Stages

```dockerfile
FROM node:18 AS builder
FROM node:18 AS tester
FROM node:18-alpine AS production
```

### Copying from Stages

```dockerfile
# Copy from named stage
COPY --from=builder /app/dist ./dist

# Copy from external image
COPY --from=nginx:alpine /etc/nginx/nginx.conf /etc/nginx/
```

### Building Specific Targets

```bash
# Build only up to a specific stage
docker build --target builder -t myapp:builder .
docker build --target production -t myapp:prod .
```

---

## 📚 Common Patterns

### Pattern 1: Build + Production

```dockerfile
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

### Pattern 2: Dev + Test + Prod

```dockerfile
FROM node:18-alpine AS base
WORKDIR /app
COPY package*.json ./

FROM base AS development
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]

FROM base AS test
RUN npm ci
COPY . .
CMD ["npm", "test"]

FROM base AS production
RUN npm ci --only=production
COPY . .
CMD ["npm", "start"]
```

### Pattern 3: Binary Apps (Go/Rust)

```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server

FROM scratch
COPY --from=builder /app/server /
ENTRYPOINT ["/server"]
```

**Result: ~5-10 MB image!**

---

## 🔒 Security Best Practices

### Don't Run as Root

```dockerfile
FROM node:18-alpine

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app
COPY --chown=nodejs:nodejs . .

# Switch to non-root
USER nodejs

CMD ["node", "app.js"]
```

### Use Minimal Base Images

| Image | Size |
|-------|------|
| node:18 | ~1 GB |
| node:18-slim | ~200 MB |
| node:18-alpine | ~130 MB |
| scratch | 0 MB (empty) |

### No Secrets in Images

```dockerfile
# ❌ BAD - Secret in image
COPY .env /app/.env
RUN echo $API_KEY

# ✅ GOOD - Pass at runtime
CMD ["node", "app.js"]
# docker run -e API_KEY=secret myapp
```

---

## 📊 Size Comparison Example

| Build Type | Image Size | Comparison |
|------------|------------|------------|
| Single-stage (node:18) | 1.14 GB | 100% |
| Multi-stage (node:18-alpine) | 126 MB | 11% |
| Go binary (scratch) | ~10 MB | <1% |

---

## ✅ Best Practices Summary

```
1. NAME YOUR STAGES
   FROM node:18 AS builder

2. USE SMALL BASE FOR FINAL
   FROM node:18-alpine
   FROM python:3.11-slim
   FROM scratch (for binaries)

3. COPY ONLY WHAT YOU NEED
   COPY --from=builder /app/dist ./dist

4. DON'T RUN AS ROOT
   USER nodejs

5. USE .dockerignore
   node_modules/
   .git/
   *.md

6. MULTI-TARGET FOR DEV/TEST/PROD
   docker build --target production .
```

---

## 📝 Key Takeaways

1. **Multi-stage** = Multiple FROM statements
2. **Copy selectively** = Only production artifacts
3. **Use Alpine/slim** = Smaller base images
4. **Non-root user** = Security best practice
5. **Named stages** = Easier to reference
6. **Build targets** = Different images from one Dockerfile

---

## 🔗 Navigation

[← Chapter 6: Docker Compose](./06-Docker-Compose.md) | [Chapter 8: Docker Security →](./08-Docker-Security.md)

