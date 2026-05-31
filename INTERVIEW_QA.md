# 🎯 Docker Interview Questions & Answers

## Beginner Level

### Q1: What is Docker and why do we use it?

**Answer:**
Docker is an open-source containerization platform that packages applications and all their dependencies into lightweight, portable units called **containers**. 

Key benefits:
- **Consistency** — "works on my machine" problem solved; same container runs everywhere
- **Isolation** — each app runs in its own container, no conflicts
- **Portability** — runs on any OS that supports Docker
- **Speed** — containers start in seconds vs minutes for VMs
- **Efficiency** — shares host OS kernel, uses fewer resources than VMs

---

### Q2: What's the difference between an Image and a Container?

**Answer:**

| Image | Container |
|-------|-----------|
| Read-only blueprint/template | Running instance of an image |
| Like a class in OOP | Like an object (instance) |
| Stored on disk | Exists in memory (while running) |
| Built with `docker build` | Created with `docker run` |
| Can be shared via registry | Lives on one host |
| Immutable | Has writable layer on top |

---

### Q3: What's the difference between Docker and a Virtual Machine?

**Answer:**

| Feature | Docker Container | Virtual Machine |
|---------|-----------------|-----------------|
| OS | Shares host OS kernel | Full OS per VM |
| Size | MB | GB |
| Boot time | Seconds | Minutes |
| Performance | Near-native | Has overhead |
| Isolation | Process-level | Hardware-level |
| Use case | Microservices, apps | Full OS isolation |

Docker is faster and lighter because it shares the host OS kernel instead of running a full OS.

---

### Q4: What is a Dockerfile?

**Answer:**
A Dockerfile is a text file with step-by-step instructions to build a Docker image. Each instruction creates a layer.

```dockerfile
FROM node:18-alpine       # Base image
WORKDIR /app              # Set working directory
COPY package.json ./      # Copy files
RUN npm install           # Install dependencies
COPY . .                  # Copy source code
EXPOSE 3000               # Document port
CMD ["node", "server.js"] # Default command
```

---

### Q5: What is Docker Hub?

**Answer:**
Docker Hub is a public cloud-based registry where Docker images are stored and shared. It's like GitHub but for Docker images. You can:
- Pull official images: `docker pull nginx`
- Push your images: `docker push username/myapp`
- Store private images (paid plan)
- Set up automated builds

---

### Q6: What is the difference between CMD and ENTRYPOINT?

**Answer:**

- **CMD** sets the default command. It can be overridden when running the container.
- **ENTRYPOINT** sets a fixed command that always runs. CMD becomes its default arguments.

```dockerfile
# CMD only
CMD ["node", "server.js"]
# docker run myapp python other.py → runs python other.py (CMD overridden)

# ENTRYPOINT + CMD
ENTRYPOINT ["node"]
CMD ["server.js"]
# docker run myapp → runs: node server.js
# docker run myapp other.js → runs: node other.js (only CMD overridden)
```

Use ENTRYPOINT when you want a fixed executable. Use CMD for default arguments.

---

### Q7: What is docker-compose?

**Answer:**
Docker Compose is a tool to define and run **multi-container applications** using a YAML file. Instead of running multiple `docker run` commands, you define all services in `docker-compose.yml` and start everything with one command:

```bash
docker compose up -d      # Start all services
docker compose down       # Stop everything
```

---

## Intermediate Level

### Q8: Explain Docker networking modes.

**Answer:**

| Mode | Description | Use Case |
|------|-------------|----------|
| `bridge` | Default; isolated network on host | Most containers |
| `host` | Shares host network; no isolation | High performance |
| `none` | No networking | Maximum isolation |
| `overlay` | Multi-host networking | Docker Swarm |
| `macvlan` | Real MAC address from LAN | Legacy apps |

Custom bridge networks support DNS resolution by container name — containers can reach each other using their name as hostname.

---

### Q9: What are Docker volumes and why are they important?

**Answer:**
Volumes provide **persistent storage** that survives container restarts and removal. Without volumes, all data inside a container is lost when it stops.

Three types:
- **Named volumes** (`-v mydata:/app/data`) — Managed by Docker, best for databases
- **Bind mounts** (`-v /host/path:/container/path`) — Map host directory into container, best for development
- **tmpfs mounts** (`--tmpfs /tmp`) — In-memory only, for secrets or temp data

---

### Q10: What is a multi-stage build and why use it?

**Answer:**
Multi-stage builds use multiple `FROM` statements in a single Dockerfile. Each stage can copy artifacts from previous stages. This lets you:
- Use full build tools in build stages (compilers, dev dependencies)
- Copy only the final artifact into a minimal production image
- Result: **dramatically smaller images** (e.g., 900MB → 50MB for Node.js apps)

```dockerfile
# Stage 1: Build (full environment)
FROM node:18 AS builder
RUN npm install && npm run build

# Stage 2: Production (minimal)
FROM node:18-alpine
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/server.js"]
```

---

### Q11: How do you pass secrets to Docker containers securely?

**Answer:**
- **Never** put secrets in Dockerfile (baked into image layers)
- **Never** commit `.env` files with real values to git
- **Correct approaches:**
  1. Runtime env vars: `docker run -e SECRET=$SECRET myapp`
  2. `--env-file`: `docker run --env-file .env myapp`
  3. Docker Secrets (Swarm): stored encrypted, mounted at `/run/secrets/`
  4. External secret managers: HashiCorp Vault, AWS Secrets Manager

---

### Q12: What is Docker layer caching and how do you optimize it?

**Answer:**
Each Dockerfile instruction creates a layer. Docker caches layers and reuses them if nothing has changed. Cache is invalidated for a layer and all subsequent layers when its input changes.

**Optimization:** Order instructions from least to most frequently changed:

```dockerfile
FROM node:18-alpine               # Never changes → cached
WORKDIR /app                      # Rarely changes → cached
COPY package*.json ./             # Changes when deps change
RUN npm ci                        # Re-runs only when package.json changes
COPY . .                          # Changes on every code edit
RUN npm run build
```

This way, `npm ci` (slow) is only re-run when `package.json` actually changes.

---

### Q13: How does container health checking work?

**Answer:**
The `HEALTHCHECK` instruction tells Docker how to test if a container is working. It returns:
- `0` = healthy
- `1` = unhealthy

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

Compose can use health status in `depends_on`:
```yaml
depends_on:
  db:
    condition: service_healthy
```

---

## Advanced Level

### Q14: What is the difference between COPY and ADD?

**Answer:**
Both copy files into the image but ADD has extra features:
- **ADD** can download files from URLs
- **ADD** automatically extracts `.tar` archives

**Best practice:** Use `COPY` unless you specifically need URL download or tar extraction. ADD with URLs can have security implications and poor cache behavior.

---

### Q15: How do you handle container orchestration at scale?

**Answer:**
For production at scale:
- **Docker Swarm** — Built into Docker, simpler, good for smaller deployments
- **Kubernetes (K8s)** — Industry standard, complex but powerful, used by most large orgs
- **AWS ECS / EKS** — Managed container services on AWS
- **Google GKE** — Managed Kubernetes on GCP
- **Azure AKS** — Managed Kubernetes on Azure

Key features needed at scale: auto-scaling, rolling updates, service discovery, load balancing, self-healing.

---

### Q16: How do you implement rolling updates with zero downtime?

**Answer:**

With Docker Swarm:
```bash
docker service update \
  --image myapp:v2 \
  --update-parallelism 1 \
  --update-delay 30s \
  myapp
```

With Compose + manual blue-green:
1. Start new version containers
2. Health check passes
3. Switch load balancer to new containers
4. Remove old containers

With Kubernetes: built-in rolling update strategy in Deployment spec.

---

### Q17: What is the Docker build context?

**Answer:**
The build context is the set of files sent to the Docker daemon when running `docker build`. By default it's the current directory (`.`). Docker can only access files within this context during build.

```bash
docker build .           # Context = current dir
docker build /path/to/   # Context = specific path
docker build -           # Context = stdin (Dockerfile from stdin)
```

**Problem:** Large contexts (e.g., with `node_modules`) slow down builds. **Fix:** Use `.dockerignore` to exclude unnecessary files.

---

### Q18: Explain Docker's Union File System (UnionFS).

**Answer:**
Docker uses a Union File System (like OverlayFS on Linux) to create images from layers:

- Each Dockerfile instruction creates a **read-only layer**
- Layers are stacked — upper layers can modify lower layers' files
- When a container runs, Docker adds a thin **writable layer** on top
- Multiple containers can share the same base layers (no duplication)
- When a container modifies a file from a read-only layer, it uses **Copy-on-Write (CoW)** — copies the file to the writable layer first

This is why images are small and start fast — you only download/copy layers you don't have.

---

### Q19: How do you debug a container that won't start?

**Answer:**

```bash
# 1. Check container status and exit code
docker ps -a
docker inspect <container> | grep -E "Status|ExitCode|Error"

# 2. Check logs (even from crashed container)
docker logs <container>
docker logs --previous <container>  # Previous run's logs

# 3. Run interactively with shell
docker run -it --entrypoint bash myimage
docker run -it --entrypoint sh myimage    # Alpine

# 4. Override CMD to keep container alive
docker run -d --entrypoint tail myimage -f /dev/null
docker exec -it <container> bash

# 5. Check for missing env vars, wrong paths, permission issues
docker inspect <container> | grep -A 10 "Env"
```

---

### Q20: What are some Docker security best practices?

**Answer:**

1. **Run as non-root** — `USER appuser` in Dockerfile
2. **Minimal base images** — alpine, distroless, scratch
3. **No secrets in images** — use runtime vars or Docker Secrets
4. **Drop capabilities** — `--cap-drop ALL --cap-add NET_BIND_SERVICE`
5. **Read-only filesystem** — `--read-only`
6. **Scan images** — Trivy, Docker Scout, Snyk
7. **No privileged containers** — avoid `--privileged`
8. **Limit resources** — `--memory --cpus`
9. **Use content trust** — `DOCKER_CONTENT_TRUST=1`
10. **Pin image tags** — use SHA digests in production
11. **No new privileges** — `--security-opt no-new-privileges:true`
12. **Internal networks** — `--internal` for backend services

---

## Quick Fire Questions

| Question | Answer |
|----------|--------|
| Default network driver? | bridge |
| Command to enter running container? | `docker exec -it <c> bash` |
| How to persist data? | Volumes |
| Difference between `docker stop` vs `docker kill`? | stop = SIGTERM (graceful), kill = SIGKILL (immediate) |
| What does `--rm` flag do? | Auto-removes container on exit |
| Where are Docker images stored? | `/var/lib/docker/` |
| What is dangling image? | Image with no tag (`<none>:<none>`) |
| Remove all stopped containers? | `docker container prune` |
| Show container resource usage? | `docker stats` |
| What is EXPOSE? | Documents port — doesn't publish it |
| Difference between CMD and RUN? | RUN executes at build time, CMD at runtime |
| What is Docker Content Trust? | Image signing and verification |
| Default restart policy? | `no` (never restart) |
| How to run container as specific user? | `docker run --user 1000:1000` |
| What is `docker system prune -a`? | Remove ALL unused images, containers, networks, volumes |
