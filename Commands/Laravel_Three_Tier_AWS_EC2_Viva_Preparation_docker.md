# Laravel Three-Tier Docker Deployment on AWS EC2

## Viva Preparation Guide

**Project:** Deploy a Laravel application using Docker Compose on an AWS
EC2 Ubuntu instance\
**Architecture:** Nginx web tier, Laravel/PHP-FPM application tier, and
MySQL database tier\
**Repository:** `https://github.com/samikhan5055/laravel-project.git`\
**Docker Hub username used in the lab:** `sami1800`

> **Important:** This guide explains the deployment setup and the
> commands used in the lab. Keep passwords, the Laravel `APP_KEY`, and
> Docker Hub credentials private. The lab `.env` values are examples for
> this exercise, not recommended production credentials.

------------------------------------------------------------------------

# 1. Project Overview

## 1.1 What is the objective of this project?

The objective is to containerize a Laravel application and deploy it on
an AWS EC2 Ubuntu server using Docker Compose. The application is split
into three logical tiers:

1.  **Web tier --- Nginx:** Accepts HTTP requests from users and
    forwards PHP requests to PHP-FPM.
2.  **Application tier --- Laravel with PHP-FPM:** Executes the Laravel
    application and its business logic.
3.  **Database tier --- MySQL:** Stores the application's relational
    data.

Docker Compose defines and manages the services, networks, and
persistent volumes used by the application.

## 1.2 Explain the request flow

When a user opens `http://<EC2-public-IP>/`:

1.  The browser sends an HTTP request to the EC2 public IP on port `80`.
2.  The EC2 security group must permit inbound HTTP traffic on port
    `80`.
3.  Docker publishes host port `80` to the Nginx container's port `80`.
4.  Nginx serves static files or forwards a PHP request to the `app`
    service at `app:9000`.
5.  PHP-FPM executes the Laravel PHP application.
6.  Laravel connects to MySQL using the Compose service name `db` and
    port `3306`.
7.  The response returns through PHP-FPM and Nginx to the browser.

**Flow:** Browser → EC2 port 80 → Nginx → PHP-FPM/Laravel → MySQL →
response back to browser.

## 1.3 Why call it a three-tier architecture?

It separates presentation/web handling, application processing, and data
storage into distinct services. This improves separation of concerns and
makes each tier easier to configure and troubleshoot. In this lab, the
tiers are separate containers on one EC2 instance; they are **not**
three separate servers.

------------------------------------------------------------------------

# 2. Key Technologies and Their Roles

  -----------------------------------------------------------------------
  Technology                          Purpose in this project
  ----------------------------------- -----------------------------------
  AWS EC2                             Virtual server that hosts the
                                      deployment

  Ubuntu                              Operating system on the EC2
                                      instance

  Docker Engine                       Builds and runs containers

  Docker Compose                      Defines and coordinates multiple
                                      containers

  Dockerfile                          Instructions for building the
                                      application and web images

  Nginx                               Web server and reverse/proxy
                                      gateway to PHP-FPM

  PHP-FPM                             Runs PHP processes that handle
                                      Laravel requests

  Laravel                             PHP web application framework

  MySQL 8.0                           Relational database

  Docker Hub                          Registry for storing and retrieving
                                      container images

  Git and GitHub                      Version control and remote source
                                      repository

  AWS Security Group                  Instance-level virtual firewall
                                      controlling inbound/outbound
                                      traffic

  Named volumes                       Persist data independently of a
                                      container's lifecycle

  Docker networks                     Control communication between
                                      containers
  -----------------------------------------------------------------------

## 2.1 What is the difference between an image and a container?

-   **Image:** A packaged, read-only template containing the application
    and required runtime files.
-   **Container:** A running or stopped instance created from an image.
-   One image can be used to create multiple containers.
-   Rebuilding an image does not automatically mean an existing
    container is replaced; Compose may need to recreate it.

## 2.2 What is the difference between Docker and Docker Compose?

Docker Engine builds and runs containers. Docker Compose uses a YAML
file---here, `compose.yaml`---to define and manage a multi-container
application with one set of commands.

------------------------------------------------------------------------

# 3. Important Files and Their Responsibilities

## 3.1 `Dockerfile`

Defines how Docker builds the images. In this project, it uses a Node.js
stage to compile frontend assets, a PHP-FPM stage for Laravel, and an
Nginx stage for serving HTTP requests.

## 3.2 `compose.yaml`

Defines the `db`, `app`, and `web` services, environment variables,
image names, networks, published ports, health checks, startup
dependencies, and named volumes.

## 3.3 `docker/nginx/default.conf`

Configures Nginx to use Laravel's public directory, route requests to
`index.php` when appropriate, and pass PHP requests to `app:9000`.

## 3.4 `.env`

Stores configuration values used by Docker Compose, including the Docker
Hub username, database settings, application URL, and Laravel
application key. It must not be committed to GitHub.

## 3.5 `.dockerignore`

Excludes unnecessary or sensitive files from the Docker build context,
such as `.env`, `.git`, `vendor`, and `node_modules`.

## 3.6 `artisan` and `composer.json`

-   `artisan` is Laravel's command-line entry point.
-   `composer.json` defines PHP dependencies and project metadata.
-   `composer.lock`, when present, records resolved dependency versions
    for reproducible dependency installation.

------------------------------------------------------------------------

# 4. Commands and What Each One Does

## 4.1 Inspect the EC2 environment

### `cat /etc/os-release`

Displays operating-system information, including the Ubuntu release.

### `uname -m`

Displays the CPU architecture, commonly `x86_64` or `aarch64`.

### `whoami`

Displays the current Linux username, expected to be `ubuntu` in this
lab.

### `df -h /`

Displays disk usage for the filesystem containing `/`, using
human-readable units. Docker builds can fail if disk space is exhausted.

## 4.2 Install and verify Docker

### `sudo apt update`

Refreshes the package index so APT knows which package versions are
available.

### `sudo apt install -y ca-certificates curl git`

Installs prerequisites used to download repository information and work
with Git. The `-y` option automatically answers yes to package prompts.

### `sudo systemctl enable --now docker`

Enables Docker to start at boot and starts the service immediately.

### `sudo docker --version`

Displays the installed Docker client version.

### `sudo docker compose version`

Displays the installed Docker Compose plugin version.

### `sudo systemctl is-active docker`

Checks whether the Docker service is active.

### `sudo docker run --rm hello-world`

Downloads the small `hello-world` image if needed, runs a test
container, prints a success message, and removes the container when it
exits. `--rm` removes the stopped test container.

### `sudo docker ps`

Lists running containers. To include stopped containers, use
`sudo docker ps -a`.

**Why use `sudo docker`?** The lab deliberately uses `sudo` for Docker
commands rather than adding the Ubuntu user to the Docker group. This
avoids configuring Docker group membership. Access to Docker is
powerful, so protect the account and use trusted administrative access.

## 4.3 Navigate and inspect project files

### `cd ~/laravel-project`

Changes the current directory to the `laravel-project` directory in the
user's home directory.

### `mkdir -p docker/nginx`

Creates the `docker/nginx` directory. `-p` also creates missing parent
directories and does not fail if the directory already exists.

### `nano Dockerfile`

Opens or creates the `Dockerfile` in the Nano text editor. Save with
`Ctrl+O`, press Enter, and exit with `Ctrl+X`.

### `nano compose.yaml`

Opens or creates the Compose configuration file.

### `nano docker/nginx/default.conf`

Opens or creates the Nginx configuration file.

### `nano .env`

Opens or creates the environment file. The leading dot makes it a hidden
file in standard directory listings.

### `test -f Dockerfile`

Checks whether `Dockerfile` exists. The command exits successfully if it
is a regular file.

### `ls`

Lists files and directories in the current directory.

## 4.4 Validate Compose and build images

### `sudo docker compose config --quiet`

Parses and validates the Compose configuration without printing the
fully rendered configuration. It can report missing required variables
or YAML/configuration errors. It does not prove that the app will start
successfully.

### `sudo docker compose build app web`

Builds the `app` and `web` images defined in `compose.yaml`. The `app`
target builds the PHP-FPM/Laravel image; the `web` target builds the
Nginx image.

### `sudo docker image ls`

Lists images stored on the EC2 host.

### `sudo docker compose push app web`

Pushes the configured `app` and `web` images to the image registry
specified by their image names. The images must have been built and
tagged correctly, and the user must be authenticated to the registry.

### `sudo docker login -u sami1800`

Authenticates to Docker Hub as `sami1800`. Enter the password or access
token only at the interactive prompt; do not place it directly in the
command.

### `sudo docker compose pull`

Downloads the images referenced by the Compose configuration from the
registry.

### `sudo docker compose up -d`

Creates and starts the services in detached mode. `-d` means Docker
Compose returns control to the terminal while containers continue
running.

### `sudo docker compose up -d --no-build`

Starts or recreates services without building images. It is useful on
EC2 when the images have already been built and pushed to Docker Hub.

### `sudo docker compose up -d --force-recreate app`

Forces Compose to recreate the `app` container, even if it otherwise
considers the existing container current. It is useful after changing
environment settings that are supplied when the container is created.

### `sudo docker compose ps`

Displays the services/containers managed by the current Compose project
and their status.

### `sudo docker compose logs --tail=100 app`

Displays the latest 100 log lines from the application service. Replace
`app` with `web` or `db` to inspect Nginx or MySQL logs.

### `sudo docker compose exec app php artisan config:clear`

Runs `php artisan config:clear` inside the running `app` container to
clear Laravel's cached configuration. It does not run PHP on the EC2
host.

### `sudo docker compose exec app php artisan about`

Runs Laravel's `about` command inside the running application container
to show application/environment information.

### `curl -I http://127.0.0.1/`

Sends an HTTP request from the EC2 instance to its local port 80 and
displays response headers. A successful response is commonly
`HTTP/1.1 200 OK`; a 500 response means the HTTP request is reaching a
server that reports an application error.

## 4.5 Git commands

### `git clone https://github.com/samikhan5055/laravel-project.git`

Copies the remote repository to the local machine.

### `git pull`

Fetches and integrates changes from the configured remote branch into
the current local branch.

### `git status --short`

Shows a concise summary of changed, untracked, and staged files.

### `git check-ignore .env`

Checks whether `.env` is excluded by a Git ignore rule. If it prints a
matching rule, that rule is ignoring the file.

### `git add Dockerfile .dockerignore compose.yaml docker/nginx/default.conf`

Stages the named deployment files for the next commit. Explicitly naming
files helps avoid accidentally staging `.env`.

### `git commit -m "Add three-tier Docker deployment"`

Creates a commit containing staged changes and assigns it a message.

### `git push origin main`

Pushes the local `main` branch to the remote named `origin`. If the
current branch has a different name, use that branch name.

## 4.6 AWS and network checks

### Security Group inbound rule: SSH 22 from My IP

Allows SSH access from the administrator's current public IP.
Restricting the source is safer than allowing SSH from anywhere.

### Security Group inbound rule: HTTP 80 from `0.0.0.0/0`

Allows IPv4 clients on the internet to reach the HTTP service on port
80. HTTPS should be configured for production deployments.

### Why not expose ports 3306 and 9000?

-   MySQL port `3306` is intended for database traffic and should remain
    private.
-   PHP-FPM port `9000` is used internally by Nginx to pass PHP
    requests.
-   In this design, only Nginx publishes port `80` to the EC2 host.

------------------------------------------------------------------------

# 5. Docker Compose Architecture Questions

## Q1. What services are defined in the Compose file?

**Answer:** `web` for Nginx, `app` for Laravel running under PHP-FPM,
and `db` for MySQL.

## Q2. Why is Nginx separate from Laravel/PHP-FPM?

**Answer:** Nginx handles HTTP connections and static files, while
PHP-FPM manages PHP worker processes that execute Laravel. Separating
them makes responsibilities clearer and allows each service to be
configured independently.

## Q3. What does `fastcgi_pass app:9000;` mean?

**Answer:** Nginx forwards PHP requests using FastCGI to the service
named `app` on port `9000`. Docker Compose service discovery lets Nginx
resolve the service name `app` on a shared network.

## Q4. Why does Laravel use `DB_HOST=db` rather than `localhost`?

**Answer:** Inside the application container, `localhost` refers to that
same container. `db` is the Compose service name for the MySQL
container, so Laravel uses it to reach the database over the Docker
network.

## Q5. What is the purpose of the `frontend` and `backend` networks?

**Answer:** The `frontend` network connects Nginx to the application.
The `backend` network connects the application to MySQL. The database is
not connected to the frontend network, which limits direct network
reachability.

## Q6. What does `internal: true` mean on the backend network?

**Answer:** It creates an internally isolated Docker network. It helps
prevent external connectivity through that network, although it is not a
substitute for secure credentials and correct host
firewall/security-group rules.

## Q7. Why is port mapping written as `"80:80"`?

**Answer:** The left side is the host port and the right side is the
container port. Host port 80 forwards traffic to port 80 inside the
Nginx container.

## Q8. Why does only the web service publish a port?

**Answer:** Users need to reach Nginx over HTTP. The app and database
communicate over private Docker networks and do not need public host
ports.

## Q9. What is a named volume?

**Answer:** A Docker-managed storage location that can outlive an
individual container. It helps preserve data when a container is removed
and recreated.

## Q10. Which named volumes are used in this project?

**Answer:** `mysql_data` stores MySQL data, and `laravel_storage` stores
Laravel's storage directory. Database data in a named volume is not
normally deleted just because the database container is recreated.

## Q11. What does `depends_on` do?

**Answer:** It establishes startup ordering. In this Compose file, the
app waits for the database health check to report healthy. Startup order
alone does not guarantee that an application can never lose its database
connection later.

## Q12. What does the MySQL health check do?

**Answer:** It periodically checks whether MySQL responds to a local
ping using the configured root password. Compose can use the health
status to delay starting the application until the database is ready.

## Q13. What does `restart: unless-stopped` mean?

**Answer:** Docker restarts the service after many failures or Docker
daemon restarts unless the service was explicitly stopped by an
administrator.

## Q14. Why use `APP_DEBUG=false` in production?

**Answer:** It prevents Laravel from displaying detailed debug
information to end users. Detailed errors should be examined in
server-side logs instead.

------------------------------------------------------------------------

# 6. Laravel Configuration and HTTP 500 Troubleshooting

## Q1. What does HTTP 500 mean?

**Answer:** It is a generic server-side error. The web server received
the request but encountered a failure while processing it. The response
code alone does not identify the cause, so logs must be checked.

## Q2. What caused the 500 error observed in this lab?

**Answer:** The provided application log reported:
`Unsupported cipher or incorrect key length`. Laravel's encrypter was
configured for `AES-256-CBC`, but the configured application key was not
being interpreted as a valid key of the required length.

## Q3. How should an OpenSSL-generated key be entered in `.env`?

**Answer:** For the command `openssl rand -base64 32`, place the
complete output after the Laravel prefix, for example:

``` dotenv
APP_KEY=base64:PASTE_THE_COMPLETE_OPENSSL_OUTPUT_HERE
```

Do not leave the example placeholder in the real `.env` file. Keep the
key private.

## Q4. Why is `APP_KEY` important?

**Answer:** Laravel uses it for encryption-related features, including
encrypted cookies and other encrypted application data. If the key is
missing or invalid, requests can fail. Changing it later may invalidate
data encrypted with the old key.

## Q5. Why did `php artisan key:generate --show` fail on the EC2 host?

**Answer:** The host did not have the `php` CLI installed. In a
containerized setup, PHP is inside the `app` container, so commands can
be run through `sudo docker compose exec app php artisan ...` once the
app container is running. OpenSSL can also generate a compatible key
without installing PHP on the host.

## Q6. How do you troubleshoot a 500 error in this project?

**Answer:** 1. Run `sudo docker compose ps` to inspect service status.
2. Check `sudo docker compose logs --tail=100 app`. 3. Check
`sudo docker compose logs --tail=100 web`. 4. Check
`sudo docker compose logs --tail=100 db` if database connectivity may be
involved. 5. Inspect Laravel's application log if available. 6. Correct
the root cause and retest locally with `curl -I http://127.0.0.1/`.

## Q7. What does `config:clear` do?

**Answer:** It clears Laravel's cached configuration so the application
can reload configuration values from its current environment. It does
not fix an incorrect `APP_KEY` by itself; the underlying value must be
corrected first.

## Q8. Why should `APP_DEBUG` be false on a public server?

**Answer:** Debug pages can expose internal paths, configuration
details, and other sensitive implementation information. Use server-side
logs to diagnose errors.

## Q9. What is the purpose of `.env`?

**Answer:** It supplies environment-specific configuration without
hardcoding credentials into source files. It should be excluded from
version control and protected with restrictive file permissions.

## Q10. What should you do if Laravel reports a database access error?

**Answer:** Compare `DB_DATABASE`, `DB_USERNAME`, and `DB_PASSWORD` with
the MySQL initialization variables in Compose. Confirm the database
container is healthy and that the app uses `DB_HOST=db`. Do not expose
port 3306 publicly as a troubleshooting shortcut.

------------------------------------------------------------------------

# 7. Docker Hub and Image Deployment

## Q1. What is Docker Hub?

**Answer:** A container registry that stores Docker images so another
machine, such as EC2, can pull and run them.

## Q2. Why build images on the development computer and push them?

**Answer:** It separates image building from deployment. EC2 can pull
the already-built images rather than needing the source code and full
build toolchain. In this project, the Compose configuration can also
build images when running on the development machine.

## Q3. Why does EC2 need Docker Hub authentication?

**Answer:** Authentication is required for private repositories and may
be used for authenticated access to public repositories.
`sudo docker login` stores credentials for the Docker client context
used by that command.

## Q4. What is the difference between `build`, `push`, and `pull`?

-   `build`: Creates an image from the Dockerfile and build context.
-   `push`: Uploads a tagged image to a registry.
-   `pull`: Downloads an image from a registry.

## Q5. Why must the image name include the Docker Hub username?

**Answer:** A name such as `sami1800/laravel-app:1.0` identifies the
registry namespace, repository, and tag. EC2 must pull the same image
name that was pushed.

## Q6. What is an image tag?

**Answer:** A label used to identify a version or variant of an image.
This lab uses tag `1.0`. In production, immutable version tags or image
digests improve deployment traceability compared with relying only on
`latest`.

------------------------------------------------------------------------

# 8. GitHub, Secrets, and Reproducibility

## Q1. Why use Git?

**Answer:** Git tracks changes to project files and makes it possible to
review, commit, and revert changes. GitHub hosts the remote repository
for collaboration and deployment.

## Q2. Why must `.env` not be committed?

**Answer:** It contains secrets such as application keys and database
passwords. If committed to a public or shared repository, those values
may be exposed and difficult to fully remove from history.

## Q3. What is `.dockerignore` used for?

**Answer:** It prevents selected files from being sent in the Docker
build context. Excluding `.env`, `.git`, and local dependency folders
reduces unnecessary build data and helps prevent secrets from being
copied into images.

## Q4. What is the purpose of `composer.lock`?

**Answer:** It records exact resolved PHP dependency versions.
`composer install` uses the lock file to install those versions, making
builds more reproducible. Running `composer update` resolves dependency
versions again and can produce different results.

## Q5. Why use explicit filenames with `git add`?

**Answer:** It stages only the intended files and reduces the risk of
accidentally including `.env` or other local files.

------------------------------------------------------------------------

# 9. AWS EC2 and Security Questions

## Q1. What is EC2?

**Answer:** Amazon Elastic Compute Cloud provides virtual servers where
the operating system and applications can be installed and run.

## Q2. What is an EC2 public IP?

**Answer:** An address that external clients can use to reach the
instance when routing, network settings, and security-group rules allow
it. A public IP may change after stopping and starting an instance
unless an Elastic IP or other stable addressing arrangement is used.

## Q3. What is a security group?

**Answer:** A stateful virtual firewall associated with AWS resources
such as EC2 instances. Its rules control allowed inbound and outbound
traffic.

## Q4. Which inbound ports are required for this lab?

**Answer:** SSH port `22`, ideally restricted to the administrator's IP,
and HTTP port `80` for public web access. Do not expose MySQL port
`3306` or PHP-FPM port `9000` to the internet.

## Q5. Why might the site fail even if the containers are running?

**Answer:** Possible causes include missing security-group rules, an
incorrect public IP, Nginx configuration errors, an unhealthy app or
database, an invalid Laravel key, incorrect database credentials, or an
application exception. Check the relevant logs rather than assuming the
cause.

## Q6. Is this production-ready?

**Answer:** It demonstrates a three-tier containerized deployment, but
additional production work is needed, including HTTPS/TLS, strong
separate secrets, secure credential management, backups, monitoring,
patching, resource limits, and an appropriate database recovery plan.

------------------------------------------------------------------------

# 10. Common Troubleshooting Scenarios

  --------------------------------------------------------------------------
  Symptom                             What to check
  ----------------------------------- --------------------------------------
  `permission denied` for Docker      Use the lab's `sudo docker ...`
  socket                              command pattern

  Docker command not found            Check Docker installation and
                                      package/service status

  Compose reports a missing variable  Inspect `.env` and required variable
                                      names; run
                                      `sudo docker compose config --quiet`

  Image pull denied                   Check image name, tag, Docker Hub
                                      repository, and authentication

  Build fails at `npm run build`      Confirm `package.json` contains a
                                      valid build script and dependencies

  Build fails during Composer install Inspect Composer output and confirm
                                      `composer.json` and `composer.lock`
                                      are valid

  Nginx returns 502/504               Check whether `app` is running,
                                      Nginx's `fastcgi_pass`, and app logs

  Browser displays HTTP 500           Inspect Laravel/PHP logs and identify
                                      the first underlying exception

  Laravel reports invalid cipher/key  Correct `APP_KEY`, including the
  length                              `base64:` prefix for the OpenSSL
                                      method

  Database connection fails           Check DB health, `DB_HOST=db`,
                                      database name, username, and password

  Database data disappears            Verify that `mysql_data` is mounted
  unexpectedly                        and has not been deleted

  Browser cannot connect at all       Check EC2 public IP, security-group
                                      port 80, container status, and
                                      `sudo docker compose logs web`
  --------------------------------------------------------------------------

## Useful diagnostic commands

Run these from the directory containing `compose.yaml`:

``` bash
cd ~/laravel-project
sudo docker compose ps
sudo docker compose logs --tail=100 app
sudo docker compose logs --tail=100 web
sudo docker compose logs --tail=100 db
sudo docker compose config --quiet
curl -I http://127.0.0.1/
```

To inspect Laravel's log if it exists:

``` bash
sudo docker compose exec app sh -lc 'tail -n 100 storage/logs/laravel.log'
```

Do not share the full contents of `.env` or logs that contain
credentials.

------------------------------------------------------------------------

# 11. Quick Viva Questions and Short Answers

1.  **What is Docker?** --- A platform for packaging and running
    applications in containers.
2.  **What is Docker Compose?** --- A tool for defining and managing
    multi-container applications.
3.  **What is a container?** --- A runnable instance of an image.
4.  **What is a Docker image?** --- A template used to create
    containers.
5.  **What is Nginx doing here?** --- Handling HTTP requests and
    forwarding PHP requests to PHP-FPM.
6.  **What is PHP-FPM?** --- A FastCGI process manager that runs PHP
    workers.
7.  **What is Laravel?** --- A PHP web application framework.
8.  **What is MySQL?** --- A relational database management system.
9.  **What is a Dockerfile?** --- Instructions for building an image.
10. **What is `compose.yaml`?** --- The configuration defining services
    and how they connect.
11. **Why use three services?** --- To separate web serving, application
    processing, and data storage.
12. **Why use Docker networks?** --- To allow controlled
    service-to-service communication.
13. **Why use a named volume?** --- To preserve data beyond an
    individual container's lifecycle.
14. **What is port 80?** --- The standard HTTP port.
15. **What is port 3306?** --- MySQL's default port.
16. **What is port 9000 here?** --- The PHP-FPM FastCGI port used
    internally by Nginx.
17. **What is Docker Hub?** --- A registry for storing and distributing
    images.
18. **What is `docker compose up -d`?** --- It creates/starts services
    in detached mode.
19. **What is `docker compose ps`?** --- It displays Compose
    service/container status.
20. **What is `docker compose logs`?** --- It displays service logs.
21. **What does `--tail=100` do?** --- Limits the displayed logs to the
    latest 100 lines.
22. **What does `--force-recreate` do?** --- Forces a service container
    to be recreated.
23. **Why use `sudo docker`?** --- This lab uses administrative access
    for Docker instead of configuring Docker group membership.
24. **What is `.env`?** --- A file containing environment-specific
    configuration and secrets.
25. **Why not commit `.env`?** --- It can expose credentials and
    encryption keys.
26. **What is `APP_KEY`?** --- Laravel's application encryption key.
27. **What caused the observed 500 error?** --- The logs showed an
    invalid cipher/key length.
28. **What does `curl -I` do?** --- Requests and displays HTTP response
    headers.
29. **What is a security group?** --- An AWS virtual firewall for
    controlling traffic.
30. **Why restrict SSH to My IP?** --- To reduce exposure of remote
    administration.
31. **Why not expose MySQL publicly?** --- The database should be
    reachable only from trusted application components.
32. **What does `depends_on` do?** --- Controls service startup order
    and can use health conditions.
33. **What is a health check?** --- A periodic test used to determine
    whether a service is responding as expected.
34. **Why is `APP_DEBUG=false` used?** --- To avoid exposing detailed
    debug information publicly.
35. **What is CI/CD?** --- Continuous Integration and Continuous
    Delivery/Deployment; automating build, test, and release workflows.
36. **Does this setup itself implement CI/CD?** --- Not by itself; it
    provides a containerized deployment. A pipeline such as Jenkins or
    GitHub Actions would need to automate the workflow.
37. **What is the purpose of GitHub in this project?** --- To store and
    version-control the application and deployment configuration.
38. **What does `git push origin main` do?** --- Pushes the local `main`
    branch to the `origin` remote.
39. **What is the purpose of `composer.lock`?** --- To record resolved
    PHP dependency versions for repeatable installs.
40. **What would you improve for production?** --- HTTPS, stronger
    separate secrets, backups, monitoring, resource limits, and a more
    resilient hosting/database design.

------------------------------------------------------------------------

# 12. One-Minute Project Explanation

"My project deploys a Laravel application on an AWS EC2 Ubuntu server
using Docker and Docker Compose. I separated the application into three
services: Nginx handles incoming HTTP requests, PHP-FPM runs Laravel,
and MySQL stores application data. Docker Compose defines the services,
private networks, environment variables, health checks, and named
volumes. Only Nginx publishes port 80, while MySQL and PHP-FPM
communicate internally. I build and tag the Docker images, push them to
Docker Hub, and deploy them on EC2. I use GitHub to version-control the
deployment files and keep `.env` out of the repository. For
troubleshooting, I inspect container status and service logs, validate
the Compose configuration, and test the local HTTP response with
`curl`."

------------------------------------------------------------------------

# 13. Final Revision Checklist

Before the viva, make sure you can explain:

-   [ ] The three tiers and the request flow between them.
-   [ ] Why Nginx forwards PHP requests to `app:9000`.
-   [ ] Why Laravel connects to `DB_HOST=db`.
-   [ ] The difference between an image and a container.
-   [ ] The purpose of `Dockerfile`, `compose.yaml`, `.env`, and
    `.dockerignore`.
-   [ ] How Docker networks and named volumes are used.
-   [ ] The meaning of `build`, `push`, `pull`, `up -d`, `ps`, `exec`,
    and `logs`.
-   [ ] Why the EC2 security group permits ports 22 and 80 but not
    public ports 3306 and 9000.
-   [ ] Why `APP_KEY` must be valid and kept secret.
-   [ ] How to investigate an HTTP 500 error from logs.
-   [ ] Why `.env` must not be committed to GitHub.
-   [ ] The difference between this deployment and a complete CI/CD
    pipeline.
-   [ ] Which steps are performed on the development computer and which
    are performed on EC2.
