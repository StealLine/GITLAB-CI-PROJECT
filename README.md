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

## DB_PROXY — CI/CD Pipeline

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

> *Note: The exposed port `9092` visible in some screenshots was used for Prometheus testing only — it should not be open in a real deployment.*
