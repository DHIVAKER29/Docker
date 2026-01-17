# 🔧 Chapter 10: Docker Troubleshooting

> Master the art of debugging containers - the skill that separates beginners from experts.

---

## 🎯 Learning Objectives

- Debug container startup failures
- Understand common exit codes
- Analyze container logs effectively
- Use inspection and diagnostic commands
- Solve real-world troubleshooting scenarios

---

## 🔍 Troubleshooting Workflow

```
┌──────────────────────────────────────────────────────────────────────────┐
│              DOCKER TROUBLESHOOTING WORKFLOW                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  1. CHECK STATUS                                                         │
│     docker ps -a                                                         │
│     └── Is container running? What's the status?                        │
│                                                                           │
│  2. CHECK LOGS                                                           │
│     docker logs <container>                                              │
│     └── What errors are printed?                                        │
│                                                                           │
│  3. INSPECT CONTAINER                                                    │
│     docker inspect <container>                                           │
│     └── Configuration correct? Exit code?                               │
│                                                                           │
│  4. CHECK INSIDE CONTAINER                                               │
│     docker exec -it <container> sh                                       │
│     └── Files exist? Permissions correct?                               │
│                                                                           │
│  5. CHECK RESOURCES                                                      │
│     docker stats                                                         │
│     └── Memory/CPU issues?                                              │
│                                                                           │
│  6. CHECK NETWORKING                                                     │
│     docker network inspect                                               │
│     └── Network connectivity issues?                                    │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 📊 Understanding Exit Codes

Exit codes tell you **why** a container stopped:

| Exit Code | Meaning | Common Cause |
|-----------|---------|--------------|
| `0` | Success | Normal completion |
| `1` | Application error | Bug in code, missing file |
| `126` | Permission denied | Can't execute command |
| `127` | Command not found | Wrong CMD, missing binary |
| `137` | SIGKILL (128+9) | OOM killed, `docker kill` |
| `139` | SIGSEGV (128+11) | Segmentation fault |
| `143` | SIGTERM (128+15) | `docker stop` |

### Check Exit Code

```bash
# Check exit code
docker inspect <container> --format='{{.State.ExitCode}}'

# Check if OOM killed
docker inspect <container> --format='{{.State.OOMKilled}}'

# Full state info
docker inspect <container> --format='{{json .State}}' | jq
```

---

## 🛠️ Essential Debugging Commands

### 1. Container Status

```bash
# List all containers (including stopped)
docker ps -a

# Filter by status
docker ps -a --filter "status=exited"
docker ps -a --filter "status=running"

# Show last N containers
docker ps -a -n 5
```

### 2. Container Logs

```bash
# View logs
docker logs <container>

# Follow logs (live)
docker logs -f <container>

# Last N lines
docker logs --tail 100 <container>

# With timestamps
docker logs -t <container>

# Logs since time
docker logs --since 1h <container>
docker logs --since 2024-01-01T00:00:00 <container>
```

### 3. Container Inspection

```bash
# Full inspection
docker inspect <container>

# Specific fields
docker inspect <container> --format='{{.State.Status}}'
docker inspect <container> --format='{{.Config.Cmd}}'
docker inspect <container> --format='{{.NetworkSettings.IPAddress}}'
docker inspect <container> --format='{{json .Config.Env}}'

# Mounts
docker inspect <container> --format='{{json .Mounts}}' | jq
```

### 4. Execute Commands in Container

```bash
# Interactive shell
docker exec -it <container> sh
docker exec -it <container> /bin/bash

# Run specific command
docker exec <container> ls -la /app
docker exec <container> cat /etc/hosts
docker exec <container> env

# As root (if running as non-root)
docker exec -u root <container> sh
```

### 5. Container Processes

```bash
# Show running processes
docker top <container>

# Resource usage
docker stats <container>
```

### 6. Copy Files

```bash
# Copy from container
docker cp <container>:/path/to/file ./local/path

# Copy to container
docker cp ./local/file <container>:/path/to/destination
```

---

## 🔥 Common Problems & Solutions

### Problem 1: Container Exits Immediately

**Symptom:**
```bash
docker run myapp
docker ps -a
# STATUS: Exited (0) 2 seconds ago
```

**Causes & Solutions:**

```bash
# 1. No foreground process
# BAD: Process runs in background
CMD ["nginx"]  # Nginx daemonizes by default

# GOOD: Run in foreground
CMD ["nginx", "-g", "daemon off;"]

# 2. Command completes and exits
# BAD: Command finishes
CMD ["echo", "hello"]

# GOOD: Keep running
CMD ["tail", "-f", "/dev/null"]  # For debugging
CMD ["node", "server.js"]        # Real app

# 3. Check logs for errors
docker logs <container>
```

---

### Problem 2: Container Exits with Code 1

**Symptom:** `Exited (1)` - Application error

**Debug Steps:**

```bash
# 1. Check logs
docker logs <container>

# 2. Run interactively to see error
docker run -it myapp

# 3. Override entrypoint to debug
docker run -it --entrypoint sh myapp

# 4. Check if files exist
docker run -it myapp ls -la /app

# 5. Check environment variables
docker run -it myapp env
```

---

### Problem 3: Container Exits with Code 137 (OOM Killed)

**Symptom:** `Exited (137)` - Killed

**Debug Steps:**

```bash
# 1. Confirm OOM kill
docker inspect <container> --format='{{.State.OOMKilled}}'
# Output: true

# 2. Check memory limit
docker inspect <container> --format='{{.HostConfig.Memory}}'

# 3. Solutions:
# a. Increase memory limit
docker run -m 1g myapp

# b. Fix memory leak in application

# c. Monitor usage
docker stats <container>
```

---

### Problem 4: Container Exits with Code 127

**Symptom:** `Exited (127)` - Command not found

**Debug Steps:**

```bash
# 1. Check the CMD in Dockerfile
docker inspect <container> --format='{{.Config.Cmd}}'

# 2. Check if binary exists
docker run -it --entrypoint sh myapp
# Then: which node, ls /usr/bin, etc.

# 3. Common causes:
# - Typo in command
# - Binary not installed
# - Wrong PATH
# - Using shell form vs exec form

# BAD (needs shell):
CMD node app.js

# GOOD (direct exec):
CMD ["node", "app.js"]
```

---

### Problem 5: Cannot Connect to Container

**Symptom:** Connection refused, timeout

**Debug Steps:**

```bash
# 1. Is container running?
docker ps

# 2. Is port mapped correctly?
docker port <container>

# 3. Is app listening on correct interface?
# BAD: Listening on localhost only
app.listen(3000, '127.0.0.1')

# GOOD: Listen on all interfaces
app.listen(3000, '0.0.0.0')

# 4. Check from inside container
docker exec <container> netstat -tlnp
docker exec <container> curl localhost:3000

# 5. Check network
docker network inspect bridge
```

---

### Problem 6: Permission Denied

**Symptom:** Permission denied errors

**Debug Steps:**

```bash
# 1. Check file permissions
docker exec <container> ls -la /app

# 2. Check what user container runs as
docker exec <container> whoami
docker exec <container> id

# 3. Solutions:
# a. Fix ownership in Dockerfile
COPY --chown=node:node . .

# b. Run as root (not recommended for prod)
docker run -u root myapp

# c. Fix volume permissions
docker run -v data:/app/data myapp
docker exec -u root <container> chown -R node:node /app/data
```

---

### Problem 7: Volume Data Not Persisting

**Symptom:** Data disappears after container restart

**Debug Steps:**

```bash
# 1. Check if volume is mounted
docker inspect <container> --format='{{json .Mounts}}' | jq

# 2. Using correct path?
# Volume must match where app writes data
-v mydata:/var/lib/mysql  # Correct path for MySQL

# 3. Named volume vs bind mount
# Named volume: -v mydata:/data  ✅ Persists
# Anonymous: -v /data            ❌ May not persist

# 4. Check volume exists
docker volume ls
docker volume inspect mydata
```

---

## 🧰 Useful Debugging One-Liners

```bash
# 1. Show all exited containers with their exit codes
docker ps -a --filter "status=exited" --format "table {{.Names}}\t{{.Status}}\t{{.Image}}"

# 2. Find containers that were OOM killed
docker ps -aq | xargs docker inspect --format "{{.Name}}: OOM={{.State.OOMKilled}}" 2>/dev/null | grep "OOM=true"

# 3. Get IP of running container
docker inspect <container> --format "{{.NetworkSettings.IPAddress}}"

# 4. Show container environment variables
docker exec <container> env

# 5. Check if port is listening inside container
docker exec <container> netstat -tlnp

# 6. Real-time log following with grep
docker logs -f <container> 2>&1 | grep -i error

# 7. Export container filesystem for analysis
docker export <container> > container.tar

# 8. See what changed in container (diff)
docker diff <container>

# 9. Check Docker daemon logs (macOS with Colima)
colima ssh -- sudo journalctl -u docker -n 50

# 10. Force remove everything (nuclear option ⚠️)
docker system prune -af --volumes
```

---

## 📋 Troubleshooting Checklist

```
┌──────────────────────────────────────────────────────────────────────────┐
│              DOCKER TROUBLESHOOTING CHECKLIST                             │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  □ Container won't start?                                                │
│    → Check docker logs                                                   │
│    → Verify CMD/ENTRYPOINT                                              │
│    → Check if image exists                                              │
│                                                                           │
│  □ Container exits immediately?                                          │
│    → Check exit code                                                     │
│    → Ensure foreground process                                          │
│    → Check for startup errors in logs                                   │
│                                                                           │
│  □ Container crashes (137)?                                              │
│    → Check OOMKilled status                                             │
│    → Increase memory limit                                              │
│    → Profile memory usage                                               │
│                                                                           │
│  □ Cannot connect to container?                                          │
│    → Verify port mapping                                                │
│    → Check app binds to 0.0.0.0                                        │
│    → Check network connectivity                                         │
│                                                                           │
│  □ Permission denied?                                                    │
│    → Check file ownership                                               │
│    → Verify user running container                                      │
│    → Check volume mount permissions                                     │
│                                                                           │
│  □ Data not persisting?                                                  │
│    → Verify volume mounted correctly                                    │
│    → Check volume path matches app                                      │
│    → Use named volumes                                                  │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 🔗 Mapping to Kubernetes

| Docker Troubleshooting | Kubernetes Equivalent |
|------------------------|----------------------|
| `docker logs` | `kubectl logs` |
| `docker exec` | `kubectl exec` |
| `docker inspect` | `kubectl describe` + `kubectl get -o yaml` |
| `docker ps -a` | `kubectl get pods` |
| Exit code 137 (OOM) | Pod `OOMKilled` status |
| Container restart | Pod `CrashLoopBackOff` |
| Network issues | Service/Ingress/NetworkPolicy |
| Volume issues | PV/PVC binding issues |

---

## ✅ Summary: Key Commands

| Command | Purpose |
|---------|---------|
| `docker ps -a` | See all containers (including stopped) |
| `docker logs <container>` | View container output |
| `docker logs -f <container>` | Follow logs in real-time |
| `docker inspect <container>` | Full container details |
| `docker exec <container> sh` | Access container shell |
| `docker top <container>` | View running processes |
| `docker stats` | Resource usage |
| `docker cp` | Copy files to/from container |

### Exit Code Quick Reference

| Code | Meaning |
|------|---------|
| 0 | Success (normal exit) |
| 1 | Application error |
| 127 | Command not found |
| 137 | OOM killed / SIGKILL |
| 143 | SIGTERM (docker stop) |

---

*Next Chapter: Kubernetes Fundamentals! 🚀*

