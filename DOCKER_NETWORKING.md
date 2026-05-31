# 🌐 Docker Networking — Deep Dive

## Network Drivers

| Driver    | Use Case | Scope |
|-----------|----------|-------|
| `bridge`  | Default single-host communication | Local |
| `host`    | No isolation, maximum performance | Local |
| `none`    | No networking at all | Local |
| `overlay` | Multi-host (Docker Swarm/Kubernetes) | Swarm |
| `macvlan` | Assign real MAC addresses | Local |
| `ipvlan`  | IP-level isolation | Local |

---

## Bridge Network (Default)

Every container gets its own IP. Containers communicate by IP or name (on custom networks).

```bash
# Default bridge — no DNS resolution by name
docker run -d --name app1 nginx
docker run -d --name app2 nginx
# app2 CANNOT reach app1 by name on default bridge

# Custom bridge — DNS resolution by container name ✅
docker network create mynet
docker run -d --name app1 --network mynet nginx
docker run -d --name app2 --network mynet nginx
# app2 CAN reach app1 at hostname "app1"
```

### Creating Custom Bridge Networks

```bash
docker network create mynet

docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  --ip-range 172.20.5.0/24 \
  mynet

# Run container with static IP
docker run --network mynet --ip 172.20.0.10 nginx
```

---

## Host Network

Container shares host's network namespace — no port mapping needed, max performance.

```bash
docker run --network host nginx
# Nginx now listens on host port 80 directly
# No -p flag needed/supported with host networking
```

> ⚠️ Only works on Linux. On macOS/Windows Docker Desktop, host = VM's host, not your Mac.

---

## None Network

```bash
docker run --network none nginx
# Container has no network interfaces except loopback
# Use for security-sensitive workloads
```

---

## Container-to-Container Communication

### Same Network (Recommended)

```bash
docker network create appnet

# Database
docker run -d \
  --name postgres \
  --network appnet \
  -e POSTGRES_PASSWORD=secret \
  postgres:15

# API connects to postgres using hostname "postgres"
docker run -d \
  --name api \
  --network appnet \
  -e DB_HOST=postgres \
  myapi
```

### Connect to Multiple Networks

```bash
docker network create frontend
docker network create backend

docker run -d --name api --network backend myapi
docker network connect frontend api
# api is now on BOTH frontend and backend
```

---

## Port Publishing

```bash
# Map host port 8080 → container port 80
docker run -p 8080:80 nginx

# Bind to specific host IP
docker run -p 127.0.0.1:8080:80 nginx

# Random host port
docker run -p 80 nginx         # Docker picks host port
docker run -P nginx            # Publish ALL exposed ports

# UDP port
docker run -p 53:53/udp dns-server

# Multiple ports
docker run -p 80:80 -p 443:443 nginx

# Show port mappings
docker port <container>
docker port <container> 80
```

---

## DNS in Docker

Custom bridge networks have a built-in DNS server at `127.0.0.11`.

```bash
# Containers can reach each other by:
# 1. Container name (--name)
# 2. Network alias (--network-alias)
# 3. Service name (in Docker Compose)

docker run -d --name mydb --network appnet postgres
docker run -d --name myapi --network appnet myapi
# myapi can connect to mydb at:
# host = "mydb", port = 5432

# Multiple aliases for one container
docker run -d \
  --name db \
  --network appnet \
  --network-alias database \
  --network-alias postgres \
  postgres
```

---

## Docker Compose Networking

By default, Compose creates one network per project. All services join it automatically.

```yaml
version: '3.8'

services:
  web:
    image: nginx
    networks:
      - frontend

  api:
    image: myapi
    networks:
      - frontend
      - backend

  db:
    image: postgres
    networks:
      - backend        # db is isolated from frontend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true     # No external internet access
```

### Network Aliases in Compose

```yaml
services:
  db:
    image: postgres
    networks:
      backend:
        aliases:
          - database
          - postgres-db

networks:
  backend:
```

---

## Inspect & Debug Networks

```bash
# List all networks
docker network ls

# Detailed info including connected containers
docker network inspect mynet

# Find container's IP address
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mycontainer

# All containers and their IPs in a network
docker network inspect mynet \
  --format='{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{"\n"}}{{end}}'

# Test connectivity between containers
docker exec api ping db
docker exec api curl http://db:5432
docker exec api nslookup db

# Check container's network settings
docker inspect mycontainer | grep -A 20 "NetworkSettings"
```

---

## Overlay Network (Docker Swarm)

For multi-host communication:

```bash
# Init swarm first
docker swarm init

# Create overlay network
docker network create --driver overlay myoverlay
docker network create \
  --driver overlay \
  --subnet 10.10.0.0/16 \
  --attachable \
  myoverlay

# Deploy services on overlay
docker service create \
  --name api \
  --network myoverlay \
  myapi

docker service create \
  --name db \
  --network myoverlay \
  postgres
```

---

## Macvlan Network

Give container a real MAC address on physical network:

```bash
docker network create \
  --driver macvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --opt parent=eth0 \
  mymacvlan

docker run --network mymacvlan --ip 192.168.1.50 nginx
# Container appears as physical device on your LAN
```

---

## Firewall & iptables

Docker manages iptables rules automatically. Key points:

- Docker adds a `DOCKER` chain to iptables
- Published ports bypass UFW rules by default!
- To restrict with UFW on Ubuntu, add to `/etc/docker/daemon.json`:

```json
{
  "iptables": false
}
```

Or bind to localhost only:

```bash
docker run -p 127.0.0.1:5432:5432 postgres
# Now only accessible from localhost, not from network
```

---

## Network Troubleshooting

```bash
# Container can't reach internet
docker run --rm alpine ping google.com
# Check daemon DNS: cat /etc/docker/daemon.json

# Container can't reach other container
docker exec app1 ping app2
# Are they on the same network? docker network inspect mynet

# Port not accessible from host
docker port mycontainer
curl http://localhost:8080
# Check: docker ps | grep mycontainer

# Check what's listening inside container
docker exec mycontainer netstat -tlnp
docker exec mycontainer ss -tlnp

# Debug with temporary netshoot container
docker run --rm -it \
  --network container:mycontainer \
  nicolaka/netshoot \
  bash
```

---

## Common Network Patterns

### Frontend / Backend Separation

```
Internet → [Nginx: frontend network] → [API: frontend+backend] → [DB: backend network]
```

### Sidecar Pattern

```bash
docker run -d --name app myapp
docker run -d \
  --network container:app \   # Share app's network namespace
  --name sidecar \
  mysidecar
# Sidecar reaches app at localhost
```

### Service Mesh (basic)

```yaml
services:
  proxy:
    image: envoyproxy/envoy
    networks: [mesh]
    ports: ["10000:10000"]

  service_a:
    image: service-a
    networks: [mesh]

  service_b:
    image: service-b
    networks: [mesh]

networks:
  mesh:
    driver: bridge
```
