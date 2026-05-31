# 🔒 Docker Security — Best Practices

## Top Security Rules

1. Never run as root
2. Use minimal base images
3. Never bake secrets into images
4. Scan images regularly
5. Restrict capabilities
6. Use read-only filesystems where possible
7. Keep Docker and images updated

---

## 1. Don't Run as Root

### In Dockerfile

```dockerfile
# ❌ Bad — runs as root by default
FROM node:18
COPY . .
CMD ["node", "server.js"]

# ✅ Good — create and use non-root user
FROM node:18-alpine

# node:18 already has a 'node' user
RUN mkdir -p /app && chown -R node:node /app
WORKDIR /app
USER node

COPY --chown=node:node . .
RUN npm ci --only=production
CMD ["node", "server.js"]
```

```dockerfile
# Create user from scratch (Alpine)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Create user from scratch (Debian/Ubuntu)
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser
```

### At Runtime

```bash
docker run --user 1000:1000 nginx
docker run --user nobody nginx

# Check current user inside container
docker exec myapp whoami
docker exec myapp id
```

---

## 2. Use Minimal Base Images

```dockerfile
# ❌ Large attack surface
FROM ubuntu:22.04

# ✅ Minimal Alpine
FROM node:18-alpine        # ~50MB vs ~900MB

# ✅ Slim variants
FROM python:3.11-slim
FROM debian:bookworm-slim

# ✅ Distroless (Google) — no shell, no package manager
FROM gcr.io/distroless/nodejs18-debian11

# ✅ Scratch — empty, for static Go binaries
FROM scratch
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

### Image Size Comparison

| Base Image | Approx Size |
|-----------|-------------|
| `ubuntu:22.04` | ~70MB |
| `debian:bookworm-slim` | ~75MB |
| `alpine:3.18` | ~7MB |
| `node:18` | ~950MB |
| `node:18-slim` | ~240MB |
| `node:18-alpine` | ~50MB |
| `gcr.io/distroless/nodejs18` | ~30MB |
| `scratch` | 0MB |

---

## 3. Never Bake Secrets into Images

### ❌ Wrong Ways

```dockerfile
# BAD: Secret visible in image layer and history
ENV DB_PASSWORD=mysecret123

# BAD: Secret baked into ARG layer
ARG DB_PASSWORD
RUN echo $DB_PASSWORD > /app/config

# BAD: Copied .env file
COPY .env .
```

### ✅ Right Ways

```bash
# Pass at runtime
docker run -e DB_PASSWORD=$DB_PASSWORD myapp

# From file (not committed to git)
docker run --env-file .env myapp

# Docker Secrets (Swarm)
echo "mysecret" | docker secret create db_password -
docker service create --secret db_password myapp
# Read at /run/secrets/db_password

# Kubernetes: use Secrets or external vaults
# AWS: use Secrets Manager or Parameter Store
# HashiCorp Vault integration
```

```dockerfile
# Read secret from file in entrypoint
ENTRYPOINT ["/entrypoint.sh"]
```

```bash
#!/bin/sh
# entrypoint.sh
export DB_PASSWORD=$(cat /run/secrets/db_password)
exec "$@"
```

---

## 4. Restrict Linux Capabilities

Containers by default have some capabilities. Remove what you don't need.

```bash
# Drop ALL, add only what's needed
docker run \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \    # Allow binding to port <1024
  nginx

# Common safe caps to add:
# NET_BIND_SERVICE — bind to ports < 1024
# CHOWN — change file ownership
# SETUID/SETGID — change uid/gid

# Never add in production:
# SYS_ADMIN — almost full root
# NET_ADMIN — full network control
# SYS_PTRACE — can debug other processes
```

```yaml
# Docker Compose
services:
  app:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

---

## 5. Read-Only Filesystem

```bash
docker run --read-only nginx

# Allow writes only to specific dirs
docker run \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /var/run \
  --tmpfs /var/cache/nginx \
  nginx
```

```yaml
services:
  app:
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
```

---

## 6. Seccomp, AppArmor, SELinux

```bash
# Use default seccomp profile (blocks ~300 syscalls)
# Already applied by default in Docker

# Custom seccomp profile
docker run \
  --security-opt seccomp=/path/to/seccomp.json \
  nginx

# Disable seccomp (don't do this in production)
docker run --security-opt seccomp=unconfined nginx

# AppArmor
docker run --security-opt apparmor=docker-default nginx

# No new privileges (prevent privilege escalation)
docker run --security-opt no-new-privileges:true nginx
```

```yaml
services:
  app:
    security_opt:
      - no-new-privileges:true
      - seccomp:/path/to/seccomp.json
```

---

## 7. Image Scanning

```bash
# Docker Scout (built into Docker Desktop)
docker scout cves myapp:latest
docker scout recommendations myapp:latest
docker scout compare myapp:v1 myapp:v2

# Trivy (open source, widely used)
trivy image nginx:latest
trivy image --severity HIGH,CRITICAL myapp:latest
trivy image --exit-code 1 --severity CRITICAL myapp  # Fail on critical

# Snyk
snyk container test myapp:latest

# Grype
grype myapp:latest
```

---

## 8. Content Trust (Image Signing)

```bash
# Enable Docker Content Trust
export DOCKER_CONTENT_TRUST=1

# Now pull/push only signed images
docker pull nginx
docker push myuser/myapp:v1.0
```

---

## 9. Network Security

```bash
# Don't expose ports on 0.0.0.0 unnecessarily
# ❌ Exposed to all interfaces
docker run -p 5432:5432 postgres

# ✅ Only accessible locally
docker run -p 127.0.0.1:5432:5432 postgres

# Use internal networks — no external access
docker network create --internal backend
```

```yaml
services:
  db:
    networks:
      - backend

networks:
  backend:
    internal: true    # No internet access
```

---

## 10. Rootless Docker

Run Docker daemon as non-root user:

```bash
# Install rootless Docker
dockerd-rootless-setuptool.sh install

# Run as current user
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
docker run nginx
```

---

## 11. Resource Limits

Prevent DoS from runaway containers:

```bash
docker run \
  --memory="512m" \
  --memory-swap="512m" \         # Disable swap
  --cpus="0.5" \
  --pids-limit=100 \             # Limit processes (fork bomb prevention)
  --ulimit nofile=1024:1024 \    # File descriptors
  nginx
```

---

## 12. Daemon Security

`/etc/docker/daemon.json`:

```json
{
  "icc": false,                        // Disable inter-container communication
  "userns-remap": "default",           // User namespace remapping
  "no-new-privileges": true,           // Global no-new-privileges
  "seccomp-profile": "/path/seccomp.json",
  "live-restore": true,                // Keep containers running on daemon restart
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

---

## Security Checklist

```
Image Security:
☐ Use minimal base image (alpine/slim/distroless)
☐ Pin base image to specific digest or tag
☐ No secrets in image layers
☐ Run image scanner (Trivy/Scout/Snyk)
☐ Label and sign images

Runtime Security:
☐ Run as non-root user
☐ Drop all unnecessary capabilities
☐ Read-only filesystem (+ tmpfs for needed writes)
☐ Set memory and CPU limits
☐ No --privileged flag
☐ no-new-privileges:true
☐ Bind ports to 127.0.0.1 if only needed locally

Network Security:
☐ Use internal networks for backend services
☐ Minimize exposed ports
☐ Use mTLS for service-to-service communication

Operational Security:
☐ Regularly update base images
☐ Enable Docker Content Trust
☐ Restrict Docker socket access
☐ Monitor container activity
☐ Use Docker Secrets for sensitive data
```
