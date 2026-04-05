# GitLab CI Project

> A pet project demonstrating a simplified end-to-end DevOps workflow — build, test, containerize, SAST scan, preview deploy, and production deploy.

This project consists of **two applications**:
- [DB_PROXY](https://github.com/StealLine/PROXY_DB_APP)
- [Frontend — Crypto Rate Checker](https://github.com/StealLine/Frontend-Crypto-Rate-Checker)

---

> ⚠️ **Disclaimer**
>
> The GitLab CI configuration and deployment logic here are **not intended for production use**. Some variables are intentionally hardcoded and certain values may be exposed. The primary goal was to demonstrate the ability to design and implement a **simplified end-to-end DevOps workflow**.
>
> Implementing a fully production-ready infrastructure (proper secret management, environment separation, IaC, etc.) would significantly exceed the scope of a single-developer pet project. **Intentional simplifications and trade-offs were made** to keep the project manageable while showcasing core CI/CD concepts. AND THIS IS MY FIRST GITLAB PROJECT!!!

---

# [DB_PROXY](https://github.com/StealLine/PROXY_DB_APP) — CI/CD Pipeline

The CI/CD pipeline for DB_PROXY is implemented using GitLab CI.

### Merge Request Pipeline

![MR Pipeline](https://github.com/user-attachments/assets/90fc4707-116e-41e4-9c2a-d59cd8cb8ed0)

### Main Branch Pipeline

![Main Pipeline](https://github.com/user-attachments/assets/739f8437-d754-4e67-bffc-d14ba6e8b876)

The `.gitlab-ci.yml` in [DB_PROXY](https://github.com/StealLine/PROXY_DB_APP) references a **separate CI/CD configuration project**. This allows pipeline changes without unnecessary commits to the main codebase.

> **Note:** Some pipeline stages such as dependency scanning and DAST testing are intentionally omitted. Images like `mcr.microsoft.com/dotnet/sdk:8.0-alpine` could optionally be stored in the GitLab registry for better performance.

---

### Configuration Files

| File | Description |
|------|-------------|
| `.default_gitlab-ci-configurations.yml` | See explanation and code in this repository |
| `.pre_config_settings.yml` | See explanation and code in this repository |
| `.gitlab-ci.yml` | See explanation and code in this repository |

---

### `.gitlab-ci.yml` in the Main Project (Developer Push Target)

![Main project CI config](https://github.com/user-attachments/assets/a0425615-cc7f-409f-bed7-c6382b2723b2)
```yaml
include:
  - project: StealLine-group/CI-cd_project
    ref: main
    file: .gitlab-ci.yml
```

*This simply pulls all pipeline configurations from the CI/CD configuration project.*

---

## GitLab CI/CD Variables

![CI/CD Variables](https://github.com/user-attachments/assets/d89c267d-59dc-4564-9e6c-d75222ee6cef)

### Required Variables

| Variable | Description |
|----------|-------------|
| `KNOWN_HOSTS` | Public key of the deployment server. Generated via `ssh-keyscan <server-ip>` |
| `PRIVATE_K` | Private SSH key for connecting to the deployment server *(note: can be exposed by team members — fixable by restricting SSH command access)* |
| `SETTINGS_ENV` | Configuration file required by [DB_PROXY](https://github.com/StealLine/PROXY_DB_APP) |
| `SONAR_HOST_URL` | SonarQube server URL |
| `SONAR_TOKEN` | SonarQube authentication token |

---

## How the Workflow Works

### 1. Developer Creates a Feature Branch

A developer creates a new feature branch and opens a merge request targeting `main`.

![Issue / MR creation](https://github.com/user-attachments/assets/14003d9e-a3dc-43f3-983b-56207960b20e)

---

### 2. MR Pipeline Runs — Preview Environment Deployed

When the merge request pipeline starts, a **preview environment** is automatically deployed.

![Preview deploy](https://github.com/user-attachments/assets/fac18dc3-fdb1-4a9d-9568-aa3975711630)

---

### 3. Preview Environment Visible in GitLab UI

![Preview in GitLab](https://github.com/user-attachments/assets/7fa99dad-d7f8-4ed6-b65d-87e761c8db24)

---

### 4. Authentication Is Auto-Generated

For each preview deployment, a **login** (GitLab username) and **password** are automatically generated.

Since this simulates a preview server, the DB_PROXY API is exposed so developers can test it directly. *(In this example, everything is deployed to a single server.)*

Authentication and routing are managed by **Traefik**.

![Auth credentials](https://github.com/user-attachments/assets/5045de90-8e7b-4c73-b282-04db569d6443)

---

### 5. Developer Logs In and Accesses the API

![API access](https://github.com/user-attachments/assets/0fc8f5e7-842c-43d9-b07e-1f9cc3eab786)

---

### 6. Each Preview Environment Runs Independently

Each preview is **isolated** in its own container stack. The database is ephemeral — it exists only while the environment is active, and is removed when stopped.

*New preview folder appearing on the server:*

![Server folders](https://github.com/user-attachments/assets/669506f7-8521-42c8-b421-0280c693903e)

![Container stack](https://github.com/user-attachments/assets/3fdc19b4-e5fc-431c-aed4-b55ba1a7294d)

---

### 7. SAST Results via SonarQube

Results are available in MR comments and directly in the SonarQube UI.

![SAST comment](https://github.com/user-attachments/assets/0fd374a6-229b-4e74-a81b-b8533fb9b985)

![SonarQube UI](https://github.com/user-attachments/assets/02a7defa-aa6f-46fb-982a-814219767586)

---

### 8. Code Approved → Merge → Main Pipeline Triggered

Once all checks pass and the MR is approved, merging automatically:
- Stops the preview environment (`stop_preview` job)
- Triggers the main branch pipeline

![Merge approval](https://github.com/user-attachments/assets/4ad67af0-1c5c-4b3b-aa2a-711c79cc8134)

![Stop preview](https://github.com/user-attachments/assets/a0172f7a-23be-4c47-9e79-7473d2b4f732)

![Main pipeline run](https://github.com/user-attachments/assets/6893cab7-a97e-4098-a913-ad1885784b8e)

---

## ✅ Deployment Complete

The release has been successfully deployed to `main`. A **rollback job** is available if needed.

![Rollback job](https://github.com/user-attachments/assets/ab6bda4e-4c21-4c23-957a-f37d71434e89)

![Rollback pipeline](https://github.com/user-attachments/assets/bfaebb04-4be0-4c47-8411-dc765e42fce9)

![Deployment result](https://github.com/user-attachments/assets/4bf5f8af-bcc7-465a-8c93-fd5da02639d7)

---

## Traefik Configuration

See the `compose.yml` file in this repository for the full Traefik setup and explanation.

> *Note: The exposed port `9092` visible in some screenshots was used for Prometheus testing only for different project — it should not be open in a real deployment.*



# Frontend — Crypto Rate Checker

> *This part of the application's DevOps cycle is built on the same principles as described in the previous section. Only the key aspects are covered here — for full implementation details, feel free to open a pull request to the repository.*

---

## CI/CD Pipelines

### Merge Request Pipeline

The merge request pipeline is triggered automatically on every MR and runs a sequence of validation and preview deployment stages, ensuring code quality before it ever reaches the main branch.

![Merge Request Pipeline](https://github.com/user-attachments/assets/85043ec6-9454-421f-94bd-326d69336b21)

---

### Main Branch Pipeline

Once a merge request is approved and merged, the main branch pipeline takes over and handles the full production deployment flow.

![Main Branch Pipeline](https://github.com/user-attachments/assets/413cf623-ef0f-4cf5-8ef0-7849e591e8b0)

---

## Developer Workflow

Repository for developers to push their changes in

![Developer Workflow](https://github.com/user-attachments/assets/2cf7c88d-15b1-4433-a135-ebb354be6a3e)

---

## Configurations Repository

All pipeline configuration is centralized in a dedicated **configurations repository**, keeping the developer-facing repository clean and the CI/CD logic versioned separately.

![Configurations Repository](https://github.com/user-attachments/assets/b6072315-06f9-4a19-a9bb-6a3c4beda2dc)

### Key Configuration Files

| File name in this repo | Real name in ci-cd-2 gitlab project |
|---|---|
| `conf_gitlab-ci.yml` | .gitlab-ci.yml |
| `.conf-default_gitlab-ci-configurations.yml` | .default_gitlab-ci-configurations.yml |

The `.gitlab-ci.yml` file in each developer repository simply **references** these centralized files — identical to the approach used in the DB Proxy setup described earlier. 

### GitLab CI/CD Variables

Sensitive values and environment-specific settings are managed through **GitLab CI/CD variables**, mirroring the same conventions used across the project.

![GitLab Variables](https://github.com/user-attachments/assets/ee4fd8ce-50d1-40eb-8bb3-19ac21318b00)

---

## End-to-End Deployment Demo

### 1. Creating a Feature Branch

To demonstrate the full pipeline flow, a new branch was created with a trivial change — adding whitespace to the `.dockerignore` file.

![Branch Creation](https://github.com/user-attachments/assets/00ab3cc3-089d-4f0b-ac9d-3bf458395891)

![Commit](https://github.com/user-attachments/assets/d5ba77ac-320c-439a-a944-3e422892c6d8)

### 2. Merge Request & Pipeline Execution

After opening and merging the MR, the pipeline ran automatically. All stages completed successfully.

![Pipeline Run](https://github.com/user-attachments/assets/1a31bc7d-2d78-4de0-bf19-9ce63b42b200)

### 3. Accessing the Preview Environment

Once the pipeline succeeds, the preview environment becomes accessible via a generated password paired with the user's GitLab username — the same authentication mechanism used for the DB Proxy.

![Preview Access](https://github.com/user-attachments/assets/cdac02c2-134e-41c9-a348-9ba77bd63247)

### 4. Verifying the Deployed Application

After authenticating, the website is reachable and fully functional in the preview environment.

![Website](https://github.com/user-attachments/assets/57d2a493-bc99-444e-b31f-10b04e37aa19)

### 5. Server State

All application containers are confirmed running on the server.

![Running Containers](https://github.com/user-attachments/assets/cd5ddc2f-06da-408e-8de0-b9d6c7acbabb)

The preview environment's configuration files are also present on the host.

![Server Files](https://github.com/user-attachments/assets/5e5b8259-7905-4574-b702-2a49d026eaae)

---

## Preview Environment Teardown

> ⚠️ **Note:** The preview environment currently points to the **production database**. It was easier to implement the code like that but it is not recommended for production-grade workflows.

When a preview environment is stopped, the pipeline sends a `curl` request to a dedicated endpoint that wipes the associated database records.

![Stop Job](https://github.com/user-attachments/assets/ff2d02ac-96d9-4a60-b4ff-95c00aa5acd3)

---

## Production Deployment

Merging the feature branch into `main` triggers the full production deployment pipeline.

![Production Pipeline](https://github.com/user-attachments/assets/c7943f14-9ca8-4c50-bda1-8a66639af91a)

Once the pipeline completes, the updated application is live in production.

---
## additional details

## Automated Cleanup

Unused Docker containers and images are pruned nightly via a **cron job** on the server host, preventing disk space from accumulating over time.

```cron
0 3 * * * ubuntu docker system prune -af --volumes
```

This runs at **03:00 UTC** daily, removing all stopped containers, dangling images, and unused volumes.

## https, bot blocking, anti ddos etc managed by cloudflare
<img width="1320" height="394" alt="image" src="https://github.com/user-attachments/assets/bd121ba8-4d22-45b0-91af-0b6ef24e6bc6" />






