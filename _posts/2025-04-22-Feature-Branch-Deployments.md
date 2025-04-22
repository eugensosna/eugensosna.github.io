## Automated Deployment of Feature Branches with Git and GitLab CI/CD

In today's fast-paced software development world, the rapid and efficient delivery of new features is a key factor in success. One crucial aspect of this process is the ability to quickly deploy an isolated version of a feature branch for testing, demonstration to clients, or QA engineers.

In this article, we will explore an approach to automate the deployment of each feature branch using Git and GitLab CI/CD, leveraging Docker Compose for environment isolation and standardization.

### Why is this needed?

Automating the deployment of feature branches offers a range of benefits:

- **Fast Feedback:** Clients and QA engineers can access new functionality immediately after it's committed, allowing for quick feedback and early detection of potential issues.
- **Improved Communication:** Demonstrating a live version of the functionality significantly simplifies the understanding of how the developer has implemented the client's requirements.
- **Early Detection of Integration Issues:** Running all dependent services in an isolated environment helps identify integration problems before the branch is merged into the main branch.
- **Standardized Environment:** Docker Compose ensures a consistent environment for each feature branch, eliminating problems related to differences in local machine configurations.
- **Reduced Risks:** Early testing and feedback help mitigate risks associated with integrating large volumes of code.

### How it works (for developers and DevOps engineers)

**For the Developer:**

1. The developer creates a new feature branch in the main repository (e.g., `feature/123-new-ui`).
2. After committing and pushing changes to this branch, GitLab CI/CD automatically triggers a pipeline.
3. The pipeline builds a Docker image of the application, tagging it with a unique identifier based on the branch name (e.g., `spring.petclinic:feature123newui`).
4. The pipeline triggers a separate pipeline in a helper repository (`deploystatus`).

**For the DevOps Engineer (described in the context of system setup):**

1. In the helper repository (`deploystatus`), a directory structure and template files are set up in the `.template` directory (`status.conf.j2`, `compose.yml.j2`, `envfile.env.j2`).
2. GitLab CI/CD in the helper repository receives information about the new branch (e.g., `UPSTREAM_BRANCH=feature/123-new-ui`).
3. The `init_update.py` (Python) script checks if a directory for this branch already exists. If not, it creates a new directory with the branch name (e.g., `feature123newui`), copies the template files from `.template` into it, and populates them based on CI/CD environment variables (e.g., branch name for `URL`, service name, image version).
4. The created files (`status.conf`, `compose.yml`, `envfile.env`) are committed and pushed to the `deploystatus` repository.
5. The main pipeline in `deploystatus` periodically or after each commit checks all directories. If `active` is found in the `status.conf` file, `docker compose up -d --env-file feature123newui/envfile.env` is executed in the corresponding directory.
6. Docker Compose spins up the containers, using the image with the appropriate tag and environment variables from `envfile.env`.
7. The container automatically registers with the Traefik network thanks to labels in `compose.yml`, and Traefik configures routing on a third-level subdomain (e.g., `feature123newui.testserver.sosna.internal`).

### Instructions for Organizing Automated Feature Branch Deployment

Here's a step-by-step guide to setting up a similar system:

**1. Create two repositories in GitLab:**

- **Main repository (e.g., `spring-petclinic`):** Contains the code for your main product.
- **Helper repository (e.g., `deploystatus`):** Responsible for managing the deployment of feature branches.

**2. Configure Dockerfile for your application in the main repository:**

- The Dockerfile should build a Docker image of your application.
- Ensure your application can be configured using environment variables.

**3. Configure GitLab CI/CD in the main repository (`.gitlab-ci.yml`):**

```yaml
stages:
  - build
  - trigger

build:
  stage: build
  tags:
    - linux
  script:
    - export SRC_BRANCH_SERVICE=$(echo "$CI_COMMIT_BRANCH" | tr '[:upper:]' '[:lower:]' | sed 's/[^[:alnum:]]\+//g')
    - echo "Building the docker image ..."
    - docker build -t spring.petclinic:$CI_COMMIT_SHORT_SHA .
    - docker image tag spring.petclinic:$CI_COMMIT_SHORT_SHA spring.petclinic:$SRC_BRANCH_SERVICE
    - echo "Build complete. Image tagged as spring.petclinic:$SRC_BRANCH_SERVICE"

upstream-trigger-job:
  stage: trigger
  variables:
    UPSTREAM_PROJECT_ID: $CI_PROJECT_ID
    UPSTREAM_BRANCH: $CI_COMMIT_REF_NAME
    UPSTREAM_SO_COMMIT_BRANCH: $CI_COMMIT_BRANCH
    UPSTREAM_ORIGIN_BRANCH: $CI_COMMIT_REF_NAME
  trigger:
    project: autofeaturedeploy/deploystatus # Replace with the path to your helper repository
