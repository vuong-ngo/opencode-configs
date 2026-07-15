# Dockerfile Templates

Optimized Dockerfile templates for popular languages and frameworks.

---

## Node.js (Express / NestJS / Next.js)

```dockerfile
# ============================================
# Stage 1: Install dependencies
# ============================================
FROM node:22-alpine AS deps

WORKDIR /app

# Copy dependency manifests first (cache layer)
COPY package.json package-lock.json* yarn.lock* pnpm-lock.yaml* ./

# Install dependencies
RUN \
  if [ -f pnpm-lock.yaml ]; then \
    corepack enable && pnpm install --frozen-lockfile; \
  elif [ -f yarn.lock ]; then \
    yarn install --frozen-lockfile; \
  elif [ -f package-lock.json ]; then \
    npm ci; \
  else \
    echo "No lockfile found." && exit 1; \
  fi

# ============================================
# Stage 2: Build application
# ============================================
FROM node:22-alpine AS builder

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN npm run build

# ============================================
# Stage 3: Production runtime
# ============================================
FROM node:22-alpine AS runner

WORKDIR /app

# Create non-root user
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 appuser

# Copy build output and production dependencies
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/main.js"]
```

### Next.js (Standalone)

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci

FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Next.js standalone output
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Copy standalone build
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/ || exit 1

CMD ["node", "server.js"]
```

> **Note**: You need to add `output: 'standalone'` in `next.config.js`

---

## Python (FastAPI / Django / Flask)

### FastAPI / Flask

```dockerfile
# ============================================
# Stage 1: Build dependencies
# ============================================
FROM python:3.13-slim AS builder

WORKDIR /app

# Install build tools if C extensions need to be compiled
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ============================================
# Stage 2: Runtime
# ============================================
FROM python:3.13-slim AS runner

WORKDIR /app

# Runtime libraries (if libpq is needed for psycopg2)
RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq5 curl && \
    rm -rf /var/lib/apt/lists/*

# Copy installed packages
COPY --from=builder /install /usr/local

# Non-root user
RUN useradd --create-home --shell /bin/bash appuser
COPY --chown=appuser:appuser . .

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

# FastAPI with uvicorn
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]

# Flask with gunicorn
# CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "app:create_app()"]
```

### Django

```dockerfile
FROM python:3.13-slim AS builder
WORKDIR /app
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.13-slim AS runner
WORKDIR /app

RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq5 curl && \
    rm -rf /var/lib/apt/lists/*

COPY --from=builder /install /usr/local

RUN useradd --create-home --shell /bin/bash appuser
COPY --chown=appuser:appuser . .

# Collect static files
RUN python manage.py collectstatic --noinput

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:8000/health/ || exit 1

CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

---

## Go

```dockerfile
# ============================================
# Stage 1: Build
# ============================================
FROM golang:1.24-alpine AS builder

WORKDIR /app

# Download dependencies first (cache layer)
COPY go.mod go.sum ./
RUN go mod download && go mod verify

COPY . .

# Build static binary
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o /app/server ./cmd/server

# ============================================
# Stage 2: Minimal runtime
# ============================================
FROM scratch

# Import CA certificates for HTTPS calls
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

COPY --from=builder /app/server /server

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD ["/server", "healthcheck"]

ENTRYPOINT ["/server"]
```

> **Note**: Go binaries can be built as static, running on `scratch` image (~0
> MB overhead). If you need shell/debugging capabilities, use `alpine` instead
> of `scratch`.

---

## Java (Spring Boot / Quarkus)

```dockerfile
# ============================================
# Stage 1: Build with Maven
# ============================================
FROM maven:3.9-eclipse-temurin-21 AS builder

WORKDIR /app

# Cache dependencies
COPY pom.xml .
RUN mvn dependency:go-offline -B

COPY src ./src
RUN mvn package -DskipTests -B

# ============================================
# Stage 2: Runtime
# ============================================
FROM eclipse-temurin:21-jre-alpine AS runner

WORKDIR /app

RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --ingroup appgroup appuser

COPY --from=builder --chown=appuser:appgroup /app/target/*.jar app.jar

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# JVM tuning for containers
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-jar", "app.jar"]
```

---

## Rust

```dockerfile
# ============================================
# Stage 1: Build
# ============================================
FROM rust:1.87-slim AS builder

WORKDIR /app

# Cache dependencies with a dummy build
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs && \
    cargo build --release && \
    rm -rf src

COPY . .
RUN touch src/main.rs && cargo build --release

# ============================================
# Stage 2: Minimal runtime
# ============================================
FROM debian:bookworm-slim AS runner

RUN apt-get update && \
    apt-get install -y --no-install-recommends ca-certificates curl && \
    rm -rf /var/lib/apt/lists/*

RUN useradd --create-home appuser

COPY --from=builder --chown=appuser /app/target/release/app /usr/local/bin/app

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

CMD ["app"]
```

---

## PHP (Laravel)

```dockerfile
FROM php:8.4-fpm-alpine AS builder

# Install extensions
RUN apk add --no-cache \
    libpng-dev libjpeg-turbo-dev freetype-dev \
    libzip-dev icu-dev oniguruma-dev && \
    docker-php-ext-configure gd --with-freetype --with-jpeg && \
    docker-php-ext-install pdo_mysql gd zip intl opcache

# Composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --optimize-autoloader

COPY . .
RUN composer dump-autoload --optimize && \
    php artisan config:cache && \
    php artisan route:cache && \
    php artisan view:cache

# Runtime
FROM php:8.4-fpm-alpine

RUN apk add --no-cache \
    libpng libjpeg-turbo freetype libzip icu curl && \
    docker-php-ext-configure gd --with-freetype --with-jpeg && \
    docker-php-ext-install pdo_mysql gd zip intl opcache

WORKDIR /var/www/html

RUN adduser -D -u 1001 appuser

COPY --from=builder --chown=appuser /var/www/html .

# OPcache config
RUN echo "opcache.enable=1" >> /usr/local/etc/php/conf.d/opcache.ini && \
    echo "opcache.memory_consumption=128" >> /usr/local/etc/php/conf.d/opcache.ini && \
    echo "opcache.validate_timestamps=0" >> /usr/local/etc/php/conf.d/opcache.ini

USER appuser

EXPOSE 9000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:9000/health || exit 1

CMD ["php-fpm"]
```

---

## .dockerignore (Universal Template)

Always create a `.dockerignore` file alongside the Dockerfile:

```dockerignore
# Version control
.git
.gitignore

# Docker
Dockerfile*
docker-compose*.yml
docker-stack*.yml
.dockerignore

# IDE
.idea
.vscode
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Dependencies (will be reinstalled inside the container)
node_modules
vendor
__pycache__
*.pyc
.venv
venv

# Build output (will be rebuilt inside the container)
dist
build
target
out

# Test & docs
*.test.*
*.spec.*
tests
__tests__
coverage
.nyc_output
docs

# Environment (do NOT copy secrets into the image)
.env
.env.*
!.env.example

# Logs
*.log
logs
```
