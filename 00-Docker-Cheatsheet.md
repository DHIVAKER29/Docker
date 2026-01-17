# 🐳 Docker Cheat Sheet

> Quick reference for all Docker commands

---

## 📦 Container Commands

```bash
# Run container
docker run <image>                    # Basic run
docker run -d <image>                 # Detached (background)
docker run -it <image> sh             # Interactive shell
docker run -p 8080:80 <image>         # Port mapping
docker run -v data:/app <image>       # With volume
docker run --name myapp <image>       # Named container
docker run --rm <image>               # Remove after exit
docker run -e VAR=value <image>       # With env variable
docker run --network net <image>      # On specific network

# Manage containers
docker ps                             # List running
docker ps -a                          # List all
docker stop <container>               # Stop
docker start <container>              # Start stopped
docker restart <container>            # Restart
docker rm <container>                 # Remove
docker rm -f <container>              # Force remove

# Inspect containers
docker logs <container>               # View logs
docker logs -f <container>            # Follow logs
docker exec -it <container> sh        # Shell into container
docker inspect <container>            # Detailed info
docker stats                          # Resource usage
docker top <container>                # Running processes
```

---

## 🖼️ Image Commands

```bash
docker images                         # List images
docker pull <image>                   # Download image
docker push <image>                   # Upload image
docker build -t name:tag .            # Build image
docker rmi <image>                    # Remove image
docker image prune                    # Remove unused
docker tag <image> <new-tag>          # Tag image
docker history <image>                # Show layers
docker save -o file.tar <image>       # Export to file
docker load -i file.tar               # Import from file
```

---

## 💾 Volume Commands

```bash
docker volume create <name>           # Create volume
docker volume ls                      # List volumes
docker volume inspect <name>          # Detailed info
docker volume rm <name>               # Remove volume
docker volume prune                   # Remove unused

# Using volumes
-v volume:/path                       # Named volume
-v /host/path:/container/path         # Bind mount
-v /path:/path:ro                     # Read-only
--tmpfs /path                         # tmpfs mount
```

---

## 🌐 Network Commands

```bash
docker network create <name>          # Create network
docker network ls                     # List networks
docker network inspect <name>         # Detailed info
docker network rm <name>              # Remove network
docker network prune                  # Remove unused
docker network connect <net> <cont>   # Connect container
docker network disconnect <net> <con> # Disconnect
```

---

## 🧹 Cleanup Commands

```bash
docker system prune                   # Remove all unused
docker system prune -a                # Remove ALL unused (including images)
docker container prune                # Remove stopped containers
docker image prune                    # Remove dangling images
docker volume prune                   # Remove unused volumes
docker network prune                  # Remove unused networks
```

---

## 📝 Dockerfile Instructions

```dockerfile
FROM image:tag          # Base image
WORKDIR /app            # Working directory
COPY src dest           # Copy files
ADD src dest            # Copy + extract
RUN command             # Execute during build
ENV KEY=value           # Environment variable
ARG KEY=value           # Build argument
EXPOSE port             # Document port
CMD ["cmd", "arg"]      # Default command
ENTRYPOINT ["cmd"]      # Main executable
USER username           # Run as user
VOLUME /path            # Define mount point
HEALTHCHECK CMD ...     # Health check
```

---

## 🐙 Docker Compose Commands

```bash
docker compose up                     # Start services
docker compose up -d                  # Start detached
docker compose up --build             # Rebuild and start
docker compose down                   # Stop and remove
docker compose down -v                # Also remove volumes
docker compose ps                     # List services
docker compose logs                   # View logs
docker compose logs -f <service>      # Follow service logs
docker compose exec <service> sh      # Shell into service
docker compose restart <service>      # Restart service
docker compose stop                   # Stop (keep containers)
docker compose start                  # Start stopped
docker compose config                 # Validate file
```

---

## 📋 docker-compose.yml Structure

```yaml
services:
  service-name:
    image: image:tag
    build: ./path
    ports:
      - "host:container"
    volumes:
      - volume:/path
    environment:
      - KEY=value
    env_file:
      - .env
    depends_on:
      - other-service
    networks:
      - network-name
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "localhost"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  volume-name:

networks:
  network-name:
```

---

## 🔍 Useful Patterns

### Run and Remove

```bash
docker run --rm -it alpine sh
```

### One-liner MySQL

```bash
docker run -d --name mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=mydb \
  -v mysql-data:/var/lib/mysql \
  -p 3306:3306 \
  mysql:8.0
```

### One-liner Redis

```bash
docker run -d --name redis \
  -v redis-data:/data \
  redis:7-alpine
```

### One-liner PostgreSQL

```bash
docker run -d --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=mydb \
  -v pg-data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:15-alpine
```

### Check Container Health

```bash
docker inspect --format='{{.State.Health.Status}}' <container>
```

### Copy Files From Container

```bash
docker cp <container>:/path/file ./local/path
```

### View Container IP

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container>
```

---

## 🎯 Pro Tips

1. **Use Alpine images** - Much smaller
2. **Copy package.json first** - Better layer caching
3. **Use .dockerignore** - Exclude unnecessary files
4. **Use specific tags** - Not `latest`
5. **Don't run as root** - Use `USER` instruction
6. **Use multi-stage builds** - Smaller production images
7. **Health checks** - Monitor container health
8. **Read-only mounts** - For config files (`:ro`)

