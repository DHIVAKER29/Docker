# Chapter 9: Docker Resource Management

> CPU and memory limits for containers

---

## 🎯 Learning Objectives

- Set CPU and memory limits
- Monitor container resource usage
- Understand requests vs limits
- Connect to Kubernetes resource concepts

---

## 🤔 The Problem

Without limits, one container can consume ALL host resources:

```
Container A (memory leak): ████████████████████████ 14 GB
Container B: 💀 (killed - no memory)
Container C: 💀 (killed - no memory)

Result: All apps down because of one bad container!
```

---

## 💾 Memory Limits

### Commands

```bash
# Hard limit (512MB max)
docker run -m 512m myapp
docker run --memory 512m myapp

# With swap
docker run -m 512m --memory-swap 1g myapp

# No swap
docker run -m 512m --memory-swap 512m myapp
```

### Memory Options

| Flag | Description |
|------|-------------|
| `-m` / `--memory` | Hard memory limit |
| `--memory-swap` | Memory + swap total |
| `--memory-reservation` | Soft limit (hint) |

### What Happens at Limit?

- Container process killed (OOM)
- Exit code: 137 (128 + 9 = SIGKILL)
- Check: `docker inspect --format='{{.State.OOMKilled}}'`

---

## 🔄 CPU Limits

### CPU Options

```bash
# Limit to 0.5 CPU
docker run --cpus 0.5 myapp

# Limit to 2 CPUs
docker run --cpus 2 myapp

# Relative weight (default 1024)
docker run --cpu-shares 512 myapp

# Pin to specific CPUs
docker run --cpuset-cpus "0,1" myapp
```

### CPU Options Summary

| Flag | Description | Example |
|------|-------------|---------|
| `--cpus` | Number of CPUs | `--cpus 0.5` |
| `--cpu-shares` | Relative weight | `--cpu-shares 512` |
| `--cpuset-cpus` | Pin to CPUs | `--cpuset-cpus "0-3"` |

---

## 📊 Monitoring Resources

### Docker Stats

```bash
# Live stats (like top)
docker stats

# Single snapshot
docker stats --no-stream

# Specific container
docker stats mycontainer
```

### Output Example

```
NAME     CPU %    MEM USAGE / LIMIT    MEM %
api      0.50%    128MiB / 512MiB      25%
mysql    2.34%    456MiB / 1GiB        44%
redis    0.10%    12MiB / 256MiB       5%
```

### Inspect Limits

```bash
docker inspect container --format '{{.HostConfig.Memory}}'
docker inspect container --format '{{.HostConfig.NanoCpus}}'
```

---

## 📝 Docker Compose

```yaml
services:
  api:
    image: myapp
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

### Alternative Syntax

```yaml
services:
  api:
    image: myapp
    mem_limit: 512m
    mem_reservation: 256m
    cpus: 0.5
```

---

## 🔗 Kubernetes Mapping

| Docker | Kubernetes |
|--------|------------|
| `--memory 512m` | `limits.memory: 512Mi` |
| `--cpus 0.5` | `limits.cpu: 500m` |
| `--memory-reservation` | `requests.memory` |
| `--cpu-shares` | `requests.cpu` |

### Kubernetes Pod Spec

```yaml
spec:
  containers:
  - name: api
    resources:
      requests:          # Guaranteed minimum
        memory: "256Mi"
        cpu: "250m"      # 0.25 CPU
      limits:            # Maximum allowed
        memory: "512Mi"
        cpu: "500m"      # 0.5 CPU
```

### CPU Units

| K8s | Meaning |
|-----|---------|
| `1` | 1 CPU |
| `500m` | 0.5 CPU |
| `100m` | 0.1 CPU |

### Memory Units

| K8s | Meaning |
|-----|---------|
| `256Mi` | 256 Mebibytes |
| `1Gi` | 1 Gibibyte |
| `512M` | 512 Megabytes |

---

## 📊 Requests vs Limits

```
REQUESTS: Guaranteed minimum (for scheduling)
LIMITS:   Maximum allowed

    requests     current      limits
        │           │            │
        ▼           ▼            ▼
├───────────────────────────────────────┤
│░░░░░░░│██████████████│                │
├───────────────────────────────────────┤
0      256Mi        384Mi            512Mi

Container using 384Mi:
✅ Above request (256Mi)
✅ Below limit (512Mi)
```

### Behavior

| Resource | At Limit |
|----------|----------|
| Memory | OOM Kill (hard limit) |
| CPU | Throttled (soft limit) |

---

## 💡 Best Practices

1. **Always set limits**
   - Prevents runaway containers
   - Required in Kubernetes

2. **Set realistic requests**
   - Too low → killed under load
   - Too high → waste resources

3. **limits ≥ requests**
   - Common ratio: limits = 2x requests

4. **Monitor and adjust**
   - Use `docker stats`
   - Right-size based on actual usage

5. **Be generous with memory**
   - Memory = hard limit (kill)
   - CPU = soft limit (throttle)

---

## 📋 Quick Reference

```bash
# Memory limit
docker run -m 512m myapp

# CPU limit
docker run --cpus 0.5 myapp

# Both
docker run -m 512m --cpus 0.5 myapp

# Monitor
docker stats

# Check limits
docker inspect container --format '{{.HostConfig.Memory}}'
```

---

## 📝 Key Takeaways

1. **Set limits** - Prevent resource exhaustion
2. **Monitor usage** - `docker stats`
3. **Memory = hard** - Container killed if exceeded
4. **CPU = soft** - Container throttled if exceeded
5. **Kubernetes** - Same concepts, different syntax

---

## 🔗 Navigation

[← Chapter 8: Docker Security](./08-Docker-Security.md) | [Chapter 10: Troubleshooting →](./10-Docker-Troubleshooting.md)

