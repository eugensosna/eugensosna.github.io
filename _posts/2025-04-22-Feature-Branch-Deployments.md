## Automated Deployment of Feature Branches with Git and GitLab CI/CD

In today's fast-paced software development world, the rapid and efficient delivery of new features is a key factor in success. One crucial aspect of this process is the ability to quickly deploy an isolated version of a feature branch for testing, demonstration to clients, or QA engineers.

In this article, we will explore an approach to automate the deployment of each feature branch using Git and GitLab CI/CD, leveraging Docker Compose for environment isolation and standardization.

### Why is this needed?

Automating the deployment of feature branches offers a range of benefits:

- **Fast Feedback:** Clients and QA engineers can access new functionality immediately after it's committed, allowing for quick feedback and early detection of potential issues.
- **Improved Communication:** Providing a live, accessible version of the feature allows stakeholders to directly experience and understand the implemented functionality, reducing ambiguity..
- **Early Detection of Integration Issues:** Running all dependent services in an isolated environment helps identify integration problems before the branch is merged into the main branch.
- **Standardized Environment:** Docker Compose ensures a consistent environment for each feature branch, eliminating problems related to differences in local machine configurations.
- **Reduced Risks:** Early testing and feedback help mitigate risks associated with integrating large volumes of code.

### How it works (for developers and DevOps engineers)

**For the Developer:**

1. The developer creates a new feature branch in the main repository (e.g., `feature/123-new-ui`).
2. After committing and pushing changes to this branch, GitLab CI/CD automatically triggers a pipeline.
3. The pipeline builds a Docker image of the application, tagging it with a unique identifier based on the branch name (e.g., `spring.petclinic:feature123newui`).
4. The pipeline triggers a separate pipeline in a helper repository (`deploystatus`).
5. In `deploystatus` will create configuration and deploy  application

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
  
  ```

  **4. Create the directory structure and template files in the helper repository (deploystatus):**
```
deploystatus/
├── traefik/
│   └── docker-compose.yml  # Traefik configuration
└── .template/
    ├── compose.yml.j2
    ├── envfile.env.j2
    └── status.conf.j2
├── init_update.py
└── run_compose.py
```
**5. Populate the Jinja2 template files (.j2) in the .template directory:**

compose.yml.j2 (example):
```YAML

version: '3.8'
services:
  app:
    image: "spring.petclinic:{LATEST_VERSION:-latest}"
    environment:
      {{ ENV_VARS | indent(8) }}
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.{{ SRC_BRANCH_SERVICE }}.rule=Host(`{{ URL }}`)"
      - "traefik.http.services.{{ SRC_BRANCH_SERVICE }}.loadbalancer.server.port=8080" # Replace with your application port
    networks:
      - web

networks:
  web:
    external: true # Connect to the external Traefik network
  ```
envfile.env.j2 (example):
```
LATEST_VERSION={{ SRC_BRANCH_SERVICE }}
URL={{ URL }}
SRC_BRANCH_SERVICE={{ SRC_BRANCH_SERVICE }}
UPSTREAM_ORIGIN_BRANCH={{ UPSTREAM_ORIGIN_BRANCH }}
# Add other necessary environment variables
```
status.conf.j2 (example):
`active`

**6. Write the init_update.py (Python) script to initialize the branch configuration:**
11 or 2 promt to AI make this script self

```Python

import os
import git
import shutil
from jinja2 import Environment, FileSystemLoader

def main():
    upstream_branch = os.environ.get("UPSTREAM_BRANCH")
    upstream_origin_branch = os.environ.get("UPSTREAM_ORIGIN_BRANCH")
    upstream_project_id = os.environ.get("UPSTREAM_PROJECT_ID")
    ci_commit_branch = os.environ.get("CI_COMMIT_BRANCH")

    if not all([upstream_branch, upstream_origin_branch, upstream_project_id, ci_commit_branch]):
        print("Not all necessary environment variables are present.")
        return

    src_branch_service = "".join(filter(str.isalnum, ci_commit_branch.lower()))
    branch_dir = os.path.join(os.getcwd(), src_branch_service)

    if os.path.exists(branch_dir):
        print(f"Directory for branch {upstream_branch} already exists.")
        return

    os.makedirs(branch_dir, exist_ok=True)

    template_dir = os.path.join(os.getcwd(), ".template")
    env = Environment(loader=FileSystemLoader(template_dir))

    env_vars = {
        "SRC_BRANCH_SERVICE": src_branch_service,
        "URL": f"{src_branch_service}.testserver.sosna.internal", # Configure your domain
        "UPSTREAM_ORIGIN_BRANCH": upstream_origin_branch,
        "ENV_VARS": "" # You can add additional environment variables here if needed
    }

    for template_file in os.listdir(template_dir):
        if template_file.endswith(".j2"):
            template = env.get_template(template_file)
            output_path = os.path.join(branch_dir, template_file[:-3])
            rendered_content = template.render(**env_vars)
            with open(output_path, "w") as f:
                f.write(rendered_content)

    try:
        repo = git.Repo(".")
        repo.index.add([os.path.join(src_branch_service, f) for f in os.listdir(branch_dir)])
        repo.index.commit(f"Initialize configuration for branch {upstream_branch}")
        origin = repo.remote(name='origin')
        origin.push()
        print(f"Configuration for branch {upstream_branch} created and pushed.")
    except Exception as e:
        print(f"Error during commit and push: {e}")
        shutil.rmtree(branch_dir) # Remove the created directory in case of an error

if __name__ == "__main__":
    main()
7. Write the run_compose.py (Python) script to run Docker Compose:

Python

import os
import subprocess

def main():
    for branch_dir in os.listdir("."):
        status_file = os.path.join(branch_dir, "status.conf")
        compose_file = os.path.join(branch_dir, "compose.yml")
        env_file = os.path.join(branch_dir, "envfile.env")

        if os.path.isfile(status_file) and os.path.isfile(compose_file) and os.path.isfile(env_file):
            with open(status_file, "r") as f:
                if "active" in f.read().strip():
                    src_branch_service = branch_dir
                    compose_path = os.path.abspath(compose_file)
                    envfile_path = os.path.abspath(env_file)

                    cmdup = [
                        "docker",
                        "compose",
                        "-f", compose_path,
                        "--env-file", envfile_path,
                        "--project-name", src_branch_service,
                        "up",
                        "-d"
                    ]

                    print(f"Running Docker Compose for branch {src_branch_service}...")
                    try:
                        subprocess.run(cmdup, check=True)
                        print(f"Docker Compose started for branch {src_branch_service}")
                    except subprocess.CalledProcessError as e:
                        print(f"Error running Docker Compose for branch {src_branch_service}: {e}")
                    except FileNotFoundError:
                        print("Command 'docker compose' not found. Ensure Docker Compose is installed.")

if __name__ == "__main__":
    main()

```

**8. Configure GitLab CI/CD in the helper repository (deploystatus/.gitlab-ci.yml):**

```YAML

stages:
  - init
  - deploy

init:
  stage: init
  tags:
    - linux
  script:
    - python init_update.py
  rules:
    - if: '$CI_PIPELINE_SOURCE == "trigger"' # Run only when triggered from the main repository

deploy:
  stage: deploy
  tags:
    - linux
  script:
    - python run_compose.py
  dependencies:
    - init
```
**9. Configure DNS for the domain *.testserver.sosna.internal (or your domain) to point to the IP address of your test server.**
I like use  Pi-hole for local 
For more explicit wildcard handling, you can directly configure dnsmasq, which is the DNS server Pi-hole uses under the hood. This involves editing a configuration file via the command line on your Pi-hole device.

  1. Access your Pi-hole via SSH: Connect to your Pi-hole device using SSH.

  2. Create or Edit a Custom dnsmasq Configuration File: Create a new configuration file (or edit an existing one if you have other custom dnsmasq settings) in the `/etc/dnsmasq.d/ directory`. Name could be something like `05-test-wildcard.conf`.
  ```Bash
  sudo nano /etc/dnsmasq.d/05-test-wildcard.conf
  ```
  3. Add the Wildcard address Directive: In this file, add the following line, replacing <your_test_server_ip> with the actual IP address of your test server:
  ```
  address=/.testserver.sosna.internal/<your_test_server_ip>
  ```

  * address=/: This specifies that the following rule applies to any query.
  * `.testserver.sosna.internal/`: This is the wildcard domain. The leading dot ensures it matches any subdomain under `testserver.sosna.internal`.
  * `<your_test_server_ip>`: The IP address to which these subdomains should resolve. For me is it  `100.61.0.115`
  
  
  4. Save and Close the File: Press `Ctrl+X`, then `Y` to confirm saving, and Enter.

  5. Restart the Pi-hole DNS Resolver: To apply the changes, restart the dnsmasq service:
  `sudo systemctl restart dnsmasq`

`
**10. Configure Traefik (or another reverse proxy) on your test server to route requests to the appropriate containers based on subdomains**.
 The Traefik configuration might look something like this in traefik/docker-compose.yml:
```YAML

version: '3.8'
services:
  traefik:
    image: traefik:v2.10
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/etc/traefik/traefik.yml
    networks:
      - web

networks:
  web:
    external: true
And the traefik/traefik.yml configuration file:

```YAML

entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

providers:
  docker:
    exposedByDefault: false

certificatesResolvers:
  myResolver:
    acme:
      email: "your.email@example.com" # Replace with your email
      storage: "/etc/traefik/acme.json"
      # Add configuration for your TLS provider (e.g., Let's Encrypt)
```
Ensure that the web network in traefik/docker-compose.yml is external and used in the compose.yml of each feature branch.

Conclusion
With the described system, every commit to a feature branch in the main repository automatically leads to the deployment of an isolated environment with the latest version of the code. Developers can easily share URLs with clients and QA engineers, who can directly evaluate the new functionality. DevOps engineers gain a flexible and automated system for managing test environments, significantly simplifying the development process and improving the quality of the final product.

This configuration is an example and can be adapted to the specific needs of your project. The key is to understand the fundamental principles of automation and the use of Git, GitLab CI/CD, and Docker Compose to achieve rapid and efficient software delivery.
