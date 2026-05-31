# 🔧 Docker Troubleshooting Guide

## Quick Diagnosis Checklist

```bash
docker version                      # Is Docker running?
docker info                         # Any system warnings?
docker ps -a                        # Container state?
docker logs <container>             # What's the error?
docker inspect <container>          # Full config details
docker stats --no-stream            # Resource usage?
docker system df                    # Disk space?
```

---

## Container Issues

### Container Exits Immediately

```bash
# Check logs
docker logs <container>
docker logs --tail 50 <container>

# Run interactively to debug
docker run -it myapp bash
docker run -it myapp sh             # Alpine

# Override CMD
docker run -it --entrypoint bash myapp
docker run -it --entrypoint sh myapp

# Common causes:
# 1. Application crash — check logs
# 2. Missing env variable
# 3. CMD is wrong
# 4. No foreground process (daemon mode exits)
```

**Fix: Keep container alive for debugging**
```dockerfile
# Temporary: override CMD
CMD ["tail", "-f", "/dev/null"]
```

### Container Keeps Restarting (Restart Loop)

```bash
# Check restart count
docker inspect --format='{{.RestartCount}}' <container>

# View logs from previous run
docker logs --previous <container>

# Disable restart policy temporarily
docker update --restart=no <container>
docker stop <container>
docker logs <container>
```

### Container Won't Stop

```bash
docker stop <container>           # Sends SIGTERM, waits 10s
docker kill <container>           # Sends SIGKILL immediately
docker rm -f <container>          # Force remove

# If daemon is hung
sudo systemctl restart docker
```

### Can't Remove Container

```bash
# Error: container is running
docker stop <container> && docker rm <container>

# Error: container has volumes
docker rm -v <container>

# Force remove everything
docker rm -f $(docker ps -aq)
```

---

## Port & Networking Issues

### "Port already in use"

```bash
# Find what's using the port
sudo lsof -i :8080
sudo netstat -tulpn | grep 8080
sudo ss -tulpn | grep 8080

# Kill the process OR use a different port
docker run -p 8081:80 nginx       # Different host port

# Find which container is using it
docker ps --format "table {{.Names}}\t{{.Ports}}"
```

### Can't Access Container from Host

```bash
# Check port mapping
docker port <container>
docker ps | grep <container>

# Test from inside container
docker exec <container> curl http://localhost:80

# Check container is listening
docker exec <container> netstat -tlnp
docker exec <container> ss -tlnp

# Is it binding to 0.0.0.0 or 127.0.0.1?
docker exec <container> ss -tlnp | grep LISTEN

# Check firewall
sudo iptables -L DOCKER -n
sudo ufw status
```

### Containers Can't Talk to Each Other

```bash
# Are they on the same network?
docker network inspect mynet
docker inspect <container1> | grep -A 5 "Networks"
docker inspect <container2> | grep -A 5 "Networks"

# Test connectivity
docker exec container1 ping container2
docker exec container1 curl http://container2:3000

# Connect them to same network
docker network connect mynet container1
docker network connect mynet container2

# Default bridge doesn't support DNS — use custom network!
docker network create mynet
docker run --network mynet --name app1 myimage1
docker run --network mynet --name app2 myimage2
```

### DNS Resolution Failing

```bash
# Test DNS inside container
docker exec <container> nslookup google.com
docker exec <container> cat /etc/resolv.conf
docker exec <container> cat /etc/hosts

# Fix Docker DNS
# Add to /etc/docker/daemon.json:
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}
sudo systemctl restart docker

# Test with explicit DNS
docker run --dns 8.8.8.8 alpine nslookup google.com
```

### Container Can't Reach Internet

```bash
# Test from container
docker run --rm alpine ping 8.8.8.8
docker run --rm alpine wget -qO- https://google.com

# Check iptables
sudo iptables -t nat -L POSTROUTING -n
# Should see: MASQUERADE  all  -- 172.17.0.0/16

# Restart Docker
sudo systemctl restart docker

# Check if IP forwarding is enabled
cat /proc/sys/net/ipv4/ip_forward    # Should be 1
sudo sysctl -w net.ipv4.ip_forward=1
```

---

## Image Issues

### "Image not found" / Pull Errors

```bash
# Check exact image name and tag
docker search nginx
docker pull nginx:latest

# Login for private images
docker login
docker login registry.example.com

# Check disk space
docker system df
df -h /var/lib/docker

# Clear and retry
docker rmi nginx
docker pull nginx
```

### Build Fails

```bash
# Verbose build output
docker build --progress=plain .

# No cache (fresh build)
docker build --no-cache .

# Check Dockerfile syntax
docker build --check .           # Docker 25+

# Common errors:
# "COPY failed: file not found" — check .dockerignore and paths
# "RUN returned non-zero" — command failed, check logs
# "permission denied" — use RUN chmod or correct USER
```

### Build Context Too Large / Slow

```bash
# Check build context size
du -sh .
docker build . 2>&1 | head -5    # Shows "Sending build context..."

# Create/fix .dockerignore
cat > .dockerignore << 'EOF'
node_modules
.git
dist
build
*.log
.env
EOF

# Use BuildKit for faster builds
DOCKER_BUILDKIT=1 docker build .
```

### "No space left on device"

```bash
# Check space
df -h /var/lib/docker
docker system df

# Clean up
docker system prune -a           # Remove all unused
docker volume prune              # Remove unused volumes
docker image prune -a            # Remove all unused images

# Remove large unused images
docker images --format "{{.Size}}\t{{.Repository}}:{{.Tag}}" | sort -hr | head -20

# Change Docker storage location
# /etc/docker/daemon.json
{
  "data-root": "/mnt/bigdisk/docker"
}
```

---

## Volume Issues

### Permission Denied on Volume

```bash
# Check ownership of mounted dir
ls -la /host/path/

# Fix: match UID/GID in container
docker run --user $(id -u):$(id -g) myapp

# Fix in Dockerfile
RUN mkdir -p /app/data && chown -R 1000:1000 /app/data

# Fix with init container
docker run --rm -v myvolume:/data alpine chown -R 1000:1000 /data
```

### Volume Data Not Persisting

```bash
# Check if correct volume is mounted
docker inspect <container> | grep -A 10 "Mounts"

# Anonymous vs named volume
docker run -v /data myapp           # Anonymous — different each time!
docker run -v mydata:/data myapp    # Named — persists ✅

# In Compose — volumes must be declared
volumes:
  mydata:                           # Must be here!
```

### Can't Delete Volume

```bash
# Error: volume is in use
docker ps -a --filter volume=mydata   # Find containers using it
docker rm <container>                  # Remove container first
docker volume rm mydata

# Remove all unused
docker volume prune

# Nuclear option
docker system prune -a --volumes
```

---

## Resource Issues

### Container Using Too Much Memory (OOMKilled)

```bash
# Check if OOMKilled
docker inspect <container> | grep -i oom
docker inspect --format='{{.State.OOMKilled}}' <container>

# View memory stats
docker stats <container> --no-stream

# Fix: increase limit
docker run --memory="1g" myapp
docker run --memory="1g" --memory-swap="1g" myapp    # Disable swap

# In Compose
deploy:
  resources:
    limits:
      memory: 1G
```

### High CPU Usage

```bash
# Find CPU hogs
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}"

# Limit CPU
docker run --cpus="0.5" myapp        # Half a core
docker run --cpu-shares=512 myapp    # Relative weight

# Check processes inside container
docker exec <container> top
docker exec <container> ps aux
docker top <container>
```

### Slow Container Start

```bash
# Time the startup
time docker run --rm myapp echo "hello"

# Use smaller base image
# alpine vs ubuntu: 7MB vs 70MB

# Pre-pull images
docker pull myapp:latest

# Check health check start_period
# If healthcheck fails too early, container may restart
healthcheck:
  start_period: 30s    # Give app time to start
```

---

## Docker Compose Issues

### Services Not Starting in Order

```bash
# Use depends_on with health checks
depends_on:
  db:
    condition: service_healthy    # Wait for healthy, not just started

# Add healthcheck to db service
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres"]
  interval: 5s
  retries: 10

# Wait script in entrypoint
# Use wait-for-it.sh or dockerize
```

### "Bind for 0.0.0.0:5432 failed: port is already in use"

```bash
# Check what's using the port
sudo lsof -i :5432

# Is it a local PostgreSQL?
sudo systemctl stop postgresql

# OR change the host port in compose
ports:
  - "5433:5432"    # Different host port
```

### Compose File Not Found

```bash
# Must be in same directory or specify file
docker compose -f /path/to/docker-compose.yml up

# Common mistake: wrong directory
ls docker-compose.yml              # Check it exists
cd /path/to/project && docker compose up
```

### Environment Variables Not Working

```bash
# Debug: print env in container
docker compose exec myservice env

# Check .env file exists and is correct
cat .env
docker compose config              # Shows resolved config with vars

# Variable not in .env? Set in shell
export MY_VAR=value
docker compose up

# Check variable substitution
# ${VAR}         — error if not set
# ${VAR:-default} — use default if not set
# ${VAR:?error}  — print error and exit if not set
```

---

## Docker Daemon Issues

### Docker Daemon Not Running

```bash
# Check status
sudo systemctl status docker

# Start daemon
sudo systemctl start docker
sudo systemctl enable docker

# View daemon logs
sudo journalctl -u docker -f
sudo journalctl -u docker --since "1 hour ago"

# macOS Docker Desktop
open -a Docker
```

### "Got permission denied while connecting to Docker socket"

```bash
# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker                       # Apply immediately
# OR log out and back in

# Verify
groups $USER | grep docker
docker ps                           # Should work now
```

### Daemon Configuration Errors

```bash
# Validate daemon.json
sudo cat /etc/docker/daemon.json
# Must be valid JSON!

# Test config
sudo dockerd --validate

# Reset to defaults
sudo mv /etc/docker/daemon.json /etc/docker/daemon.json.bak
sudo systemctl restart docker
```

---

## Debugging Techniques

### Debug with Shell Access

```bash
# Running container
docker exec -it <container> bash
docker exec -it <container> sh      # Alpine

# Stopped container (start it first)
docker start <container>
docker exec -it <container> bash

# Container that exits immediately
docker run -it --entrypoint bash myimage
docker run -it --entrypoint sh myimage
```

### Debug with Netshoot (Networking)

```bash
# Powerful network debugging container
docker run --rm -it \
  --network container:<target_container> \
  nicolaka/netshoot \
  bash

# Tools available: curl, wget, ping, nslookup, dig,
# netstat, ss, tcpdump, nmap, iperf3, and more
```

### Inspect Everything

```bash
# Full container details
docker inspect <container> | less

# Specific fields
docker inspect --format='{{.State.Status}}' <container>
docker inspect --format='{{.NetworkSettings.Networks}}' <container>
docker inspect --format='{{range .Mounts}}{{.Source}} → {{.Destination}}{{"\n"}}{{end}}' <container>
docker inspect --format='{{json .Config.Env}}' <container>

# Image details
docker image inspect myapp
docker history myapp               # See all layers and sizes
```

### Copy Files for Analysis

```bash
# Copy logs out
docker cp <container>:/var/log/app.log ./app.log

# Copy entire directory
docker cp <container>:/app/data ./debug-data/

# View file inside container
docker exec <container> cat /app/config.json
docker exec <container> ls -la /app/
```

---

## Common Error Reference

| Error Message | Cause | Fix |
|--------------|-------|-----|
| `docker: command not found` | Docker not installed | Install Docker |
| `permission denied` (socket) | Not in docker group | `sudo usermod -aG docker $USER` |
| `port is already allocated` | Host port in use | Change host port or stop conflicting process |
| `no such container` | Wrong name/ID | `docker ps -a` to find correct name |
| `image not found` | Wrong image name/tag | Check spelling, `docker pull` first |
| `no space left on device` | Disk full | `docker system prune -a` |
| `OOMKilled` | Out of memory | Increase `--memory` limit |
| `connection refused` | App not listening or wrong port | Check app logs, verify EXPOSE |
| `name already in use` | Container name taken | `docker rm <name>` or use different name |
| `bind: address already in use` | Port taken | Find and stop conflicting process |
| `network not found` | Network doesn't exist | Create network first |
| `volume is in use` | Container using volume | Remove container first |
| `Dockerfile not found` | Wrong context or path | Check `-f` flag or file location |
| `COPY failed: file not found` | File not in context | Check `.dockerignore`, file path |
| `unauthorized: authentication required` | Not logged in | `docker login` |
