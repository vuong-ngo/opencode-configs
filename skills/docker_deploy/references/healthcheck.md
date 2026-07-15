# Health Check & Monitoring

Guide for setting up health checks and monitoring for Docker containers.

---

## 1. Health Check in Dockerfile

### HTTP-based (most common)

```dockerfile
# Using wget (available by default in Alpine)
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Using curl (needs to be installed separately in Alpine)
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
```

### TCP-based (for services without HTTP)

```dockerfile
# PostgreSQL
HEALTHCHECK --interval=10s --timeout=5s --retries=5 \
  CMD pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB} || exit 1

# Redis
HEALTHCHECK --interval=10s --timeout=5s --retries=5 \
  CMD redis-cli ping | grep PONG || exit 1

# MySQL
HEALTHCHECK --interval=10s --timeout=5s --retries=5 \
  CMD mysqladmin ping -h localhost -u root -p${MYSQL_ROOT_PASSWORD} || exit 1

# Generic TCP check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD nc -z localhost 8080 || exit 1
```

### Custom health check script

```dockerfile
COPY docker/scripts/healthcheck.sh /usr/local/bin/healthcheck.sh
RUN chmod +x /usr/local/bin/healthcheck.sh

HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD /usr/local/bin/healthcheck.sh
```

```bash
#!/bin/sh
# docker/scripts/healthcheck.sh

# Check HTTP endpoint
if ! wget --no-verbose --tries=1 --spider http://localhost:3000/health 2>/dev/null; then
  echo "HTTP health check failed"
  exit 1
fi

# Check disk space (>90% = unhealthy)
DISK_USAGE=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$DISK_USAGE" -gt 90 ]; then
  echo "Disk usage critical: ${DISK_USAGE}%"
  exit 1
fi

# Check memory (optional)
# MEM_USAGE=$(free | grep Mem | awk '{print int($3/$2 * 100)}')
# if [ "$MEM_USAGE" -gt 90 ]; then
#   echo "Memory usage critical: ${MEM_USAGE}%"
#   exit 1
# fi

echo "All health checks passed"
exit 0
```

---

## 2. Health Check Parameters

| Parameter        | Description                                           | Recommended Value     |
| ---------------- | ----------------------------------------------------- | --------------------- |
| `--interval`     | Time between each check                               | 15s – 30s            |
| `--timeout`      | Timeout for each check                                | 5s – 10s             |
| `--start-period` | Grace period before checks start (startup time)       | 10s – 60s (app-dependent) |
| `--retries`      | Number of failures before marking as unhealthy        | 3 – 5                |

**Principles:**

- `start-period` should be large enough for the app to finish starting (Java/Spring typically needs 30-60s)
- `interval` should not be too short to avoid adding load to the app
- `timeout` must be less than `interval`

---

## 3. Health Check Endpoint in Application

### Node.js (Express)

```javascript
// routes/health.js
const express = require("express");
const router = express.Router();

router.get("/health", async (req, res) => {
  const checks = {};
  let status = "healthy";

  // Check database connection
  try {
    await db.query("SELECT 1");
    checks.database = "ok";
  } catch (err) {
    checks.database = "error";
    status = "unhealthy";
  }

  // Check Redis connection
  try {
    await redis.ping();
    checks.redis = "ok";
  } catch (err) {
    checks.redis = "error";
    status = "unhealthy";
  }

  const statusCode = status === "healthy" ? 200 : 503;
  res.status(statusCode).json({
    status,
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    checks,
  });
});
```

### Python (FastAPI)

```python
from fastapi import FastAPI, Response
from datetime import datetime
import time

app = FastAPI()
start_time = time.time()

@app.get("/health")
async def health_check():
    checks = {}
    status = "healthy"

    # Check database
    try:
        await database.execute("SELECT 1")
        checks["database"] = "ok"
    except Exception:
        checks["database"] = "error"
        status = "unhealthy"

    # Check Redis
    try:
        await redis_client.ping()
        checks["redis"] = "ok"
    except Exception:
        checks["redis"] = "error"
        status = "unhealthy"

    response = {
        "status": status,
        "timestamp": datetime.utcnow().isoformat(),
        "uptime": time.time() - start_time,
        "checks": checks,
    }

    status_code = 200 if status == "healthy" else 503
    return Response(
        content=json.dumps(response),
        status_code=status_code,
        media_type="application/json",
    )
```

### Go

```go
package main

import (
	"encoding/json"
	"net/http"
	"time"
)

var startTime = time.Now()

type HealthResponse struct {
	Status    string            `json:"status"`
	Timestamp string            `json:"timestamp"`
	Uptime    float64           `json:"uptime"`
	Checks    map[string]string `json:"checks"`
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
	checks := map[string]string{}
	status := "healthy"

	// Check database
	if err := db.Ping(); err != nil {
		checks["database"] = "error"
		status = "unhealthy"
	} else {
		checks["database"] = "ok"
	}

	resp := HealthResponse{
		Status:    status,
		Timestamp: time.Now().UTC().Format(time.RFC3339),
		Uptime:    time.Since(startTime).Seconds(),
		Checks:    checks,
	}

	statusCode := http.StatusOK
	if status == "unhealthy" {
		statusCode = http.StatusServiceUnavailable
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(statusCode)
	json.NewEncoder(w).Encode(resp)
}
```

---

## 4. Restart Policies

```yaml
# docker-compose.yml
services:
  app:
    restart: unless-stopped   # Recommended for production

# Available values:
#   no              — Never restart (default)
#   always          — Always restart, even when manually stopped
#   on-failure      — Restart only when exit code != 0
#   unless-stopped  — Restart unless manually stopped (RECOMMENDED)
```

In Swarm:

```yaml
deploy:
  restart_policy:
    condition: on-failure     # none | on-failure | any
    delay: 5s                 # Wait before restarting
    max_attempts: 3           # Maximum number of restart attempts
    window: 120s              # Within this time window
```

---

## 5. Logging Configuration

### JSON File Driver (default)

```yaml
services:
  app:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"       # Maximum 10MB per log file
        max-file: "5"         # Keep up to 5 files
        compress: "true"      # Compress old log files
```

### Structured Logging in the App

```javascript
// Node.js: structured JSON logging
const logger = {
  info: (msg, meta = {}) =>
    console.log(
      JSON.stringify({
        level: "info",
        msg,
        timestamp: new Date().toISOString(),
        ...meta,
      })
    ),
  error: (msg, meta = {}) =>
    console.error(
      JSON.stringify({
        level: "error",
        msg,
        timestamp: new Date().toISOString(),
        ...meta,
      })
    ),
};
```

---

## 6. Monitoring Stack (Optional)

If the user requests monitoring, add Prometheus + Grafana:

```yaml
# docker-compose.monitoring.yml
# Usage: docker compose -f docker-compose.yml -f docker-compose.monitoring.yml up -d
services:
  prometheus:
    image: prom/prometheus:v3.4.1
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./docker/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - monitoring
      - app-network
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=15d"

  grafana:
    image: grafana/grafana:11.6.0
    container_name: grafana
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_USER=${GRAFANA_USER:-admin}
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD:-admin}
    volumes:
      - grafana-data:/var/lib/grafana
    ports:
      - "3001:3000"
    depends_on:
      - prometheus
    networks:
      - monitoring

  # Node Exporter — system metrics (CPU, RAM, Disk)
  node-exporter:
    image: prom/node-exporter:v1.9.1
    container_name: node-exporter
    restart: unless-stopped
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--path.rootfs=/rootfs"
    networks:
      - monitoring

  # cAdvisor — container metrics
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.52.1
    container_name: cadvisor
    restart: unless-stopped
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - monitoring

volumes:
  prometheus-data:
  grafana-data:

networks:
  monitoring:
    driver: bridge
```

### Prometheus Config

```yaml
# docker/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Prometheus self-monitoring
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  # Node Exporter (system metrics)
  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]

  # cAdvisor (container metrics)
  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]

  # Application metrics (if the app exposes a /metrics endpoint)
  - job_name: "app"
    static_configs:
      - targets: ["app:3000"]
    metrics_path: /metrics
    scrape_interval: 10s
```

---

## 7. Quick Status Check

```bash
# View health status of all containers
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# View detailed health check info
docker inspect --format='{{json .State.Health}}' <container_name> | jq

# View health check logs
docker inspect --format='{{range .State.Health.Log}}{{.Output}}{{end}}' <container_name>

# Test health endpoint directly
curl -s http://localhost:3000/health | jq
```
