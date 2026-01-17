# Chapter 5: Docker Volumes & Persistent Data

> How to persist data beyond container lifecycle

---

## 🎯 Learning Objectives

- Understand why containers lose data
- Learn three types of Docker storage
- Use volumes for databases
- Use bind mounts for development
- Backup and restore volume data

---

## 🤔 The Problem: Containers Are Ephemeral

When a container is removed, **all data inside is lost**.

```
Day 1: MySQL container running
┌─────────────────────────────────────┐
│  MySQL Container                     │
│  /var/lib/mysql/                    │
│  ├── users.ibd (1000 users)         │
│  └── orders.ibd (50000 orders)      │
└─────────────────────────────────────┘

Day 2: Container removed
docker rm mysql-container

💥 ALL DATA GONE! 💥
```

**Problem for:**
- Databases (MySQL, PostgreSQL)
- User uploads (images, files)
- Application logs
- Any persistent data

---

## 📦 Three Types of Docker Storage

### 1. Volumes (Recommended)

```
docker run -v mydata:/app/data nginx
            ──────  ─────────
            volume  container
            name    path
```

- **Managed by Docker**
- Stored in `/var/lib/docker/volumes/`
- Best for: **Databases, persistent app data**
- Easy backup and migration
- Can be shared between containers

### 2. Bind Mounts

```
docker run -v /home/user/code:/app nginx
            ─────────────────  ────
            host path          container
            (absolute)         path
```

- **You control the location**
- Maps host directory to container
- Best for: **Development (live code sync)**
- Best for: **Config files**

### 3. tmpfs Mounts (Linux only)

```
docker run --tmpfs /app/temp nginx
```

- **Stored in memory (RAM)**
- Never written to disk
- Lost when container stops
- Best for: **Sensitive temp data**

---

## 🎨 Visual Comparison

```
            CONTAINER
       ┌─────────────────┐
       │  /app/data ─────┼────────▶ VOLUME (Docker-managed)
       │  /app/code ─────┼────────▶ BIND MOUNT (Your directory)
       │  /app/temp ─────┼────────▶ TMPFS (Memory/RAM)
       └─────────────────┘

VOLUME:      /var/lib/docker/volumes/mydata/
BIND MOUNT:  /home/user/code/
TMPFS:       (no disk, in RAM)
```

---

## 🔧 Volume Commands

```bash
# Create a volume
docker volume create my-volume

# List all volumes
docker volume ls

# Inspect a volume
docker volume inspect my-volume

# Remove a volume
docker volume rm my-volume

# Remove all unused volumes
docker volume prune

# Remove volume with container
docker rm -v my-container
```

---

## 🗄️ Using Volumes with Containers

### Named Volume

```bash
# Create and use volume
docker run -d \
  --name mysql \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8.0
```

### Anonymous Volume

```bash
# Docker generates random name
docker run -d -v /var/lib/mysql mysql:8.0
```

### Read-Only Volume

```bash
# Container can read but not write
docker run -v myconfig:/etc/config:ro nginx
```

---

## 📁 Bind Mounts for Development

Perfect for **live code sync** during development:

```bash
# Mount current directory to /app
docker run -v $(pwd):/app node:18

# Changes on host are instantly visible in container!
```

**Example: Node.js Development**

```bash
docker run -d \
  --name dev-server \
  -v $(pwd)/src:/app/src \
  -v $(pwd)/package.json:/app/package.json:ro \
  -p 3000:3000 \
  node:18 npm run dev
```

---

## 🗃️ Database Example (Critical!)

```bash
# 1. Create volume for MySQL data
docker volume create mysql-data

# 2. Start MySQL with volume
docker run -d \
  --name mysql \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8.0

# 3. Add some data...
docker exec -it mysql mysql -p
# INSERT INTO users VALUES (...)

# 4. Destroy the container
docker stop mysql
docker rm mysql
# Container is GONE!

# 5. Start NEW container with SAME volume
docker run -d \
  --name mysql-new \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8.0

# 6. Data is still there!
docker exec -it mysql-new mysql -p
# SELECT * FROM users; → Data preserved!
```

---

## 💾 Backup and Restore

### Backup a Volume

```bash
docker run --rm \
  -v mysql-data:/source:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/mysql-backup.tar.gz -C /source .
```

### Restore a Volume

```bash
docker run --rm \
  -v mysql-data:/target \
  -v $(pwd):/backup \
  alpine tar xzf /backup/mysql-backup.tar.gz -C /target
```

---

## 👥 Sharing Volumes Between Containers

```bash
# Writer container
docker run -d \
  --name writer \
  -v shared-data:/data \
  alpine sh -c "while true; do date >> /data/log.txt; sleep 5; done"

# Reader container (same volume!)
docker run --rm \
  -v shared-data:/data:ro \
  alpine cat /data/log.txt
```

---

## 📋 When to Use What?

| Use Case | Storage Type | Example |
|----------|--------------|---------|
| Database | Named Volume | `-v mysql-data:/var/lib/mysql` |
| Dev code | Bind Mount | `-v ./src:/app/src` |
| Config files | Bind Mount (ro) | `-v ./nginx.conf:/etc/nginx.conf:ro` |
| User uploads | Named Volume | `-v uploads:/app/uploads` |
| Temp/cache | tmpfs | `--tmpfs /tmp` |
| Secrets (dev) | tmpfs | `--tmpfs /run/secrets` |

---

## 📝 Key Takeaways

1. **Containers are ephemeral** - data lost when removed
2. **Volumes** = Docker-managed, best for databases
3. **Bind mounts** = Host directories, best for development
4. **tmpfs** = In-memory, for sensitive temp data
5. **:ro** = Read-only mount for security
6. **Data survives** container removal with volumes

---

## 🔗 Kubernetes Connection

| Docker | Kubernetes |
|--------|------------|
| Volume | PersistentVolume (PV) |
| Volume mount | PersistentVolumeClaim (PVC) |

---

## 🔗 Navigation

[← Chapter 4: Docker Networking](./04-Docker-Networking.md) | [Chapter 6: Docker Compose →](./06-Docker-Compose.md)

