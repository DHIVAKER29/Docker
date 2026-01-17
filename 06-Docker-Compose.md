# Chapter 6: Docker Compose

> Running multi-container applications with ease

---

## 🎯 Learning Objectives

- Understand what Docker Compose is
- Write docker-compose.yml files
- Run multi-container applications
- Manage application stacks
- Connect Compose concepts to Kubernetes

---

## 🤔 The Problem: Too Many Commands!

Running a typical app requires multiple containers:

```bash
# Without Compose - TOO MANY COMMANDS!
docker network create app-network
docker run -d --name mysql --network app-network -v mysql-data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=secret mysql:8.0
docker run -d --name redis --network app-network redis:7-alpine
docker run -d --name api --network app-network -p 3000:3000 -e DB_HOST=mysql my-api
```

**Problems:**
- Too many commands to remember
- Easy to make mistakes
- Hard to share with team
- Starting/stopping is painful

---

## ✅ The Solution: Docker Compose

One YAML file describes everything:

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DB_HOST=mysql
    depends_on:
      - mysql
      - redis

  mysql:
    image: mysql:8.0
    volumes:
      - mysql-data:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=secret

  redis:
    image: redis:7-alpine

volumes:
  mysql-data:
```

```bash
# Start EVERYTHING with ONE command!
docker compose up -d

# Stop EVERYTHING with ONE command!
docker compose down
```

---

## 📋 Docker Compose File Structure

```yaml
# Version (optional in newer versions)
version: "3.8"

# Services (containers)
services:
  service-name:
    image: image:tag          # Use existing image
    build: ./path             # OR build from Dockerfile
    ports:                    # Port mapping
      - "host:container"
    volumes:                  # Data persistence
      - volume:/path
      - ./host:/container
    environment:              # Environment variables
      - KEY=value
    env_file:                 # Load from file
      - .env
    depends_on:               # Startup order
      - other-service
    networks:                 # Network connections
      - my-network
    restart: unless-stopped   # Restart policy
    command: custom command   # Override CMD
    healthcheck:              # Health monitoring
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3

# Named volumes
volumes:
  volume-name:

# Custom networks
networks:
  my-network:
    driver: bridge
```

---

## 🔧 Essential Compose Commands

### Starting/Stopping

```bash
# Start all services (detached)
docker compose up -d

# Start and rebuild images
docker compose up -d --build

# Stop and remove containers
docker compose down

# Stop, remove, AND delete volumes
docker compose down -v

# Stop (but keep containers)
docker compose stop

# Start stopped containers
docker compose start
```

### Viewing Status

```bash
# List running services
docker compose ps

# View all logs
docker compose logs

# Follow logs for specific service
docker compose logs -f api

# Show resource usage
docker compose top
```

### Service Management

```bash
# Restart a service
docker compose restart api

# Scale a service (multiple instances)
docker compose up -d --scale api=3

# Execute command in service
docker compose exec api sh

# Run one-off command
docker compose run --rm api npm test
```

### Debugging

```bash
# Validate compose file
docker compose config

# Show service configuration
docker compose config --services

# View images used
docker compose images
```

---

## 📝 Complete Example

### Project Structure

```
my-app/
├── docker-compose.yml
├── backend/
│   ├── Dockerfile
│   ├── app.js
│   └── package.json
└── frontend/
    ├── index.html
    └── nginx.conf
```

### docker-compose.yml

```yaml
services:
  # Frontend - Nginx
  frontend:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./frontend/index.html:/usr/share/nginx/html/index.html:ro
      - ./frontend/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - backend
    networks:
      - app-network

  # Backend - Node.js
  backend:
    build: ./backend
    environment:
      - NODE_ENV=production
      - DB_HOST=postgres
      - REDIS_HOST=redis
    depends_on:
      - postgres
      - redis
    networks:
      - app-network
    # No ports - accessed via frontend proxy

  # Database - PostgreSQL
  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=app
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=myapp
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  # Cache - Redis
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - app-network

volumes:
  postgres-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

---

## 🔌 Service Communication

Services communicate using **service names as hostnames**:

```yaml
services:
  api:
    environment:
      - DB_HOST=postgres    # ← Service name!
      - REDIS_HOST=redis    # ← Service name!
  
  postgres:
    # ...
  
  redis:
    # ...
```

```javascript
// In your code
const dbHost = process.env.DB_HOST; // "postgres"
// Connects to postgres:5432 automatically!
```

---

## ⏱️ depends_on and Health Checks

### Basic depends_on

```yaml
services:
  api:
    depends_on:
      - postgres  # Just waits for container to START
```

### With Health Check (Better!)

```yaml
services:
  api:
    depends_on:
      postgres:
        condition: service_healthy  # Waits until HEALTHY

  postgres:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s
      retries: 5
```

---

## 📂 Environment Variables

### Inline

```yaml
services:
  api:
    environment:
      - NODE_ENV=production
      - API_KEY=secret123
```

### From .env File

```yaml
services:
  api:
    env_file:
      - .env
```

**.env file:**
```
NODE_ENV=production
API_KEY=secret123
```

### Variable Substitution

```yaml
services:
  api:
    image: myapp:${VERSION:-latest}  # Default to "latest"
```

---

## 🔄 Restart Policies

| Policy | Behavior |
|--------|----------|
| `no` | Never restart (default) |
| `always` | Always restart |
| `on-failure` | Restart on error |
| `unless-stopped` | Restart unless manually stopped |

```yaml
services:
  api:
    restart: unless-stopped
```

---

## 📊 Compose to Kubernetes Mapping

| Docker Compose | Kubernetes |
|----------------|------------|
| service | Deployment + Service |
| ports | Service (NodePort/LoadBalancer) |
| volumes | PersistentVolumeClaim |
| environment | ConfigMap / Secret |
| depends_on | Init containers / Probes |
| networks | Network Policies |
| docker-compose.yml | Kubernetes manifests |

---

## 📝 Key Takeaways

1. **One file** describes entire application stack
2. **One command** to start/stop everything
3. **Service names** = hostnames for communication
4. **depends_on** controls startup order
5. **Health checks** ensure dependencies are ready
6. **Volumes** persist data across restarts
7. **Great stepping stone** to Kubernetes

---

## 🔗 Navigation

[← Chapter 5: Docker Volumes](./05-Docker-Volumes.md) | [Chapter 7: Multi-Stage Builds →](./07-Multi-Stage-Builds.md)

