# 🚀 Docker in CI/CD

## GitHub Actions

### Build & Push to Docker Hub

```yaml
# .github/workflows/docker.yml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags: ['v*.*.*']
  pull_request:
    branches: [main]

env:
  REGISTRY: docker.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix={{branch}}-

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Push to GitHub Container Registry (GHCR)

```yaml
name: Push to GHCR

on:
  push:
    branches: [main]

jobs:
  push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
```

### Push to AWS ECR

```yaml
name: Push to ECR

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, push
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: myapp
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
                     $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
```

### Full CI Pipeline

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests in Docker
        run: |
          docker compose -f docker-compose.test.yml up \
            --abort-on-container-exit \
            --exit-code-from tests

      - name: Clean up
        if: always()
        run: docker compose -f docker-compose.test.yml down -v

  security-scan:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Scan for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH

      - name: Upload scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: trivy-results.sarif

  build-push:
    runs-on: ubuntu-latest
    needs: [test, security-scan]
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: myuser/myapp:latest,myuser/myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

test:
  stage: test
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker compose -f docker-compose.test.yml up --abort-on-container-exit
  after_script:
    - docker compose -f docker-compose.test.yml down -v

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $IMAGE .
    - docker push $IMAGE
    - docker tag $IMAGE $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main

deploy:
  stage: deploy
  image: alpine
  script:
    - apk add --no-cache openssh-client
    - ssh -o StrictHostKeyChecking=no deploy@$SERVER_HOST "
        docker pull $IMAGE &&
        docker stop myapp || true &&
        docker rm myapp || true &&
        docker run -d --name myapp -p 80:3000 $IMAGE
      "
  only:
    - main
```

---

## docker-compose.test.yml

```yaml
version: '3.8'

services:
  tests:
    build:
      context: .
      target: builder     # Use build stage that has dev deps
    command: npm test
    environment:
      - NODE_ENV=test
      - DATABASE_URL=postgresql://test:test@testdb:5432/testdb
    depends_on:
      testdb:
        condition: service_healthy

  testdb:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: testdb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 5s
      retries: 10
    tmpfs:
      - /var/lib/postgresql/data    # In-memory for speed
```

---

## Docker BuildKit Cache

```bash
# Enable BuildKit
export DOCKER_BUILDKIT=1

# Mount cache in Dockerfile
RUN --mount=type=cache,target=/root/.npm \
    npm ci

RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

RUN --mount=type=cache,target=/go/pkg \
    go build ./...

# GitHub Actions cache
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

---

## Deployment Scripts

### Simple Deploy Script

```bash
#!/bin/bash
# deploy.sh
set -e

IMAGE="myuser/myapp"
TAG="${1:-latest}"
CONTAINER="myapp"
PORT="3000"

echo "🐳 Deploying $IMAGE:$TAG..."

# Pull latest
docker pull $IMAGE:$TAG

# Stop existing container
docker stop $CONTAINER 2>/dev/null || true
docker rm $CONTAINER 2>/dev/null || true

# Run new container
docker run -d \
  --name $CONTAINER \
  --restart unless-stopped \
  -p $PORT:3000 \
  --env-file /etc/myapp/.env \
  $IMAGE:$TAG

# Wait and check health
echo "Waiting for health check..."
sleep 10
if docker inspect --format='{{.State.Health.Status}}' $CONTAINER | grep -q "healthy"; then
    echo "✅ Deployment successful!"
else
    echo "❌ Health check failed!"
    docker logs $CONTAINER
    exit 1
fi
```

### Blue-Green Deployment

```bash
#!/bin/bash
# blue-green.sh
IMAGE="myuser/myapp:$1"
NGINX_CONF="/etc/nginx/sites-enabled/myapp.conf"

# Determine current color
if docker ps | grep -q "myapp-blue"; then
    CURRENT="blue"
    NEW="green"
    NEW_PORT="3001"
else
    CURRENT="green"
    NEW="blue"
    NEW_PORT="3000"
fi

echo "Deploying to $NEW (current: $CURRENT)"

# Start new version
docker run -d \
    --name "myapp-$NEW" \
    -p $NEW_PORT:3000 \
    $IMAGE

# Wait for health
sleep 15
if ! curl -sf http://localhost:$NEW_PORT/health; then
    echo "New version unhealthy, rolling back"
    docker rm -f "myapp-$NEW"
    exit 1
fi

# Switch nginx to new version
sed -i "s/3000/3001/g; s/3001/3000/g" $NGINX_CONF  # Toggle ports
nginx -s reload

# Remove old version
docker rm -f "myapp-$CURRENT"
docker image prune -f
echo "✅ Switched to $NEW"
```
