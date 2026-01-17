# 🔬 Chapter 12: Docker Internals

> Understanding what happens under the hood - Linux namespaces, cgroups, and container runtimes.

---

## 🎯 Learning Objectives

- Understand Linux namespaces and how they provide isolation
- Learn how cgroups limit resources
- Understand union filesystems and image layers
- Know the container runtime stack (containerd, runc)
- See how it all connects to Kubernetes

---

## 🧠 The Big Picture

Docker isn't magic - it uses **Linux kernel features** that have existed for years:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DOCKER INTERNALS ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     Your Application                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                               │                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     Docker CLI / API                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                               │                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     Docker Daemon (dockerd)                       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                               │                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     containerd (Container Runtime)                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                               │                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     runc (OCI Runtime)                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                               │                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     Linux Kernel                                  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │   │
│  │  │ Namespaces  │  │   cgroups   │  │  Union Filesystem       │  │   │
│  │  │ (Isolation) │  │ (Limits)    │  │  (Layers)               │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🏠 Linux Namespaces (Isolation)

**Namespaces** provide isolation - they make a container think it's the only thing running on the system.

### Types of Namespaces

| Namespace | What It Isolates | Example |
|-----------|-----------------|---------|
| **PID** | Process IDs | Container sees PID 1 as its init process |
| **NET** | Network stack | Container has its own IP, ports, routes |
| **MNT** | Mount points | Container has its own filesystem view |
| **UTS** | Hostname | Container can have its own hostname |
| **IPC** | Inter-process comm | Isolated shared memory, semaphores |
| **USER** | User/Group IDs | Root in container ≠ root on host |
| **CGROUP** | Cgroup visibility | Container sees only its cgroup |

### Real-World Analogy

Think of namespaces like **apartments in a building**:
- Each apartment (container) has its own address (network namespace)
- Each apartment has its own rooms (mount namespace)
- Residents can't see into other apartments (isolation)
- But they all share the same building structure (kernel)

### See It In Action

```bash
# On host - see all processes
ps aux | wc -l
# Output: 200+ processes

# Inside container - isolated view
docker run --rm alpine ps aux
# Output: Only 1-2 processes (the container's processes)

# Container thinks it's PID 1
docker run --rm alpine cat /proc/1/cmdline
# Output: /bin/sh (or whatever CMD is)
```

### PID Namespace Example

```bash
# Run a container
docker run -d --name test alpine sleep 1000

# View process from HOST perspective
docker top test
# PID on host might be 12345

# View from CONTAINER perspective
docker exec test ps aux
# PID inside container is 1

# Both point to the same process, different views!
```

### Network Namespace Example

```bash
# Host has its own network
ip addr
# Shows eth0, docker0, etc.

# Container has isolated network
docker run --rm alpine ip addr
# Shows only lo and eth0 with container IP

# Different network namespaces!
```

---

## ⚖️ Control Groups (cgroups) - Resource Limits

**cgroups** limit how much resources a container can use.

### What cgroups Control

| Resource | cgroup Control |
|----------|---------------|
| CPU | Time slices, cores |
| Memory | RAM limit, swap |
| I/O | Disk read/write bandwidth |
| Network | Bandwidth (with tc) |
| PIDs | Number of processes |

### Real-World Analogy

Think of cgroups like **utility meters in apartments**:
- Each apartment has a power limit (CPU)
- Each apartment has a water limit (memory)
- If you exceed, you get cut off (OOM killed)

### See cgroups in Action

```bash
# Run container with limits
docker run -d --name limited -m 128m --cpus 0.5 nginx

# View cgroup settings
docker exec limited cat /sys/fs/cgroup/memory.max
# Shows memory limit in bytes

docker exec limited cat /sys/fs/cgroup/cpu.max
# Shows CPU quota
```

### How Memory Limit Works

```bash
# Set 100MB limit
docker run -m 100m myapp

# What happens:
# 1. cgroup memory.max is set to 104857600 bytes
# 2. If process exceeds, kernel sends SIGKILL
# 3. Container exits with code 137 (OOMKilled)
```

### How CPU Limit Works

```bash
# Set 0.5 CPU (50% of one core)
docker run --cpus 0.5 myapp

# What happens:
# 1. cgroup sets cpu.max to "50000 100000" (50ms per 100ms)
# 2. Container gets throttled if it exceeds
# 3. Process slows down but doesn't get killed
```

---

## 📚 Union Filesystem (Layers)

Docker images are made of **layers** stacked on top of each other using a **union filesystem**.

### How Layers Work

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         UNION FILESYSTEM                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Container Layer (R/W)     ← Your changes go here (ephemeral)           │
│  ──────────────────────                                                  │
│  App Code Layer (R/O)      ← COPY app.js .                              │
│  ──────────────────────                                                  │
│  Dependencies Layer (R/O)  ← RUN npm install                            │
│  ──────────────────────                                                  │
│  Base Image Layer (R/O)    ← FROM node:18-alpine                        │
│  ──────────────────────                                                  │
│  Alpine Base (R/O)         ← Alpine Linux files                         │
│                                                                          │
│  All layers merged into single filesystem view                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Real-World Analogy

Think of layers like **transparent slides stacked on a projector**:
- Each slide (layer) adds something new
- When you look through all slides, you see the complete picture
- You can reuse slides (layers) across different presentations (images)

### Key Concepts

1. **Image Layers are Read-Only**: Once created, never change
2. **Container Layer is Read-Write**: Changes stored here
3. **Copy-on-Write**: File modified = copied to container layer first
4. **Layer Sharing**: Multiple containers share same base layers

### See Layers

```bash
# View image layers
docker history nginx:alpine

# Output:
# IMAGE          CREATED       SIZE      COMMAND
# abc123         2 days ago    0B        CMD ["nginx"...]
# def456         2 days ago    1.2MB     COPY nginx.conf
# ghi789         2 days ago    24MB      RUN apk add nginx
# ...

# Each line is a layer!
```

### Union Filesystem Types

| Filesystem | Description |
|------------|-------------|
| **overlay2** | Default, recommended, fast |
| **aufs** | Older, Ubuntu legacy |
| **devicemapper** | Block-level, RHEL |
| **btrfs** | If using btrfs filesystem |
| **zfs** | If using ZFS filesystem |

```bash
# Check which one you're using
docker info | grep "Storage Driver"
# Output: Storage Driver: overlay2
```

---

## 🏃 Container Runtimes

### The Runtime Stack

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      CONTAINER RUNTIME STACK                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Docker CLI                                                       │   │
│  │  (User interface - docker run, docker build, etc.)               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                               │                                          │
│                               ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Docker Daemon (dockerd)                                          │   │
│  │  (API server, image management, networking, volumes)             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                               │                                          │
│                               ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  containerd (High-Level Runtime)                                  │   │
│  │  (Container lifecycle, image pull/push, storage)                 │   │
│  │  ⭐ Kubernetes uses containerd directly!                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                               │                                          │
│                               ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  runc (Low-Level Runtime - OCI Compliant)                         │   │
│  │  (Actually creates and runs containers using kernel features)    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                               │                                          │
│                               ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Linux Kernel (namespaces, cgroups, seccomp, capabilities)       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### containerd

- **What**: Industry-standard container runtime
- **Role**: Manages container lifecycle, images, storage
- **Used by**: Docker, Kubernetes, cloud providers
- **Key point**: Kubernetes talks to containerd directly, bypassing Docker

```bash
# containerd is running as a service
# You can interact with it directly using ctr
ctr --address /run/containerd/containerd.sock containers list
```

### runc

- **What**: Low-level OCI runtime
- **Role**: Actually creates the container using Linux primitives
- **Implements**: OCI (Open Container Initiative) specification

```bash
# runc creates containers from OCI bundles
# Usually you don't interact with it directly
```

### OCI (Open Container Initiative)

Standards for containers:
1. **Runtime Spec**: How to run a container
2. **Image Spec**: How container images are formatted
3. **Distribution Spec**: How images are distributed

This ensures **interoperability** - images work across different runtimes.

---

## 🔐 Security Features

### Seccomp (Secure Computing)

Limits which **system calls** a container can make:

```bash
# Docker applies a default seccomp profile
# Blocks dangerous syscalls like:
# - reboot
# - mount
# - ptrace
# - etc.

# Run without seccomp (dangerous!)
docker run --security-opt seccomp=unconfined myapp
```

### AppArmor / SELinux

**Mandatory Access Control** - extra layer of security:

```bash
# Check AppArmor profile
docker inspect mycontainer | grep AppArmor

# Run with custom AppArmor profile
docker run --security-opt apparmor=my-profile myapp
```

### Capabilities

Fine-grained root permissions:

```bash
# Drop all capabilities
docker run --cap-drop ALL myapp

# Add only what's needed
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp
```

---

## 🔗 Mapping to Kubernetes

| Docker Internal | Kubernetes Equivalent |
|-----------------|----------------------|
| containerd | Default container runtime for K8s |
| runc | Same - OCI runtime |
| Namespaces | Pod network/PID/IPC namespaces |
| cgroups | Resource requests/limits |
| Seccomp | Pod securityContext.seccompProfile |
| Capabilities | securityContext.capabilities |
| Union FS | Same - image layers |

### Kubernetes Container Runtime Interface (CRI)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  Kubernetes (kubelet)                                                   │
│         │                                                                │
│         │ CRI (Container Runtime Interface)                             │
│         ▼                                                                │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  containerd  │  CRI-O  │  Docker (deprecated)                    │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│         │                                                                │
│         ▼                                                                │
│       runc                                                              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Note**: Docker as a runtime is deprecated in Kubernetes 1.24+. 
Kubernetes now uses containerd directly. But Docker images still work!

---

## ✅ Summary

### Key Concepts

| Concept | Purpose | Kernel Feature |
|---------|---------|----------------|
| **Namespaces** | Isolation | pid, net, mnt, uts, ipc, user |
| **cgroups** | Resource limits | memory, cpu, io, pids |
| **Union FS** | Layered images | overlay2 |
| **Seccomp** | Syscall filtering | seccomp-bpf |
| **Capabilities** | Fine-grained root | Linux capabilities |

### The Container Formula

```
Container = Namespaces (isolation)
          + cgroups (resource limits)
          + Union FS (layered filesystem)
          + Seccomp/Capabilities (security)
```

### Why This Matters

1. **Troubleshooting**: Understanding internals helps debug issues
2. **Security**: Know what's actually isolating your containers
3. **Performance**: Understand resource limiting behavior
4. **CKA**: May be asked about container security and isolation

---

*Docker internals complete! Now you understand what's really happening under the hood.*

