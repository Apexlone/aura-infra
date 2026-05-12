# AuraSkill Infra 🚀

This repository manages the orchestration of the **AuraSkill** ecosystem development environment using native Docker Engine on WSL2 and GitOps practices with Portainer.

## 🛠️ Technologies Used
*   **Environment:** WSL2 (Ubuntu) - Native Docker Engine
*   **Orchestration:** Docker Compose
*   **Management:** Portainer (Recommended for GitOps)
*   **Identity Provider:** Keycloak 24+
*   **Database (Application):** PostgreSQL 16 (Local via Docker)
*   **Database (Keycloak):** PostgreSQL (Remote via Supabase)

---

## 🏗️ Environment Architecture
The infrastructure is configured to ensure data isolation and long-term persistence:

1.  **`auraskill-db-app`**: Local Postgres container dedicated to Java/Spring Boot application.
    *   **Port:** 5432
    *   **Persistence:** Data is mapped to `./data/app-db`. Since you use WSL, verify that write permissions in the folder are correct (`chmod 777` or `chown`).
2.  **`auraskill-keycloak`**: Authentication server persisted in **Supabase**.
    *   **Port:** 8080
    *   **Advantage:** Realms and users are saved in the cloud, facilitating local resets.

---

## 🚀 Quick Start Guide (WSL Ubuntu Environment)

### 1. Prerequisites
*   **Docker Engine** and **Docker Compose** installed directly on Ubuntu.
*   **Remove Native Postgres:** You need to uninstall or stop Postgres that runs directly on Ubuntu to free port 5432.
    ```bash
    sudo service postgresql stop
    # Or to uninstall completely:
    sudo apt-get --purge remove postgresql\*
    ```
*   **Portainer:** Ensure Portainer is installed by mapping the Docker socket:
    ```bash
    docker run -d -p 9443:9443 --name portainer --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/data portainer/portainer-ce:latest
    ```

---

## 🔄 Developer Workflow (CI/CD)

### Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                   DEVELOPER ENVIRONMENTS                         │
│                                                                   │
│  LOCAL (Your PC)       →    HML (Staging)    →    PROD          │
│                                                                   │
│  feature/* branch         hml branch            main branch       │
│  (development)           (testing)             (production)      │
└─────────────────────────────────────────────────────────────────┘
         ↓                      ↓                      ↓
     docker-compose        GitHub Actions          GitHub Actions
     (dev)                 Self-hosted Runner       Self-hosted Runner
     
     Hot reload            Real build              Real build
     Local volumes         Containers              Containers
     next dev             Automated deploy        Automated deploy
     spring-boot:run      docker-compose up       docker-compose up
```

### 📋 Complete Workflow Steps

#### **PHASE 1: Local Development (Branch feature/*)**

1. **Clone the required repositories:**
   ```bash
   cd ~/devtools/repos
   
   # If you haven't cloned them yet
   git clone https://github.com/guiassys/aura-api.git
   git clone https://github.com/guiassys/aura-front.git
   git clone https://github.com/guiassys/aura-infra.git
   ```

2. **Start the local development environment:**
   ```bash
   cd ~/devtools/repos/aura-infra
   
   # Pull to ensure you have the latest changes
   git pull origin main
   
   # Create a .env file if you don't have one
   # See "Environment Variables Configuration" section below
   
   # Stop all running containers before restarting the stack
   docker compose -f docker/dev/docker-compose.yml down
   
   # Start containers in development mode
   docker compose -f docker/dev/docker-compose.yml up -d
   ```

3. **Check if containers are running:**
   ```bash
   docker compose -f docker/dev/docker-compose.yml ps
   
   # Or via Portainer: https://localhost:9443
   ```

4. **Make changes to aura-api or aura-front:**
   ```bash
   # Example: editing aura-api
   cd ~/devtools/repos/aura-api
   
   # Create a branch for your feature
   git checkout -b feature/my-feature
   
   # Make your changes...
   # DEV containers will hot reload automatically
   ```

5. **Test your changes locally:**
   - Backend: http://localhost:8081
   - Frontend: http://localhost:3000
   - Keycloak: http://localhost:8080
   - Database: localhost:5432

6. **Commit and push:**
   ```bash
   cd ~/devtools/repos/aura-api
   git add .
   git commit -m "feat: description of your change"
   git push origin feature/my-feature
   ```

---

#### **PHASE 2: Pull Request and Testing (Branch hml)**

1. **Create a Pull Request (PR) to the hml branch:**
   - Go to: https://github.com/guiassys/aura-api/pulls
   - Click "New pull request"
   - Compare: `feature/my-feature` → `hml`
   - Describe your changes
   - Create the PR

2. **Wait for the pipeline to execute (if configured):**
   - The pipeline runs tests automatically
   - Check the status in the PR

3. **Merge the PR to the hml branch:**
   - After approval, click "Merge pull request"
   - Delete the feature branch after merge

4. **Trigger the HML deploy pipeline:**
   ```bash
   # The pipeline triggers automatically when you push to hml
   # But you can also force it by making a commit to the hml branch
   
   cd ~/devtools/repos/aura-infra
   git pull origin hml
   
   # Or make a small test commit
   git commit --allow-empty -m "trigger: deploy hml"
   git push origin hml
   ```

---

#### **PHASE 3: HML Environment - Staging (Production Testing)**

The HML environment is **triggered automatically** when you push to the `hml` branch in any repository.

**What happens:**

1. ✅ GitHub Actions triggers the `hml-deploy.yml` workflow
2. ✅ Self-hosted Runner (your WSL) receives the job
3. ✅ Runs git pull for aura-api
4. ✅ Runs git pull for aura-front
5. ✅ Executes: `docker compose -f docker/hml/docker-compose.yml down`
6. ✅ Executes: `docker compose -f docker/hml/docker-compose.yml up -d --build`
7. ✅ Containers are rebuilt and deployed

**Monitor the HML deployment:**

- **Option 1: Monitor via GitHub Actions (Recommended)**
  1. Go to: https://github.com/guiassys/aura-infra/actions
  2. Click the latest "HML Deploy" workflow
  3. See steps being executed in real-time

- **Option 2: Monitor via Portainer**
  1. Access: https://localhost:9443
  2. Log in with your credentials
  3. Go to: Containers
  4. Search for containers prefixed with "auraskill"
  5. Check logs in real-time

- **Option 3: Via Terminal (WSL)**
  ```bash
  # Check container status
  docker compose -f ~/devtools/repos/aura-infra/docker/hml/docker-compose.yml ps
  
  # View logs in real-time
  docker compose -f ~/devtools/repos/aura-infra/docker/hml/docker-compose.yml logs -f
  
  # View logs for a specific container
  docker compose -f ~/devtools/repos/aura-infra/docker/hml/docker-compose.yml logs -f auraskill-api
  ```

**Test the HML environment:**
- Backend: http://localhost:8081
- Frontend: http://localhost:3000
- Keycloak: http://localhost:8080
- Portainer: https://localhost:9443

---

#### **PHASE 4: Production (Branch main - FUTURE)**

When the infrastructure is ready for production:

```bash
# Merging hml to main triggers deploy to PROD
cd ~/devtools/repos/aura-infra
git checkout hml
git pull origin hml
git checkout main
git pull origin main
git merge hml
git push origin main
```

The `prod-deploy.yml` pipeline would be triggered automatically.

---

## 📊 Pipeline Monitoring

### Via GitHub Actions (Recommended for Development)

**Access in real-time:**
1. https://github.com/guiassys/aura-infra/actions
2. Click the latest workflow
3. You will see:
   - ✅ Checkout infra
   - ✅ Update Backend
   - ✅ Update Frontend
   - ✅ Deploy Containers
   - Status (success/failure)
   - Execution time

**Interpreting Results:**
- ✅ Green = Success
- ❌ Red = Failure (check the logs)

### Via Portainer (Recommended for Monitoring)

1. Access: https://localhost:9443
2. Go to: **Containers**
3. Search for "auraskill" containers
4. Monitor:
   - Status (Running/Stopped)
   - CPU/Memory usage
   - Real-time logs

### Via Terminal (WSL)

```bash
# Monitor execution in real-time
watch -n 1 'docker compose -f ~/devtools/repos/aura-infra/docker/hml/docker-compose.yml ps'

# View logs with timestamp
docker compose -f ~/devtools/repos/aura-infra/docker/hml/docker-compose.yml logs --timestamps -f

# Filter logs for a specific service
docker compose -f ~/devtools/repos/aura-infra/docker/hml/docker-compose.yml logs auraskill-api -f
```

---

## 🔐 Environment Variables Configuration

Create a `.env` file in the root of `aura-infra/docker/dev`, `aura-infra/docker/hml` and `aura-infra/docker/prd`:


```bash
# Database - Application
APP_DB_USER=postgres
APP_DB_PASSWORD=your_secure_password
APP_DB_NAME=auraskill
APP_DB_PORT=5432

# Keycloak
KC_ADMIN_USER=admin
KC_ADMIN_PASSWORD=your_secure_password
KC_PORT=8080
KC_DB_URL_SUPABASE=jdbc:postgresql://...supabase...
KC_DB_USER_SUPABASE=postgres
KC_DB_PASSWORD_SUPABASE=your_supabase_password

# Application
KEYCLOAK_CLIENT_ID=your_client_id
KEYCLOAK_CLIENT_SECRET=your_client_secret
KEYCLOAK_ISSUER=http://localhost:8080/realms/your_realm
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=your_secure_secret
NEXT_PUBLIC_API_URL=http://localhost:8081
INTERNAL_API_URL=http://auraskill-api:8081
```

---

## 🚨 Troubleshooting

### Pipeline fails on HML deploy

1. **Check if the runner is running:**
   ```bash
   ps aux | grep Runner.Listener
   ```

2. **Check if Docker Compose is available:**
   ```bash
   docker compose --version
   ```

3. **Check runner logs:**
   ```bash
   tail -100 ~/actions-runner/_diag/Runner_*.log
   ```

### Containers won't start

1. **Check data permissions:**
   ```bash
   sudo chmod -R 777 ~/devtools/repos/aura-infra/data
   ```

2. **Clean containers and volumes:**
   ```bash
   docker compose -f docker/hml/docker-compose.yml down -v
   docker compose -f docker/hml/docker-compose.yml up -d
   ```

3. **Check available ports:**
   ```bash
   netstat -tuln | grep -E "3000|5432|8080|8081"
   ```

### Code changes don't appear in DEV

1. **Check if volumes are correct:**
   ```bash
   docker compose -f docker/dev/docker-compose.yml ps
   docker inspect auraskill-api | grep -A 5 Mounts
   ```

2. **Restart the container:**
   ```bash
   docker compose -f docker/dev/docker-compose.yml restart auraskill-api
   ```

---

## 📞 Support

For more information:
- 📖 Project documentation: See `aiprompts/tasks/001__create_CICD_pipeline.md`
- 🐳 Docker Compose: https://docs.docker.com/compose
- 🔐 Keycloak: https://www.keycloak.org/documentation
- 🚀 GitHub Actions: https://docs.github.com/en/actions
