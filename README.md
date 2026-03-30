# GITLAB-CI-PROJECT

## This project consist of total 2 apps
[DB_PROXY](https://github.com/StealLine/PROXY_DB_APP)
[Frontend](https://github.com/StealLine/Frontend-Crypto-Rate-Checker)
---
PROXY_DB_APP ci/cd pipeline realization was implemented using Gitlab
This is how merge request  to main branch pipeline looks like
<img width="2916" height="367" alt="image" src="https://github.com/user-attachments/assets/90fc4707-116e-41e4-9c2a-d59cd8cb8ed0" />
And for main
<img width="1991" height="742" alt="image" src="https://github.com/user-attachments/assets/739f8437-d754-4e67-bffc-d14ba6e8b876" />

.gitlab-ci.yml in [DB_PROXY](https://github.com/StealLine/PROXY_DB_APP) and [Frontend](https://github.com/StealLine/Frontend-Crypto-Rate-Checker) refers to the projects where configuration of the ci/cd pipeline stored. This helps to dynamycally change configuration of the pipeline without doing unneccessary commits to the main branch

This is ci/cd configuration for DB_PROXY](https://github.com/StealLine/PROXY_DB_APP) 
<img width="919" height="179" alt="image" src="https://github.com/user-attachments/assets/6a6f9e5a-1823-4a6c-840a-c7940f6b5177" />

Logically how it works.

1) lint stage checking for style, it is not neccessary so it can be removed
2) build stage building the project and saving publish/ folder and Dockerfile as an artifact for future jobs also caching dependencies.
3) Xunit test is running integration testing, it is running postgresql as well as test code in the one job, than saving reports
4) build sonar jobs testing code using sonarqube server implemented
5) docker build preview job building  





