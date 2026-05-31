# 🏗️ Docker Real-World Project Stacks

## Stack 1: MERN (MongoDB + Express + React + Node)

### Project Structure

```
mern-app/
├── docker-compose.yml
├── docker-compose.dev.yml
├── .env
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       └── index.js
└── frontend/
    ├── Dockerfile
    ├── package.json
    └── src/
```

### `backend/Dockerfile`

```dockerfile
FROM node:18-alpine AS base
WORKDIR /app

FROM base AS deps
COPY package*.json ./
RUN npm ci --only=production

FROM base AS builder
COPY package*.json ./
RUN npm ci
COPY . .

FROM base AS production
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/src ./src
COPY package.json .
RUN addgroup -S app && adduser -S app -G app
USER app
EXPOSE 5000
HEALTHCHECK CMD wget -qO- http://localhost:5000/health || exit 1
CMD ["node", "src/index.js"]
```

### `frontend/Dockerfile`

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### `docker-compose.yml`

```yaml
version: '3.8'

services:
  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    depends_on:
      - backend
    networks:
      - webnet

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - MONGO_URI=mongodb://mongo:27017/mydb
      - JWT_SECRET=${JWT_SECRET}
      - NODE_ENV=production
    depends_on:
      mongo:
        condition: service_healthy
    networks:
      - webnet
      - dbnet
    restart: unless-stopped

  mongo:
    image: mongo:7
    volumes:
      - mongodata:/data/db
    networks:
      - dbnet
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  mongo-express:
    image: mongo-express
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_SERVER: mongo
    depends_on:
      - mongo
    networks:
      - dbnet
    profiles: ["tools"]

volumes:
  mongodata:

networks:
  webnet:
  dbnet:
```

---

## Stack 2: Django + PostgreSQL + Celery + Redis

### Project Structure

```
django-app/
├── docker-compose.yml
├── .env
├── Dockerfile
├── requirements.txt
├── manage.py
├── myproject/
│   └── settings.py
└── nginx/
    └── nginx.conf
```

### `Dockerfile`

```dockerfile
FROM python:3.11-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app

FROM base AS deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev gcc && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM base AS production
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 && rm -rf /var/lib/apt/lists/*
COPY --from=deps /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH
COPY . .
RUN python manage.py collectstatic --noinput
RUN useradd -m appuser && chown -R appuser /app
USER appuser
EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "myproject.wsgi:application"]
```

### `docker-compose.yml`

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - staticfiles:/static:ro
      - mediafiles:/media:ro
    depends_on:
      - web
    networks:
      - frontend

  web:
    build: .
    volumes:
      - staticfiles:/app/staticfiles
      - mediafiles:/app/media
    environment:
      - DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
      - REDIS_URL=redis://redis:6379/0
      - SECRET_KEY=${SECRET_KEY}
      - ALLOWED_HOSTS=${ALLOWED_HOSTS}
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - frontend
      - backend
    restart: unless-stopped

  celery:
    build: .
    command: celery -A myproject worker --loglevel=info
    environment:
      - DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - db
      - redis
    networks:
      - backend
    restart: unless-stopped

  celery-beat:
    build: .
    command: celery -A myproject beat --loglevel=info
    depends_on:
      - celery
    networks:
      - backend
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redisdata:/data
    networks:
      - backend

volumes:
  pgdata:
  redisdata:
  staticfiles:
  mediafiles:

networks:
  frontend:
  backend:
```

### `nginx/nginx.conf`

```nginx
upstream django {
    server web:8000;
}

server {
    listen 80;
    client_max_body_size 100M;

    location /static/ {
        alias /static/;
        expires 30d;
    }

    location /media/ {
        alias /media/;
    }

    location / {
        proxy_pass http://django;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }
}
```

---

## Stack 3: Microservices (API Gateway + Services)

### Structure

```
microservices/
├── docker-compose.yml
├── gateway/
│   └── nginx.conf
├── auth-service/
│   └── Dockerfile
├── user-service/
│   └── Dockerfile
└── product-service/
    └── Dockerfile
```

### `docker-compose.yml`

```yaml
version: '3.8'

services:
  # API Gateway
  gateway:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./gateway/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - auth-service
      - user-service
      - product-service
    networks:
      - gateway

  auth-service:
    build: ./auth-service
    environment:
      - DB_URL=postgresql://auth:auth@auth-db:5432/authdb
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      auth-db:
        condition: service_healthy
    networks:
      - gateway
      - auth-net
    restart: unless-stopped

  user-service:
    build: ./user-service
    environment:
      - DB_URL=postgresql://user:user@user-db:5432/userdb
    depends_on:
      user-db:
        condition: service_healthy
    networks:
      - gateway
      - user-net
    restart: unless-stopped

  product-service:
    build: ./product-service
    environment:
      - MONGO_URL=mongodb://mongo:27017/products
    depends_on:
      - mongo
    networks:
      - gateway
      - product-net
    restart: unless-stopped

  # Databases — isolated per service
  auth-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: auth
      POSTGRES_PASSWORD: auth
      POSTGRES_DB: authdb
    volumes:
      - auth-pgdata:/var/lib/postgresql/data
    networks:
      - auth-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U auth"]
      interval: 5s
      retries: 10

  user-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: user
      POSTGRES_DB: userdb
    volumes:
      - user-pgdata:/var/lib/postgresql/data
    networks:
      - user-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      retries: 10

  mongo:
    image: mongo:7
    volumes:
      - mongodata:/data/db
    networks:
      - product-net

  # Message Queue
  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "15672:15672"   # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER:-admin}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASS:-admin}
    volumes:
      - rabbitmqdata:/var/lib/rabbitmq
    networks:
      - gateway
    profiles: ["messaging"]

volumes:
  auth-pgdata:
  user-pgdata:
  mongodata:
  rabbitmqdata:

networks:
  gateway:
  auth-net:
    internal: true
  user-net:
    internal: true
  product-net:
    internal: true
```

### `gateway/nginx.conf`

```nginx
upstream auth {
    server auth-service:3000;
}
upstream users {
    server user-service:3000;
}
upstream products {
    server product-service:3000;
}

server {
    listen 80;

    location /api/auth/ {
        proxy_pass http://auth/;
    }

    location /api/users/ {
        proxy_pass http://users/;
    }

    location /api/products/ {
        proxy_pass http://products/;
    }
}
```

---

## Stack 4: ELK Stack (Elasticsearch + Logstash + Kibana)

```yaml
version: '3.8'

services:
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
      - xpack.security.enabled=false
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - elk
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      retries: 5

  logstash:
    image: logstash:8.11.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
    ports:
      - "5044:5044"    # Beats input
      - "5000:5000"    # TCP input
    environment:
      LS_JAVA_OPTS: -Xmx256m -Xms256m
    depends_on:
      elasticsearch:
        condition: service_healthy
    networks:
      - elk

  kibana:
    image: kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_URL: http://elasticsearch:9200
    depends_on:
      - elasticsearch
    networks:
      - elk

volumes:
  esdata:

networks:
  elk:
```

---

## Stack 5: WordPress + MySQL + Redis

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - wordpress:/var/www/html:ro
      - ./nginx/wordpress.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - wordpress
    networks:
      - frontend

  wordpress:
    image: wordpress:php8.2-fpm-alpine
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
      WORDPRESS_DB_NAME: ${DB_NAME}
      WORDPRESS_CONFIG_EXTRA: |
        define('WP_REDIS_HOST', 'redis');
        define('WP_REDIS_PORT', 6379);
    volumes:
      - wordpress:/var/www/html
    depends_on:
      db:
        condition: service_healthy
    networks:
      - frontend
      - backend
    restart: unless-stopped

  db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - dbdata:/var/lib/mysql
    networks:
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redisdata:/data
    networks:
      - backend
    restart: unless-stopped

  phpmyadmin:
    image: phpmyadmin:latest
    ports:
      - "8080:80"
    environment:
      PMA_HOST: db
    depends_on:
      - db
    networks:
      - backend
    profiles: ["tools"]

volumes:
  wordpress:
  dbdata:
  redisdata:

networks:
  frontend:
  backend:
    internal: true
```

---

## Development Environment Templates

### Any Language Dev Environment

```yaml
# docker-compose.dev.yml — generic dev environment
version: '3.8'

services:
  app:
    build:
      context: .
      target: development
    volumes:
      - .:/app                      # Live code sync
      - deps:/app/node_modules      # Preserve deps
    ports:
      - "${PORT:-3000}:3000"
      - "9229:9229"                 # Debug port
    environment:
      - NODE_ENV=development
    stdin_open: true
    tty: true
    command: npm run dev

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: dev
      POSTGRES_USER: dev
      POSTGRES_DB: devdb
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  mailhog:                          # Catch emails in dev
    image: mailhog/mailhog
    ports:
      - "1025:1025"                 # SMTP
      - "8025:8025"                 # Web UI

volumes:
  deps:
  pgdata:
```
