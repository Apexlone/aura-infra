# AuraSkill Infra 🚀 | Apexlone Organization

This repository manages the orchestration of the AuraSkill ecosystem development and staging environments. It implements GitOps practices using GitHub Actions and a self-hosted runner integrated with Docker Engine on WSL2.

## Technologies Used
- Host Environment: WSL2 (Ubuntu 22.04/24.04)
- Container Runtime: Native Docker Engine
- Orchestration: Docker Compose
- Management: Portainer (Visual Stack Monitoring)
- Identity Provider: Keycloak 24+
- Databases:
    - auraskill-db-app: PostgreSQL 16 (Application data)
    - auraskill-db-keycloak: PostgreSQL 16 (Identity data)
- CI/CD: GitHub Actions + Self-hosted Runner (apexlone-srv)

---

## Environment Architecture

The infrastructure is designed for isolation and high availability within the local network.

1. Network Isolation: All containers communicate through a dedicated internal bridge network.
2. Persistence: Database files are mapped to ./data/ on the host to ensure no data loss during container recreation.
3. Security: External access is restricted to essential ports (3000, 8080, 8081, 9443).

---

## CI/CD Workflow

### Deployment Pipeline
Developer Push > GitHub Actions > Self-hosted Runner > Docker Build > System Live

### Phase-by-Phase Process

#### Phase 1: Local Development
Developers should use the docker/dev/docker-compose.yml for hot-reloading features.
Command: docker compose -f docker/dev/docker-compose.yml up -d

#### Phase 2: Staging (HML)
Pushes to the hml or main branch trigger the automated deployment.
1. Runner Pickup: The apexlone-srv runner detects the job.
2. Environment Sync: The runner pulls aura-api and aura-front into the workspace.
3. Re-provisioning: docker compose -f docker/hml/docker-compose.yml up -d --build

---

## Setup and Installation

### 1. WSL2 Prerequisites
Ensure Docker is installed natively inside Ubuntu (not just via Docker Desktop).
Check versions:
- docker --version
- docker compose version

### 2. GitHub Runner Configuration
To allow the runner to execute jobs for public repositories in the Apexlone organization:
1. Go to Organization Settings > Actions > Runner Groups.
2. Select the Default group.
3. Check the box "Allow public repositories".
4. In the aura-infra Actions tab, manually approve the first run if requested.

---

## Environment Variables (.env and secrets.env)

To manage environment variables and secrets securely, the project now uses two separate files:
- `.env`: For non-sensitive configuration variables.
- `secrets.env`: For sensitive information like passwords and API keys.

Both files should be created in `docker/hml/` and `docker/dev/` based on the templates below. **Ensure `secrets.env` is properly secured and not committed to version control.**

### Template for `.env` (Non-sensitive variables)
```
# DATABASE CONFIG:
APP_DB_USER=postgres
APP_DB_NAME=auraskill

# KEYCLOAK CONFIG:
KC_DB_URL=jdbc:postgresql://auraskill-db-keycloak:5432/keycloak
KC_DB_USERNAME=postgres
KC_HOSTNAME=localhost
KC_ADMIN_USER=admin

# FRONTEND CONFIG (NextAuth):
NEXT_PUBLIC_API_URL=http://localhost:8081
NEXT_PUBLIC_KEYCLOAK_ISSUER=http://localhost:8080/realms/auraskill
```

### Template for `secrets.env` (Sensitive variables - DO NOT COMMIT!)
```
# DATABASE SECRETS:
APP_DB_PASSWORD=secure_app_password_here

# KEYCLOAK SECRETS:
KC_DB_PASSWORD=secure_keycloak_db_password_here
KC_ADMIN_PASSWORD=secure_keycloak_admin_password_here

# REDIS SECRETS:
REDIS_PASSWORD=redis_secure_password_here

# FRONTEND SECRETS (NextAuth):
NEXTAUTH_SECRET=your_long_and_stable_secret_here
KEYCLOAK_CLIENT_SECRET=your_keycloak_client_secret_here
```
---

## Troubleshooting

### Common Build Failures
- TypeScript "Property does not exist": Usually occurs in the Frontend build when NextAuth sessions are not properly augmented. Ensure next-auth.d.ts is present in the aura-front repository.
- Runner Offline: If the pipeline is stuck in "Queued", check the WSL terminal by running ./run.sh in the actions-runner folder.
- Port Conflicts: If containers fail to start, ensure no native services are using ports 5432 or 8080. Check with: sudo lsof -i :5432

---

## Monitoring
- GitHub Actions: View Pipeline Status on the Actions tab of this repo.
- Local Management: Access Portainer at https://localhost:9443
- Logs: docker compose -f docker/hml/docker-compose.yml logs -f --tail=100

---

## Extended Documentation
For system design patterns, OIDC flows, and API specifications, please visit the documentation repository:
Apexlone Documentation (aura-doc) at https://github.com/Apexlone/aura-doc

Maintained by Apexlone Organization.

## Test pipeline
- 01 - Forcing github actions