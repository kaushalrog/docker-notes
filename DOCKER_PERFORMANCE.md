# ⚡ Docker Performance Optimization

## Image Size Optimization

### Use Minimal Base Images

```dockerfile
# ❌ Bloated
FROM ubuntu:22.04                # ~70MB base

# ✅ Slim
FROM debian:bookworm-slim        # ~75MB
FROM python:3.11-slim            # ~130MB

# ✅ Alpine (smallest)
FROM alpine:3.18                 # ~7MB
FROM node:18-alpine              # ~50MB
FROM python:3.11-alpine          # ~55MB

# ✅ Distroless (no shell/package manager)
FROM gcr.io/distroless/python3   # ~20MB

# ✅ Scratch (for static binaries)
FROM scratch                     # 0MB
```

### Multi-Stage Builds

```dockerfile
# Stage 1: Full build environment (~900MB)
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

# Stage 2: Minimal runtime (~50MB)
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]

# Result: 50MB instead of 900MB+ ✅
```

### Remove Unnecessary Files in Same Layer

```dockerfile
# ❌ Bad — intermediate files stay in layers
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ Good — all in one layer, no leftover files
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      curl \
      wget && \
    rm -rf /var/lib/apt/lists/* && \
    apt-get clean
```

### .dockerignore — Reduce Build Context

```
# Without .dockerignore: 500MB build context (node_modules!)
# With .dockerignore: 2MB build context

node_modules
.git
dist
build
coverage
*.log
.env
.DS_Store
```

### Check Image Size

```bash
docker images myapp
docker history myapp                    # Size per layer
docker image inspect myapp | grep Size

# Find largest layers
docker history --no-trunc myapp | sort -k5 -rh | head -10

# Analyze with dive
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive myapp
```

---

## Layer Caching Optimization

### Order Dockerfile Instructions by Change Frequency

```dockerfile
# ✅ Correct order — most stable first
FROM node:18-alpine                        # Never changes

WORKDIR /app

# Changes only when package.json changes
COPY package*.json ./
RUN npm ci

# Changes often — put LAST
COPY . .

RUN npm run build
```

```dockerfile
# ❌ Bad — invalidates cache on every code change
FROM node:18-alpine
COPY . .                                   # Everything copied first
RUN npm ci                                 # Re-runs every time!
```

### BuildKit Cache Mounts

```dockerfile
# Cache pip downloads across builds
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Cache npm downloads
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Cache apt packages
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get install -y curl

# Cache Go modules
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

# Cache Cargo (Rust)
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    cargo build --release
```

```bash
# Enable BuildKit
export DOCKER_BUILDKIT=1
docker build .

# Or permanently in daemon.json
{
  "features": { "buildkit": true }
}
```

### Registry Cache

```bash
# Use registry for cross-machine caching
docker build \
  --cache-from myapp:latest \
  --cache-to type=registry,ref=myapp:buildcache \
  -t myapp:latest .

# GitHub Actions cache
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

---

## Build Speed Optimization

### Parallel Builds (Multi-Stage)

```dockerfile
# These stages run in PARALLEL with BuildKit
FROM node:18-alpine AS frontend-builder
COPY frontend/ .
RUN npm ci && npm run build

FROM python:3.11-slim AS backend-builder
COPY backend/requirements.txt .
RUN pip install -r requirements.txt

# Final stage waits for both
FROM python:3.11-slim AS production
COPY --from=frontend-builder /frontend/dist /app/static
COPY --from=backend-builder /root/.local /root/.local
COPY backend/ /app/
CMD ["python", "app.py"]
```

```bash
# Enable parallel builds
DOCKER_BUILDKIT=1 docker build .
```

### Build Specific Stage

```bash
# Only build up to the "test" stage
docker build --target test .
docker build --target development .
```

---

## Runtime Performance

### Resource Limits (Prevent Noisy Neighbors)

```bash
# CPU
docker run --cpus="2.0" myapp          # 2 CPU cores
docker run --cpu-shares=1024 myapp     # Higher priority

# Memory
docker run --memory="512m" myapp
docker run --memory="1g" --memory-swap="1g" myapp  # Disable swap

# I/O
docker run --device-read-bps /dev/sda:10mb myapp
docker run --device-write-bps /dev/sda:10mb myapp
docker run --blkio-weight=500 myapp    # Block I/O weight
```

### tmpfs for Temp Data

```bash
# Write to RAM instead of disk
docker run --tmpfs /tmp:size=100m myapp

# Dramatically faster for temp files
```

### Host Networking (Max Performance)

```bash
# Skip Docker's network layer
docker run --network host myapp
# WARNING: No port mapping, no isolation
# Use only for performance-critical or latency-sensitive apps
```

---

## Compose Performance

### Parallel Service Startup

```yaml
# Docker Compose starts services in dependency order
# Independent services start in PARALLEL automatically

services:
  api:
    depends_on: [db, redis]     # api waits for db and redis

  db:                           # db and redis start together ✅
    image: postgres

  redis:                        # starts in parallel with db ✅
    image: redis
```

### Build Cache in Compose

```yaml
services:
  api:
    build:
      context: .
      cache_from:
        - myapp:latest          # Use registry image as cache source
```

```bash
docker compose build --parallel  # Build all services in parallel
```

---

## Volume Performance

### Named Volumes vs Bind Mounts

| Type | Linux Performance | macOS Performance |
|------|-------------------|-------------------|
| Named Volume | ⚡ Native | ⚡ Fast (in VM) |
| Bind Mount | ⚡ Native | 🐌 Slow (file sync overhead) |

```yaml
# macOS: Use named volumes for heavy file I/O
services:
  app:
    volumes:
      - node_modules:/app/node_modules  # Named volume — fast ✅
      - .:/app                           # Bind mount for source — ok for dev
      
volumes:
  node_modules:
```

### Delegated Mounts (macOS Legacy)

```yaml
# Older Docker Desktop — improve bind mount performance
volumes:
  - .:/app:delegated        # Container writes don't sync immediately
  - .:/app:cached           # Host writes don't sync immediately
  - .:/app:consistent       # Default — fully consistent
```

---

## Logging Performance

### Use JSON File with Rotation

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "compress": "true"
  }
}
```

```bash
# Or per container
docker run \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp
```

### Use Syslog or External Driver for Production

```bash
# Syslog (lower overhead than json-file for high-volume logs)
docker run --log-driver=syslog myapp

# Fluentd
docker run --log-driver=fluentd myapp

# AWS CloudWatch
docker run --log-driver=awslogs \
  --log-opt awslogs-group=/myapp \
  myapp
```

---

## Monitoring

### Live Stats

```bash
docker stats                              # All containers
docker stats myapp                        # Single container
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"
```

### Prometheus + cAdvisor

```yaml
# docker-compose.monitoring.yml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
```

---

## Image Optimization Summary

```bash
# Before optimization: 950MB
FROM node:18
COPY . .
RUN npm install
CMD ["node", "server.js"]

# After optimization: ~50MB
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=builder --chown=app:app /app/dist ./dist
COPY --from=builder --chown=app:app /app/node_modules ./node_modules
USER app
CMD ["node", "dist/server.js"]

# 95% smaller image = faster pulls, less storage, smaller attack surface ✅
```
