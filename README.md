# GITLAB-CI-PROJECT

## This project consist of total 2 apps
[DB_PROXY](https://github.com/StealLine/PROXY_DB_APP)  AND
[Frontend](https://github.com/StealLine/Frontend-Crypto-Rate-Checker)

---
⚠️ **Note about the CI/CD configuration**

The GitLab CI configuration and deployment logic in this repository are **not intended to represent a production-grade setup**. Some variables are intentionally hardcoded and certain values may be exposed in the configuration.

This project is a **personal pet project** whose primary goal was to demonstrate the ability to design and implement a **simplified end-to-end DevOps workflow**, including build, test, containerization, sast testing, preview deployment, and actual deployment stages .

The focus of this repository was on **showcasing practical DevOps skills and understanding of the overall pipeline**, rather than achieving a perfectly secure or fully optimized production configuration.

Implementing a completely production-ready infrastructure (with proper secret management, environment separation, infrastructure automation, etc.) would significantly increase the scope and complexity of the project and require considerably more time than is reasonable for a single-developer pet project.

Therefore, some **simplifications and trade-offs were intentionally made** to keep the project manageable while still demonstrating the core concepts of CI/CD and deployment automation.

---
## DB_PROXY DEVOPS LOGIC
DB_PROXY ci/cd pipeline realization was implemented using Gitlab
This is how merge request  to main branch pipeline looks like
<img width="2916" height="367" alt="image" src="https://github.com/user-attachments/assets/90fc4707-116e-41e4-9c2a-d59cd8cb8ed0" />
And for main
<img width="1991" height="742" alt="image" src="https://github.com/user-attachments/assets/739f8437-d754-4e67-bffc-d14ba6e8b876" />

The `.gitlab-ci.yml` file in [DB_PROXY](https://github.com/StealLine/PROXY_DB_APP) references a separate project where the CI/CD pipeline configuration is stored. This approach allows dynamic pipeline configuration changes without unnecessary commits to the main project.
 
> **Note:** These configurations may not include all best practices. Some pipeline stages such as dependency scanning and DAST testing are intentionally omitted. Additionally, images like `mcr.microsoft.com/dotnet/sdk:8.0-alpine` could be stored in the GitLab registry for improved performance.
### `.default_gitlab-ci-configurations.yml`
> look for explanation and the code of the file in this repository 

### `.pre_config_settings.yml`
> look for explanation and the code of the file in this repository 

### `.gitlab-ci.yml`
> look for explanation and the code of the file in this repository 

And now .gitlab-ci.yml file where the source code is stored (in this repository all developers pushing their code)
<img width="788" height="600" alt="image" src="https://github.com/user-attachments/assets/a0425615-cc7f-409f-bed7-c6382b2723b2" />


### `.gitlab-ci.yml in the main project (to where is developers pushing)`

```yaml
include:
  - project: StealLine-group/CI-cd_project
    ref: main
    file: .gitlab-ci.yml 
```
*It is just taking all the configurations from the project above*

---
## Now GitLab CI/CD variables:
<img width="1365" height="543" alt="image" src="https://github.com/user-attachments/assets/d89c267d-59dc-4564-9e6c-d75222ee6cef" />

## Required Variables and Secrets

The following variables are required for the CI/CD pipeline to work properly.

- **KNOWN_HOSTS** – public key of the deployment server.  
  Generated using:

```bash
ssh-keyscan <server-ip>
```

- **PRIVATE_K** – private SSH key used for connecting to the deployment server. (in this project can be revealed if someone from the dev team will try to, can be fixed with restricting command access via this ssh)

- **SETTINGS_ENV** – configuration file required for  
  [DB_PROXY](https://github.com/StealLine/PROXY_DB_APP).

- **SONAR_HOST_URL** and **SONAR_TOKEN** – required for integration with the SonarQube server, and usually generated with SonarQube

---

# How Everything Works

## 1. Developer creates an issue or work item in milestone or etc.

A developer creates a new feature branch and opens a merge request to the main branch.

<img width="1354" height="1161" alt="image" src="https://github.com/user-attachments/assets/14003d9e-a3dc-43f3-983b-56207960b20e" />



---

## 2. Merge request pipeline runs and preview environment is deployed

When the merge request pipeline starts, a **preview environment** is automatically deployed.

<img width="1375" height="939" alt="image" src="https://github.com/user-attachments/assets/fac18dc3-fdb1-4a9d-9568-aa3975711630" />



## 3. You will be able to see preview environment in gitlab ui 
<img width="1336" height="455" alt="image" src="https://github.com/user-attachments/assets/7fa99dad-d7f8-4ed6-b65d-87e761c8db24" />


---
## 4. Authentication is generated automatically

For each preview deployment:

- a **login** (just  using gitlab user)
- a **password**

are automatically generated for the user. (As its preview environment and we imagine it is deployed on different server we are openning its DB_PROXY api so developers can test it and do their thigs) 
p.s in this example everything deployed to a single server

Authentication and routing are managed by **Traefik**.

<img width="1206" height="109" alt="image" src="https://github.com/user-attachments/assets/5045de90-8e7b-4c73-b282-04db569d6443" />


## 5. Developer logged in and now they can see desired API
<img width="1577" height="348" alt="image" src="https://github.com/user-attachments/assets/0fc8f5e7-842c-43d9-b07e-1f9cc3eab786" />


## 6. Preview environments runs on the server

Each preview environment is **independent** and runs in its own container stack. (means for preview environment database lives only when environment is working, after environment is stopped it is removed)
*as we can see new folder with our preview appeared*
<img width="1137" height="786" alt="image" src="https://github.com/user-attachments/assets/669506f7-8521-42c8-b421-0280c693903e" />
<img width="3269" height="227" alt="image" src="https://github.com/user-attachments/assets/3fdc19b4-e5fc-431c-aed4-b55ba1a7294d" />

## 7. Sast results can be checked in comments or directly in Sonarqube UI

<img width="957" height="641" alt="image" src="https://github.com/user-attachments/assets/0fd374a6-229b-4e74-a81b-b8533fb9b985" />
<img width="1692" height="1062" alt="image" src="https://github.com/user-attachments/assets/02a7defa-aa6f-46fb-982a-814219767586" />

## 8. After staging is done all code checked and approved, time for merge. (Merge will autmatically start stop_preview job and launch main pipeline)
<img width="1388" height="1091" alt="image" src="https://github.com/user-attachments/assets/4ad67af0-1c5c-4b3b-aa2a-711c79cc8134" />
<img width="1568" height="165" alt="image" src="https://github.com/user-attachments/assets/a0172f7a-23be-4c47-9e79-7473d2b4f732" />
<img width="2062" height="715" alt="image" src="https://github.com/user-attachments/assets/6893cab7-a97e-4098-a913-ad1885784b8e" />

# It is done, our deployment successfully have been commited to main, we can now easily roll back to previos version by running this special job, but i wont do it for now.
<img width="475" height="230" alt="image" src="https://github.com/user-attachments/assets/ab6bda4e-4c21-4c23-957a-f37d71434e89" />

<img width="1253" height="647" alt="image" src="https://github.com/user-attachments/assets/bfaebb04-4be0-4c47-8411-dc765e42fce9" />

<img width="1611" height="223" alt="image" src="https://github.com/user-attachments/assets/4bf5f8af-bcc7-465a-8c93-fd5da02639d7" />

*P.S. dont look at opened 9092 port, i was just testing prometheus, in your deploy it should`nt be opened*










---

тут ще про траефік і тп






