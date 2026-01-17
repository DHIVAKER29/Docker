# Chapter 2: Docker Architecture

> Understanding Docker components and running your first container

---

## 🎯 Learning Objectives

- Understand Docker's client-server architecture
- Learn the difference between images and containers
- Run your first container
- Master essential Docker commands

---

## 🏗️ Docker Architecture

Docker uses a **client-server** architecture with three main components:

```
┌─────────────────┐         ┌─────────────────────────────────────┐
│  DOCKER CLIENT  │         │         DOCKER HOST                  │
│  (docker CLI)   │         │                                      │
│                 │         │  ┌─────────────────────────────────┐ │
│  You type:      │   REST  │  │      DOCKER DAEMON (dockerd)    │ │
│  docker run     │───API──▶│  │                                 │ │
│  docker build   │         │  │  Manages:                       │ │
│  docker pull    │         │  │  • Images                       │ │
│                 │         │  │  • Containers                   │ │
└─────────────────┘         │  │  • Networks                     │ │
                            │  │  • Volumes                      │ │
                            │  └─────────────────────────────────┘ │
                            └─────────────────────────────────────┘
                                           │
                                           │ pull/push
                                           ▼
                            ┌─────────────────────────────────────┐
                            │        DOCKER REGISTRY              │
                            │        (Docker Hub)                 │
                            └─────────────────────────────────────┘
```

### 1. Docker Client

- Command-line tool (`docker` command)
- Sends commands to Docker daemon
- Can connect to local or remote daemon

### 2. Docker Daemon (dockerd)

- Background service doing the actual work
- Builds images, runs containers
- Manages all Docker objects

### 3. Docker Registry

- Storage for Docker images
- Docker Hub (public), ECR, GCR (private)

---

## 🖼️ Images vs Containers (CRITICAL CONCEPT!)

| Docker Image | Docker Container |
|--------------|------------------|
| 📋 Blueprint/Template | 🏃 Running Instance |
| 📀 Like a CD/DVD | ▶️ Like playing the CD |
| Read-only | Read-write |
| Stored on disk | Running process |
| ONE image | MANY containers from same image |

```
┌─────────────────┐         ┌─────────────────┐
│   nginx:latest  │── run ─▶│  Container 1    │
│     (IMAGE)     │         └─────────────────┘
│                 │── run ─▶│  Container 2    │
│                 │         └─────────────────┘
│                 │── run ─▶│  Container 3    │
└─────────────────┘         └─────────────────┘
```

---

## 🚀 Your First Container

### Run Hello World

```bash
docker run hello-world
```

**What happens:**

1. Docker looks for image locally → Not found
2. Downloads from Docker Hub
3. Creates container from image
4. Container runs and prints message
5. Container exits (job done)

### Run a Web Server

```bash
docker run -d -p 8080:80 --name my-nginx nginx
```

**Flags explained:**

| Flag | Meaning |
|------|---------|
| `-d` | Detached mode (run in background) |
| `-p 8080:80` | Map host port 8080 → container port 80 |
| `--name my-nginx` | Give container a name |
| `nginx` | Image to run |

---

## 📝 Essential Docker Commands

### Image Commands

```bash
# List all images
docker images

# Download an image
docker pull nginx

# Remove an image
docker rmi nginx

# Remove unused images
docker image prune
```

### Container Commands

```bash
# List running containers
docker ps

# List ALL containers (including stopped)
docker ps -a

# Run a container
docker run nginx

# Run in background
docker run -d nginx

# Stop a container
docker stop <container_name>

# Start a stopped container
docker start <container_name>

# Remove a container
docker rm <container_name>

# View container logs
docker logs <container_name>

# Execute command in running container
docker exec -it <container_name> sh
```

### Cleanup Commands

```bash
# Remove all stopped containers
docker container prune

# Remove all unused images
docker image prune

# Remove everything unused
docker system prune
```

---

## 🔍 Inspecting Containers

```bash
# Detailed container info
docker inspect <container_name>

# Live resource usage
docker stats

# Processes running in container
docker top <container_name>

# View logs (follow mode)
docker logs -f <container_name>
```

---

## 🌐 Port Mapping Explained

```
-p HOST_PORT:CONTAINER_PORT

Example: -p 8080:80

┌─────────────────┐          ┌─────────────────┐
│   YOUR LAPTOP   │          │    CONTAINER    │
│                 │          │                 │
│  Browser ───────┼──8080───▶│──────▶ 80      │
│                 │          │    (nginx)      │
└─────────────────┘          └─────────────────┘

Access: http://localhost:8080
```

---

## 📝 Key Takeaways

1. **Client-Server**: Docker CLI talks to Docker daemon via API
2. **Images**: Read-only templates (like a recipe)
3. **Containers**: Running instances of images (like a cooked dish)
4. **Registry**: Where images are stored (Docker Hub)
5. **Port Mapping**: `-p` connects host ports to container ports

---

## 🔗 Navigation

[← Chapter 1: Why Containers?](./01-Why-Containers.md) | [Chapter 3: Dockerfile →](./03-Dockerfile.md)

