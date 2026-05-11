# Execution Plan — Local CI/CD with GitHub Actions + WSL + Docker

## Prerequisites

Before executing this task, you MUST read and use the following files as mandatory context and instruction sources:

- #file:EXPERT_DEVOPS_CICD_PIPELINE_SPECIALIST.md
- #file:COR_INSTRUCTIONS.md

All analysis, decisions, implementations, and outputs must strictly follow the standards, rules, architecture principles, and execution guidelines defined in these files.

## Objective

Implement a local CI/CD pipeline (On-Premise) using:

- GitHub Actions
- Self-hosted Runner on WSL Ubuntu
- Docker + Docker Compose
- Portainer
- Spring Boot Backend
- Next.js Frontend
- Keycloak
- PostgreSQL

Desired flow:

```text
Pull Request → Branch hml → Pipeline → Automatic Deploy
```

---

# Recommended Architecture

```text
GitHub
   ↓
GitHub Actions
   ↓
Self-hosted Runner (WSL Ubuntu)
   ↓
Docker Compose
   ↓
Local Deploy
   ↓
Portainer (Monitoring)
```

---

# Architecture Objectives

## Requirements Met

- [x] Free software
- [x] On-Premise
- [x] Simple
- [x] Low maintenance
- [x] Compatible with multiple repositories
- [x] Automatic deploy
- [x] Scalable in the future

---

# Current Structure

## Repositories

### Backend
https://github.com/guiassys/aura-api

### Frontend
https://github.com/guiassys/aura-front

### Infrastructure
https://github.com/guiassys/aura-infra

---

# Problem with Current Structure

Currently containers use:

```yaml
volumes:
  - /home/guiassys/devtools/repos/auraskill-api:/app
```

This approach only works for local development.

## Problems

- Does not generate real Docker images
- Does not allow clean automated deploy
- Couples container to local filesystem
- Does not allow image versioning
- Hinders CI/CD

---

# Correct Strategy

Separate environments:

## DEV Environment

Objective:

- Hot reload
- Local development
- Mounted volumes
- next dev
- spring-boot:run

## HML Environment

Objective:

- Real build
- Real Docker images
- Automated deploy
- Isolated containers
- Reproducible environment

---

# Recommended Structure

```text
aura-infra/

docker/
├── dev/
│   └── docker-compose.yml
│
├── hml/
│   └── docker-compose.yml
│
└── prod/
    └── docker-compose.yml
```

---

# Step 1 — Create Backend Dockerfile

## Repository

https://github.com/guiassys/aura-api

## File

```text
aura-api/Dockerfile
```

## Content

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build

WORKDIR /app

COPY . .

RUN mvn clean package -DskipTests

FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=build /app/target/*.jar app.jar

EXPOSE 8081

ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

# Step 2 — Create Frontend Dockerfile

## Repository

https://github.com/guiassys/aura-front

## File

```text
aura-front/Dockerfile
```

## Content

```dockerfile
FROM node:20-alpine AS deps

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

RUN npm run build

FROM node:20-alpine

WORKDIR /app

COPY --from=deps /app ./

EXPOSE 3000

CMD ["npm", "start"]
```

---

# Step 3 — Create DEV Docker Compose

## Objective

Local development environment.

## Features

- Hot reload
- Local volumes
- next dev
- spring-boot:run

## File

```text
docker/dev/docker-compose.yml
```

---

# Step 4 — Create HML Docker Compose

## Objective

Automated environment for CI/CD.

## Features

- No local volumes
- Real build
- Real containers
- Automatic deploy

## File

```text
docker/hml/docker-compose.yml
```

## Example

```yaml
services:

  db-app:
    image: postgres:16-alpine
    container_name: auraskill-db-app
    restart: always
    environment:
      POSTGRES_USER: ${APP_DB_USER}
      POSTGRES_PASSWORD: ${APP_DB_PASSWORD}
      POSTGRES_DB: ${APP_DB_NAME}
    ports:
      - "5432:5432"

  keycloak:
    image: quay.io/keycloak/keycloak:latest
    container_name: auraskill-keycloak
    restart: always
    command: start-dev
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME: ${KC_ADMIN_USER}
      KC_BOOTSTRAP_ADMIN_PASSWORD: ${KC_ADMIN_PASSWORD}
    ports:
      - "8080:8080"

  auraskill-api:
    build:
      context: ../../aura-api
    container_name: auraskill-api
    restart: always
    ports:
      - "8081:8081"

  auraskill-frontend:
    build:
      context: ../../aura-front
    container_name: auraskill-frontend
    restart: always
    ports:
      - "3000:3000"
```

---

# Step 5 — Install GitHub Actions Runner

## Objective

Allow GitHub to execute pipelines directly on WSL.

---

# Create Directory

```bash
mkdir ~/actions-runner
cd ~/actions-runner
```

---

# Download Runner

Official documentation:

https://docs.github.com/en/actions/hosting-your-own-runners

---

# Configure Runner

```bash
./config.sh
```

---

# Run Runner

```bash
./run.sh
```

---

# Expected Result

GitHub Actions will start executing jobs directly inside WSL Ubuntu.

---

# Step 6 — Create GitHub Actions Workflow

## Repository

https://github.com/guiassys/aura-infra

## File

```text
.github/workflows/hml-deploy.yml
```

---

# Initial Pipeline

```yaml
name: HML Deploy

on:
  push:
    branches:
      - hml

jobs:

  deploy:
    runs-on: self-hosted

    steps:

      - name: Checkout infra
        uses: actions/checkout@v4

      - name: Update Backend
        run: |
          cd ../aura-api
          git pull

      - name: Update Frontend
        run: |
          cd ../aura-front
          git pull

      - name: Deploy Containers
        run: |
          docker compose -f docker/hml/docker-compose.yml down
          docker compose -f docker/hml/docker-compose.yml up -d --build
```

---

# Complete Flow

## Development

```text
feature/* → local development
```

---

## Pull Request

```text
feature/* → hml
```

Pipeline executes:

- Backend build
- Frontend build
- Basic tests

---

## Merge on branch hml

Pipeline executes:

- docker compose build
- docker compose up -d
- Automatic deploy

---

# Portainer Role

Portainer will continue to be used for:

- Container visualization
- Logs
- Manual restart
- Visual administration
- Monitoring

Deploy will be the responsibility of GitHub Actions.

---

# Future Improvements

## Phase 2

- Automated tests
- Pipeline per repository
- Build cache
- Private Docker registry

---

## Phase 3

- Semantic versioning
- Automatic rollback
- Blue/Green deploy
- Traefik
- Local HTTPS

---

## Phase 4

- Observability
- Prometheus
- Grafana
- Loki

---

# Recommended Technologies

## CI/CD

- GitHub Actions
- Self-hosted Runner

## Containers

- Docker
- Docker Compose
- Portainer

## Database

- PostgreSQL

## Auth

- Keycloak

## Backend

- Spring Boot

## Frontend

- Next.js

---

# What NOT to Use Right Now

## Not recommended for this phase

- Kubernetes
- ArgoCD
- Jenkins
- GitLab Self-hosted
- Helm
- Terraform

---

# Final Expected Result

```text
GitHub PR/Merge
        ↓
GitHub Actions
        ↓
Runner on WSL
        ↓
Docker Compose
        ↓
Automatic Deploy
        ↓
Portainer monitors containers
```

---

# Next Steps

## Recommended Order

1. Create Dockerfiles
2. Separate DEV/HML
3. Create HML compose
4. Install self-hosted runner
5. Create workflow
6. Execute first automated deploy

---

# Important Notes

## Branches

Suggestion:

```text
main    → production
hml     → staging
feature → development
```

---

## Initial Strategy

In this first phase:

- Do NOT use Docker registry
- Do NOT use Kubernetes
- Do NOT use complex microservices

Objective:

- Simplicity
- Automation
- Reproducibility
- Low maintenance

---

# Final Infrastructure State

```text
WSL Ubuntu
│
├── aura-api
├── aura-front
├── aura-infra
│
├── actions-runner
│
└── docker
```