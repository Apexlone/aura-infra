# AuraSkill Infra 🚀

Este repositório centraliza a orquestração do ambiente de desenvolvimento do projeto AuraSkill utilizando Docker Compose.

## 🛠️ Tecnologias Utilizadas
*   **Ambiente:** WSL2 (Ubuntu)
*   **Orquestração:** Docker Compose
*   **Identity Provider:** Keycloak 24+
*   **Database:** PostgreSQL (Instâncias para Backend e Keycloak)

## 🏗️ Estrutura do Ambiente
A infraestrutura local é composta pelos seguintes serviços:
1.  **Postgres (Local):** Banco de dados para a aplicação Java/Spring Boot.
2.  **Postgres (Keycloak):** Banco de dados dedicado para persistência do Keycloak.
3.  **Keycloak:** Servidor de autenticação e autorização.

## 🚀 Como Iniciar

### Pré-requisitos
*   Docker Desktop (com integração WSL2 ativada) ou Docker Engine instalado no Ubuntu.

### Passo a Passo
1. Clone este repositório:
   ```bash
   git clone [https://github.com/guiassys/auraskill-infra.git](https://github.com/guiassys/auraskill-infra.git)
   cd auraskill-infra