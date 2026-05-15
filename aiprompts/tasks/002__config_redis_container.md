# Execution Plan — Redis Container Configuration

## Prerequisites

- Docker and Docker Compose installed and running
- Access to `aura-infra` repository
- Basic understanding of Redis and environment variables

**References:**
- #file:EXPERT_DEVOPS_CICD_PIPELINE_SPECIALIST.md
- #file:COR_INSTRUCTIONS.md

## Objective

Add a Redis container to the HML docker-compose stack as a distributed cache server. Redis must:

- Integrate with `auraskill-network`
- Be accessible from `auraskill-api` and `auraskill-frontend`
- Persist data using absolute volume paths (consistent with other services)
- Include health checks and authentication
- Follow existing service patterns (matching db-app and db-keycloak)

---

## Implementation Steps

### Step 1: Update `.env` File

Add Redis configuration before `# --- BACKEND (SPRING BOOT) ---` section:

```dotenv
# --- REDIS CACHE SERVER ---
REDIS_HOST=auraskill-redis
REDIS_PORT=6379
REDIS_PASSWORD=redis_secure_password_hml
REDIS_DB=0
REDIS_TIMEOUT=60000
REDIS_MAX_RETRIES=3
```

---

### Step 2: Update `docker-compose.yml`

Insert this Redis service before `# --- BACKEND API (Spring Boot) ---` section:

```yaml
  # --- REDIS CACHE SERVER ---
  auraskill-redis:
    image: redis:7-alpine
    container_name: auraskill-redis
    restart: always
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 512mb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - /home/guiassys/docker-data/aura-infra/redis-data:/data
    networks:
      - auraskill-network
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
```

**Note:** Volume path `/home/guiassys/docker-data/aura-infra/redis-data` matches the pattern used by `db-app` and `db-keycloak` for consistency.

---

### Step 3: Update API Dependencies

Modify `auraskill-api` service's `depends_on:`:

```yaml
depends_on:
  db-app:
    condition: service_healthy
  keycloak:
    condition: service_started
  auraskill-redis:
    condition: service_healthy
```

---

## Success Criteria

- ✅ Redis container starts without errors
- ✅ Healthcheck passes
- ✅ API connects to Redis successfully
- ✅ Data persists after container restart
- ✅ No critical errors in logs

---

## Best Practices Applied

- Infrastructure as Code (version-controlled configuration)
- Container Orchestration (Docker Compose with health checks)
- Security (password authentication, network isolation)
- Persistence (absolute volume paths for explicit data management)
- Observability (health checks and logging)
- Standardization (consistent with db-app and db-keycloak patterns)
