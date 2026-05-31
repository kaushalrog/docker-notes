# 🐳 Docker — Complete Reference Notes

> A complete, production-ready Docker reference with notes, guides, examples, troubleshooting, CI/CD, security, and interview prep — all in one place.

![Docker](https://img.shields.io/badge/Docker-2CA5E0?style=for-the-badge&logo=docker&logoColor=white)
![Docker Compose](https://img.shields.io/badge/Docker_Compose-2CA5E0?style=for-the-badge&logo=docker&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)

---

## 📁 Repository Structure

```
docker-notes/
│
├── README.md                                    ← You are here (core notes)
├── INTERVIEW_QA.md                              ← 20 interview Q&As (beginner → advanced)
├── UPLOAD_GUIDE.md                              ← How to upload this to GitHub
│
├── cheatsheet/
│   └── DOCKER_COMMANDS.md                       ← All CLI commands reference
│
├── dockerfile-guide/
│   └── DOCKERFILE_REFERENCE.md                 ← Every instruction + language examples
│
├── compose/
│   └── DOCKER_COMPOSE.md                        ← Full Compose guide + all options
│
├── networking/
│   └── DOCKER_NETWORKING.md                     ← Networks, DNS, ports, debugging
│
├── volumes/
│   └── DOCKER_VOLUMES.md                        ← Volumes, bind mounts, backup/restore
│
├── security/
│   └── DOCKER_SECURITY.md                       ← Security best practices + checklist
│
├── performance/
│   └── DOCKER_PERFORMANCE.md                    ← Image optimization, caching, monitoring
│
├── cicd/
│   └── DOCKER_CICD.md                           ← GitHub Actions, GitLab CI, deploy scripts
│
├── troubleshooting/
│   └── DOCKER_TROUBLESHOOTING.md               ← Common errors & how to fix them
│
├── examples/
│   ├── DOCKERFILES_BY_LANGUAGE.md              ← Node, Python, Go, Java, PHP, Ruby, Rust
│   └── REAL_WORLD_STACKS.md                    ← MERN, Django, Microservices, ELK, WordPress
│
├── Dockerfile.example                           ← Production multi-stage Dockerfile
├── docker-compose.example.yml                   ← Full-stack Compose example
└── .dockerignore.example                        ← Comprehensive .dockerignore template
```

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

Docker is an **open-source platform** that packages applications and their dependencies into lightweight, portable units called **containers**.

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
sudo apt remove docker docker-engine docker.io containerd runc
sudo apt update && sudo apt install ca-certificates curl gnupg lsb-release

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin

sudo usermod -aG docker $USER && newgrp docker
docker --version && docker run hello-world
```

### macOS / Windows
Download **Docker Desktop** from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)

---

## 4. Docker CLI — Essential Commands

### System
```bash
docker version && docker info
docker system df
docker system prune && docker system prune -a
```

### Images
```bash
docker images && docker pull nginx:alpine
docker rmi nginx && docker rmi $(docker images -q)
docker image inspect nginx && docker image history nginx
docker tag myapp:latest myapp:v1.0
```

### Containers
```bash
docker run -d -p 8080:80 --name web nginx
docker run -it ubuntu bash
docker run --rm -e MY_VAR=value --env-file .env nginx

docker ps && docker ps -a
docker stop web && docker kill web
docker rm web && docker rm $(docker ps -aq)

docker exec -it web bash
docker logs -f web && docker logs --tail 100 web
docker inspect web && docker stats && docker top web
docker cp web:/path ./local && docker cp ./local web:/path
```

### Build
```bash
docker build -t myapp:latest .
docker build -t myapp:v1 -f Dockerfile.prod .
docker build --no-cache --build-arg NODE_ENV=prod .
```

---

## 5. Dockerfile

### Basic Structure

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

### All Instructions

| Instruction    | Description | Example |
|----------------|-------------|---------|
| `FROM`         | Base image | `FROM ubuntu:22.04` |
| `WORKDIR`      | Set working directory | `WORKDIR /app` |
| `COPY`         | Copy files from host | `COPY . .` |
| `ADD`          | Copy + URL/tar support | `ADD app.tar.gz /app` |
| `RUN`          | Execute at build time | `RUN npm install` |
| `CMD`          | Default runtime command | `CMD ["npm", "start"]` |
| `ENTRYPOINT`   | Fixed entrypoint | `ENTRYPOINT ["python"]` |
| `ENV`          | Environment variable | `ENV NODE_ENV=production` |
| `ARG`          | Build-time variable | `ARG VERSION=1.0` |
| `EXPOSE`       | Document port | `EXPOSE 8080` |
| `VOLUME`       | Mount point | `VOLUME ["/data"]` |
| `USER`         | Set user | `USER node` |
| `LABEL`        | Metadata | `LABEL version="1.0"` |
| `HEALTHCHECK`  | Health check | `HEALTHCHECK CMD curl -f http://localhost/` |
| `SHELL`        | Override shell | `SHELL ["/bin/bash", "-c"]` |
| `STOPSIGNAL`   | Stop signal | `STOPSIGNAL SIGTERM` |

### CMD vs ENTRYPOINT

```dockerfile
ENTRYPOINT ["node"]
CMD ["server.js"]
# docker run myapp           → node server.js
# docker run myapp other.js  → node other.js
```

### .dockerignore

```
node_modules
.git
.env
*.log
dist
.DS_Store
__pycache__
```

---

## 6. Docker Images

### Layer Caching (Best Practice Order)

```
Layer 5: COPY . .            ← Changes often → put LAST
Layer 4: RUN npm install     ← Changes when package.json changes
Layer 3: COPY package*.json  ← Rarely changes
Layer 2: WORKDIR /app
Layer 1: FROM node:18        ← Never changes → put FIRST
```

---

## 7. Docker Containers

### Restart Policies

```bash
docker run --restart=no nginx           # Never (default)
docker run --restart=always nginx       # Always
docker run --restart=on-failure nginx   # On crash
docker run --restart=unless-stopped nginx
```

### Health Check

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

---

## 8. Docker Volumes

| Type       | Command | Description |
|------------|---------|-------------|
| Named Vol  | `-v myvolume:/data` | Docker-managed, persistent |
| Bind Mount | `-v $(pwd):/app` | Maps host directory |
| tmpfs      | `--tmpfs /tmp` | RAM only, temporary |

```bash
docker volume create mydata && docker volume ls
docker run -v mydata:/app/data nginx
docker run -v $(pwd):/app:ro nginx
docker volume prune
```

---

## 9. Docker Networking

| Driver  | Description |
|---------|-------------|
| `bridge` | Default; isolated; DNS by container name (custom networks) |
| `host`   | Shares host network stack |
| `none`   | No networking |
| `overlay`| Multi-host (Swarm) |

```bash
docker network create mynet
docker run --network mynet --name db postgres
docker run --network mynet --name api myapi
# api can reach db at hostname "db"
docker run -p 8080:80 nginx
```

---

## 10. Docker Compose

### Example

```yaml
version: '3.8'
services:
  web:
    build: .
    ports: ["3000:3000"]
    environment: [NODE_ENV=development]
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      retries: 5

volumes:
  pgdata:
```

```bash
docker compose up -d && docker compose down
docker compose logs -f && docker compose exec web bash
docker compose build && docker compose ps
```

---

## 11. Multi-Stage Builds

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./ && RUN npm ci
COPY . . && RUN npm run build

FROM node:18-alpine AS production
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=builder --chown=app:app /app/dist ./dist
COPY --from=builder --chown=app:app /app/node_modules ./node_modules
USER app
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

---

## 12. Docker Registry

```bash
docker login && docker logout
docker tag myapp:latest myuser/myapp:latest
docker push myuser/myapp:latest
docker pull myuser/myapp:latest
```

---

## 13. Environment Variables & Secrets

```bash
docker run -e MY_VAR=value myapp
docker run --env-file .env myapp
```

```dockerfile
ARG VERSION=1.0         # Build time only
ENV NODE_ENV=production # Runtime
```

**Never** bake passwords into images. Use `--env-file` or Docker Secrets.

---

## 14. Resource Limits

```bash
docker run --memory="512m" --cpus="1.0" --pids-limit=100 nginx
```

```yaml
deploy:
  resources:
    limits:
      cpus: '0.5'
      memory: 512M
```

---

## 15. Logging & Monitoring

```bash
docker logs -f <container>
docker logs --tail 50 --timestamps <container>
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
docker top <container>
docker events --filter container=myapp
```

---

## 16. Docker Best Practices

```dockerfile
# ✅ Pin tags
FROM node:18.19-alpine3.19

# ✅ Layer cache optimization
COPY package*.json ./
RUN npm ci --only=production
COPY . .

# ✅ Non-root user
RUN addgroup -S app && adduser -S app -G app
USER app

# ✅ Exec form CMD
CMD ["node", "server.js"]

# ✅ Health check
HEALTHCHECK --interval=30s CMD curl -f http://localhost:3000/health

# ✅ Labels
LABEL maintainer="you@email.com" version="1.0.0"
```

**Security rules:** No secrets in images · Drop capabilities · Read-only FS · Scan images · Internal networks for backends

---

## 17. Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `permission denied` | Not in docker group | `sudo usermod -aG docker $USER` |
| `port already allocated` | Port in use | Change host port |
| `OOMKilled` | Out of memory | Increase `--memory` |
| Container exits immediately | CMD fails | `docker logs <container>` |
| `connection refused` between containers | Different networks | Use custom network with both containers |
| `no space left on device` | Disk full | `docker system prune -a` |
| `image not found` | Wrong name/tag | `docker pull` first |
| `COPY failed: file not found` | Bad path | Check `.dockerignore` and paths |

---

## 18. Quick Reference Cheat Sheet

```bash
# ── Images ──────────────────────────────────────
docker pull nginx:alpine && docker images
docker build -t myapp:v1 . && docker push myapp:v1
docker rmi nginx && docker image prune -a

# ── Containers ──────────────────────────────────
docker run -d -p 8080:80 --name web --restart=unless-stopped nginx
docker ps && docker ps -a
docker exec -it web bash && docker logs -f web
docker stop web && docker rm web
docker rm $(docker ps -aq)

# ── Volumes ──────────────────────────────────────
docker volume create mydata
docker run -v mydata:/data nginx
docker volume ls && docker volume prune

# ── Networks ─────────────────────────────────────
docker network create mynet
docker run --network mynet nginx
docker network ls && docker network prune

# ── Compose ──────────────────────────────────────
docker compose up -d && docker compose down -v
docker compose logs -f && docker compose exec web bash

# ── Cleanup ──────────────────────────────────────
docker system prune -a --volumes -f
```

---

## 📚 Further Learning

- [Official Docker Docs](https://docs.docker.com)
- [Docker Hub](https://hub.docker.com)
- [Play with Docker](https://labs.play-with-docker.com) — Free online playground
- [Docker Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Compose Specification](https://compose-spec.io)
- [Docker Security Guide](https://docs.docker.com/engine/security/)

---

## 🤝 Contributing

1. Fork the repo
2. Create a branch: `git checkout -b feat/my-addition`
3. Commit: `git commit -m "feat: add xyz notes"`
4. Push: `git push origin feat/my-addition`
5. Open a Pull Request

---

*Happy containerizing! 🐳*
