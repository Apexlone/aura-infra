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
REDIS_PASSWORD=<password_seguro>
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
# Execution Plan — Redis (HML)

## Prerequisites

- Docker and Docker Compose installed and running
- Access to the `aura-infra` repository
- Basic knowledge of Redis and environment variables

**References:**
- #file:EXPERT_DEVOPS_CICD_PIPELINE_SPECIALIST.md
- #file:COR_INSTRUCTIONS.md

## Objective

Bring Redis into the HML docker-compose stack as a cache server using the same host-mounted data pattern used by the database services.

Key requirements:
- Integrate with `auraskill-network`
- Be reachable by `auraskill-api` and `auraskill-frontend`
- Persist data to host path `/home/guiassys/docker-data/aura-infra/redis-data`
- Provide password authentication and a healthcheck
- Align `auraskill-api` environment with Redis connection vars

---

## Success Criteria

- Redis container starts and passes its healthcheck
- `auraskill-api` connects to Redis using the configured SPRING_DATA_REDIS_* variables
- Redis data is persisted under `/home/guiassys/docker-data/aura-infra/redis-data`

---

## Best Practices Applied

- Consistency with existing host-mounted data directories
- Password-protected Redis and healthchecks
- Clear application environment variables for runtime connectivity
- CI/CD alignment: predictable host paths for backups and operations
