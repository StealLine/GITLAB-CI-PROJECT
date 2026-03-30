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

PROXY_DB_APP ci/cd pipeline realization was implemented using Gitlab
This is how merge request  to main branch pipeline looks like
<img width="2916" height="367" alt="image" src="https://github.com/user-attachments/assets/90fc4707-116e-41e4-9c2a-d59cd8cb8ed0" />
And for main
<img width="1991" height="742" alt="image" src="https://github.com/user-attachments/assets/739f8437-d754-4e67-bffc-d14ba6e8b876" />

.gitlab-ci.yml in [DB_PROXY](https://github.com/StealLine/PROXY_DB_APP) and [Frontend](https://github.com/StealLine/Frontend-Crypto-Rate-Checker) refers to the projects where configuration of the ci/cd pipeline stored. This helps to dynamycally change configuration of the pipeline without doing unneccessary commits to the main branch

This is ci/cd configuration for DB_PROXY](https://github.com/StealLine/PROXY_DB_APP) 
<img width="919" height="179" alt="image" src="https://github.com/user-attachments/assets/6a6f9e5a-1823-4a6c-840a-c7940f6b5177" />


### `.default_gitlab-ci-configurations.yml`

```yaml
default: # default configuration for most of the jobs
  image: mcr.microsoft.com/dotnet/sdk:8.0-alpine
  interruptible: true

.default_dind_images:
  image: docker:29.3.1-cli-alpine3.23
  services:
    - docker:29.3.1-dind-alpine3.23

variables:
  NUGET_PACKAGES: "$CI_PROJECT_DIR/.nuget/packages"
  POSTGRES_HOST: postg
  POSTGRES_PORT: 5432
  POSTGRES_PASSWORD: postgres
  POSTGRES_USER: postgres
  POSTGRES_DB: postgres
  XUNIT_REPORT_PATH: "$CI_PROJECT_DIR/test-results.xml"
  PATH_TO_TEST_CSPROJ: TEST_PROXY/TEST_PROXY.csproj
  SERVER_IP: # ip-of-server-where-deploying
  SERVER_ACCOUNT: # deploying-user
  PREVIEW_USER: $GITLAB_USER_LOGIN
  PREVIEW_PASS: $CI_COMMIT_SHORT_SHA
  DEPLOY_PATH: # path for preview deployment
.default_before_script: # some utils req to join server via ssh
  script:
     - apk add --no-cache curl
     - apk add --no-cache openssh-client
     - apk add --no-cache apache2-utils
     - mkdir -p ~/.ssh
     - chmod 700 ~/.ssh

     - echo "$PRIVATE_K" | base64 -d > MyKey.pem
     - chmod 400 MyKey.pem

     - echo "$KNOWN_HOSTS" | base64 -d > ~/.ssh/known_hosts
     - chmod 644 ~/.ssh/known_hosts

     - eval $(ssh-agent)
     - ssh-add MyKey.pem
     - echo "$SETTINGS_ENV" | base64 -d > settings.env

.default_after_script_filecopy_andrun: # default deploy script
  script:
      - ssh $SERVER_ACCOUNT@$SERVER_IP mkdir -p $DEPLOY_PATH

      - scp docker-compose.yml settings.env $SERVER_ACCOUNT@$SERVER_IP:$DEPLOY_PATH

      - |
        ssh $SERVER_ACCOUNT@$SERVER_IP "
          cd $DEPLOY_PATH &&
          echo \"$CI_REGISTRY_PASSWORD\" | docker login -u \"$CI_REGISTRY_USER\" --password-stdin $CI_REGISTRY &&
          docker compose pull &&
          docker compose up -d
        "

```

### `.pre_config_settings.yml`

```yaml

workflow:
  name: 'Pipeline for branch: $CI_COMMIT_BRANCH' # name of the workflow
  auto_cancel: # we can interrupt jobs in this pipeline
    on_new_commit: interruptible # default interruptible is true
  rules: # rules when to run the pipeline
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH # first rule if the commit is on the default branch
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event" && $CI_COMMIT_REF_NAME =~ /^feature\-.+$/' #only if
    #branch name feature-... and only for merge requests
    - when: never  # not running for other cases

stages: # stages of the pipeline
  - lint
  - build
  - test
  - docker_build
  - preview_deploy
  - deploy
```

### `.gitlab-ci.yml`

```yaml
include:
  - local: .default_gitlab-ci-configurations.yml 
  - local: .pre_config_settings.yml

code_lint: # linting job, just checking style, not formatting code

    stage: lint
    inherit: # not inheriting any variables 
      variables: false
    needs: []
    
    script:
      - dotnet format --verify-no-changes # command for running lint test

    allow_failure: true #  this job is not critical, can be failed without failing the whole pipeline

building_code:

  stage: build
  inherit: # inheriting nuget packages path so they will be saved localy in directory not globaly
    variables:
      - NUGET_PACKAGES
  needs: []
      
  before_script:
    - dotnet restore # restoring dependencies for all projects in solutions
  script:
    - dotnet publish -c Release -o publish # publishing only main application to publish/ folder

  artifacts: # publishing publish ready app as an artifact for the next stages
    paths:
      - "publish/" # ready application folder generated by publish command
      - "Dockerfile" # dockerfile for deploy job

    when: on_success # only publish artifact if the build was successful
    access: none # artifact will not be available for download, only for the next stages
    expire_in: 1 day # artifact will be stored for 1 week, after that it will be deleted

  cache:
    key: cache-dependencies-$CI_COMMIT_REF_SLUG # saving packets to cache specific for every branch
    paths:
      - $NUGET_PACKAGES # path where they saved


Xunit_test: # integration testing for code
    
  stage: test
  inherit:
    variables:
      - POSTGRES_DB # variablle for postgres to be working
      - POSTGRES_USER # variablle for postgres to be working
      - POSTGRES_PASSWORD # variablle for postgres to be working
      - POSTGRES_HOST # custom variable for our app, to say the host for the app
      - POSTGRES_PORT # custom variable specifying port on which postgres working
      - XUNIT_REPORT_PATH # custom variable showing where to save report
      - NUGET_PACKAGES # custom variable where the dependencies should be taken from
      - PATH_TO_TEST_CSPROJ # custom variable where the path csproj of xunit project is stored
  needs:
    - job: building_code # waiting for build job to be finished
      artifacts: false

  services: # it inherited all the variables above
    - name: postgres:14.22-alpine # additional db service
      alias: $POSTGRES_HOST # docker network alias so we can connect to it through our app

  script:
    - dotnet test $PATH_TO_TEST_CSPROJ --logger "junit;LogFilePath=$XUNIT_REPORT_PATH" # running test and saving
    # result into file

  artifacts:
    reports:
      junit:
        - "$XUNIT_REPORT_PATH" # publishing junit report 
    
  cache:
    key: cache-dependencies-$CI_COMMIT_REF_SLUG # pulling same cache dependencies but not pushing them
    paths:
      - $NUGET_PACKAGES
    policy: pull # policy not to push
  
build-sonar-mr:

  stage: test

  inherit:
    variables: false
  
  needs:
    - job: building_code
      artifacts: false
    
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"

  cache:
    - key: "sonar-cache-$CI_COMMIT_REF_SLUG"
      paths:
        - "${SONAR_USER_HOME}/cache"
      policy: pull-push
    - key: cache-dependencies-$CI_COMMIT_REF_SLUG  # pulling nuget packages for build in sonarq
      paths:
        - $NUGET_PACKAGES
      policy: pull

  script:
    - "dotnet tool install --global dotnet-sonarscanner"
    - "export PATH=\"$PATH:$HOME/.dotnet/tools\""
    - "dotnet sonarscanner begin /k:\"stealline-group_proxy_app_project_2_b169dc8f-d99c-494e-b6b9-02d5c1b7b6ab\" /d:sonar.token=\"$SONAR_TOKEN\" /d:\"sonar.host.url=$SONAR_HOST_URL\" /d:sonar.pullrequest.key=\"$CI_MERGE_REQUEST_IID\" /d:sonar.pullrequest.branch=\"$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME\" /d:sonar.pullrequest.base=\"$CI_MERGE_REQUEST_TARGET_BRANCH_NAME\""
    - "dotnet build"

    - "dotnet sonarscanner end /d:sonar.token=\"$SONAR_TOKEN\""

  allow_failure: true

  rules:
    - if: $CI_PIPELINE_SOURCE == 'merge_request_event'

build-sonar-main:

  stage: test

  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: "0"

  inherit:
    variables: false
  
  needs:
    - job: building_code
      artifacts: false

  cache:
    - key: "sonar-cache-main"
      policy: pull-push
      paths:
        - "${SONAR_USER_HOME}/cache"
    - key: cache-dependencies-$CI_COMMIT_REF_SLUG  # pulling nuget packages for build in sonarq
      paths:
        - $NUGET_PACKAGES
      policy: pull
  
  script:
    - "dotnet tool install --global dotnet-sonarscanner"
    - "export PATH=\"$PATH:$HOME/.dotnet/tools\""
    - "dotnet sonarscanner begin /k:\"stealline-group_proxy_app_project_2_b169dc8f-d99c-494e-b6b9-02d5c1b7b6ab\" /d:sonar.token=\"$SONAR_TOKEN\" /d:\"sonar.host.url=$SONAR_HOST_URL\""
    - "dotnet build"

    - "dotnet sonarscanner end /d:sonar.token=\"$SONAR_TOKEN\""
    
  allow_failure: true
  rules:
    - if: $CI_COMMIT_BRANCH == 'main'



docker_build_preview: # building image for deploy/preview

    extends: .default_dind_images # default docker  cli and dind

    stage: docker_build

    needs:
      - job: Xunit_test 
        artifacts: false       
      - job: build-sonar-mr
        artifacts: false
      - job: building_code     # taking artefacts from build
        artifacts: true

    inherit:
      variables: false # not inheriting any variable

    variables:
      GIT_STRATEGY: none # we don't need git repository in this job, so we can skip cloning it


    script:
      # loggining to gitlab container registry
      - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin $CI_REGISTRY 
      # pulling image if exists for our branch
      - docker pull $CI_REGISTRY_IMAGE/previewproxy:$CI_COMMIT_REF_SLUG || true
      # building our image and caching from previous image if it exists, creating tag for an image, 
      # that refers to our branch
      - | 
        docker build \
        --cache-from $CI_REGISTRY_IMAGE/previewproxy:$CI_COMMIT_REF_SLUG \
        -t $CI_REGISTRY_IMAGE/previewproxy:$CI_COMMIT_REF_SLUG \
        -t $CI_REGISTRY_IMAGE/previewproxy:$CI_COMMIT_SHA \
        .
      # pushing our image to the registry
      - docker push $CI_REGISTRY_IMAGE/previewproxy:$CI_COMMIT_REF_SLUG
      - docker push $CI_REGISTRY_IMAGE/previewproxy:$CI_COMMIT_SHA

    rules:
      - if: $CI_PIPELINE_SOURCE == "merge_request_event"

    
  
preview:
    stage: preview_deploy

    image: alpine:3.23
    resource_group: preview_deploy-$CI_COMMIT_REF_SLUG

    interruptible: false
    
    # we need docker build process to be finished to start this one
    needs: 
      - job: docker_build_preview
        artifacts: false

    rules:
      - if: $CI_PIPELINE_SOURCE == "merge_request_event"

    inherit:
      variables: 
        - SERVER_IP
        - SERVER_ACCOUNT
        - POSTGRES_USER
        - POSTGRES_PASSWORD
        - POSTGRES_DB 
        - PREVIEW_USER
        - PREVIEW_PASS
        - DEPLOY_PATH # path where we will deploy preview version of app, can be used in before and after script
    
    variables:
      GIT_STRATEGY: none # we don't need git repository in this job, so we can skip cloning it


    # here we are loggining into our server through ssh
    before_script:
      - !reference [.default_before_script,script]
    script:
      - | # changing resource group policy
        curl --request PUT \
        --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
        --data "process_mode=oldest_first" \
        "$CI_API_V4_URL/projects/$CI_PROJECT_ID/resource_groups/preview_deploy-$CI_COMMIT_REF_SLUG"
        
      - export PREVIEW_PASS_HASH=$(htpasswd -nb "$PREVIEW_USER" "$PREVIEW_PASS" | sed -e 's/\$/\$\$/g') # generating password for preview
      - |
        cat > docker-compose.yml <<EOF
        
        services:
          proxydb:
            networks:
              - preview_network_${CI_COMMIT_REF_SLUG}
              - traefik_network
            container_name: proxy_app_${CI_COMMIT_REF_SLUG}
            image: $CI_REGISTRY_IMAGE/previewproxy:$CI_COMMIT_SHA
            env_file:
              - settings.env
            depends_on:
              pgdb:
                condition: service_healthy 
            labels:
              - "traefik.enable=true"
              - "traefik.http.routers.${CI_COMMIT_REF_SLUG}.rule=Host(\`${CI_COMMIT_REF_SLUG}.superapp.website\`)"
              - "traefik.http.services.${CI_COMMIT_REF_SLUG}.loadbalancer.server.port=8080"
              - "traefik.http.middlewares.${CI_COMMIT_REF_SLUG}-auth.basicauth.users=${PREVIEW_PASS_HASH}"
              - "traefik.http.routers.${CI_COMMIT_REF_SLUG}.middlewares=${CI_COMMIT_REF_SLUG}-auth"
              - "traefik.http.routers.${CI_COMMIT_REF_SLUG}.tls.certresolver=le"
              - "traefik.http.routers.${CI_COMMIT_REF_SLUG}.tls=true"
              - "traefik.http.routers.${CI_COMMIT_REF_SLUG}.entrypoints=websecure"

          pgdb:
            networks: 
              - preview_network_${CI_COMMIT_REF_SLUG}
            container_name: db_postgres_${CI_COMMIT_REF_SLUG}
            image: postgres:14.22
            environment:
              POSTGRES_USER: $POSTGRES_USER
              POSTGRES_PASSWORD: $POSTGRES_PASSWORD
              POSTGRES_DB: $POSTGRES_DB
            healthcheck:
              test: [ "CMD", "pg_isready", "-q", "-d", "postgres", "-U", "postgres" ]
              interval: 10s
              timeout: 5s
              retries: 5
        networks:
          preview_network_${CI_COMMIT_REF_SLUG}:
            external: false
          traefik_network:
            external: true 
        EOF

      - !reference [.default_after_script_filecopy_andrun,script]
      - |
          echo "Preview URL: https://$CI_COMMIT_REF_SLUG.superapp.website/Home/Hello"
          echo "Login: $PREVIEW_USER / $PREVIEW_PASS"


    environment:
      deployment_tier: staging
      name: "preview/$CI_COMMIT_REF_SLUG"
      url: "http://$CI_COMMIT_REF_SLUG.superapp.website/Home/Hello"
      on_stop: stop_env_job
      auto_stop_in: 1 hour  

stop_env_job:

    interruptible: false

    image: alpine:3.23
    stage: preview_deploy 

    inherit:
      variables: 
        - SERVER_IP
        - SERVER_ACCOUNT
        - DEPLOY_PATH

    needs:
      - job: preview
        artifacts: false

    variables:
      GIT_STRATEGY: none # no need git repo

    when: manual
    manual_confirmation: This will stop current container in $DEPLOY_PATH
    

    before_script:
      - !reference [.default_before_script,script]

    script:
      - |
        ssh $SERVER_ACCOUNT@$SERVER_IP "
          echo \"$CI_REGISTRY_PASSWORD\" | docker login -u \"$CI_REGISTRY_USER\" --password-stdin $CI_REGISTRY &&
          cd $DEPLOY_PATH &&
          docker compose down -v --remove-orphans
        "

    rules:
      - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  
    environment:
      name: "preview/$CI_COMMIT_REF_SLUG"  
      action: stop
    

docker_build_main: # deploying main images

  interruptible: false 

  stage: deploy 
  resource_group: deployment # can`t build in 2 different pipelines

  extends: .default_dind_images # default docker  cli and dind
 
  variables:
      GIT_STRATEGY: none
      DEPLOY_PATH: /home/deploy-app-cryptovault/proxy-db/deploy

  needs:
      - job: Xunit_test      
        artifacts: false       
      - job: build-sonar-main
        artifacts: false
      - job: building_code     # taking from build
        artifacts: true

  inherit:
      variables:
        - SERVER_IP
        - SERVER_ACCOUNT
        - POSTGRES_USER
        - POSTGRES_PASSWORD
        - POSTGRES_DB


  before_script:
    - !reference [.default_before_script,script]
  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin $CI_REGISTRY

    - docker pull $CI_REGISTRY_IMAGE/deployproxy:stable || true
    - docker tag $CI_REGISTRY_IMAGE/deployproxy:stable $CI_REGISTRY_IMAGE/deployproxy:previous-stable || true
    - docker push $CI_REGISTRY_IMAGE/deployproxy:previous-stable || true
    - |
      docker build \
      -t $CI_REGISTRY_IMAGE/deployproxy:$CI_COMMIT_SHORT_SHA \
      --cache-from $CI_REGISTRY_IMAGE/deployproxy:stable \
      .

    - docker push $CI_REGISTRY_IMAGE/deployproxy:$CI_COMMIT_SHORT_SHA

    - docker tag $CI_REGISTRY_IMAGE/deployproxy:$CI_COMMIT_SHORT_SHA $CI_REGISTRY_IMAGE/deployproxy:stable
    - docker tag $CI_REGISTRY_IMAGE/deployproxy:$CI_COMMIT_SHORT_SHA $CI_REGISTRY_IMAGE/deployproxy:latest
    - docker push $CI_REGISTRY_IMAGE/deployproxy:stable
    - docker push $CI_REGISTRY_IMAGE/deployproxy:latest
    - |
        cat > docker-compose.yml <<EOF
        services:
          proxydb:
            container_name: proxy_app_deploy
            image: $CI_REGISTRY_IMAGE/deployproxy:stable
            env_file:
              - settings.env
            depends_on:
              pgdb:
                condition: service_healthy 
            networks:
              - deploy_net
            labels:
              - "traefik.enable=true"

          pgdb:
            container_name: db_postgres_deploy
            image: postgres:14.22
            environment:
              POSTGRES_USER: $POSTGRES_USER
              POSTGRES_PASSWORD: $POSTGRES_PASSWORD
              POSTGRES_DB: $POSTGRES_DB
            healthcheck:
              test: [ "CMD", "pg_isready", "-q", "-d", "postgres", "-U", "postgres" ]
              interval: 10s
              timeout: 5s
              retries: 5
            volumes:
              - db_data:/var/lib/postgresql/data  
            networks:
              - deploy_net
            labels:
              - "traefik.enable=true"

        volumes:
          db_data:
            external: true

        networks:
          deploy_net:
            external: true
        EOF
    - !reference [.default_after_script_filecopy_andrun, script]

  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH # only if main branch
    
  environment:
    name: deployment
    deployment_tier: production

rollback_main: 
 
  stage: deploy
  interruptible: false
  resource_group: deployment

  needs:
    - job: docker_build_main
      artifacts: false

  extends: .default_dind_images # default docker  cli and dind

  variables:
      GIT_STRATEGY: none
      DEPLOY_PATH: /home/deploy-app-cryptovault/proxy-db/deploy

  inherit:
      variables:
        - SERVER_IP
        - SERVER_ACCOUNT

  when: manual  
  manual_confirmation: "After running this job your latest deployment will be moved 1 step back"

  before_script:
    - !reference [.default_before_script,script]
 
  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin $CI_REGISTRY

    - |
      docker pull $CI_REGISTRY_IMAGE/deployproxy:previous-stable || {
        echo "No previous-stable image found! Aborting rollback."
        exit 1
      }
    - docker tag $CI_REGISTRY_IMAGE/deployproxy:previous-stable $CI_REGISTRY_IMAGE/deployproxy:latest
    - docker tag $CI_REGISTRY_IMAGE/deployproxy:previous-stable $CI_REGISTRY_IMAGE/deployproxy:stable
    - docker push $CI_REGISTRY_IMAGE/deployproxy:stable
    - docker push $CI_REGISTRY_IMAGE/deployproxy:latest 

    - |
      ssh $SERVER_ACCOUNT@$SERVER_IP "
        echo \"$CI_REGISTRY_PASSWORD\" | docker login -u \"$CI_REGISTRY_USER\" --password-stdin $CI_REGISTRY &&
        cd $DEPLOY_PATH &&
        docker compose pull &&
        docker compose up -d --remove-orphans
      "
      
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

And now .gitlab-ci.yml file where the source code is stored (in this repository all developers pushing)
<img width="1057" height="539" alt="image" src="https://github.com/user-attachments/assets/75885a74-af34-420b-b302-28a18c59d709" />

### `.gitlab-ci.yml`

```yaml
include:
  - project: # your proj name here
    ref: main
    file: .gitlab-ci.yml 
```
It is just taking all the configurations from the project above

---

now Gitlab-ci variables 
<img width="1365" height="543" alt="image" src="https://github.com/user-attachments/assets/d89c267d-59dc-4564-9e6c-d75222ee6cef" />

Known_hosts is key of server where deploying (got from ssh-keyskan command)
private_k for connecting to server
settings_env file required for [DB_PROXY](https://github.com/StealLine/PROXY_DB_APP)
Sonar host and sonar token required for sonarqube server to be working properly

How all works:
1) Developer creating new feature and  merging his branch to main
<img width="1940" height="1054" alt="image" src="https://github.com/user-attachments/assets/f7afa1b0-e837-46fc-a522-9b5c0edb82f0" />
2) Merge pipeline is running and preview environment is deployed 
   <img width="1153" height="173" alt="image" src="https://github.com/user-attachments/assets/b1a36738-5a54-4b8f-8482-be400feb306d" />
3) Preview environment running on server, each preview is independent
  <img width="675" height="442" alt="image" src="https://github.com/user-attachments/assets/2a35b75f-119a-4337-af0f-be68670938fd" />
4) For each deployment password and login creating for user this all managed by traefik
   <img width="1192" height="104" alt="image" src="https://github.com/user-attachments/assets/2c76ac85-7bf8-4c6c-811b-89cb8212d2ef" />








Logically how it works.

1) lint stage checking for style, it is not neccessary so it can be removed
2) build stage building the project and saving publish/ folder and Dockerfile as an artifact for future jobs also caching dependencies.
3) Xunit test is running integration testing, it is running postgresql as well as test code in the one job, than saving reports
4) build sonar jobs testing code using sonarqube server implemented
5) docker build preview job building  





