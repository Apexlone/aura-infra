# AuraSkill Infra 🚀

Este repositório gerencia a orquestração do ambiente de desenvolvimento do ecossistema **AuraSkill** utilizando Docker Engine nativo no WSL2 e práticas de GitOps com Portainer.

## 🛠️ Tecnologias Utilizadas
*   **Ambiente:** WSL2 (Ubuntu) - Docker Engine Nativo
*   **Orquestração:** Docker Compose
*   **Gerenciamento:** Portainer (Recomendado para GitOps)
*   **Identity Provider:** Keycloak 24+
*   **Database (Application):** PostgreSQL 16 (Local via Docker)
*   **Database (Keycloak):** PostgreSQL (Remoto via Supabase)

---

## 🏗️ Estrutura do Ambiente
A infraestrutura é configurada para garantir isolamento de dados e persistência de longo prazo:

1.  **`auraskill-db-app`**: Container Postgres local dedicado à aplicação Java/Spring Boot.
    *   **Porta:** 5432
    *   **Persistência:** Os dados são mapeados em `./data/app-db`. Como você usa WSL, verifique se as permissões de escrita na pasta estão corretas (`chmod 777` ou `chown`).
2.  **`auraskill-keycloak`**: Servidor de autenticação persistido no **Supabase**.
    *   **Porta:** 8080
    *   **Vantagem:** Realms e usuários são salvos na nuvem, facilitando resets locais.

---

## 🚀 Guia de Início Rápido (Ambiente WSL Ubuntu)

### 1. Pré-requisitos
*   **Docker Engine** e **Docker Compose** instalados diretamente no Ubuntu.
*   **Postgres Nativo Removido:** É necessário desinstalar ou parar o Postgres que roda direto no Ubuntu para liberar a porta 5432.
    ```bash
    sudo service postgresql stop
    # Ou para desinstalar de vez:
    sudo apt-get --purge remove postgresql\*
    ```
*   **Portainer:** Certifique-se de que o Portainer foi instalado mapeando o socket do Docker:
    ```bash
    docker run -d -p 9443:9443 --name portainer --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/data portainer/portainer-ce:latest