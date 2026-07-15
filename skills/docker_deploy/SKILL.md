---
name: docker-deploy
description: >
  Deploy systems using Docker. This skill supports automatic Dockerfile
  generation, docker-compose.yml, Docker Swarm config, health check &
  monitoring. Activate when the user requests containerization, deployment,
  or Docker setup for a project.
---

# Docker Deploy Skill

This skill supports system deployment using Docker with the following
capabilities:

1. **Docker Basics** — Build image, push to registry, run containers
2. **Docker Compose** — Multi-container on a single server
3. **Docker Swarm** — Cluster orchestration, service scaling
4. **Health Check & Monitoring** — Ensure uptime and observability

---

## Workflow

### Step 1: Analyze the Project

Before creating any Docker files, you **MUST** analyze the project:

1. Identify the language/framework (Node.js, Python, Go, Java, Rust, PHP,
   .NET…)
2. Read dependency management files (`package.json`, `requirements.txt`,
   `go.mod`, `pom.xml`, `Cargo.toml`, `composer.json`, `*.csproj`…)
3. Determine the entry point and build process
4. Check whether `Dockerfile`, `docker-compose.yml`, `.dockerignore` already
   exist
5. Identify dependent services (database, cache, message queue…)

### Step 2: Create Dockerfile

Refer to the appropriate template in
[references/dockerfiles.md](references/dockerfiles.md).

**Mandatory principles when writing Dockerfiles:**

- ✅ Use **multi-stage builds** to reduce image size
- ✅ Use **Alpine** or **slim** base images when possible
- ✅ Set an explicit `WORKDIR`
- ✅ Copy dependency files first, install, then copy source code (leverage Docker
  layer caching)
- ✅ Run processes as a **non-root user**
- ✅ Add `HEALTHCHECK` instruction
- ✅ Use `.dockerignore` to exclude unnecessary files
- ❌ Do NOT use `latest` tag for base images — always pin a specific version
- ❌ Do NOT store secrets in Dockerfile — use build args or env files
- ❌ Do NOT run multiple processes in one container (unless using a supervisor)

### Step 3: Create Docker Compose (if needed)

Refer to patterns in
[references/compose-patterns.md](references/compose-patterns.md).

**Principles:**

- Use clear, meaningful service names
- Use `depends_on` with `condition: service_healthy` to ensure startup order
- Use volumes for data persistence (database, uploads…)
- Create separate networks for each service group
- Environment variables via `.env` file (do NOT hardcode)
- Resource limits (`deploy.resources.limits`)

### Step 4: Docker Swarm (if cluster deployment is requested)

Refer to [references/swarm.md](references/swarm.md).

**Principles:**

- Use `docker stack deploy` instead of `docker-compose up`
- Configure replicas, update policy, rollback policy
- Use placement constraints for sensitive services
- Manage secrets with `docker secret`
- Use overlay networks for inter-service communication

### Step 5: Health Check & Monitoring

Refer to [references/healthcheck.md](references/healthcheck.md).

**Required configurations:**

- `HEALTHCHECK` in Dockerfile
- Health check endpoint in the application (`/health` or `/healthz`)
- Restart policy (`restart: unless-stopped` or `on-failure`)
- Logging driver configuration
- Guide for integrating monitoring stack (Prometheus + Grafana) if requested by
  the user

---

## Recommended Output File Structure

```
project/
├── Dockerfile                  # Main application
├── .dockerignore               # Exclude unnecessary files
├── docker-compose.yml          # Development / single-server deploy
├── docker-compose.prod.yml     # Production overrides
├── docker-stack.yml            # Swarm deployment (if using Swarm)
├── .env.example                # Template environment variables
└── docker/                     # Additional Docker configs
    ├── nginx/
    │   ├── Dockerfile
    │   └── nginx.conf
    └── scripts/
        ├── entrypoint.sh       # Custom entrypoint
        └── healthcheck.sh      # Health check script
```

---

## Checklist Before Completion

- [ ] Dockerfile uses multi-stage build
- [ ] `.dockerignore` has been created
- [ ] Non-root user in container
- [ ] Health check has been configured
- [ ] Environment variables are not hardcoded
- [ ] Volume mapping for persistent data
- [ ] Build & run instructions have been provided to the user

---

## Important Notes

- **Always ask the user** about port mapping, volume mounts, and required
  environment variables before creating configs
- **Do not assume** database credentials — ask the user or use clear
  placeholders in `.env.example`
- When the user has an existing project, **read the source code carefully** to
  understand the correct build and run process
- Explain each design decision in the Dockerfile with comments
