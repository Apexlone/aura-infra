# Execution Plan — Integrate `aura-eureka` into CI/CD pipeline (Infra & Docker)

## Overview (Checklist)
- [ ] Verify prerequisites and access permissions to the `Apexlone/aura-eureka` repository.
- [ ] Update `docker/hml/.env` to add variables required for `aura-eureka` and the project path.
- [ ] Update `docker/hml/docker-compose.yml` to add the `aura-eureka` service (build or image, volumes, healthcheck, network).
- [ ] Update `.github/workflows/infra-deploy.yml` to check out the `aura-eureka` repository and include the service in the pipeline's `docker compose up`.
- [ ] Test locally with `docker compose --env-file docker/hml/.env -f docker/hml/docker-compose.yml up -d aura-eureka` and validate the healthcheck.

> Note: This document follows the best practices in `#file:COR_INSTRUCTIONS.md` and applies the expertise from the `#file:EXPERT_DEVOPS_CICD_PIPELINE_SPECIALIST.md` persona.

---

## Prerequisites

- A self-hosted GitHub Actions runner with Docker and Docker Compose installed and allowed to build images and run containers.
- Access to the `Apexlone/aura-eureka` repository via the runner's token or public access. If private, ensure appropriate credentials (`GITHUB_TOKEN` or runner credentials) are available.
- Host directory structure on the runner should be compatible with host-mounted paths used by the project (`/home/guiassys/...`) or adjust environment variables accordingly.
- Basic familiarity with Spring Boot (Eureka Server) and the `Dockerfile` present in the `aura-eureka` repository.

---

## Objective

Add the `aura-eureka` project (Eureka Server) to the infrastructure managed by `docker/hml/docker-compose.yml` and integrate it into the CI/CD pipeline (`.github/workflows/infra-deploy.yml`) so the service is built and deployed automatically when the infra workflow runs.

Expected outcomes:
- `aura-eureka` is built (or pulled) from the `Apexlone/aura-eureka` repository on the runner.
- It starts and attaches to the `auraskill-network` so other services (e.g. `auraskill-api`, `auraskill-frontend`) can register.
- It exposes a healthcheck and configuration is provided via `docker/hml/.env`.

---

## Implementation Steps (detailed)

Below are the changes required per file, with example snippets to apply.

1) Update `docker/hml/.env`

- Goal: add variables that point to the `aura-eureka` source on the runner and Eureka-specific settings (port, app name, health endpoint).

Add this block (place before the `# --- BACKEND (SPRING BOOT) ---` section or another suitable location):

```dotenv
# --- EUREKA SERVER ---
AURA_EUREKA_PATH=/home/guiassys/actions-runner/_work/aura-eureka/aura-eureka
EUREKA_APP_NAME=eureka-server
EUREKA_PORT=8761
EUREKA_HEALTH_ENDPOINT=/actuator/health
```

Notes:
- `AURA_EUREKA_PATH` must match the `path` used by the checkout step in the workflow (see `infra-deploy.yml` changes below).
- Adjust `EUREKA_PORT` to match the `application.properties` or `application.yml` of the `aura-eureka` project.


2) Update `docker/hml/docker-compose.yml`

- Goal: add a new `aura-eureka` service that builds from `${AURA_EUREKA_PATH}` (or uses an image). Include host-mounted volumes if needed, attach to `auraskill-network`, add a healthcheck and restart policy.

Example service block (insert before `auraskill-api` or in a logical place):

```yaml
  # --- EUREKA SERVER ---
  aura-eureka:
    build:
      context: ${AURA_EUREKA_PATH}
      dockerfile: Dockerfile
    container_name: auraskill-eureka
    restart: always
    ports:
      - "${EUREKA_PORT}:${EUREKA_PORT}"
    environment:
      - SPRING_APPLICATION_NAME=${EUREKA_APP_NAME}
      - SERVER_PORT=${EUREKA_PORT}
      - MANAGEMENT_ENDPOINT_HEALTH_SHOWDETAILS=always
    networks:
      - auraskill-network
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:${EUREKA_PORT}${EUREKA_HEALTH_ENDPOINT} || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

Implementation notes and alternatives:
- If `aura-eureka` has a published image in a registry, replace the `build:` block with `image: repo/aura-eureka:tag` for faster deployments.
- Ensure the repository's `Dockerfile` exposes the configured port and includes the correct Java runtime/base image.


3) Update `.github/workflows/infra-deploy.yml`

- Goal: make sure the runner checks out the `aura-eureka` repository before running `docker compose up`, and include the `aura-eureka` service in the list of services to start.

Recommended changes:

A) Add a checkout step for the `Apexlone/aura-eureka` repository (place before the `Deploy Infra Stack` step):

```yaml
      - name: Checkout aura-eureka repository
        uses: actions/checkout@v4
        with:
          repository: Apexlone/aura-eureka
          path: aura-eureka
          ref: main
```

This will place the `aura-eureka` code at `${{ github.workspace }}/aura-eureka`. Update `AURA_EUREKA_PATH` in `.env` to point to this path or use the absolute runner path.

B) Update the `Deploy Infra Stack` step to include `aura-eureka` in the `docker compose up` command, for example:

```yaml
      - name: Deploy Infra Stack
        run: |
          docker compose \
            --env-file /home/guiassys/devtools/repos/aura-infra/docker/hml/.env \
            -f ${{ github.workspace }}/docker/hml/docker-compose.yml \
            up -d db-app db-keycloak keycloak auraskill-redis aura-eureka auraskill-api auraskill-frontend
```

Important notes:
- If `AURA_EUREKA_PATH` points to a different location (e.g. `actions-runner/_work/aura-eureka/aura-eureka`), either adapt the checkout `path` or update `.env` accordingly.
- On self-hosted runners, verify permissions and disk space. The workflow already contains a `Pre-checkout cleanup` step — avoid removing the directory where `aura-eureka` will be checked out. Consider narrowing the cleanup or adjusting step order.


4) Validations and dependency adjustments

- To ensure startup ordering, update `depends_on` for services that rely on Eureka (such as `auraskill-api` and `auraskill-frontend`) to wait for `aura-eureka` using `condition: service_healthy` (or `service_started` if the service lacks a healthcheck):

```yaml
    depends_on:
      aura-eureka:
        condition: service_healthy
      db-app:
        condition: service_started
      keycloak:
        condition: service_started
      auraskill-redis:
        condition: service_healthy
```

- Ensure `auraskill-api` and `auraskill-frontend` have environment variables configured to locate Eureka (for example: `EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://auraskill-eureka:${EUREKA_PORT}/eureka`). Add these variables to `.env` and to the services' `environment` blocks.

## Success Criteria

- ✅ The `infra-deploy.yml` workflow checks out the `aura-eureka` repository and starts the service via `docker compose` without errors.
- ✅ The `auraskill-eureka` container reports `healthy` (healthcheck passes).
- ✅ `auraskill-api` and `auraskill-frontend` can discover/register with the Eureka Server.
- ✅ Any required volumes persist data (if applicable) and host paths are correctly configured on the runner.


---

## Risks and Mitigations

- Access/permissions to `Apexlone/aura-eureka`: ensure tokens/credentials are configured in the runner.
- Path mismatches between the runner checkout and `AURA_EUREKA_PATH`: prefer relative paths inside `${{ github.workspace }}` and keep `.env` in sync.
- The `Pre-checkout cleanup` step may remove the target checkout directory: consider narrowing the cleanup scope or moving the checkout step before cleanup where appropriate.


---

## Final Notes / Best Practices (applying `COR_INSTRUCTIONS`)

- Be explicit about paths and environment variables; document any assumptions.
- Use `depends_on` for ordering but rely on healthchecks for readiness.
- Prefer versioned images over `latest` to improve deployment reproducibility.
- Test changes in an integration branch before applying to `main`.


---

If you agree, I will proceed to apply the changes to the following files:
- `.github/workflows/infra-deploy.yml`
- `docker/hml/docker-compose.yml`
- `docker/hml/.env`

Tell me if you want me to apply the edits automatically (I will modify the files and run basic validations in the workspace), or if you prefer to review this English document first.
