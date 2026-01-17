# Chapter 4: Docker Networking

> How containers communicate with each other

---

## 🎯 Learning Objectives

- Understand Docker network types
- Learn how containers communicate
- Master port mapping vs internal access
- Connect multiple containers together
- Understand DNS resolution in Docker

---

## 🤔 The Problem: Container Isolation

By default, containers are **isolated** - they can't see each other.

```
Container A                    Container B
┌─────────────┐               ┌─────────────┐
│   API       │  ── ❌ ──     │   MySQL     │
│             │  Can't        │             │
│             │  connect!     │             │
└─────────────┘               └─────────────┘
```

**Solution**: Docker Networks connect containers together.

---

## 🌐 Docker Network Types

| Driver | Description | Use Case |
|--------|-------------|----------|
| **bridge** | Default. Private network on host | Most common |
| **host** | Uses host's network directly | Maximum performance |
| **none** | No networking | Security-critical jobs |
| **overlay** | Multi-host networking | Kubernetes, Swarm |

### Bridge Network (Default)

```
┌──────────────────────────────────────────┐
│         BRIDGE NETWORK                    │
│  ┌──────┐  ┌──────┐  ┌──────┐           │
│  │ API  │  │ MySQL│  │ Redis│           │
│  └──┬───┘  └──┬───┘  └──┬───┘           │
│     └─────────┼─────────┘                │
│           ┌───▼───┐                      │
│           │bridge0│ (172.17.0.x)         │
│           └───────┘                      │
└──────────────────────────────────────────┘
```

---

## 🔧 Network Commands

```bash
# List networks
docker network ls

# Create a network
docker network create my-network

# Inspect a network
docker network inspect my-network

# Connect container to network
docker network connect my-network my-container

# Disconnect from network
docker network disconnect my-network my-container

# Remove a network
docker network rm my-network

# Remove all unused networks
docker network prune
```

---

## 🚀 Running Containers on a Network

```bash
# Create network
docker network create app-network

# Run MySQL on the network
docker run -d \
  --name mysql \
  --network app-network \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8.0

# Run API on the same network
docker run -d \
  --name api \
  --network app-network \
  -e DB_HOST=mysql \
  my-api

# API can now connect to "mysql:3306"!
```

---

## 🔮 DNS Resolution (The Magic!)

When containers are on the **same network**, Docker provides automatic DNS:

```
Container Name = Hostname

api container connects to:
  - mysql:3306   ← Docker resolves "mysql" to its IP
  - redis:6379   ← Docker resolves "redis" to its IP
```

**This is exactly how Kubernetes works!**

```bash
# Proof: From inside api container
docker exec api ping mysql
# Output: PING mysql (172.18.0.2): 56 data bytes
```

---

## 🚪 Port Mapping vs Internal Access

### Two Different Worlds

```
┌──────────────────────────────────────────────────────────┐
│  YOUR LAPTOP (Outside World)                              │
│                                                           │
│  localhost:8080  ─────┐                                  │
│  localhost:3306  ─────┼─── Can only access if MAPPED     │
│                       │                                  │
├═══════════════════════╪══════════════════════════════════┤
│  DOCKER NETWORK       │ (Inside World)                   │
│                       ▼                                  │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐    │
│  │    API      │   │   MySQL     │   │   Redis     │    │
│  │  port 3000  │──▶│  port 3306  │   │  port 6379  │    │
│  │  MAPPED     │   │  INTERNAL   │   │  INTERNAL   │    │
│  └─────────────┘   └─────────────┘   └─────────────┘    │
│                                                           │
│  Containers can talk to each other freely!               │
└──────────────────────────────────────────────────────────┘
```

### Comparison

| Scenario | Port Mapped (`-p`) | No Port Mapping |
|----------|-------------------|-----------------|
| Access from laptop | ✅ Yes | ❌ No |
| Access from same network container | ✅ Yes | ✅ Yes |
| Access from different network | ❌ No | ❌ No |
| Security | Exposed | Protected |

### When to Use What?

| Service Type | Port Mapping? | Why? |
|--------------|---------------|------|
| API/Web Server | ✅ Yes | Users need access |
| Database | ❌ No | Internal only (security) |
| Cache (Redis) | ❌ No | Internal only |
| Admin Tools | Maybe | Only for debugging |

---

## 📊 Network Isolation

Different networks **cannot communicate** (security feature):

```bash
# Container on network-a
docker run --network network-a --name app-a alpine

# Container on network-b
docker run --network network-b --name app-b alpine

# app-a CANNOT reach app-b!
docker exec app-a ping app-b
# Error: bad address 'app-b'
```

---

## 🔌 Connecting Existing Container to Network

```bash
# Connect to additional network
docker network connect my-network existing-container

# Container is now on BOTH networks
docker inspect existing-container | grep Networks
```

---

## 📝 Practical Example: API + MySQL + Redis

```bash
# 1. Create network
docker network create app-network

# 2. Start MySQL (no port mapping - internal only!)
docker run -d \
  --name mysql \
  --network app-network \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=myapp \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0

# 3. Start Redis (no port mapping)
docker run -d \
  --name redis \
  --network app-network \
  redis:7-alpine

# 4. Start API (with port mapping for external access)
docker run -d \
  --name api \
  --network app-network \
  -p 3000:3000 \
  -e DB_HOST=mysql \
  -e REDIS_HOST=redis \
  my-api

# Result:
# - API accessible at localhost:3000
# - MySQL only accessible from containers
# - Redis only accessible from containers
```

---

## 📝 Key Takeaways

1. **Bridge network** = Default, containers on same host
2. **Same network** = Containers communicate via names
3. **DNS resolution** = Container name becomes hostname
4. **Port mapping** = Expose to outside world
5. **No port mapping** = Internal only (more secure)
6. **Different networks** = Isolated, can't communicate

---

## 🔗 Navigation

[← Chapter 3: Dockerfile](./03-Dockerfile.md) | [Chapter 5: Docker Volumes →](./05-Docker-Volumes.md)

