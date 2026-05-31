# 💾 Docker Volumes — Deep Dive

## Types of Storage

```
┌───────────────────────────────────────────────────────────┐
│                       Host Machine                         │
│                                                           │
│  /var/lib/docker/volumes/  ←── Named Volumes (managed)   │
│  /your/custom/path/        ←── Bind Mounts (you control) │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                   Container                         │  │
│  │   /data        ←── Volume Mount Point               │  │
│  │   /tmp         ←── tmpfs (RAM only)                 │  │
│  └─────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────┘
```

| Type | Persistence | Performance | Use Case |
|------|-------------|-------------|----------|
| Named Volume | ✅ Yes | Good | Databases, app data |
| Bind Mount | ✅ Yes | Best (native) | Dev, config files |
| tmpfs | ❌ RAM only | Fastest | Secrets, temp files |
| Anonymous Volume | ✅ Until removed | Good | One-off containers |

---

## Named Volumes

Managed by Docker. Stored in `/var/lib/docker/volumes/`.

```bash
# Create
docker volume create mydata
docker volume create \
  --driver local \
  --label project=myapp \
  mydata

# Use
docker run -v mydata:/app/data nginx
docker run --mount type=volume,source=mydata,target=/app/data nginx

# Inspect
docker volume ls
docker volume inspect mydata
# Output shows Mountpoint: /var/lib/docker/volumes/mydata/_data

# Remove
docker volume rm mydata
docker volume prune          # Remove all unused volumes
docker volume prune -f       # Without confirmation
```

---

## Bind Mounts

Map a host directory or file directly into the container.

```bash
# Directory
docker run -v /host/path:/container/path nginx
docker run -v $(pwd):/app nginx                     # Current dir
docker run -v $(pwd)/config:/app/config:ro nginx    # Read-only

# File
docker run -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro nginx

# Using --mount (explicit, preferred)
docker run \
  --mount type=bind,source=$(pwd),target=/app \
  nginx

# Read-only bind mount
docker run \
  --mount type=bind,source=$(pwd),target=/app,readonly \
  nginx
```

### Dev Workflow with Bind Mount

```bash
# Live code reload in development
docker run -d \
  --name dev \
  -v $(pwd):/app \
  -v /app/node_modules \   # Anonymous volume — don't overwrite node_modules
  -p 3000:3000 \
  node:18 \
  npm run dev
```

---

## tmpfs Mounts (In-Memory)

Data is stored in RAM. Lost when container stops.

```bash
docker run --tmpfs /tmp nginx
docker run --tmpfs /tmp:size=100m,mode=1777 nginx

# Using --mount
docker run \
  --mount type=tmpfs,destination=/tmp \
  nginx

docker run \
  --mount type=tmpfs,destination=/tmp,tmpfs-size=100m \
  nginx

# Use cases:
# - Secrets that shouldn't touch disk
# - High-performance temp files
# - Session data
```

---

## Volume in Docker Compose

```yaml
version: '3.8'

services:
  db:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data       # Named volume
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro  # Bind mount
      - /tmp/pgtemp:/tmp                      # Bind mount (absolute)

  app:
    image: myapp
    volumes:
      - .:/app                                # Dev bind mount
      - /app/node_modules                     # Anonymous (keep container's)
      - appdata:/app/data                     # Named volume
      - type: tmpfs
        target: /tmp                          # tmpfs

volumes:
  pgdata:                                     # Docker-managed
  appdata:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/myapp                     # Bind to specific host path
  external_vol:
    external: true                            # Pre-existing volume
    name: my-existing-volume
```

---

## Volume Drivers

```bash
# Local driver (default)
docker volume create --driver local mydata

# NFS
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/nfs/share \
  nfs-volume

# CIFS / SMB (Windows shares)
docker volume create \
  --driver local \
  --opt type=cifs \
  --opt o=addr=server,username=user,password=pass \
  --opt device=//server/share \
  smb-volume

# tmpfs
docker volume create \
  --driver local \
  --opt type=tmpfs \
  --opt o=size=100m \
  tmpfs-volume
```

---

## Backup & Restore

### Backup Named Volume

```bash
# Method 1: Using a temp container
docker run --rm \
  -v mydata:/source \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/mydata-backup.tar.gz -C /source .

# Method 2: Copy from running container
docker exec mycontainer tar czf - /data > backup.tar.gz
```

### Restore Named Volume

```bash
# Create volume first
docker volume create mydata

# Restore
docker run --rm \
  -v mydata:/target \
  -v $(pwd):/backup \
  alpine \
  tar xzf /backup/mydata-backup.tar.gz -C /target

# Verify
docker run --rm -v mydata:/data alpine ls /data
```

### Backup Database Volume

```bash
# PostgreSQL
docker exec postgres pg_dump -U postgres mydb > backup.sql

# MySQL
docker exec mysql mysqldump -u root -p mydb > backup.sql

# Restore PostgreSQL
docker exec -i postgres psql -U postgres mydb < backup.sql
```

---

## Volume Permissions

Common issue: file permission conflicts between host and container.

```bash
# Check container UID
docker run --rm node:18 id
# uid=1000(node) gid=1000(node)

# Set ownership in Dockerfile
RUN mkdir -p /app/data && chown -R node:node /app/data
VOLUME /app/data
USER node

# Fix at runtime
docker run -v mydata:/app/data \
  --user $(id -u):$(id -g) \
  myapp

# Named volume with correct permissions
docker volume create mydata
docker run --rm \
  -v mydata:/data \
  alpine \
  chown -R 1000:1000 /data
```

---

## Inspect Volume Data

```bash
# View volume location on host
docker volume inspect mydata
# "Mountpoint": "/var/lib/docker/volumes/mydata/_data"

# Browse volume contents (via temp container)
docker run --rm -it \
  -v mydata:/data \
  alpine sh

# Or directly (as root)
ls /var/lib/docker/volumes/mydata/_data

# Check what containers use a volume
docker ps -a --filter volume=mydata
```

---

## Volume Patterns

### Database Persistence

```yaml
services:
  db:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./scripts:/docker-entrypoint-initdb.d:ro  # Init scripts

volumes:
  pgdata:
```

### Config Files

```yaml
services:
  nginx:
    image: nginx
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/ssl/certs:ro
      - ./html:/usr/share/nginx/html:ro
```

### Shared Storage Between Services

```yaml
services:
  uploader:
    image: uploader
    volumes:
      - uploads:/app/uploads

  processor:
    image: processor
    volumes:
      - uploads:/app/input:ro   # Read-only access

volumes:
  uploads:
```

### Development Hot Reload

```yaml
services:
  app:
    image: node:18
    working_dir: /app
    command: npm run dev
    volumes:
      - .:/app                    # Source code
      - /app/node_modules         # Preserve container's node_modules
      - node_cache:/root/.npm     # NPM cache
    ports:
      - "3000:3000"

volumes:
  node_cache:
```

---

## Cleanup

```bash
# Remove specific volume
docker volume rm mydata

# Remove all unused volumes
docker volume prune

# Remove volumes when stopping compose
docker compose down -v

# Remove anonymous volumes when removing container
docker rm -v mycontainer

# Find orphaned volumes (not attached to any container)
docker volume ls -qf dangling=true
docker volume rm $(docker volume ls -qf dangling=true)
```
