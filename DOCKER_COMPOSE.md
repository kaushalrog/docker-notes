# 🐙 Docker Compose — Complete Guide

## What is Docker Compose?

Docker Compose is a tool to define and run **multi-container applications** using a single YAML file (`docker-compose.yml`).

```
docker-compose.yml → defines services, networks, volumes
docker compose up  → creates and starts everything
docker compose down → stops and removes everything
```

---

## File Structure

```yaml
version: '3.8'          # Compose file format version

services:               # Container definitions
  service_name:
    ...

volumes:                # Named volume definitions
  volume_name:
    ...

networks:               # Network definitions
  network_name:
    ...

configs:                # Config file definitions (Swarm)
  config_name:
    ...

secrets:                # Secret definitions (Swarm)
  secret_name:
    ...
```

---

## Service Configuration — All Options

```yaml
services:
  myapp:
    # ── Image ──────────────────────────────────
    image: nginx:alpine                    # Use existing image
    # OR build from Dockerfile
    build:
      context: .                           # Build context path
      dockerfile: Dockerfile.prod          # Custom Dockerfile
      args:                               # Build args
        - NODE_ENV=production
        - VERSION=1.0
      target: production                   # Multi-stage target
      labels:
        - "com.example.version=1.0"
      cache_from:
        - myapp:latest
      platforms:
        - linux/amd64
        - linux/arm64

    # ── Identity ───────────────────────────────
    container_name: myapp                  # Custom container name
    hostname: myapp-host                   # Container hostname

    # ── Ports ──────────────────────────────────
    ports:
      - "3000:3000"                        # host:container
      - "127.0.0.1:8080:80"               # Bind to host IP
      - "3001-3005:3001-3005"             # Range
      - target: 80                         # Verbose form
        published: 8080
        protocol: tcp
        mode: host

    # ── Environment ────────────────────────────
    environment:
      - NODE_ENV=production
      - PORT=3000
      DB_HOST: db                          # YAML map form
      DB_PORT: 5432
    env_file:
      - .env
      - .env.production

    # ── Volumes ────────────────────────────────
    volumes:
      - mydata:/app/data                   # Named volume
      - ./config:/app/config:ro            # Bind mount (read-only)
      - /tmp:/tmp                          # Absolute bind mount
      - type: bind                         # Verbose form
        source: ./config
        target: /app/config
        read_only: true
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 100m

    # ── Networks ───────────────────────────────
    networks:
      - frontend
      - backend
    # OR with options:
    networks:
      frontend:
        aliases:
          - web
          - myapp
        ipv4_address: 172.16.0.10

    # ── Dependencies ───────────────────────────
    depends_on:
      - db                                  # Simple
    depends_on:                             # With condition
      db:
        condition: service_healthy
      redis:
        condition: service_started
      migrate:
        condition: service_completed_successfully

    # ── Restart ────────────────────────────────
    restart: "no"               # Never restart (default)
    restart: always             # Always restart
    restart: on-failure         # On non-zero exit
    restart: unless-stopped     # Always except manual stop

    # ── Health Check ───────────────────────────
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
      disable: false            # Set true to disable

    # ── Resources ──────────────────────────────
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3

    # ── Logging ────────────────────────────────
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

    # ── Commands ───────────────────────────────
    command: npm start                      # Override CMD
    entrypoint: ["/entrypoint.sh"]         # Override ENTRYPOINT
    working_dir: /app                       # Working directory
    user: "1000:1000"                       # User:Group

    # ── Security ───────────────────────────────
    read_only: true                         # Read-only filesystem
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    privileged: false

    # ── Labels ─────────────────────────────────
    labels:
      - "com.example.env=production"
      - "traefik.enable=true"

    # ── Extra Hosts ────────────────────────────
    extra_hosts:
      - "host.docker.internal:host-gateway"
      - "myhost:192.168.1.100"

    # ── Sysctls ────────────────────────────────
    sysctls:
      - net.core.somaxconn=1024
      - net.ipv4.tcp_syncookies=0

    # ── TTY / STDIN ────────────────────────────
    tty: true
    stdin_open: true            # -i flag

    # ── Init ───────────────────────────────────
    init: true                  # Use tini as PID 1
```

---

## Complete Production Example

### Project Structure

```
myapp/
├── docker-compose.yml
├── docker-compose.override.yml     # Dev overrides
├── docker-compose.prod.yml         # Prod overrides
├── .env
├── .env.example
├── Dockerfile
├── nginx/
│   └── nginx.conf
└── scripts/
    └── init.sql
```

### `docker-compose.yml` (Base)

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      - NODE_ENV=${NODE_ENV:-production}
      - DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD} --appendonly yes
    volumes:
      - redisdata:/data
    networks:
      - backend
    restart: unless-stopped

volumes:
  pgdata:
  redisdata:

networks:
  backend:
    driver: bridge
```

### `docker-compose.override.yml` (Dev — auto-loaded)

```yaml
version: '3.8'

services:
  api:
    build:
      target: development
    volumes:
      - .:/app
      - /app/node_modules
    ports:
      - "3000:3000"
      - "9229:9229"    # Node.js debugger
    environment:
      - NODE_ENV=development
      - DEBUG=app:*
    command: npm run dev

  db:
    ports:
      - "5432:5432"    # Expose for local tools

  redis:
    ports:
      - "6379:6379"    # Expose for local tools

  # Dev tools
  adminer:
    image: adminer
    ports:
      - "8080:8080"
    networks:
      - backend
```

### `docker-compose.prod.yml` (Production)

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/ssl/certs:ro
      - staticfiles:/usr/share/nginx/html:ro
    depends_on:
      - api
    networks:
      - frontend
      - backend
    restart: always

  api:
    image: ${REGISTRY}/myapp:${TAG:-latest}    # Pre-built image in prod
    volumes:
      - staticfiles:/app/public
    networks:
      - frontend
      - backend
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

volumes:
  staticfiles:

networks:
  frontend:
    driver: bridge
```

---

## `.env` File

```env
# Application
NODE_ENV=development
APP_PORT=3000

# Database
DB_USER=admin
DB_PASSWORD=strongpassword123
DB_NAME=myappdb

# Redis
REDIS_PASSWORD=redispassword123

# Registry (for production)
REGISTRY=123456789.dkr.ecr.us-east-1.amazonaws.com
TAG=latest
```

---

## Compose Commands Reference

```bash
# Using multiple files
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Specific service
docker compose up api
docker compose stop db
docker compose restart redis

# Scale
docker compose up --scale api=3

# Override entrypoint
docker compose run --entrypoint /bin/sh api

# Run migrations before starting
docker compose run --rm api npm run migrate
docker compose up -d
```

---

## Profiles (Conditional Services)

```yaml
version: '3.8'

services:
  api:
    image: myapi           # Always runs

  adminer:
    image: adminer
    profiles: ["tools"]    # Only with --profile tools

  debug:
    image: myapi
    profiles: ["debug"]
    command: npm run debug
```

```bash
docker compose up                          # Only api
docker compose --profile tools up          # api + adminer
docker compose --profile tools --profile debug up
```

---

## Extends (Reusable Config)

```yaml
# base.yml
services:
  base:
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"

# docker-compose.yml
services:
  api:
    extends:
      file: base.yml
      service: base
    image: myapi
```

---

## Healthcheck Patterns

```yaml
# HTTP
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
  interval: 30s

# PostgreSQL
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres"]
  interval: 10s

# MySQL
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  interval: 10s

# Redis
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 10s

# Custom script
healthcheck:
  test: ["CMD", "/health-check.sh"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

---

## Common Patterns

### Run Database Migrations

```yaml
services:
  migrate:
    build: .
    command: npm run migrate
    depends_on:
      db:
        condition: service_healthy
    restart: "no"    # Run once and exit

  api:
    build: .
    depends_on:
      migrate:
        condition: service_completed_successfully
```

### Seed Data

```yaml
  seed:
    build: .
    command: npm run seed
    depends_on:
      migrate:
        condition: service_completed_successfully
    profiles: ["seed"]
```

### Sidecar Logging

```yaml
services:
  app:
    image: myapp
    volumes:
      - logs:/var/log/app

  log-shipper:
    image: fluentd
    volumes:
      - logs:/var/log/app:ro
    depends_on:
      - app

volumes:
  logs:
```
