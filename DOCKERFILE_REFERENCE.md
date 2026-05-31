# Dockerfile Reference Guide

## All Instructions with Examples

---

### FROM — Base Image

```dockerfile
FROM ubuntu:22.04
FROM python:3.11-slim
FROM node:18-alpine
FROM scratch                    # Empty image (for Go binaries)
FROM busybox:latest

# Multi-stage: name a stage
FROM node:18 AS builder
FROM nginx:alpine AS production

# Build for specific platform
FROM --platform=linux/amd64 node:18
```

---

### WORKDIR — Working Directory

```dockerfile
WORKDIR /app                    # Created if doesn't exist
WORKDIR /usr/src/app
WORKDIR /home/node/app

# Subsequent instructions run from WORKDIR
WORKDIR /app
RUN pwd                         # prints /app
```

---

### COPY — Copy Files

```dockerfile
COPY . .                        # All from context to WORKDIR
COPY src/ /app/src/             # Specific directory
COPY package*.json ./           # Wildcard
COPY --chown=node:node . .      # With ownership
COPY --from=builder /app/dist ./dist   # From another stage
COPY ["file name.txt", "/app/"] # Paths with spaces
```

---

### ADD — Advanced Copy

```dockerfile
ADD . .                         # Same as COPY
ADD https://example.com/file.tar.gz /tmp/   # Download URL
ADD app.tar.gz /app/            # Auto-extract tar
# Prefer COPY unless you need URL download or tar extraction
```

---

### RUN — Execute Commands at Build Time

```dockerfile
# Shell form (runs in /bin/sh -c)
RUN apt-get update && apt-get install -y curl wget

# Exec form (no shell, no variable expansion)
RUN ["apt-get", "install", "-y", "curl"]

# Best practice: chain commands, clean up in same layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      curl \
      wget \
      git && \
    rm -rf /var/lib/apt/lists/*

# Alpine
RUN apk add --no-cache curl wget git

# Python
RUN pip install --no-cache-dir -r requirements.txt

# Node
RUN npm ci --only=production
```

---

### CMD — Default Container Command

```dockerfile
# Exec form (preferred)
CMD ["node", "server.js"]
CMD ["python", "app.py"]
CMD ["nginx", "-g", "daemon off;"]
CMD ["/bin/bash"]

# Shell form
CMD node server.js
CMD python app.py

# Only one CMD per Dockerfile (last one wins)
# Can be overridden at runtime: docker run myapp python other.py
```

---

### ENTRYPOINT — Fixed Entrypoint

```dockerfile
# Exec form (preferred)
ENTRYPOINT ["python"]
ENTRYPOINT ["docker-entrypoint.sh"]
ENTRYPOINT ["/bin/sh", "-c"]

# Combine with CMD for default args
ENTRYPOINT ["python"]
CMD ["app.py"]
# docker run myapp          → python app.py
# docker run myapp other.py → python other.py

# Override at runtime with --entrypoint
# docker run --entrypoint /bin/bash myapp
```

---

### ENV — Environment Variables

```dockerfile
ENV NODE_ENV=production
ENV PORT=3000
ENV DATABASE_URL=postgresql://localhost/mydb

# Multiple in one instruction
ENV NODE_ENV=production \
    PORT=3000 \
    LOG_LEVEL=info

# Used in subsequent instructions
ENV APP_HOME=/app
WORKDIR $APP_HOME
COPY . $APP_HOME

# Available at runtime too
```

---

### ARG — Build-Time Variables

```dockerfile
ARG VERSION=1.0
ARG NODE_ENV=development
ARG REGISTRY=docker.io

# Use in build
FROM node:${NODE_ENV:-18}-alpine
RUN echo "Building version $VERSION"

# Pass at build time
# docker build --build-arg VERSION=2.0 .

# ARG before FROM is available in FROM only
ARG BASE_TAG=18-alpine
FROM node:${BASE_TAG}
# ARG must be declared again after FROM to be available
ARG BASE_TAG
RUN echo $BASE_TAG

# Unlike ENV, ARG values are NOT available at runtime
```

---

### EXPOSE — Document Port

```dockerfile
EXPOSE 80
EXPOSE 443
EXPOSE 3000
EXPOSE 8080/tcp
EXPOSE 9229/tcp     # Node.js debug port

# EXPOSE is documentation only — doesn't actually publish
# Use -p or -P at runtime to publish
```

---

### VOLUME — Mount Points

```dockerfile
VOLUME ["/data"]
VOLUME ["/data", "/logs"]
VOLUME /var/lib/mysql
VOLUME /app/uploads

# Creates anonymous volume at runtime if not mounted
# Useful for databases, file uploads
```

---

### USER — Set User

```dockerfile
# Create user first
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Switch to user
USER appuser
USER node                   # node user exists in node images
USER 1000                   # By UID
USER 1000:1000              # UID:GID

# Root for installs, then switch
RUN apt-get install -y curl
USER nobody
```

---

### LABEL — Metadata

```dockerfile
LABEL maintainer="dev@example.com"
LABEL version="1.0.0"
LABEL description="My application"

# Multiple labels
LABEL maintainer="dev@example.com" \
      version="1.0.0" \
      description="My application" \
      org.opencontainers.image.source="https://github.com/user/repo"
```

---

### HEALTHCHECK — Container Health

```dockerfile
# HTTP check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# TCP check
HEALTHCHECK --interval=10s --timeout=5s \
  CMD nc -z localhost 5432 || exit 1

# Command check
HEALTHCHECK CMD ["python", "-c", "import app; app.health_check()"]

# Disable health check from base image
HEALTHCHECK NONE

# Options:
# --interval=30s    (default: 30s) how often to check
# --timeout=30s     (default: 30s) timeout per check
# --start-period=0s (default: 0s)  grace period on start
# --retries=3       (default: 3)   failures before unhealthy
```

---

### SHELL — Override Default Shell

```dockerfile
# Default is ["/bin/sh", "-c"] on Linux
SHELL ["/bin/bash", "-c"]
SHELL ["/bin/bash", "-o", "pipefail", "-c"]    # Strict mode
SHELL ["powershell", "-command"]               # Windows

RUN echo $HOME    # Now uses bash
```

---

### ONBUILD — Trigger on Child Image

```dockerfile
# In base image:
ONBUILD COPY . /app
ONBUILD RUN npm install

# When someone does: FROM mybase
# These instructions run automatically
```

---

### STOPSIGNAL — Container Stop Signal

```dockerfile
STOPSIGNAL SIGTERM    # Default
STOPSIGNAL SIGINT
STOPSIGNAL 9          # SIGKILL (numeric)
```

---

## Complete Real-World Examples

### Node.js (Production)

```dockerfile
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine AS production
LABEL maintainer="dev@example.com"
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json .
ENV NODE_ENV=production PORT=3000
USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

### Python (FastAPI)

```dockerfile
FROM python:3.11-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app

FROM base AS deps
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM base AS production
COPY --from=deps /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH
COPY . .
RUN adduser --disabled-password --gecos "" appuser && chown -R appuser /app
USER appuser
EXPOSE 8000
HEALTHCHECK CMD curl -f http://localhost:8000/health || exit 1
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Go (Minimal)

```dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o server .

FROM scratch
COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
ENTRYPOINT ["/server"]
```

### Nginx (Static Site)

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Java (Spring Boot)

```dockerfile
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app
COPY .mvn/ .mvn
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline
COPY src ./src
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S spring && adduser -S spring -G spring
COPY --from=builder /app/target/*.jar app.jar
USER spring
EXPOSE 8080
HEALTHCHECK CMD curl -f http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]
```
