# Docker Container-Based CI/CD Deployment with Jenkins
## Complete Start-to-Finish Lab Guide — Fresh AWS EC2 Ubuntu Instance

**Repository:** https://github.com/samikhan5055/laravel-project.git  
**Docker Hub namespace:** `sami1800`  
**Application:** Laravel three-tier application (Nginx web tier, Laravel/PHP-FPM application tier, MySQL database tier)  
**Deployment target:** The same fresh EC2 instance, with the website exposed on HTTP port `80`  
**Jenkins pipeline file:** `Jenkinsfile.docker` — deliberately separate from the repository's existing `Jenkinsfile`

---

## 1. What this lab will deliver

At the end, a Jenkins Pipeline will:

1. Check out the Laravel project from GitHub.
2. Validate the Compose configuration.
3. Build the Docker images for the Laravel app and Nginx web tier.
4. Perform container-level smoke checks.
5. Authenticate to Docker Hub using a Jenkins credential.
6. Push versioned images to Docker Hub.
7. Pull those exact versioned images and deploy the three-tier application using Docker Compose.
8. Run Laravel database migrations.
9. Check the running containers and make an HTTP request to the deployed website.

### Pipeline flow

```text
GitHub main branch
       |
       v
Jenkins Pipeline (Jenkinsfile.docker)
       |
       +--> Checkout source
       +--> Validate Compose configuration
       +--> Build app image and web image
       +--> Smoke-check both images
       +--> Push commit-tagged images to Docker Hub
       +--> Deploy those exact images with Docker Compose
       +--> Run migrations and HTTP smoke test
       |
       v
http://EC2-PUBLIC-IP/
```

### Existing repository details used by this guide

The repository already contains:

- `Dockerfile` with the multi-stage targets `app` and `web`.
- `compose.yaml` with `app`, `web`, and `db` services.
- `docker/nginx/default.conf`, which sends FastCGI requests to `app:9000`.
- MySQL 8.0, a MySQL health check, and named volumes for MySQL and Laravel storage.
- `composer.lock` and Laravel tests.

The Compose file publishes Nginx on host port `80`, uses `mysql_data` and `laravel_storage` volumes, and expects database/application settings from an `.env` file.

**Important:** This guide does not replace the existing `Jenkinsfile`. It adds `Jenkinsfile.docker` and selects that file in the new Jenkins job, so your previous pipeline remains available.

---

## 2. Create a fresh EC2 instance

Use the AWS Console to launch a new Ubuntu EC2 instance.

### Recommended lab sizing

- **AMI:** Ubuntu Server 24.04 LTS (64-bit x86).
- **Instance type:** at least 2 vCPU and 4 GiB RAM for Jenkins plus Docker image builds.
- **Root EBS volume:** 30–50 GiB gp3. Prefer 50 GiB if available; Jenkins workspaces and Docker layers can consume disk space quickly.
- **Key pair:** select or create a key pair that you can access.
- **Public IPv4:** enable/assign a public IPv4 address so you can access Jenkins and the website.

A very small root disk can cause `No space left on device` errors during Docker builds. Check storage before starting:

```bash
df -h
free -h
```

### Security Group inbound rules

Add these rules. Replace `YOUR_PUBLIC_IP/32` with your own public IP address in the AWS Console.

| Type | Port | Source | Purpose |
|---|---:|---|---|
| SSH | 22 | `YOUR_PUBLIC_IP/32` | Terminal access |
| Custom TCP | 8080 | `YOUR_PUBLIC_IP/32` | Jenkins UI |
| HTTP | 80 | Your IP for a private demo, or `0.0.0.0/0` if the website must be publicly viewable | Laravel website |

Do not expose Jenkins port `8080` to the whole internet for this lab. Do not add inbound rules for MySQL port `3306` or PHP-FPM port `9000`; both remain internal to Docker networking.

Connect to the instance using the EC2 connection instructions. The commands below assume the default Ubuntu user:

```bash
ssh -i YOUR_KEY.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Run commands from the EC2 terminal unless a step explicitly says to use the browser or AWS Console.

---

## 3. Update Ubuntu and install basic utilities

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg git unzip openssl
```

Confirm the operating system and available disk space:

```bash
cat /etc/os-release
df -h /
free -h
```

If the root disk is nearly full, increase the EBS volume in AWS before proceeding. Do not run Docker cleanup commands that delete volumes; this lab's MySQL volume will hold database data.

---

## 4. Install Docker Engine and the Docker Compose plugin

Use Docker's official APT repository. Do not install the old standalone `docker-compose` binary.

### 4.1 Add Docker's official package repository

```bash
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

### 4.2 Install Docker and Compose

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

Enable Docker and verify it:

```bash
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
sudo docker version
sudo docker compose version
sudo docker run --rm hello-world
```

Use `sudo docker` and `sudo docker compose` throughout this guide. Do **not** add your user to the Docker group or run `newgrp docker`. Access to the Docker daemon is effectively root-level access.

Confirm the Docker executable path:

```bash
command -v docker
```

On the expected Ubuntu installation, it should be `/usr/bin/docker`. The Jenkins sudo configuration below assumes that path; if your command prints a different path, use the actual path consistently.

---

## 5. Install Java 21 and Jenkins LTS

Jenkins requires a supported Java runtime. Install Java before installing Jenkins.

### 5.1 Install Java 21

```bash
sudo apt install -y fontconfig openjdk-21-jre
java -version
```

Confirm that the output identifies Java 21.

### 5.2 Add the Jenkins LTS repository

```bash
sudo install -m 0755 -d /etc/apt/keyrings

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list >/dev/null

sudo apt update
sudo apt install -y jenkins
```

Enable and start Jenkins:

```bash
sudo systemctl enable --now jenkins
sudo systemctl status jenkins --no-pager
```

If Jenkins fails to start, inspect the service log rather than reinstalling immediately:

```bash
sudo journalctl -u jenkins.service -n 100 --no-pager
```

Check that port 8080 is listening:

```bash
sudo ss -lntp | grep ':8080'
```

### 5.3 Open Jenkins in your browser

In your browser, visit:

```text
http://YOUR_EC2_PUBLIC_IP:8080
```

Find the initial administrator password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Copy it into the Jenkins setup page. Select **Install suggested plugins**, create your administrator account, and complete setup.

Install/confirm these plugins under **Manage Jenkins → Plugins** if they are not already available:

- Pipeline
- Git
- Credentials Binding

The Docker pipeline below uses the installed Docker CLI through shell commands, so the Jenkins Docker Pipeline plugin is not required.

---

## 6. Allow Jenkins to run Docker commands without a Docker-group change

Jenkins runs as the Linux account `jenkins`. Docker commands normally require elevated privileges. This guide uses `sudo` and a narrowly named sudoers entry instead of adding Jenkins to the Docker group.

> **Security note:** Permission to run Docker as root can provide root-equivalent control over the EC2 host. Use this only on the isolated lab instance and do not reuse it as a production security design.

Check the Docker path:

```bash
command -v docker
```

If it is `/usr/bin/docker`, create the sudoers rule:

```bash
echo 'jenkins ALL=(root) NOPASSWD: /usr/bin/docker' \
  | sudo tee /etc/sudoers.d/jenkins-docker >/dev/null

sudo chmod 440 /etc/sudoers.d/jenkins-docker
sudo visudo -cf /etc/sudoers.d/jenkins-docker
```

Test the rule as the Jenkins user:

```bash
sudo -u jenkins sudo -n /usr/bin/docker version
sudo -u jenkins sudo -n /usr/bin/docker compose version
```

Both commands should display version information. If the test says the command is not permitted, re-check the executable path and the `visudo` validation output.

---

## 7. Create the deployment environment file

The Compose file requires these values:

- `DOCKERHUB_USERNAME`
- `APP_KEY`
- `APP_URL`
- `DB_DATABASE`
- `DB_USERNAME`
- `DB_PASSWORD`
- `MYSQL_ROOT_PASSWORD`

For this lab, the application database username is `laraveluser`, the database is `laravel`, and the requested application database password is `sami1234`. A separate random root password is generated below. Do not reuse the application password as the MySQL root password.

### 7.1 Get the EC2 public IPv4 address

The following uses EC2 instance metadata with a token:

```bash
TOKEN=$(curl -fsS -X PUT \
  'http://169.254.169.254/latest/api/token' \
  -H 'X-aws-ec2-metadata-token-ttl-seconds: 21600')

PUBLIC_IP=$(curl -fsS \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  'http://169.254.169.254/latest/meta-data/public-ipv4')

printf 'EC2 public IP: %s\n' "$PUBLIC_IP"
unset TOKEN
```

If metadata access is disabled or no public IP is assigned, get the public IPv4 value from the EC2 Console and use it in the `APP_URL` value in the next step.

### 7.2 Generate a new APP_KEY and a separate MySQL root password

```bash
APP_KEY="base64:$(openssl rand -base64 32 | tr -d '\n')"
MYSQL_ROOT_PASSWORD="$(openssl rand -hex 24)"

sudo install -d -o root -g jenkins -m 750 /opt/laravel-cicd

sudo tee /opt/laravel-cicd/.env >/dev/null <<EOF
DOCKERHUB_USERNAME=sami1800
APP_KEY=$APP_KEY
APP_URL=http://$PUBLIC_IP
DB_DATABASE=laravel
DB_USERNAME=laraveluser
DB_PASSWORD=sami1234
MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD
EOF

sudo chown root:jenkins /opt/laravel-cicd/.env
sudo chmod 640 /opt/laravel-cicd/.env

unset APP_KEY MYSQL_ROOT_PASSWORD PUBLIC_IP
```

This command does not print the generated passwords. Keep `/opt/laravel-cicd/.env` on the EC2 instance; never commit it to GitHub or paste its contents into a public log.

Check only that the file exists and has restrictive permissions—do not print its contents:

```bash
sudo ls -l /opt/laravel-cicd/.env
```

The Jenkins pipeline will pass this file to Compose with `--env-file`. It is separate from the Git checkout and will not be overwritten by later builds.

---

## 8. Prepare Docker Hub

Sign in to Docker Hub in your browser.

Under the `sami1800` namespace, create these two repositories if they do not already exist:

- `laravel-app`
- `laravel-web`

For this lab, the repositories can be public so the published images are easy to inspect. If you keep them private, the Jenkins Docker Hub credential must have permission to push and pull them.

Create a **Docker Hub access token** with the permissions needed to push images. Use the token in Jenkins credentials; do not put it in the repository, `.env`, or Jenkinsfile.

The image names used in the pipeline will be:

```text
sami1800/laravel-app:<Git-commit-tag>
sami1800/laravel-web:<Git-commit-tag>
```

The pipeline also pushes a `latest` tag for inspection, but deployment uses the commit-specific tag.

---

## 9. Create the Jenkins Docker Hub credential

In Jenkins:

1. Open **Manage Jenkins → Credentials**.
2. Select the appropriate global/system credential store.
3. Add a credential of type **Username with password**.
4. Username: `sami1800`.
5. Password: paste the Docker Hub access token, not your account password.
6. ID: `dockerhub-creds`.
7. Description: `Docker Hub push token for Laravel CI/CD`.
8. Save.

The ID must match `dockerhub-creds` in the Jenkinsfile below. Jenkins masks the credential in normal logs; the pipeline also disables shell tracing around login.

---

## 10. Clone the repository and confirm its files

The Jenkins job will check out the repository itself, but clone it once on the EC2 host so you can add and commit the new pipeline file.

```bash
mkdir -p ~/devops-practice
cd ~/devops-practice

git clone https://github.com/samikhan5055/laravel-project.git
cd laravel-project
```

If the directory already exists, use it rather than cloning into it again:

```bash
cd ~/devops-practice/laravel-project
git pull origin main
```

Confirm the repository's important files:

```bash
git branch --show-current
ls -l Dockerfile compose.yaml docker/nginx/default.conf composer.lock
```

Inspect the actual Compose services and Dockerfile targets:

```bash
grep -nE '^(FROM|services:|  app:|  web:|  db:|  target:|    ports:)' Dockerfile compose.yaml
```

The Dockerfile should have `app` and `web` targets. The Compose file should define `app`, `web`, and `db`. If these files have been changed in the repository, stop and reconcile the differences before using the pipeline below.

---

## 11. Add a separate `Jenkinsfile.docker`

Do not overwrite the repository's existing `Jenkinsfile`. Create `Jenkinsfile.docker` in the repository root:

```bash
cd ~/devops-practice/laravel-project

cat > Jenkinsfile.docker <<'JENKINSFILE'
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        DOCKERHUB_NAMESPACE = 'sami1800'
        APP_IMAGE = 'sami1800/laravel-app'
        WEB_IMAGE = 'sami1800/laravel-web'
        DEPLOY_DIR = '/opt/laravel-cicd'
        COMPOSE_PROJECT_NAME = 'laravel-cicd'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh '''
                    set -eu
                    test -f Dockerfile
                    test -f compose.yaml
                    test -f docker/nginx/default.conf
                    test -f composer.lock
                    echo "Required repository files are present."
                '''
            }
        }

        stage('Validate Compose Configuration') {
            steps {
                sh '''
                    set -eu
                    test -f "$DEPLOY_DIR/.env"
                    sudo docker compose \
                      --env-file "$DEPLOY_DIR/.env" \
                      -p "$COMPOSE_PROJECT_NAME" \
                      -f "$WORKSPACE/compose.yaml" \
                      config -q
                    echo "Compose configuration is valid."
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()
                }

                sh '''
                    set -eu

                    echo "Building Laravel PHP-FPM image..."
                    sudo docker build --pull \
                      --target app \
                      -t "$APP_IMAGE:$IMAGE_TAG" \
                      -t "$APP_IMAGE:latest" \
                      .

                    echo "Building Nginx web image..."
                    sudo docker build --pull \
                      --target web \
                      -t "$WEB_IMAGE:$IMAGE_TAG" \
                      -t "$WEB_IMAGE:latest" \
                      .

                    echo "Built image tag: $IMAGE_TAG"
                '''
            }
        }

        stage('Container Smoke Checks') {
            steps {
                sh '''
                    set -eu

                    echo "Checking Laravel image..."
                    sudo docker run --rm \
                      --entrypoint php \
                      "$APP_IMAGE:$IMAGE_TAG" \
                      artisan --version

                    echo "Checking Nginx configuration..."
                    sudo docker run --rm \
                      --entrypoint nginx \
                      "$WEB_IMAGE:$IMAGE_TAG" \
                      -t

                    echo "Both container smoke checks passed."
                '''
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu
                        set +x

                        printf '%s' "$DOCKERHUB_TOKEN" |
                          sudo docker login \
                            --username "$DOCKERHUB_USER" \
                            --password-stdin

                        sudo docker push "$APP_IMAGE:$IMAGE_TAG"
                        sudo docker push "$APP_IMAGE:latest"
                        sudo docker push "$WEB_IMAGE:$IMAGE_TAG"
                        sudo docker push "$WEB_IMAGE:latest"
                    '''
                }
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                sh '''
                    set -eu

                    # Use the exact images built for this Git commit.
                    cat > "$WORKSPACE/compose.cicd.yaml" <<EOF
services:
  app:
    image: ${APP_IMAGE}:${IMAGE_TAG}
  web:
    image: ${WEB_IMAGE}:${IMAGE_TAG}
EOF

                    echo "Pulling the commit-tagged images..."
                    sudo docker compose \
                      --env-file "$DEPLOY_DIR/.env" \
                      -p "$COMPOSE_PROJECT_NAME" \
                      -f "$WORKSPACE/compose.yaml" \
                      -f "$WORKSPACE/compose.cicd.yaml" \
                      pull app web

                    echo "Starting/updating the three-tier application..."
                    sudo docker compose \
                      --env-file "$DEPLOY_DIR/.env" \
                      -p "$COMPOSE_PROJECT_NAME" \
                      -f "$WORKSPACE/compose.yaml" \
                      -f "$WORKSPACE/compose.cicd.yaml" \
                      up -d --no-build --remove-orphans

                    echo "Running Laravel database migrations..."
                    sudo docker compose \
                      --env-file "$DEPLOY_DIR/.env" \
                      -p "$COMPOSE_PROJECT_NAME" \
                      -f "$WORKSPACE/compose.yaml" \
                      -f "$WORKSPACE/compose.cicd.yaml" \
                      exec -T app php artisan migrate --force

                    echo "Current Compose service status:"
                    sudo docker compose \
                      --env-file "$DEPLOY_DIR/.env" \
                      -p "$COMPOSE_PROJECT_NAME" \
                      -f "$WORKSPACE/compose.yaml" \
                      -f "$WORKSPACE/compose.cicd.yaml" \
                      ps
                '''
            }
        }

        stage('HTTP Smoke Test') {
            steps {
                sh '''
                    set -eu
                    echo "Checking the website on local port 80..."
                    curl --fail --silent --show-error \
                      --retry 10 \
                      --retry-delay 3 \
                      --retry-connrefused \
                      http://127.0.0.1/ \
                      --output /dev/null

                    echo "HTTP smoke test passed."
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: images built, checked, pushed, deployed, and HTTP-tested.'
            echo 'Website: http://EC2_PUBLIC_IP/'
        }

        failure {
            echo 'FAILURE: inspect the first failed stage in Console Output.'
        }

        always {
            sh 'sudo docker logout >/dev/null 2>&1 || true'
            echo 'Docker CI/CD pipeline finished.'
        }
    }
}
JENKINSFILE
```

### What this Jenkinsfile does

- Uses the Git commit ID as an immutable deployment tag.
- Builds the `app` and `web` targets already defined in your Dockerfile.
- Checks that Laravel can run an Artisan command and that Nginx configuration passes `nginx -t`.
- Pushes the app and web images to Docker Hub.
- Creates a small Compose override file in the Jenkins workspace to select the commit-specific image tags without changing the existing `compose.yaml`.
- Uses a fixed Compose project name (`laravel-cicd`) so repeated builds reuse the same Compose-managed named volumes.
- Runs database migrations and tests the website over local HTTP.

**Testing limitation:** The repository's production Dockerfile installs Composer dependencies with `--no-dev`, and the PHP image installs `pdo_mysql` rather than `pdo_sqlite`. Therefore this pipeline uses image/configuration smoke checks, not `php artisan test`. A full PHPUnit stage needs a separate test image/environment with development dependencies and a compatible test database.

---

## 12. Commit and push the new pipeline file

The repository is public, so no GitHub credential is needed for this initial checkout. You need permission to push changes to the repository.

```bash
cd ~/devops-practice/laravel-project

git status --short
git add Jenkinsfile.docker
git commit -m "ci: add Docker container deployment pipeline"
git push origin main
```

If Git asks for your identity, configure the name and email you use for GitHub. If push authentication fails, use your existing Git authentication method; do not place a personal access token directly in the command or commit it to a file.

Verify that `Jenkinsfile.docker` appears in the `main` branch on GitHub.

---

## 13. Create the Jenkins Pipeline job

Open Jenkins at `http://YOUR_EC2_PUBLIC_IP:8080`.

1. Select **New Item**.
2. Name it `Laravel-Docker-CI-CD`.
3. Select **Pipeline**.
4. Click **OK**.
5. Under **General**, optionally add a description such as `Build, publish, and deploy the Laravel three-tier Docker application`.
6. Under **Build Triggers**, select **Poll SCM** and use:

   ```text
   H/2 * * * *
   ```

   This checks for repository changes approximately every two minutes. It avoids needing a public GitHub webhook to reach Jenkins. You can also use **Build Now** for immediate testing.

7. Under **Pipeline**:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/samikhan5055/laravel-project.git`
   - Branch specifier: `*/main`
   - Script Path: `Jenkinsfile.docker`
8. Leave Git credentials empty if the repository remains public.
9. Save the job.

The script path is important: choose `Jenkinsfile.docker`, not the existing `Jenkinsfile`.

---

## 14. Run the first build

In Jenkins, open `Laravel-Docker-CI-CD` and select **Build Now**.

Open the build number, then **Console Output**. Follow these stages in order:

1. Checkout
2. Validate Compose Configuration
3. Build Docker Images
4. Container Smoke Checks
5. Push Images to Docker Hub
6. Deploy with Docker Compose
7. HTTP Smoke Test

The first build may take longer because it downloads base images, Composer dependencies, and Node packages.

If it fails, fix the first failing stage before rerunning the entire pipeline.

---

## 15. Verify the deployment on EC2

SSH into the same EC2 instance and run:

### 15.1 Check the Compose services

```bash
sudo docker compose \
  --env-file /opt/laravel-cicd/.env \
  -p laravel-cicd \
  -f ~/devops-practice/laravel-project/compose.yaml \
  ps
```

The command above assumes the checked-out directory is `~/devops-practice/laravel-project`. Jenkins uses its own workspace for deployments; if the SSH checkout path does not exist, run the equivalent command with the actual `compose.yaml` path from Jenkins's workspace, or inspect the containers directly as below.

### 15.2 Inspect running containers

```bash
sudo docker ps
```

Expected service roles:

- `web`: Nginx, publishing host port `80`.
- `app`: Laravel running under PHP-FPM on port `9000` internally.
- `db`: MySQL 8.0 on the internal Compose network.

Do not publish MySQL port `3306` or PHP-FPM port `9000` to the public host.

### 15.3 Inspect recent logs

```bash
sudo docker logs --tail 100 "$(sudo docker ps -q --filter name=laravel-cicd-web)"
sudo docker logs --tail 100 "$(sudo docker ps -q --filter name=laravel-cicd-app)"
sudo docker logs --tail 100 "$(sudo docker ps -q --filter name=laravel-cicd-db)"
```

Container names can include Compose-generated suffixes. If a filter returns no container, list names with `sudo docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'` and use the actual names.

### 15.4 Test locally on the EC2 host

```bash
curl -I http://127.0.0.1/
```

A successful response should show an HTTP status such as `200 OK` or another expected success/redirect response.

### 15.5 Open the website

Visit:

```text
http://YOUR_EC2_PUBLIC_IP/
```

Use the EC2 instance's current public IPv4 address from the AWS Console. The EC2 Security Group must allow inbound HTTP on port 80 for your source IP (or for the intended public demo).

---

## 16. Demonstrate automatic CI/CD

To prove the pipeline works beyond the initial build:

1. Make a harmless change in the Laravel project—for example, update a comment or text on an existing page.
2. Commit and push the change to `main`.
3. Wait for the Poll SCM interval or start the job manually.
4. Confirm Jenkins checks out the new commit.
5. Confirm the new commit-specific Docker images are built and pushed.
6. Confirm the deployment stage pulls and runs those exact tags.
7. Confirm the HTTP smoke test passes.
8. Check the new image tags in Docker Hub.

The commit-specific tag should change when the Git commit changes. The deployed app and web services should use the new commit tag, while the Compose project name and named volumes remain stable.

---

## 17. Common errors and exact fixes

### Error A — `docker: permission denied` or Docker socket access denied

Use `sudo docker ...` for host commands. For Jenkins, run:

```bash
sudo -u jenkins sudo -n /usr/bin/docker version
sudo visudo -cf /etc/sudoers.d/jenkins-docker
```

If the executable is not `/usr/bin/docker`, update the sudoers file to match `command -v docker`. This guide intentionally does not use Docker group membership or `newgrp docker`.

### Error B — `docker compose: command not found`

Check the Compose plugin:

```bash
sudo docker compose version
```

If it is missing, install the official `docker-compose-plugin` package from the Docker APT repository in Section 4. Use `docker compose`, with a space, not the old `docker-compose` command.

### Error C — Jenkins cannot start

```bash
java -version
sudo systemctl status jenkins --no-pager
sudo journalctl -u jenkins.service -n 100 --no-pager
```

Confirm Java 21 is installed. Check whether port 8080 is already in use and whether the EC2 Security Group allows your IP to reach it.

### Error D — `dockerhub-creds` credential not found

Open **Manage Jenkins → Credentials** and verify that the credential ID is exactly `dockerhub-creds`. The token belongs in the credential, not in the Jenkinsfile.

### Error E — Docker Hub push returns `denied`

Check all three items:

1. The credential username is `sami1800`.
2. The credential password is a valid Docker Hub access token with push permission.
3. The repositories `sami1800/laravel-app` and `sami1800/laravel-web` exist and the account can push to them.

### Error F — Compose reports a required variable is unset

The Compose file requires an environment file. Confirm that the file exists without displaying its contents:

```bash
sudo ls -l /opt/laravel-cicd/.env
```

The pipeline supplies it using:

```text
--env-file /opt/laravel-cicd/.env
```

Do not commit this file to Git.

### Error G — `No space left on device`

```bash
df -h
sudo docker system df
```

Increase the EC2 EBS root volume if required. Avoid `docker system prune --volumes` and avoid deleting named volumes, because the MySQL volume stores the database.

### Error H — MySQL `Access denied`

Check the database variables in `/opt/laravel-cicd/.env` without printing the password. The app values must agree:

- Database: `laravel`
- User: `laraveluser`
- App password: `sami1234`

The MySQL root password must be a separate value. On a fresh instance, Compose initializes the database with the values supplied on its first start. Changing a Secret or `.env` later does not automatically change the password of a MySQL account in an already initialized data directory. If credentials need rotation, perform a deliberate MySQL account-password update and coordinate the app configuration; do not delete the database volume to bypass the issue.

### Error I — Nginx returns `502 Bad Gateway`

Inspect the web and app logs:

```bash
sudo docker ps
sudo docker logs --tail 100 CONTAINER_NAME
```

The existing Nginx config uses `fastcgi_pass app:9000;`. Compose must therefore keep the service name `app` and connect the web and app services to the shared frontend network. Do not use a Pod/host IP in this Docker Compose Nginx configuration.

### Error J — Website works locally but not from the browser

Check all of the following:

```bash
sudo docker ps
sudo ss -lntp | grep ':80'
curl -I http://127.0.0.1/
```

Then verify the EC2 Security Group permits inbound TCP port 80 from your client IP. Do not open ports 3306 or 9000 publicly.

### Error K — Pipeline reports that Artisan or Composer tests are missing

The production app image is intentionally built with `composer install --no-dev`; it is not a full test-runner image. This guide checks `php artisan --version` and `nginx -t`. For PHPUnit coverage, create a separate CI test image with development dependencies and the needed database driver, then run `php artisan test` before publishing production images.

### Error L — `git push` is rejected

Run:

```bash
git status
git branch --show-current
git remote -v
```

Make sure the change is on `main` and that your GitHub account has push permission. Authenticate through your normal Git credential method; never paste a token into a tracked file or a command that will be saved in shell history.

---

## 18. Safety and cleanup rules for this lab

- Never commit `.env`, Docker Hub tokens, passwords, or application secrets.
- Do not expose Jenkins port `8080` publicly for convenience.
- Do not publish MySQL `3306` or PHP-FPM `9000` on the EC2 host.
- Do not run `docker compose down -v`, `docker volume prune`, or `docker system prune --volumes` as routine troubleshooting. Those operations can destroy database data.
- Keep the Compose project name fixed as `laravel-cicd`; do not deploy different build workspaces under different project names.
- The app and web images are tagged with the Git commit ID. Avoid relying on `latest` for deployment.
- The migration stage changes the database schema. Review migration files before pushing application changes.
- This is a learning environment. Production deployment would need stronger secret management, backups, TLS, monitoring, access controls, and a deliberate rollback/database migration strategy.

---

## 19. Completion checklist

Mark each item only after verifying it:

- [ ] Fresh Ubuntu EC2 instance created with adequate RAM and disk.
- [ ] Security Group limits SSH and Jenkins access to my IP.
- [ ] Docker Engine and Docker Compose plugin installed.
- [ ] Java 21 and Jenkins LTS installed and running.
- [ ] Jenkins can run `sudo docker version`.
- [ ] `/opt/laravel-cicd/.env` created with restrictive permissions and not committed.
- [ ] Docker Hub repositories `sami1800/laravel-app` and `sami1800/laravel-web` exist.
- [ ] Jenkins credential ID `dockerhub-creds` created.
- [ ] `Jenkinsfile.docker` committed to the repository without overwriting `Jenkinsfile`.
- [ ] Jenkins Pipeline job uses `Jenkinsfile.docker` and the `main` branch.
- [ ] Compose validation succeeds.
- [ ] Both Docker images build and pass smoke checks.
- [ ] Commit-tagged images are pushed to Docker Hub.
- [ ] Compose runs the `web`, `app`, and `db` services.
- [ ] Laravel migrations complete successfully.
- [ ] HTTP smoke test passes.
- [ ] Website opens at `http://EC2_PUBLIC_IP/`.
- [ ] A subsequent Git push triggers a new pipeline run and deployment.

## 20. Useful reference documentation

- Docker Engine on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Docker Compose: https://docs.docker.com/compose/
- Jenkins installation on Linux: https://www.jenkins.io/doc/book/installing/linux/
- Jenkins Pipeline: https://www.jenkins.io/doc/book/pipeline/
- Jenkins credentials: https://www.jenkins.io/doc/book/using/using-credentials/
- Docker Hub: https://hub.docker.com/

---

**Expected final result:** A fresh EC2 Ubuntu host runs Jenkins and Docker. A push to the Laravel GitHub repository starts a Jenkins Pipeline that builds and validates the app and web images, pushes commit-tagged images to Docker Hub, deploys those images with the existing three-tier Compose configuration, runs migrations, and verifies the website over HTTP.
