# 🐳 Docker Commands Cheat Sheet

## 🔧 System & Info

```bash
docker version                      # Client & server version
docker info                         # System-wide info
docker system df                    # Disk usage by Docker
docker system prune                 # Remove stopped containers, dangling images, unused networks
docker system prune -a              # + remove all unused images
docker system prune -a --volumes    # + also remove volumes
docker events                       # Real-time Docker events
docker events --since 1h            # Events from last 1 hour
```

---

## 📦 Image Commands

```bash
# Pull
docker pull ubuntu                          # Pull latest
docker pull ubuntu:22.04                    # Pull specific tag
docker pull --all-tags nginx                # Pull all tags

# List
docker images                               # List local images
docker images -a                            # Include intermediate
docker images --filter "dangling=true"      # Dangling images only
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Remove
docker rmi nginx                            # Remove by name
docker rmi nginx:1.25                       # Remove by tag
docker rmi $(docker images -q)              # Remove all images
docker image prune                          # Remove dangling images
docker image prune -a                       # Remove all unused images

# Inspect & Info
docker inspect nginx                        # Full JSON details
docker image history nginx                  # Show layers
docker image inspect nginx                  # Image details
docker manifest inspect nginx              # Multi-arch manifest

# Tag & Push
docker tag myapp:latest myapp:v1.0
docker tag myapp:v1.0 myuser/myapp:v1.0
docker push myuser/myapp:v1.0

# Save & Load (offline transfer)
docker save -o myapp.tar myapp:latest       # Save to tar
docker load -i myapp.tar                    # Load from tar
docker export <container> -o container.tar  # Export container filesystem
docker import container.tar myapp:imported  # Import as image
```

---

## 🚀 Container Commands

### Create & Run

```bash
docker run nginx                                   # Foreground (attached)
docker run -d nginx                                # Detached (background)
docker run -it ubuntu bash                         # Interactive terminal
docker run --name myapp -d nginx                   # Custom name
docker run -p 8080:80 nginx                        # Port: host:container
docker run -p 127.0.0.1:8080:80 nginx              # Bind to localhost only
docker run -P nginx                                # Publish all exposed ports
docker run -v myvolume:/data nginx                 # Named volume
docker run -v $(pwd):/app nginx                    # Bind mount
docker run -e MY_VAR=value nginx                   # Environment variable
docker run --env-file .env nginx                   # Env from file
docker run --rm nginx                              # Auto-remove on exit
docker run --restart=always nginx                  # Auto-restart
docker run --network mynet nginx                   # Custom network
docker run --cpus="1.0" --memory="512m" nginx      # Resource limits
docker run --read-only nginx                       # Read-only filesystem
docker run --user 1000:1000 nginx                  # Run as specific user
docker run --workdir /app nginx                    # Set working directory
docker run --hostname myhost nginx                 # Custom hostname
docker run --entrypoint /bin/sh nginx              # Override entrypoint
docker run -d --log-driver json-file \
  --log-opt max-size=10m nginx                     # Custom log driver
```

### Manage

```bash
docker ps                           # Running containers
docker ps -a                        # All containers
docker ps -q                        # Only IDs
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

docker start myapp                  # Start stopped container
docker stop myapp                   # Graceful stop (SIGTERM)
docker kill myapp                   # Force stop (SIGKILL)
docker restart myapp                # Stop + start
docker pause myapp                  # Pause (freeze) container
docker unpause myapp                # Resume container

docker rm myapp                     # Remove stopped container
docker rm -f myapp                  # Force remove running container
docker rm $(docker ps -aq)          # Remove all containers
docker container prune              # Remove all stopped containers
```

### Interact

```bash
docker exec -it myapp bash          # Shell into running container
docker exec -it myapp sh            # sh (for Alpine-based)
docker exec myapp ls /app           # Run command non-interactively
docker exec -u root myapp bash      # Exec as root user

docker attach myapp                 # Attach to container's STDIN/STDOUT
docker cp myapp:/app/file.txt .     # Copy from container
docker cp ./file.txt myapp:/app/    # Copy to container
```

### Inspect & Monitor

```bash
docker logs myapp                   # Show all logs
docker logs -f myapp                # Follow (stream) logs
docker logs --tail 50 myapp         # Last 50 lines
docker logs --since 10m myapp       # Logs from last 10 minutes
docker logs --timestamps myapp      # Include timestamps

docker inspect myapp                # Full JSON info
docker inspect --format='{{.NetworkSettings.IPAddress}}' myapp
docker inspect --format='{{.State.Health.Status}}' myapp

docker stats                        # Live resource stats (all)
docker stats myapp                  # Live stats (one container)
docker stats --no-stream            # One-time snapshot
docker top myapp                    # Processes inside container
docker diff myapp                   # File changes since creation
docker port myapp                   # Port mappings
```

---

## 🏗️ Build Commands

```bash
docker build .                              # Build from current dir
docker build -t myapp:latest .              # With tag
docker build -t myapp:v1 -f Dockerfile.prod .  # Custom Dockerfile
docker build --no-cache -t myapp .          # Skip cache
docker build --build-arg NODE_ENV=prod .    # Pass build arg
docker build --target builder .             # Build specific stage
docker build --platform linux/amd64 .      # Target platform
docker build --progress=plain .             # Verbose output

# BuildKit (faster, recommended)
DOCKER_BUILDKIT=1 docker build .
```

---

## 💾 Volume Commands

```bash
docker volume create mydata                 # Create volume
docker volume ls                            # List volumes
docker volume inspect mydata                # Inspect volume
docker volume rm mydata                     # Remove volume
docker volume prune                         # Remove unused volumes

# Use volume
docker run -v mydata:/app/data nginx        # Named volume
docker run -v $(pwd)/data:/app/data nginx   # Bind mount
docker run -v mydata:/app/data:ro nginx     # Read-only
docker run --tmpfs /tmp nginx               # In-memory tmpfs
```

---

## 🌐 Network Commands

```bash
docker network create mynet                         # Bridge network
docker network create --driver host mynet           # Host network
docker network create --subnet 172.20.0.0/16 mynet  # Custom subnet
docker network create \
  --driver bridge \
  --subnet 172.18.0.0/16 \
  --gateway 172.18.0.1 mynet

docker network ls                           # List networks
docker network inspect mynet                # Inspect network
docker network rm mynet                     # Remove network
docker network prune                        # Remove unused networks

docker network connect mynet myapp          # Connect container
docker network disconnect mynet myapp       # Disconnect container

docker run --network mynet nginx            # Run in network
docker run --network host nginx             # Use host network
docker run --network none nginx             # No network
```

---

## 📋 Docker Compose Commands

```bash
# Start & Stop
docker compose up                   # Start in foreground
docker compose up -d                # Start in background
docker compose up --build           # Rebuild before starting
docker compose up --force-recreate  # Recreate containers
docker compose up --scale web=3     # Scale service to 3 replicas
docker compose down                 # Stop & remove containers
docker compose down -v              # Also remove volumes
docker compose down --rmi all       # Also remove images
docker compose stop                 # Stop without removing
docker compose start                # Start stopped services
docker compose restart              # Restart all services
docker compose restart web          # Restart specific service
docker compose pause                # Pause services
docker compose unpause              # Unpause services

# Inspect
docker compose ps                   # Service containers
docker compose top                  # Running processes
docker compose images               # Images used
docker compose port web 3000        # Published port
docker compose config               # Validate & print config

# Logs
docker compose logs                 # All service logs
docker compose logs -f              # Follow all logs
docker compose logs -f web          # Follow specific service
docker compose logs --tail=50 web   # Last 50 lines

# Run & Execute
docker compose exec web bash        # Shell into service
docker compose exec -u root web sh  # As root
docker compose run web npm test     # One-off command
docker compose run --rm web npm test # Auto-remove after run

# Build
docker compose build                # Build all services
docker compose build web            # Build specific service
docker compose build --no-cache     # Skip cache
docker compose pull                 # Pull latest images
docker compose push                 # Push built images
```

---

## 🔐 Registry Commands

```bash
# Login
docker login                                # Docker Hub
docker login registry.example.com          # Private registry
docker login -u user -p pass registry.com  # Non-interactive
docker logout

# Tag & Push
docker tag myapp:latest myuser/myapp:latest
docker push myuser/myapp:latest
docker push myuser/myapp                    # Pushes all tags

# Pull
docker pull myuser/myapp:latest
docker pull --platform linux/arm64 myapp   # Specific platform
```

---

## 🧹 Cleanup Reference

```bash
# Containers
docker rm $(docker ps -aq)              # All stopped containers
docker container prune -f               # Force prune

# Images
docker rmi $(docker images -q)          # All images
docker image prune                      # Dangling only
docker image prune -a                   # All unused

# Volumes
docker volume prune -f

# Networks
docker network prune -f

# Nuclear option
docker system prune -a --volumes -f     # Remove EVERYTHING unused
```
