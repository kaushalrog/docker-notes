# 🐳 Docker — Complete Notes

> A complete reference guide for Docker: concepts, commands, Dockerfiles, Compose, networking, volumes, and best practices.

---

## 📑 Table of Contents

1. [What is Docker?](#1-what-is-docker)
2. [Core Concepts](#2-core-concepts)
3. [Installation](#3-installation)
4. [Docker CLI — Essential Commands](#4-docker-cli--essential-commands)
5. [Dockerfile](#5-dockerfile)
6. [Docker Images](#6-docker-images)
7. [Docker Containers](#7-docker-containers)
8. [Docker Volumes](#8-docker-volumes)
9. [Docker Networking](#9-docker-networking)
10. [Docker Compose](#10-docker-compose)
11. [Multi-Stage Builds](#11-multi-stage-builds)
12. [Docker Registry](#12-docker-registry)
13. [Environment Variables & Secrets](#13-environment-variables--secrets)
14. [Resource Limits](#14-resource-limits)
15. [Logging & Monitoring](#15-logging--monitoring)
16. [Docker Best Practices](#16-docker-best-practices)
17. [Common Errors & Fixes](#17-common-errors--fixes)
18. [Quick Reference Cheat Sheet](#18-quick-reference-cheat-sheet)

---

## 1. What is Docker?

Docker is a tool that allows developers to package an application and everything it needs to run (code, runtime, system tools, libraries) into a single, isolated box called a **Container**. This box can be shipped and run on any computer without changing anything.
```
Your App  →  Container  →  Runs anywhere (Dev, Staging, Prod)
```

### Docker vs Virtual Machine

| Feature        | Docker Container         | Virtual Machine         |
|----------------|--------------------------|-------------------------|
| Boot time      | Seconds                  | Minutes                 |
| Size           | MBs                      | GBs                     |
| OS             | Shares host OS kernel    | Full OS per VM          |
| Isolation      | Process-level            | Hardware-level          |
| Performance    | Near-native              | Overhead                |

---

## 2. Core Concepts

| Term              | Description |
|-------------------|-------------|
| **Image**         | Read-only blueprint to create containers (like a class) |
| **Container**     | Running instance of an image (like an object) |
| **Dockerfile**    | Script with instructions to build an image |
| **Registry**      | Storage for images (e.g., Docker Hub, ECR, GCR) |
| **Volume**        | Persistent storage that survives container restarts |
| **Network**       | Communication layer between containers |
| **Docker Compose**| Tool to define and run multi-container applications |

### Docker Architecture

```
┌─────────────────────────────────────────┐
│            Docker Client (CLI)          │
│         docker build / run / push       │
└──────────────────┬──────────────────────┘
                   │ REST API
┌──────────────────▼──────────────────────┐
│           Docker Daemon (dockerd)        │
│  ┌──────────┐ ┌──────────┐ ┌─────────┐ │
│  │  Images  │ │Containers│ │ Volumes │ │
│  └──────────┘ └──────────┘ └─────────┘ │
└─────────────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│           Docker Registry               │
│        (Docker Hub / private)           │
└─────────────────────────────────────────┘
```

---

## 3. Installation

### Linux (Ubuntu/Debian)

```bash
# Remove old versions
sudo apt remove docker docker-engine docker.io containerd runc

# Install dependencies
sudo apt update
sudo apt install ca-certificates curl gnupg lsb-release

# Add Docker's GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Run Docker without sudo
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker run hello-world
```

### macOS / Windows
- Download **Docker Desktop** from [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
- Docker Desktop includes Docker Engine, CLI, Compose, and Kubernetes.

---

## 4. Docker CLI — Essential Commands

### System

```bash
docker version                  # Show Docker version info
docker info                     # Display system-wide information
docker system df                # Show disk usage
docker system prune             # Remove unused data (stopped containers, dangling images)
docker system prune -a          # Remove all unused data including images
```

### Images

```bash
docker images                   # List local images
docker images -a                # Include intermediate images
docker pull nginx               # Pull image from Docker Hub
docker pull nginx:1.25          # Pull specific tag
docker rmi nginx                # Remove image
docker rmi $(docker images -q)  # Remove all images
docker image inspect nginx      # Detailed image info
docker image history nginx      # Show image layers
docker tag myapp:latest myapp:v1.0  # Tag an image
```

### Containers

```bash
docker run nginx                          # Run container (foreground)
docker run -d nginx                       # Run in detached (background) mode
docker run -it ubuntu bash                # Interactive terminal
docker run --name myapp nginx             # Run with custom name
docker run -p 8080:80 nginx               # Port mapping: host:container
docker run -v /host/path:/container/path nginx  # Volume mount
docker run --rm nginx                     # Auto-remove when stopped
docker run -e MY_VAR=value nginx          # Set environment variable

docker ps                                 # List running containers
docker ps -a                              # List all containers (including stopped)
docker start <container>                  # Start stopped container
docker stop <container>                   # Gracefully stop container
docker kill <container>                   # Force stop container
docker restart <container>               # Restart container
docker rm <container>                     # Remove container
docker rm -f <container>                  # Force remove running container
docker rm $(docker ps -aq)               # Remove all containers

docker exec -it <container> bash          # Enter running container shell
docker exec <container> ls /app           # Run command in container
docker logs <container>                   # View container logs
docker logs -f <container>                # Follow (stream) logs
docker logs --tail 100 <container>        # Last 100 log lines
docker inspect <container>               # Detailed container info
docker stats                              # Live resource usage stats
docker top <container>                    # Running processes in container
docker cp <container>:/path ./local       # Copy file from container
docker cp ./local <container>:/path       # Copy file to container
docker diff <container>                   # Show file changes
docker commit <container> new-image       # Create image from container
```

### Build

```bash
docker build .                            # Build from Dockerfile in current dir
docker build -t myapp:latest .            # Build with tag
docker build -t myapp:v1 -f Dockerfile.prod .  # Custom Dockerfile
docker build --no-cache -t myapp .        # Build without cache
docker build --build-arg NODE_ENV=prod .  # Pass build argument
```

---

## 5. Dockerfile

A `Dockerfile` is a text file with instructions to build a Docker image.

### Basic Structure

```dockerfile
# Base image
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy dependency files first (for cache efficiency)
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy source code
COPY . .

# Expose port
EXPOSE 3000

# Default command
CMD ["node", "server.js"]
```

### All Dockerfile Instructions

| Instruction    | Description | Example |
|----------------|-------------|---------|
| `FROM`         | Base image (must be first) | `FROM ubuntu:22.04` |
| `WORKDIR`      | Set working directory | `WORKDIR /app` |
| `COPY`         | Copy files from host to image | `COPY . .` |
| `ADD`          | Like COPY but supports URLs and tar extraction | `ADD app.tar.gz /app` |
| `RUN`          | Execute command during build | `RUN apt-get install -y curl` |
| `CMD`          | Default command when container starts | `CMD ["npm", "start"]` |
| `ENTRYPOINT`   | Fixed entrypoint, CMD becomes args | `ENTRYPOINT ["python"]` |
| `ENV`          | Set environment variable | `ENV NODE_ENV=production` |
| `ARG`          | Build-time variable | `ARG VERSION=1.0` |
| `EXPOSE`       | Document which port app listens on | `EXPOSE 8080` |
| `VOLUME`       | Define mount point for volumes | `VOLUME ["/data"]` |
| `USER`         | Set user to run commands as | `USER node` |
| `LABEL`        | Add metadata | `LABEL version="1.0"` |
| `HEALTHCHECK`  | Check container health | `HEALTHCHECK CMD curl -f http://localhost/` |
| `ONBUILD`      | Trigger instruction in child image | `ONBUILD COPY . .` |
| `SHELL`        | Override default shell | `SHELL ["/bin/bash", "-c"]` |
| `STOPSIGNAL`   | Signal to stop container | `STOPSIGNAL SIGTERM` |

### CMD vs ENTRYPOINT

```dockerfile
# CMD only — command can be overridden
CMD ["node", "server.js"]
# docker run myapp python script.py  → runs python script.py

# ENTRYPOINT only — command is fixed
ENTRYPOINT ["node"]
# docker run myapp server.js → runs node server.js

# Both — ENTRYPOINT is fixed, CMD provides default args
ENTRYPOINT ["node"]
CMD ["server.js"]
# docker run myapp → runs: node server.js
# docker run myapp other.js → runs: node other.js
```

### Shell vs Exec Form

```dockerfile
# Shell form (runs via /bin/sh -c) — allows variable expansion
RUN echo $HOME
CMD npm start

# Exec form (preferred) — no shell, no variable expansion
RUN ["npm", "install"]
CMD ["npm", "start"]
```

### .dockerignore

Create `.dockerignore` to exclude files from the build context:

```
node_modules
.git
.env
*.log
dist
.DS_Store
**/__pycache__
**/*.pyc
coverage
.github
```

---

## 6. Docker Images

### Image Layers

Each instruction in a Dockerfile creates a new **read-only layer**. Layers are cached — if nothing changes, Docker reuses the cache.

```
Layer 5: COPY . .            ← Changes often, put last
Layer 4: RUN npm install     ← Changes when package.json changes
Layer 3: COPY package*.json  ← Rarely changes
Layer 2: WORKDIR /app
Layer 1: FROM node:18        ← Base, never changes
```

### Image Naming Convention

```
[registry/][username/]repository[:tag]

Examples:
  ubuntu                        → docker.io/library/ubuntu:latest
  nginx:1.25                    → docker.io/library/nginx:1.25
  myusername/myapp:v2.0
  gcr.io/myproject/myapp:latest
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:prod
```

---

## 7. Docker Containers

### Container Lifecycle

```
Created → Running → Paused → Running → Stopped → Removed
   ↑                                       ↑
docker create                          docker stop
docker run                             docker kill
```

### Restart Policies

```bash
docker run --restart=no nginx           # Default: never restart
docker run --restart=always nginx       # Always restart
docker run --restart=on-failure nginx   # Restart only on non-zero exit
docker run --restart=unless-stopped nginx # Always except manually stopped
```

### Health Checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

```bash
docker inspect --format='{{.State.Health.Status}}' <container>
```

---

## 8. Docker Volumes

Volumes persist data beyond container lifecycle.

### Types of Storage

| Type           | Command                               | Description |
|----------------|---------------------------------------|-------------|
| Volume         | `-v myvolume:/data`                   | Managed by Docker, best for persistence |
| Bind Mount     | `-v /host/path:/container/path`       | Maps host dir to container |
| tmpfs Mount    | `--tmpfs /tmp`                        | In-memory only, not persisted |

### Volume Commands

```bash
docker volume create mydata             # Create a volume
docker volume ls                        # List volumes
docker volume inspect mydata            # Inspect volume details
docker volume rm mydata                 # Remove volume
docker volume prune                     # Remove unused volumes

# Mount a volume
docker run -v mydata:/app/data nginx

# Bind mount (absolute path required)
docker run -v $(pwd)/data:/app/data nginx

# Read-only mount
docker run -v mydata:/app/data:ro nginx
```

### Backup & Restore Volume

```bash
# Backup
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/backup.tar.gz /data

# Restore
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  ubuntu tar xzf /backup/backup.tar.gz -C /
```

---

## 9. Docker Networking

### Network Types

| Driver    | Description |
|-----------|-------------|
| `bridge`  | Default. Isolated network on single host |
| `host`    | Shares host network stack, no isolation |
| `none`    | No networking |
| `overlay` | Multi-host networking (Docker Swarm) |
| `macvlan` | Assign MAC address to container |

### Network Commands

```bash
docker network ls                                # List networks
docker network create mynet                      # Create bridge network
docker network create --driver overlay mynet     # Create overlay network
docker network inspect mynet                     # Inspect network
docker network rm mynet                          # Remove network
docker network prune                             # Remove unused networks

# Connect container to network
docker run --network mynet nginx
docker network connect mynet <container>
docker network disconnect mynet <container>
```

### Container Communication

Containers on the same network can reach each other by **container name**:

```bash
docker network create appnet

docker run -d --name db --network appnet postgres
docker run -d --name api --network appnet myapi

# Inside api container:
# psql -h db -U postgres  ← use container name "db" as hostname
```

### Port Mapping

```bash
docker run -p 8080:80 nginx           # host:container
docker run -p 127.0.0.1:8080:80 nginx # Bind to specific host IP
docker run -P nginx                   # Map all EXPOSED ports randomly
docker port <container>               # Show port mappings
```

---

## 10. Docker Compose

Docker Compose defines and runs multi-container applications via a YAML file.

### Basic `docker-compose.yml`

```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    volumes:
      - .:/app
      - /app/node_modules
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:

networks:
  default:
    name: myapp-network
```

### Compose Commands

```bash
docker compose up                   # Start all services
docker compose up -d                # Start in background
docker compose up --build           # Rebuild images before starting
docker compose down                 # Stop and remove containers
docker compose down -v              # Also remove volumes
docker compose stop                 # Stop without removing
docker compose start                # Start stopped services
docker compose restart              # Restart services

docker compose ps                   # List service containers
docker compose logs                 # View all logs
docker compose logs -f web          # Follow logs for 'web' service
docker compose exec web bash        # Shell into service
docker compose run web npm test     # Run one-off command

docker compose build                # Build images
docker compose pull                 # Pull latest images
docker compose config               # Validate and view config
```

### Full Compose Example (Production-like)

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/ssl:ro
    depends_on:
      - api
    networks:
      - frontend

  api:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        - NODE_ENV=production
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://redis:6379
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - frontend
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redisdata:/data
    networks:
      - backend

volumes:
  pgdata:
  redisdata:

networks:
  frontend:
  backend:
```

---

## 11. Multi-Stage Builds

Multi-stage builds produce smaller, leaner production images.

### Node.js Example

```dockerfile
# ── Stage 1: Build ──────────────────────────────
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# ── Stage 2: Production ──────────────────────────
FROM node:18-alpine AS production

WORKDIR /app

# Only copy what's needed
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package.json .

# Don't run as root
USER node

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Python Example

```dockerfile
# Stage 1: Build dependencies
FROM python:3.11-slim AS deps

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2: Production
FROM python:3.11-slim AS production

WORKDIR /app
COPY --from=deps /root/.local /root/.local
COPY . .

ENV PATH=/root/.local/bin:$PATH

USER nobody
CMD ["python", "app.py"]
```

### Go Example

```dockerfile
# Stage 1: Build binary
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# Stage 2: Minimal production image
FROM scratch

COPY --from=builder /app/main /main
EXPOSE 8080
ENTRYPOINT ["/main"]
```

---

## 12. Docker Registry

### Docker Hub

```bash
# Login / logout
docker login
docker logout

# Push image
docker tag myapp:latest username/myapp:latest
docker push username/myapp:latest

# Pull image
docker pull username/myapp:latest
```

### Self-hosted Registry

```bash
# Run local registry
docker run -d -p 5000:5000 --name registry registry:2

# Push to local registry
docker tag myapp localhost:5000/myapp
docker push localhost:5000/myapp

# Pull from local registry
docker pull localhost:5000/myapp
```

### AWS ECR

```bash
# Authenticate
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS \
  --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Tag and push
docker tag myapp:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
```

---

## 13. Environment Variables & Secrets

### Passing Variables

```bash
# Single variable
docker run -e MY_VAR=value myapp

# From host environment
docker run -e MY_VAR myapp

# From file
docker run --env-file .env myapp
```

### `.env` File

```env
DB_HOST=localhost
DB_PORT=5432
DB_USER=admin
DB_PASSWORD=secret123
```

### Using in Dockerfile

```dockerfile
# Build-time argument (not in final image)
ARG VERSION=1.0
RUN echo "Building version $VERSION"

# Runtime environment variable
ENV NODE_ENV=production
ENV PORT=3000
```

### Docker Secrets (Swarm)

```bash
echo "mysecret" | docker secret create db_password -
docker service create \
  --secret db_password \
  --name myapp myimage
# Secret available at: /run/secrets/db_password
```

---

## 14. Resource Limits

```bash
# Memory limit
docker run --memory="512m" nginx
docker run --memory="1g" --memory-swap="2g" nginx

# CPU limits
docker run --cpus="1.5" nginx           # 1.5 CPU cores
docker run --cpu-shares=512 nginx        # Relative weight (default 1024)

# Combined
docker run \
  --memory="512m" \
  --cpus="0.5" \
  --name myapp \
  nginx
```

### In Docker Compose

```yaml
services:
  api:
    image: myapi
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

---

## 15. Logging & Monitoring

### Log Drivers

```bash
# Default JSON file logging
docker run nginx

# syslog
docker run --log-driver=syslog nginx

# JSON file with options
docker run \
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  nginx
```

### Monitoring Commands

```bash
docker stats                        # Live stats for all containers
docker stats <container>            # Stats for specific container
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

docker top <container>              # Running processes
docker events                       # Real-time event stream
docker events --filter container=myapp
```

---

## 16. Docker Best Practices

### Dockerfile Best Practices

```dockerfile
# ✅ Use specific base image tags — avoid :latest in production
FROM node:18.19-alpine3.19

# ✅ Use .dockerignore to exclude unnecessary files

# ✅ Combine RUN commands to reduce layers
RUN apt-get update && \
    apt-get install -y curl wget && \
    rm -rf /var/lib/apt/lists/*

# ✅ Copy dependency files before source code (cache optimization)
COPY package*.json ./
RUN npm ci --only=production
COPY . .

# ✅ Run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# ✅ Use COPY over ADD (unless you need tar/URL support)
COPY . .

# ✅ Use exec form for CMD and ENTRYPOINT
CMD ["node", "server.js"]

# ✅ Add HEALTHCHECK
HEALTHCHECK --interval=30s CMD curl -f http://localhost:3000/health

# ✅ Label your images
LABEL maintainer="your@email.com"
LABEL version="1.0.0"
```

### Security Best Practices

- Never run containers as `root` in production
- Never store secrets in environment variables or image layers
- Use `--read-only` flag for immutable containers
- Scan images: `docker scout cves myimage`
- Use minimal base images: `alpine`, `distroless`, `scratch`
- Don't expose unnecessary ports
- Use secrets management (Vault, AWS Secrets Manager, Docker Secrets)
- Regularly update base images

### Performance Best Practices

- Use multi-stage builds to minimize image size
- Leverage layer caching — put infrequently changed layers first
- Use `.dockerignore` to reduce build context size
- Prefer `npm ci` over `npm install` in Dockerfiles
- Use `--no-cache` in CI/CD to avoid stale caches

---

## 17. Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `permission denied` while running docker | User not in docker group | `sudo usermod -aG docker $USER && newgrp docker` |
| `port is already allocated` | Host port already in use | Use a different host port `-p 8081:80` |
| `No such file or directory` in container | Wrong WORKDIR or COPY path | Check paths in Dockerfile |
| `OOMKilled` | Container ran out of memory | Increase `--memory` limit |
| `Image not found` | Wrong image name or tag | `docker pull` the image first |
| Container exits immediately | CMD fails or no foreground process | Check logs: `docker logs <container>` |
| `connection refused` between containers | Not on the same network | Add both to same Docker network |
| `Build context too large` | Missing `.dockerignore` | Add `.dockerignore` file |
| `Layer already exists` during push | Push is resumable | It's fine, Docker deduplicates layers |
| `no space left on device` | Docker disk full | `docker system prune -a` |

---

## 18. Quick Reference Cheat Sheet

```bash
# ── Images ──────────────────────────────────────
docker pull nginx:alpine           # Pull image
docker images                      # List images
docker rmi nginx                   # Remove image
docker build -t myapp:v1 .         # Build image
docker push myapp:v1               # Push to registry

# ── Containers ──────────────────────────────────
docker run -d -p 8080:80 --name web nginx   # Run detached
docker ps                          # List running
docker ps -a                       # List all
docker stop web                    # Stop
docker rm web                      # Remove
docker exec -it web bash           # Shell access
docker logs -f web                 # Follow logs

# ── Volumes ──────────────────────────────────────
docker volume create mydata
docker run -v mydata:/data nginx
docker volume ls
docker volume rm mydata

# ── Networks ─────────────────────────────────────
docker network create mynet
docker run --network mynet nginx
docker network ls
docker network rm mynet

# ── Compose ──────────────────────────────────────
docker compose up -d               # Start services
docker compose down                # Stop & remove
docker compose logs -f             # Follow logs
docker compose exec web bash       # Shell into service

# ── Cleanup ──────────────────────────────────────
docker system prune                # Remove unused resources
docker system prune -a             # Remove all unused
docker volume prune                # Remove unused volumes
docker image prune                 # Remove dangling images
```

---

## 📚 Further Learning

- [Official Docker Docs](https://docs.docker.com)
- [Docker Hub](https://hub.docker.com)
- [Play with Docker](https://labs.play-with-docker.com) — Free online Docker playground
- [Docker Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Compose Specification](https://compose-spec.io)

---

*Feel free to open issues or PRs to improve these notes!*
