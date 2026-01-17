# Chapter 3: Dockerfile Deep Dive

> Building your own Docker images

---

## 🎯 Learning Objectives

- Understand what a Dockerfile is
- Learn all Dockerfile instructions
- Build custom Docker images
- Understand image layers and caching
- Apply best practices

---

## 📋 What is a Dockerfile?

A **Dockerfile** is a text file with instructions to build a Docker image.

**Analogy**: Recipe for cooking
- Recipe (Dockerfile) → Prepared dish (Image) → Served meal (Container)

---

## 🔧 Dockerfile Instructions

### Complete Reference

| Instruction | Purpose | Example |
|-------------|---------|---------|
| `FROM` | Base image to start from | `FROM node:18-alpine` |
| `WORKDIR` | Set working directory | `WORKDIR /app` |
| `COPY` | Copy files from host to image | `COPY package.json .` |
| `ADD` | Copy + extract archives + URLs | `ADD app.tar.gz /app` |
| `RUN` | Execute commands during build | `RUN npm install` |
| `ENV` | Set environment variables | `ENV NODE_ENV=production` |
| `ARG` | Build-time variables | `ARG VERSION=1.0` |
| `EXPOSE` | Document exposed ports | `EXPOSE 3000` |
| `CMD` | Default command when starting | `CMD ["node", "app.js"]` |
| `ENTRYPOINT` | Main executable (rarely changed) | `ENTRYPOINT ["python"]` |
| `USER` | Set user to run as | `USER node` |
| `VOLUME` | Define mount points | `VOLUME /data` |
| `LABEL` | Add metadata | `LABEL version="1.0"` |
| `HEALTHCHECK` | Define health check | See example below |

---

## 📝 Example Dockerfile (Annotated)

```dockerfile
# =============================================================
# DOCKERFILE EXAMPLE - Node.js Application
# =============================================================

# STEP 1: Base image
# Use Alpine for smaller size (~5MB vs ~1GB for Ubuntu)
FROM node:18-alpine

# STEP 2: Set working directory
# All subsequent commands run from /app
WORKDIR /app

# STEP 3: Copy package.json first (for layer caching!)
# If package.json unchanged, npm install layer is cached
COPY package*.json ./

# STEP 4: Install dependencies
# RUN executes during BUILD time
RUN npm install --production

# STEP 5: Copy application code
# Done AFTER npm install for better caching
COPY . .

# STEP 6: Set environment variables
ENV NODE_ENV=production
ENV PORT=3000

# STEP 7: Document exposed port
# Note: Doesn't actually publish - just documentation
EXPOSE 3000

# STEP 8: Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD wget --spider http://localhost:3000/health || exit 1

# STEP 9: Command to run when container starts
# Use array format for proper signal handling
CMD ["node", "app.js"]
```

---

## 🏗️ Building an Image

```bash
# Basic build
docker build -t my-app .

# Build with tag
docker build -t my-app:v1.0 .

# Build with different Dockerfile
docker build -f Dockerfile.prod -t my-app:prod .

# Build with build arguments
docker build --build-arg VERSION=2.0 -t my-app .
```

**Command breakdown:**

```
docker build -t my-app:v1 .
            │  │         │
            │  │         └── Build context (current directory)
            │  └── Tag: name:version
            └── Tag flag
```

---

## 📦 Image Layers (IMPORTANT!)

Each instruction creates a **layer**. Layers are **cached** and **reused**.

```
┌─────────────────────────────────────────────────┐
│  Layer 7: CMD ["node", "app.js"]                │ ← Startup
├─────────────────────────────────────────────────┤
│  Layer 6: COPY . .                              │ ← Your code
├─────────────────────────────────────────────────┤
│  Layer 5: RUN npm install                       │ ← Dependencies
├─────────────────────────────────────────────────┤
│  Layer 4: COPY package.json .                   │ ← Package info
├─────────────────────────────────────────────────┤
│  Layer 3: WORKDIR /app                          │ ← Directory
├─────────────────────────────────────────────────┤
│  Layer 2: node:18-alpine                        │ ← Base image
└─────────────────────────────────────────────────┘
```

### How Caching Works

When you rebuild:
- **Unchanged layers** → Reused from cache ✅
- **Changed layer** → Rebuilt 🔄
- **All layers after** → Also rebuilt 🔄

**Example**: If only `app.js` changes:
- Layers 1-5: CACHED (fast!)
- Layer 6+: REBUILT

---

## 💡 Best Practices

### 1. Order Instructions by Change Frequency

```dockerfile
# ❌ BAD - Everything rebuilds when code changes
COPY . .
RUN npm install

# ✅ GOOD - npm install cached when only code changes
COPY package*.json ./
RUN npm install
COPY . .
```

### 2. Use Specific Tags

```dockerfile
# ❌ BAD - "latest" can change unexpectedly
FROM node:latest

# ✅ GOOD - Specific version
FROM node:18.20-alpine
```

### 3. Minimize Layers

```dockerfile
# ❌ BAD - Multiple RUN creates multiple layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git

# ✅ GOOD - Single RUN, single layer
RUN apt-get update && \
    apt-get install -y curl git && \
    rm -rf /var/lib/apt/lists/*
```

### 4. Use .dockerignore

Create `.dockerignore` file:

```
node_modules/
.git/
*.log
.env
README.md
Dockerfile
docker-compose.yml
```

### 5. Use Alpine Base Images

```dockerfile
# node:18         → ~1GB
# node:18-slim    → ~200MB
# node:18-alpine  → ~130MB  ← Best for production
```

---

## 🔄 CMD vs ENTRYPOINT

| Aspect | CMD | ENTRYPOINT |
|--------|-----|------------|
| Purpose | Default command/arguments | Main executable |
| Override | Easy: `docker run image newcmd` | Harder: `--entrypoint` |
| Use case | Command that might change | Fixed executable |

**Common pattern:**

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]

# Runs: python app.py
# Override: docker run myimage other.py → python other.py
```

---

## 📝 Key Takeaways

1. **Dockerfile** = Recipe to build images
2. **Layers** = Each instruction creates a cached layer
3. **Order matters** = Put frequently changing things last
4. **Use Alpine** = Smaller, more secure images
5. **.dockerignore** = Exclude unnecessary files

---

## 🔗 Navigation

[← Chapter 2: Docker Architecture](./02-Docker-Architecture.md) | [Chapter 4: Docker Networking →](./04-Docker-Networking.md)

