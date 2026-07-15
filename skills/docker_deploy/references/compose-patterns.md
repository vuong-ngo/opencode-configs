# Docker Compose Patterns

Common Docker Compose patterns for various system architectures.

---

## Pattern 1: Web App + Database + Cache

The most common pattern for web applications.

```yaml
# docker-compose.yml
services:
  # ==========================================
  # Application
  # ==========================================
  app:
    build:
      context: .
      dockerfile: Dockerfile
      # Cache mount to speed up rebuilds
      args:
        - BUILDKIT_INLINE_CACHE=1
    container_name: app
    restart: unless-stopped
    ports:
      - "${APP_PORT:-3000}:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/${DB_NAME}
      - REDIS_URL=redis://redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

  # ==========================================
  # PostgreSQL Database
  # ==========================================
  postgres:
    image: postgres:17-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      # Init scripts (run once on initialization)
      # - ./docker/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "${DB_PORT:-5432}:5432"
    networks:
      - app-network
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # ==========================================
  # Redis Cache
  # ==========================================
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    command: redis-server --appendonly yes --maxmemory 128mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    ports:
      - "${REDIS_PORT:-6379}:6379"
    networks:
      - app-network
    deploy:
      resources:
        limits:
          cpus: "0.25"
          memory: 192M
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local

networks:
  app-network:
    driver: bridge
```

### Corresponding .env.example

```dotenv
# Application
APP_PORT=3000

# PostgreSQL
DB_NAME=myapp
DB_USER=myapp
DB_PASSWORD=CHANGE_ME_TO_STRONG_PASSWORD
DB_PORT=5432

# Redis
REDIS_PORT=6379
```

---

## Pattern 2: Web App + Nginx Reverse Proxy

Nginx sits in front of the app, handling static files and SSL termination.

```yaml
services:
  # ==========================================
  # Nginx Reverse Proxy
  # ==========================================
  nginx:
    image: nginx:1.27-alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./docker/nginx/conf.d:/etc/nginx/conf.d:ro
      # Static files (if serving directly)
      # - ./public:/usr/share/nginx/html:ro
      # SSL certificates
      # - ./certs:/etc/nginx/certs:ro
    depends_on:
      app:
        condition: service_healthy
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  app:
    build: .
    container_name: app
    restart: unless-stopped
    # Do not expose port externally — only Nginx accesses it
    expose:
      - "3000"
    environment:
      - NODE_ENV=production
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3

networks:
  app-network:
    driver: bridge
```

### Sample Nginx Config

```nginx
# docker/nginx/conf.d/default.conf
upstream app_backend {
    server app:3000;
    # If scaling multiple instances:
    # server app:3000 weight=5;
}

server {
    listen 80;
    server_name _;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml application/xml+rss text/javascript
               image/svg+xml;

    # Health check endpoint for Nginx
    location /health {
        access_log off;
        return 200 "OK";
        add_header Content-Type text/plain;
    }

    # Static files (long cache)
    location /static/ {
        alias /usr/share/nginx/html/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Proxy to app
    location / {
        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;

        # Buffer settings
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
}
```

---

## Pattern 3: Microservices

Multiple services communicating with each other via an internal network.

```yaml
services:
  # ==========================================
  # API Gateway
  # ==========================================
  gateway:
    build:
      context: ./services/gateway
    container_name: gateway
    restart: unless-stopped
    ports:
      - "80:3000"
    environment:
      - AUTH_SERVICE_URL=http://auth:3001
      - USER_SERVICE_URL=http://users:3002
      - ORDER_SERVICE_URL=http://orders:3003
    depends_on:
      auth:
        condition: service_healthy
      users:
        condition: service_healthy
      orders:
        condition: service_healthy
    networks:
      - frontend
      - backend
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/health"]
      interval: 15s
      timeout: 5s
      retries: 3

  # ==========================================
  # Auth Service
  # ==========================================
  auth:
    build:
      context: ./services/auth
    container_name: auth
    restart: unless-stopped
    expose:
      - "3001"
    environment:
      - DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/auth_db
      - JWT_SECRET=${JWT_SECRET}
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3001/health"]
      interval: 15s
      timeout: 5s
      retries: 3

  # ==========================================
  # User Service
  # ==========================================
  users:
    build:
      context: ./services/users
    container_name: users
    restart: unless-stopped
    expose:
      - "3002"
    environment:
      - DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/users_db
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - backend
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3002/health"]
      interval: 15s
      timeout: 5s
      retries: 3

  # ==========================================
  # Order Service
  # ==========================================
  orders:
    build:
      context: ./services/orders
    container_name: orders
    restart: unless-stopped
    expose:
      - "3003"
    environment:
      - DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/orders_db
      - REDIS_URL=redis://redis:6379/1
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - backend
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3003/health"]
      interval: 15s
      timeout: 5s
      retries: 3

  # ==========================================
  # Shared Infrastructure
  # ==========================================
  postgres:
    image: postgres:17-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./docker/postgres/init-databases.sh:/docker-entrypoint-initdb.d/init.sh
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres-data:
  redis-data:

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
```

### Init Script for Multiple Databases

```bash
#!/bin/bash
# docker/postgres/init-databases.sh
set -e

# Create a database for each service
for db in auth_db users_db orders_db; do
  psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" <<-EOSQL
    CREATE DATABASE $db;
    GRANT ALL PRIVILEGES ON DATABASE $db TO $POSTGRES_USER;
EOSQL
done
```

---

## Pattern 4: Full Stack (Frontend + Backend + DB)

```yaml
services:
  frontend:
    build:
      context: ./frontend
    container_name: frontend
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:8000
    depends_on:
      backend:
        condition: service_healthy
    networks:
      - app-network

  backend:
    build:
      context: ./backend
    container_name: backend
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@postgres:5432/${DB_NAME}
      - REDIS_URL=redis://redis:6379
      - CORS_ORIGINS=http://localhost:3000
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 15s
      timeout: 5s
      retries: 3

  postgres:
    image: postgres:17-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

---

## Production Overrides

Use `docker-compose.prod.yml` to override dev settings:

```yaml
# docker-compose.prod.yml
# Usage: docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
services:
  app:
    build:
      args:
        - NODE_ENV=production
    restart: always
    # Do not expose port directly — route through nginx
    ports: []
    expose:
      - "3000"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 1G
        reservations:
          cpus: "0.5"
          memory: 256M

  postgres:
    # Do not expose port externally in production
    ports: []
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M

  redis:
    ports: []
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
```

---

## Useful Commands Reference

```bash
# Build and start all services
docker compose up -d --build

# Production deployment
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build

# View logs
docker compose logs -f          # All services
docker compose logs -f app      # Only the app service

# Scale a service
docker compose up -d --scale app=3

# Restart a service
docker compose restart app

# Stop and remove everything (keep volumes)
docker compose down

# Stop and remove everything (including volumes)
docker compose down -v

# Check status
docker compose ps

# Exec into a container
docker compose exec app sh
```
