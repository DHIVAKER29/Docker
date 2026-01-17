# 📊 Chapter 14: Docker Logging & Monitoring

> Understanding log drivers, centralized logging, and container monitoring.

---

## 🎯 Learning Objectives

- Understand Docker logging architecture
- Configure different log drivers
- Implement centralized logging
- Monitor container health and performance
- Use Docker events for observability

---

## 🪵 Docker Logging Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DOCKER LOGGING ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Container                                                               │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Application                                                      │  │
│  │      │                                                            │  │
│  │      ├── stdout ──┐                                              │  │
│  │      └── stderr ──┤                                              │  │
│  └───────────────────┼──────────────────────────────────────────────┘  │
│                      │                                                   │
│                      ▼                                                   │
│              ┌───────────────┐                                          │
│              │  Log Driver   │                                          │
│              └───────┬───────┘                                          │
│                      │                                                   │
│    ┌─────────────────┼─────────────────────────────┐                   │
│    ▼                 ▼                             ▼                    │
│  json-file       syslog                      Cloud Services            │
│  (default)       journald                    - AWS CloudWatch          │
│  local           fluentd                     - GCP Logging             │
│                  gelf                        - Splunk                  │
│                  awslogs                     - Datadog                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 📝 Log Drivers

### Available Log Drivers

| Driver | Description | Use Case |
|--------|-------------|----------|
| `json-file` | Default, JSON format | Local development |
| `local` | Optimized local storage | Local with rotation |
| `syslog` | System syslog | Linux servers |
| `journald` | Systemd journal | Systemd-based systems |
| `fluentd` | Fluentd collector | Centralized logging |
| `gelf` | Graylog Extended Log Format | Graylog |
| `awslogs` | AWS CloudWatch | AWS deployments |
| `gcplogs` | Google Cloud Logging | GCP deployments |
| `splunk` | Splunk HTTP Event Collector | Splunk |
| `none` | No logging | When you handle logs yourself |

### Configure Log Driver

```bash
# Per container
docker run --log-driver json-file --log-opt max-size=10m nginx

# Daemon default (docker daemon config)
# /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

---

## 📄 json-file Driver (Default)

The default driver stores logs as JSON files.

### Configuration Options

```bash
docker run \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=5 \
  --log-opt compress=true \
  --log-opt labels=production_status \
  --log-opt env=os,customer \
  nginx
```

| Option | Description | Default |
|--------|-------------|---------|
| `max-size` | Max size before rotation | -1 (unlimited) |
| `max-file` | Max number of log files | 1 |
| `compress` | Compress rotated files | disabled |
| `labels` | Include container labels | |
| `env` | Include environment vars | |

### View Log Files Directly

```bash
# Find log file location
docker inspect --format='{{.LogPath}}' mycontainer

# View raw log file
sudo cat /var/lib/docker/containers/<container-id>/<container-id>-json.log

# Log file format (JSON)
{"log":"Hello World\n","stream":"stdout","time":"2024-01-15T10:30:00.123456789Z"}
```

---

## 🌐 Centralized Logging with Fluentd

### Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   Container A ─┐                                                         │
│   Container B ─┼── fluentd driver ──> Fluentd ──> Elasticsearch         │
│   Container C ─┘                         │                               │
│                                          └──> Kibana (visualization)    │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### Docker Compose with Fluentd

```yaml
version: '3.8'

services:
  fluentd:
    image: fluentd:v1.16-1
    volumes:
      - ./fluentd/conf:/fluentd/etc
    ports:
      - "24224:24224"
      - "24224:24224/udp"

  webapp:
    image: myapp
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        fluentd-async: "true"
        tag: docker.{{.Name}}

  nginx:
    image: nginx
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: docker.nginx
```

### Fluentd Configuration

```xml
<!-- fluentd/conf/fluent.conf -->
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<match docker.**>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  logstash_prefix docker
</match>
```

---

## ☁️ AWS CloudWatch Logs

### Configure awslogs Driver

```bash
docker run \
  --log-driver awslogs \
  --log-opt awslogs-region=us-east-1 \
  --log-opt awslogs-group=my-log-group \
  --log-opt awslogs-stream=my-container \
  --log-opt awslogs-create-group=true \
  myapp
```

### Docker Compose with awslogs

```yaml
services:
  webapp:
    image: myapp
    logging:
      driver: awslogs
      options:
        awslogs-region: us-east-1
        awslogs-group: /ecs/myapp
        awslogs-stream-prefix: webapp
        awslogs-create-group: "true"
```

---

## 📈 Docker Logs Command

### Basic Usage

```bash
# View all logs
docker logs mycontainer

# Follow logs (live)
docker logs -f mycontainer

# Last N lines
docker logs --tail 100 mycontainer

# With timestamps
docker logs -t mycontainer

# Since specific time
docker logs --since 1h mycontainer
docker logs --since 2024-01-15T10:00:00 mycontainer

# Until specific time
docker logs --until 2024-01-15T12:00:00 mycontainer

# Combine options
docker logs -f --tail 50 -t mycontainer
```

### Filter Logs (with grep)

```bash
# Find errors
docker logs mycontainer 2>&1 | grep -i error

# Find specific pattern
docker logs mycontainer 2>&1 | grep "user login"

# Live filtering
docker logs -f mycontainer 2>&1 | grep --line-buffered "ERROR"

# Count occurrences
docker logs mycontainer 2>&1 | grep -c "error"
```

---

## 🏥 Health Checks

### Define in Dockerfile

```dockerfile
FROM nginx:alpine

# Basic health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/ || exit 1

# Or with wget
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost/ || exit 1
```

### Health Check Options

| Option | Description | Default |
|--------|-------------|---------|
| `--interval` | Time between checks | 30s |
| `--timeout` | Max time for check | 30s |
| `--start-period` | Grace period for startup | 0s |
| `--retries` | Failures before unhealthy | 3 |

### Define at Runtime

```bash
docker run -d \
  --health-cmd="curl -f http://localhost:8080/health || exit 1" \
  --health-interval=30s \
  --health-timeout=10s \
  --health-retries=3 \
  --health-start-period=10s \
  myapp
```

### Check Health Status

```bash
# View health status
docker ps
# CONTAINER ID   STATUS
# abc123         Up 5 min (healthy)
# def456         Up 2 min (unhealthy)

# Detailed health info
docker inspect --format='{{json .State.Health}}' mycontainer | jq

# Output:
{
  "Status": "healthy",
  "FailingStreak": 0,
  "Log": [
    {
      "Start": "2024-01-15T10:30:00.123Z",
      "End": "2024-01-15T10:30:00.456Z",
      "ExitCode": 0,
      "Output": "OK"
    }
  ]
}
```

### Health Check in Docker Compose

```yaml
services:
  webapp:
    image: myapp
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    
  postgres:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

---

## 📊 Container Monitoring

### docker stats

```bash
# Live stats for all containers
docker stats

# Specific containers
docker stats container1 container2

# No stream (single snapshot)
docker stats --no-stream

# Custom format
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# JSON output
docker stats --no-stream --format "{{json .}}"
```

### Output Columns

| Column | Description |
|--------|-------------|
| `CONTAINER ID` | Container identifier |
| `NAME` | Container name |
| `CPU %` | CPU usage percentage |
| `MEM USAGE / LIMIT` | Memory used / limit |
| `MEM %` | Memory usage percentage |
| `NET I/O` | Network bytes in/out |
| `BLOCK I/O` | Disk bytes read/written |
| `PIDS` | Number of processes |

### docker system df

```bash
# Disk usage overview
docker system df

# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          50        10        15.2GB    12.1GB (79%)
# Containers      25        5         2.3GB     1.8GB (78%)
# Local Volumes   10        8         5.1GB     500MB (9%)
# Build Cache     0         0         0B        0B

# Verbose output
docker system df -v
```

### docker events

```bash
# Live event stream
docker events

# Filter by type
docker events --filter 'type=container'
docker events --filter 'type=image'
docker events --filter 'type=network'
docker events --filter 'type=volume'

# Filter by event
docker events --filter 'event=start'
docker events --filter 'event=stop'
docker events --filter 'event=die'

# Filter by container
docker events --filter 'container=myapp'

# Since/Until
docker events --since '2024-01-15T10:00:00'
docker events --until '2024-01-15T12:00:00'

# Format as JSON
docker events --format '{{json .}}'
```

### Event Types

| Event | Description |
|-------|-------------|
| `create` | Container created |
| `start` | Container started |
| `stop` | Container stopped |
| `die` | Container died (exited) |
| `kill` | Container killed |
| `oom` | Out of memory |
| `pause` | Container paused |
| `unpause` | Container unpaused |
| `health_status` | Health check status changed |

---

## 🔧 Logging Best Practices

### Application Logging

```javascript
// Log to stdout/stderr (Docker captures these)
console.log(JSON.stringify({
  level: 'info',
  message: 'User logged in',
  userId: 123,
  timestamp: new Date().toISOString()
}));

// Don't log to files inside container!
// BAD: fs.writeFileSync('/var/log/app.log', message)
```

### Structured Logging

```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "level": "error",
  "message": "Database connection failed",
  "service": "api",
  "trace_id": "abc123",
  "error": {
    "code": "ECONNREFUSED",
    "host": "postgres",
    "port": 5432
  }
}
```

### Docker Compose Logging Config

```yaml
# Default for all services
x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

services:
  webapp:
    image: myapp
    logging: *default-logging
    
  nginx:
    image: nginx
    logging: *default-logging
```

---

## 🔗 Mapping to Kubernetes

| Docker Logging | Kubernetes Equivalent |
|----------------|----------------------|
| `docker logs` | `kubectl logs` |
| Log drivers | Node-level log collection |
| Fluentd | Fluentd/Fluent Bit DaemonSet |
| Health checks | Liveness/Readiness probes |
| `docker stats` | `kubectl top pods` |
| `docker events` | `kubectl get events` |

---

## ✅ Summary

### Key Commands

| Command | Purpose |
|---------|---------|
| `docker logs -f <container>` | Follow logs |
| `docker logs --tail 100` | Last 100 lines |
| `docker logs --since 1h` | Logs from last hour |
| `docker stats` | Resource usage |
| `docker system df` | Disk usage |
| `docker events` | Event stream |
| `docker inspect --format='{{json .State.Health}}'` | Health status |

### Best Practices

1. ✅ **Log to stdout/stderr** - Docker captures these
2. ✅ **Use structured logging** - JSON format
3. ✅ **Set log rotation** - Prevent disk filling
4. ✅ **Implement health checks** - Know when containers fail
5. ✅ **Use centralized logging** - Aggregate across containers
6. ✅ **Include trace IDs** - Correlate distributed logs

---

*Next: Docker in CI/CD*

