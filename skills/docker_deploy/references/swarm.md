# Docker Swarm Deployment

Guide for deploying systems on a Docker Swarm cluster.

---

## When to Use Swarm?

| Criteria          | Docker Compose         | Docker Swarm          |
| ----------------- | ---------------------- | --------------------- |
| Server count      | 1                      | ≥ 1 (cluster)        |
| Scaling           | Manual                 | Auto / declarative    |
| High Availability | No                     | Yes (replicas)        |
| Rolling Update    | Manual restart         | Built-in              |
| Secret Management | `.env` file            | `docker secret`       |
| Load Balancing    | Requires nginx/traefik | Built-in (ingress)    |
| Complexity        | Low                    | Medium                |

---

## Initialize Swarm Cluster

```bash
# On the manager node
docker swarm init --advertise-addr <MANAGER_IP>

# The output will provide a join token — run on worker nodes:
# docker swarm join --token <TOKEN> <MANAGER_IP>:2377

# Check nodes
docker node ls

# Retrieve join token if needed
docker swarm join-token worker
docker swarm join-token manager
```

---

## Stack File (docker-stack.yml)

A stack file is similar to a compose file but includes additional deploy
configuration:

```yaml
# docker-stack.yml
version: "3.9"

services:
  # ==========================================
  # Application
  # ==========================================
  app:
    image: ${REGISTRY}/myapp:${TAG:-latest}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1          # Update 1 container at a time
        delay: 10s              # Wait 10s between each update
        failure_action: rollback
        monitor: 30s            # Monitor for 30s after update
        order: start-first      # Start new container before stopping old one
      rollback_config:
        parallelism: 1
        delay: 5s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 128M
      placement:
        constraints:
          - node.role == worker
    ports:
      - target: 3000
        published: 80
        protocol: tcp
        mode: ingress           # Swarm load balancing
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:pass@postgres:5432/myapp
      - REDIS_URL=redis://redis:6379
    secrets:
      - db_password
      - jwt_secret
    networks:
      - frontend
      - backend
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s

  # ==========================================
  # PostgreSQL
  # ==========================================
  postgres:
    image: postgres:17-alpine
    deploy:
      replicas: 1              # Database typically runs a single instance
      placement:
        constraints:
          - node.role == manager
          - node.labels.db == true
      resources:
        limits:
          cpus: "1.0"
          memory: 1G
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - postgres-data:/var/lib/postgresql/data
    secrets:
      - db_password
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ==========================================
  # Redis
  # ==========================================
  redis:
    image: redis:7-alpine
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
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

# ==========================================
# Secrets (must be created beforehand with docker secret create)
# ==========================================
secrets:
  db_password:
    external: true
  jwt_secret:
    external: true

# ==========================================
# Volumes
# ==========================================
volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local

# ==========================================
# Networks
# ==========================================
networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
    internal: true            # Not accessible from outside
```

---

## Managing Secrets

```bash
# Create secret from a string
echo "my_strong_password" | docker secret create db_password -

# Create secret from a file
docker secret create jwt_secret ./jwt_secret.key

# List secrets
docker secret ls

# Inspect a secret (cannot view the value)
docker secret inspect db_password

# Remove a secret
docker secret rm db_password
```

**Using secrets in the application:**

```python
# Python: Read secret from file
import os

def get_secret(name: str) -> str:
    """Read Docker secret, falling back to environment variable."""
    secret_path = f"/run/secrets/{name}"
    if os.path.exists(secret_path):
        with open(secret_path) as f:
            return f.read().strip()
    return os.environ.get(name.upper(), "")
```

```javascript
// Node.js: Read secret from file
const fs = require("fs");
const path = require("path");

function getSecret(name) {
  const secretPath = path.join("/run/secrets", name);
  try {
    return fs.readFileSync(secretPath, "utf8").trim();
  } catch {
    return process.env[name.toUpperCase()] || "";
  }
}
```

---

## Deploy & Manage Stacks

```bash
# ==========================================
# Deploy
# ==========================================

# Deploy a stack
docker stack deploy -c docker-stack.yml myapp

# Deploy with multiple compose files
docker stack deploy -c docker-stack.yml -c docker-stack.prod.yml myapp

# ==========================================
# Monitoring
# ==========================================

# View stack services
docker stack services myapp

# View tasks (containers) of a service
docker service ps myapp_app

# View logs
docker service logs myapp_app -f --tail 100

# ==========================================
# Scaling
# ==========================================

# Scale a service
docker service scale myapp_app=5

# ==========================================
# Update
# ==========================================

# Update image
docker service update --image ${REGISTRY}/myapp:v2.0 myapp_app

# Rollback
docker service rollback myapp_app

# ==========================================
# Cleanup
# ==========================================

# Remove stack (keep volumes)
docker stack rm myapp

# Remove stack + volumes (WARNING: data will be lost!)
docker stack rm myapp
docker volume prune -f
```

---

## Node Labels & Placement

```bash
# Assign labels to a node
docker node update --label-add db=true node-1
docker node update --label-add app=true node-2
docker node update --label-add app=true node-3

# Usage in placement constraints:
# placement:
#   constraints:
#     - node.labels.db == true
#     - node.role == worker
```

---

## Deployment Script

```bash
#!/bin/bash
# deploy.sh — Script to deploy to Docker Swarm
set -euo pipefail

STACK_NAME="${1:-myapp}"
REGISTRY="${REGISTRY:-ghcr.io/myorg}"
TAG="${TAG:-$(git rev-parse --short HEAD)}"

echo "🚀 Deploying ${STACK_NAME} with tag ${TAG}"

# 1. Build & Push image
echo "📦 Building image..."
docker build -t "${REGISTRY}/myapp:${TAG}" .
docker push "${REGISTRY}/myapp:${TAG}"

# 2. Tag as latest
docker tag "${REGISTRY}/myapp:${TAG}" "${REGISTRY}/myapp:latest"
docker push "${REGISTRY}/myapp:latest"

# 3. Deploy stack
echo "🔄 Deploying stack..."
TAG="${TAG}" REGISTRY="${REGISTRY}" \
  docker stack deploy -c docker-stack.yml "${STACK_NAME}"

# 4. Verify deployment
echo "⏳ Waiting for service to stabilize..."
sleep 10

docker stack services "${STACK_NAME}"

echo "✅ Deployment complete!"
echo "   Stack: ${STACK_NAME}"
echo "   Image: ${REGISTRY}/myapp:${TAG}"
```
