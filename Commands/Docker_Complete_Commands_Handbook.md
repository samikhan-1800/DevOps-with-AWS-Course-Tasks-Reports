# Docker Commands Handbook
## Practical reference for Docker Engine, images, containers, ports, networks, volumes, Compose, registries, troubleshooting, and deployment

**Environment:** Ubuntu/Linux with Docker Engine and Docker Compose installed. This guide uses `sudo docker ...` and `sudo docker compose ...`; it does not require adding your user to the Docker group.

**Replace placeholders** such as `CONTAINER_NAME`, `IMAGE_NAME`, `YOUR_USERNAME`, and `PROJECT_DIR` with your actual values. Run build commands from the directory containing the relevant `Dockerfile`.

**Data safety:** Volumes often contain database data. Inspect what a command will affect before removing anything. Do not use `docker compose down -v` or broad prune commands as routine troubleshooting.

---

## Contents
1. Fundamentals
2. Installation and service
3. System information and disk space
4. Images
5. Containers
6. Port mapping and changing ports
7. Dockerfiles and rebuilding
8. Networks
9. Volumes and bind mounts
10. Docker Compose
11. Updating Compose applications
12. Environment variables and secrets
13. Docker Hub
14. Build cache and build troubleshooting
15. Resource limits and health checks
16. Troubleshooting
17. EC2/public access
18. Cleanup safety
19. Common recipes
20. Cheat sheet
21. Key concepts

---

# 1. Docker fundamentals

An **image** is a template; a **container** is a running or stopped instance created from an image. A **Dockerfile** defines how to build an image. A **registry** stores images. A **volume** persists data. **Compose** defines multiple services in YAML.

```text
Source + Dockerfile --docker build--> Image --docker run--> Container
                                      ^                     |
                                      | docker pull          +-- port mapping --> Host
                                Image registry               +-- network ------> Other containers
                                                            +-- volume --------> Persistent data
```

General command pattern:

```bash
sudo docker COMMAND OPTIONS
sudo docker compose COMMAND OPTIONS
```

Get help:

```bash
sudo docker --help
sudo docker run --help
sudo docker image --help
sudo docker compose --help
sudo docker compose up --help
```

These commands list available options and syntax for the installed Docker version.

---

# 2. Installation and service status on Ubuntu

## Check installed versions

```bash
docker --version
sudo docker version
sudo docker compose version
```

The first prints the CLI version; the second checks both client and Engine; the third checks the Compose plugin.

## Manage the Docker service

```bash
sudo systemctl status docker
sudo systemctl start docker
sudo systemctl stop docker
sudo systemctl restart docker
sudo systemctl enable docker
sudo systemctl enable --now docker
sudo systemctl is-active docker
sudo systemctl is-enabled docker
```

These respectively inspect service status, start it, stop it, restart it, enable startup at boot, enable and start it immediately, check whether it is active, and check whether it is enabled. Restarting/stopping Docker can affect running workloads.

## Install Docker from Docker's official Ubuntu repository

Installation details can change; consult Docker's current official Ubuntu instructions if the repository setup differs for your Ubuntu release.

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Refreshes packages, installs prerequisites, creates the key directory, downloads Docker's signing key, and makes it readable to APT.

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo docker run --rm hello-world
```

Adds the official stable repository, refreshes package metadata, installs Docker Engine/CLI/runtime/Buildx/Compose, starts Docker, and verifies it with a disposable test container. `--rm` removes that test container after it exits.

---

# 3. System information and disk space

```bash
sudo docker info
sudo docker system df
sudo docker system df -v
df -h
free -h
sudo docker stats
sudo docker stats CONTAINER_NAME
sudo docker events
```

- `docker info`: daemon, storage driver, runtime and host details.
- `docker system df`: space used by images, containers, volumes and build cache.
- `docker system df -v`: more detailed Docker disk usage.
- `df -h`: host filesystem capacity; use when builds report `No space left on device`.
- `free -h`: host memory.
- `docker stats`: live resource use for running containers; `Ctrl+C` ends the display.
- `docker stats CONTAINER_NAME`: limits the live view to one container.
- `docker events`: streams daemon events until interrupted.

---

# 4. Images

## List, download, inspect

```bash
sudo docker images
sudo docker image ls
sudo docker images -a
sudo docker image ls --digests
sudo docker pull nginx
sudo docker pull nginx:1.27
sudo docker pull mysql:8.0
sudo docker image inspect IMAGE_NAME:TAG
sudo docker image history IMAGE_NAME:TAG
sudo docker image inspect --format='{{.Id}}' IMAGE_NAME:TAG
```

Lists local images (the first two are equivalent), includes intermediate images, shows digests, downloads an image/tag, or inspects image metadata/history/ID. Prefer a suitable version tag for reproducible deployments.

## Build an image

Run from the directory containing the Dockerfile:

```bash
sudo docker build -t myapp:1.0 .
sudo docker build -t myapp:1.0 -f Dockerfile .
sudo docker build --no-cache -t myapp:1.0 .
sudo docker build --pull -t myapp:1.0 .
sudo docker build --progress=plain -t myapp:1.0 .
```

- `-t`: assigns an image name and tag.
- `-f`: chooses a Dockerfile explicitly.
- `--no-cache`: rebuilds without reusing cached layers; slower and potentially more disk-intensive.
- `--pull`: checks for a newer base image where applicable.
- `--progress=plain`: shows detailed build output for troubleshooting.
- `.` is the build context, the files Docker can access for `COPY`/`ADD`.

For a multi-stage Dockerfile with named stages:

```bash
sudo docker build --target app -t myapp-app:1.0 .
sudo docker build --target web -t myapp-web:1.0 .
```

Builds only the `app` or `web` stage. The stage names must be defined in the Dockerfile.

## Tag, save, and load

```bash
sudo docker tag myapp:1.0 YOUR_USERNAME/myapp:1.0
sudo docker tag myapp:1.0 myapp:latest
sudo docker save -o myapp-image.tar myapp:1.0
sudo docker load -i myapp-image.tar
```

Adds another tag (does not duplicate image data), adds a `latest` label (not a guarantee of newest content), exports an image archive, or imports an archive.

## Remove images

```bash
sudo docker image rm IMAGE_NAME:TAG
sudo docker image rm IMAGE_ID
sudo docker image prune
sudo docker image prune -a
```

Removes a specific local image/tag, or prunes unused images. `image prune -a` is more aggressive: it removes all images not referenced by an existing container, potentially requiring future pulls/rebuilds. Check dependencies before removing.

---

# 5. Containers

## Run containers

```bash
sudo docker run nginx
sudo docker run -d --name web nginx
sudo docker run --name temporary-web --rm nginx
sudo docker run -it ubuntu:24.04 bash
sudo docker run -d --name web -p 8080:80 nginx
sudo docker run -d --name web-local -p 127.0.0.1:8080:80 nginx
```

- Runs Nginx attached to the terminal.
- Runs it in detached/background mode and names it `web`.
- Automatically removes the container after exit, but not the image.
- Opens an interactive Ubuntu shell; type `exit` to leave.
- Publishes host port 8080 to container port 80.
- Publishes only on host loopback, normally preventing direct remote access.

## List and inspect containers

```bash
sudo docker ps
sudo docker ps -a
sudo docker ps -q
sudo docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}'
sudo docker inspect CONTAINER_NAME
sudo docker inspect --format='{{.State.Status}}' CONTAINER_NAME
sudo docker port CONTAINER_NAME
sudo docker top CONTAINER_NAME
```

Lists running containers, all containers, running container IDs, a formatted table, full configuration/state/mounts/networks, just the state, published port mappings, or processes in the container.

## Logs and commands inside a container

```bash
sudo docker logs CONTAINER_NAME
sudo docker logs -f CONTAINER_NAME
sudo docker logs --tail 100 CONTAINER_NAME
sudo docker logs --since 10m CONTAINER_NAME
sudo docker logs -t CONTAINER_NAME
sudo docker exec -it CONTAINER_NAME bash
sudo docker exec -it CONTAINER_NAME sh
sudo docker exec CONTAINER_NAME ls -la /app
sudo docker exec -u root -it CONTAINER_NAME sh
```

Prints logs, follows them live, limits output to 100 lines, shows recent logs, adds timestamps, opens an interactive Bash or shell session, runs a single command, or opens a shell as root. `bash` may not exist in small images. `Ctrl+C` stops log-following, not the container.

## Lifecycle operations

```bash
sudo docker stop CONTAINER_NAME
sudo docker start CONTAINER_NAME
sudo docker restart CONTAINER_NAME
sudo docker pause CONTAINER_NAME
sudo docker unpause CONTAINER_NAME
sudo docker rm CONTAINER_NAME
sudo docker rm -f CONTAINER_NAME
sudo docker rename OLD_NAME NEW_NAME
sudo docker update --restart unless-stopped CONTAINER_NAME
```

Gracefully stops, starts an existing container, restarts, pauses/resumes processes, removes a stopped container, force-removes (and stops if necessary), renames, or changes restart policy. Removing a container can lose data stored only in its writable layer. Use volumes for persistent data.

## Copy files

```bash
sudo docker cp ./local-file.txt CONTAINER_NAME:/tmp/local-file.txt
sudo docker cp CONTAINER_NAME:/var/log/app.log ./app.log
```

Copies a host file into a container or a container file to the host. Changes made this way are generally not a repeatable deployment method; update source/Dockerfile and rebuild when appropriate.

---

# 6. Port mapping and changing ports

Port mapping syntax:

```text
-p HOST_PORT:CONTAINER_PORT
```

The host port is on the Docker machine; the container port is where the application listens inside the container. The application must actually listen on the container port. `EXPOSE` in a Dockerfile documents a port but does not publish it.

## Examples

```bash
sudo docker run -d --name nginx-a -p 8080:80 nginx
sudo docker run -d --name nginx-b -p 8081:80 nginx
sudo docker run -d --name app -p 8000:8000 myapp:1.0
sudo docker run -d --name local-only -p 127.0.0.1:8082:80 nginx
```

These map host 8080, host 8081, host 8000, or loopback-only 8082 to the specified container ports. The `8000:8000` example only works if the application listens on container port 8000.

## Change a published port on a regular container

You cannot change a container's published port mapping in place. Recreate the container with the new mapping. Example: change host port 8080 to 9090, keeping Nginx container port 80:

```bash
sudo docker stop nginx-a
sudo docker rm nginx-a
sudo docker run -d --name nginx-a -p 9090:80 nginx
```

This replaces the container with one using the new host port. Data stored only in the old container's writable layer may be lost; use volumes/bind mounts for persistent data.

## Change a Compose port

Before:

```yaml
services:
  web:
    image: nginx:stable
    ports:
      - "8080:80"
```

After, to use host port 9090:

```yaml
services:
  web:
    image: nginx:stable
    ports:
      - "9090:80"
```

Apply and verify:

```bash
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
```

For EC2, the security group and host firewall must also permit the new host port. Check port conflicts:

```bash
sudo ss -ltnp
sudo lsof -i :8080
```

The first lists host TCP listeners; the second (if `lsof` is installed) identifies processes using port 8080. If port 80 is used by host Nginx/Apache, check what it serves before stopping it.

---

# 7. Dockerfiles and rebuilding images

## Example Dockerfile

```dockerfile
FROM nginx:stable
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

`FROM` selects a base image; `COPY` adds a file from the build context; `EXPOSE` documents the expected container port but does not publish it.

## Build, run, rebuild, replace

```bash
sudo docker build -t my-site:1.0 .
sudo docker run -d --name my-site -p 8080:80 my-site:1.0
sudo docker build -t my-site:1.1 .
sudo docker stop my-site
sudo docker rm my-site
sudo docker run -d --name my-site -p 8080:80 my-site:1.1
```

Builds version 1.0, starts it, builds a new version after source changes, then replaces the container to run version 1.1. Building an image does not update already-running containers automatically.

If reusing the same tag:

```bash
sudo docker build -t my-site:1.0 .
sudo docker rm -f my-site
sudo docker run -d --name my-site -p 8080:80 my-site:1.0
```

Rebuilds the tag and recreates the container. Versioned tags are usually easier to audit and roll back.

## Rebuild using Compose

```bash
sudo docker compose build
sudo docker compose up -d
sudo docker compose up -d --build
sudo docker compose build --no-cache app
sudo docker compose up -d --no-deps app
sudo docker compose up -d --force-recreate
sudo docker compose up -d --build --force-recreate
```

- Builds services with a `build:` section.
- Starts/recreates services using the built images.
- Builds then starts in one step.
- Rebuilds service `app` without cache.
- Recreates `app` without starting dependencies; only use if dependencies already run and are compatible.
- Forces recreation even if Compose sees no configuration change.
- Builds and forces recreation.

Check whether the container uses the expected image:

```bash
sudo docker image ls
sudo docker inspect CONTAINER_NAME --format='{{.Image}}'
sudo docker logs --tail 100 CONTAINER_NAME
```

Shows local tags, the image ID used by the container, and recent logs. If old code appears, verify build context, `.dockerignore`, source copying, bind mounts, image tag, recreation, port, and browser/application cache.

---

# 8. Networks

```bash
sudo docker network ls
sudo docker network inspect bridge
sudo docker network create app-network
sudo docker run -d --name api --network app-network my-api:1.0
sudo docker run -d --name frontend --network app-network -p 8080:80 nginx
sudo docker network connect app-network CONTAINER_NAME
sudo docker network disconnect app-network CONTAINER_NAME
sudo docker network rm app-network
```

Lists networks, inspects the default bridge, creates a user-defined bridge, starts two containers on it, attaches/detaches a container, or removes a network if no containers remain attached.

Containers on a user-defined bridge can usually reach one another by container name. If an API listens on container port 3000, another container can use `http://api:3000`. `localhost` inside a container refers to that same container, not another container or the host.

In Compose, services normally share a project network and can reach one another by service name. A backend connecting to a Compose database usually uses `db:3306`, not `localhost:3306`.

---

# 9. Volumes and bind mounts

Containers are replaceable; data in the writable container layer may be lost when the container is removed. Use named volumes or bind mounts for persistent data.

## Named volumes

```bash
sudo docker volume ls
sudo docker volume create app-data
sudo docker volume inspect app-data
sudo docker run -d --name database -v app-data:/var/lib/mysql mysql:8.0
sudo docker volume rm app-data
sudo docker volume prune
```

Lists volumes, creates one, inspects its metadata, mounts it at MySQL's data directory, removes one unused volume, or prunes unused volumes. A real MySQL deployment needs valid environment configuration and secure credentials. Removing a volume can permanently delete stored data; inspect and back up first.

## Bind mounts

```bash
sudo docker run -d --name dev-web -p 8080:80 -v "$PWD/html:/usr/share/nginx/html:ro" nginx
sudo docker run -it --rm -v "$PWD:/workspace" -w /workspace ubuntu:24.04 bash
```

The first maps a host `html` directory read-only into Nginx; the second maps the current directory into a temporary Ubuntu shell. Bind mounts are useful for development, but a mount can hide files that were present in the image at the target path.

## Compose volume example

```yaml
services:
  db:
    image: mysql:8.0
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

Declares a named volume for MySQL data. `docker compose down` normally preserves named volumes, but `docker compose down -v` can delete the declared volumes. Back up databases before any storage maintenance.

---

# 10. Docker Compose

Compose defines services in `compose.yaml` (or another supported Compose filename).

## Minimal example

```yaml
services:
  web:
    image: nginx:stable
    ports:
      - "8080:80"
    restart: unless-stopped
```

Defines Nginx, publishes host 8080 to container 80, and requests automatic restart unless explicitly stopped.

## Validate, start, inspect

```bash
sudo docker compose config
sudo docker compose up -d
sudo docker compose up
sudo docker compose up -d web
sudo docker compose ps
sudo docker compose ps -a
sudo docker compose logs
sudo docker compose logs -f
sudo docker compose logs --tail 100
sudo docker compose logs -f web
sudo docker compose top
sudo docker compose images
```

Validates and renders configuration; starts all services detached; starts attached to the terminal; starts a named service and dependencies; lists running/all project containers; displays/follows/limits logs; shows processes; and lists service images. Rendered config/logs may expose secrets, so review before sharing.

## Execute commands

```bash
sudo docker compose exec web sh
sudo docker compose exec app php artisan about
sudo docker compose run --rm app php artisan migrate:status
```

Opens a shell in a running service, runs a Laravel Artisan command inside the `app` service, or creates a one-off container to run a command and removes that one-off container after exit.

## Lifecycle commands

```bash
sudo docker compose stop
sudo docker compose start
sudo docker compose restart
sudo docker compose restart app
sudo docker compose down
sudo docker compose down --remove-orphans
```

Stops project containers, starts previously created containers, restarts all or one service, removes project containers and networks, or additionally removes orphaned project containers. Named volumes are normally retained by `down` without `-v`.

**Destructive to persistent data—use only when intentionally deleting it:**

```bash
sudo docker compose down -v
```

Removes declared named volumes and attached anonymous volumes. This can delete database data.

## Compose file and project options

```bash
sudo docker compose -f compose.yaml -f compose.prod.yaml config
sudo docker compose -f compose.yaml -f compose.prod.yaml up -d
sudo docker compose -p myproject up -d
sudo docker compose -p myproject ps
sudo docker compose --env-file .env.production up -d
```

Combines a base and override file; sets an explicit project name to isolate environments; or loads a selected environment file for Compose interpolation.

---

# 11. Updating an existing Compose application

```bash
sudo docker compose build app
sudo docker compose up -d app
sudo docker compose build
sudo docker compose up -d
sudo docker compose pull
sudo docker compose up -d
sudo docker compose up -d --force-recreate
```

- Rebuilds `app`, then applies the new image.
- Builds all services with a build context, then starts/recreates services.
- Pulls updated registry images, then recreates services if needed.
- Forces container recreation without necessarily rebuilding images.

Changing a port: edit the service's `ports:` in `compose.yaml`, then run `sudo docker compose config`, `sudo docker compose up -d`, and `sudo docker compose ps`.

Remember: `restart` does not rebuild an image; `pull` downloads an image; `build` creates an image from a Dockerfile. A rebuilt image does not replace existing containers until you recreate/update them.

---

# 12. Environment variables and secrets

## Pass variables to a container

```bash
sudo docker run -d --name app -e APP_ENV=production myapp:1.0
sudo docker run --env-file .env -d --name app myapp:1.0
```

Sets one environment variable or loads variables from a file. Do not commit a private `.env` file.

Compose example:

```yaml
services:
  app:
    image: myapp:1.0
    env_file:
      - .env
    environment:
      APP_ENV: production
```

`env_file` passes values into the container; `environment` sets or overrides named values. Compose interpolation such as `${APP_PORT}` in YAML is a separate function.

Example `.env`:

```dotenv
APP_PORT=8080
DB_DATABASE=appdb
DB_USERNAME=appuser
DB_PASSWORD=replace_with_a_strong_password
```

Example `.gitignore` entries:

```gitignore
.env
.env.*
!.env.example
```

This ignores private environment files while allowing a safe example template. Confirm your project's actual ignore rules.

Inspect environment (may expose credentials):

```bash
sudo docker exec CONTAINER_NAME printenv
sudo docker inspect CONTAINER_NAME
```

Do not paste secret-bearing output into public issues or screenshots. Avoid baking secrets into Dockerfile `ARG`/`ENV`. Use an appropriate secret manager for production. Rotate any credential that has been exposed; removing it from the latest commit does not erase Git history.

---

# 13. Docker Hub and registries

## Authenticate

```bash
sudo docker login
sudo docker login docker.io
sudo docker logout
```

Logs in to the default registry, logs in to Docker Hub explicitly, or removes saved credentials. Prefer a personal access token where supported; do not put passwords in command-line arguments or shell history.

## Build, tag, and push

Example username: `sami1800`.

```bash
sudo docker build -t sami1800/laravel-welcome:1.0 .
sudo docker push sami1800/laravel-welcome:1.0
sudo docker build -t sami1800/laravel-welcome:1.1 .
sudo docker push sami1800/laravel-welcome:1.1
```

Builds and pushes two versioned image tags. The repository must exist or your account must have permission to create/push it.

## Pull and run elsewhere

```bash
sudo docker pull sami1800/laravel-welcome:1.0
sudo docker run -d --name laravel-welcome -p 8000:8000 sami1800/laravel-welcome:1.0
```

Downloads and starts the image. The port mapping is correct only if the application listens on container port 8000. A Laravel PHP-FPM container may instead listen on port 9000 behind an Nginx service on port 80; follow the project's actual architecture.

---

# 14. Build cache and build troubleshooting

## `.dockerignore` example

```text
.git
.env
.env.*
node_modules
vendor
build
dist
coverage
*.log
```

Excludes files from the build context. Only exclude dependency directories such as `vendor` or `node_modules` if your Dockerfile installs/builds those dependencies itself.

## Build cache commands

```bash
sudo docker buildx ls
sudo docker buildx inspect
sudo docker builder du
sudo docker builder prune
sudo docker builder prune -a
```

Lists builders, inspects the selected builder, shows build-cache usage, or removes eligible/unused build cache. The `-a` option is more aggressive. Pruning can make future builds slower; review disk use first.

## Common build errors

**`COPY failed` / file not found**
- Confirm the file exists beneath the build context.
- Check source paths relative to the build context.
- Check `.dockerignore`.
- Run the command from the correct project directory.

**Dependency installation fails**
- Read the first meaningful error above the final failure line.
- Check network, package repositories, versions and lockfiles.
- Use `--progress=plain` for detailed output.

**Disk full**
```bash
df -h
sudo docker system df
sudo docker builder du
```

Checks host disk, Docker objects, and build cache. Select only known-unneeded cleanup targets; do not delete volumes as routine cleanup.

**Architecture mismatch**
```bash
uname -m
sudo docker image inspect IMAGE_NAME:TAG
```

Checks host architecture and image metadata. For a different target architecture, use a correctly configured Buildx build and choose the intended platform.

---

# 15. Resource limits, restart policies, and health checks

## Resource limits

```bash
sudo docker run -d --name limited-app --memory=512m --cpus=1 myapp:1.0
sudo docker update --memory=768m --cpus=1.5 limited-app
```

Limits a container to 512 MiB and one CPU's worth of processing, or updates supported limits on an existing container. Choose limits based on measured application requirements.

## Restart policy

```bash
sudo docker run -d --restart unless-stopped --name web -p 8080:80 nginx
```

Configures restart behavior. Common policies: `no` (default), `on-failure`, `always`, and `unless-stopped`. Exact restart behavior depends on exit status and whether an operator explicitly stopped the container.

## Health check example

```dockerfile
FROM nginx:stable
HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD curl -f http://localhost/ || exit 1
```

Defines a periodic HTTP health check, assuming `curl` exists in the image. A health check reports status; it does not repair a failed app automatically.

```bash
sudo docker ps
sudo docker inspect --format='{{json .State.Health}}' CONTAINER_NAME
```

Shows container status and health details if a health check is configured.

---

# 16. Troubleshooting

## Container exits immediately

```bash
sudo docker ps -a
sudo docker logs CONTAINER_NAME
sudo docker inspect CONTAINER_NAME
```

Finds the exit status, prints application output, and inspects configuration. A container normally remains alive only while its main process is running.

## App is not reachable

```bash
sudo docker ps
sudo docker port CONTAINER_NAME
sudo docker logs --tail 100 CONTAINER_NAME
sudo ss -ltnp
```

Checks container status, published mappings, recent logs, and host listeners. Confirm the app listens on the correct interface and container port.

## Container cannot reach another container

```bash
sudo docker network ls
sudo docker network inspect NETWORK_NAME
sudo docker logs --tail 100 CONTAINER_NAME
```

Check shared network membership, service readiness, internal port, service name, credentials, and database configuration. Do not use `localhost` to refer to a separate container.

## Permission problem with a bind mount

```bash
ls -ln HOST_DIRECTORY
sudo docker inspect CONTAINER_NAME
```

Checks numeric host ownership and container mounts. The process may run as a non-root user and lack write permission. Fix ownership deliberately; avoid making sensitive directories world-writable.

## Restart loop

```bash
sudo docker ps -a
sudo docker logs --tail 200 CONTAINER_NAME
sudo docker inspect --format='{{.State.ExitCode}} {{.State.Error}}' CONTAINER_NAME
```

Shows repeated exits, recent logs, and last exit status/error. Fix the root cause rather than repeatedly restarting.

## Docker daemon unavailable

```bash
sudo systemctl status docker
sudo journalctl -u docker --since "30 minutes ago"
```

Checks service state and recent daemon logs. Investigate before restarting Docker on a production machine.

## Port already allocated

```bash
sudo docker ps
sudo ss -ltnp
```

Finds containers or host processes using a port. Stop only the intended conflicting service or select a different host port, for example `8081:80`.

---

# 17. EC2 and public access

Check local response on the EC2 host:

```bash
sudo docker ps
sudo docker port CONTAINER_NAME
curl -I http://127.0.0.1:HOST_PORT
```

Checks container status, published mappings, and an HTTP response from the EC2 machine. Replace `HOST_PORT` with the published host port.

For a browser on another machine to reach the app:
1. The application must listen on the expected container port.
2. Docker must publish the port on a suitable host interface.
3. The EC2 security group must allow inbound traffic to the **host port** from the intended source.
4. The host firewall must allow it, if enabled.
5. The application or reverse proxy must be configured correctly.

Example:

```bash
sudo docker run -d --name public-web -p 8080:80 nginx
```

A remote browser uses `http://EC2_PUBLIC_IP:8080`, while Nginx listens on container port 80. The security group must allow inbound TCP 8080 for this example.

```bash
sudo ufw status
sudo ss -ltnp | grep ':80'
```

Shows UFW rules and helps identify a listener on port 80. Do not change firewall rules without preserving SSH access. Do not stop host Nginx/Apache blindly if it serves another application.

Security reminders:
- Avoid exposing database ports publicly.
- Prefer a reverse proxy with TLS for public websites.
- Restrict SSH to trusted source IPs where practical.
- Keep secrets out of commands, Git repositories and screenshots.
- Use maintained image versions and back up persistent data.

---

# 18. Cleanup: safe versus destructive

Inspect first:

```bash
sudo docker system df
sudo docker ps -a
sudo docker images
sudo docker volume ls
```

Shows disk usage, all containers, images, and volumes before cleanup.

Remove a specific stopped container or image:

```bash
sudo docker rm CONTAINER_NAME
sudo docker image rm IMAGE_NAME:TAG
```

Removes a selected stopped container or selected image tag. Verify the target first.

Prune stopped containers, unused networks, dangling images, or build cache:

```bash
sudo docker container prune
sudo docker network prune
sudo docker image prune
sudo docker builder prune
```

Each removes a class of unused resources after confirmation. Inspect resources first; stopped containers can be useful for debugging.

**Use these only after understanding the impact and confirming backups:**

```bash
sudo docker system prune -a
sudo docker volume prune
sudo docker compose down -v
```

These are more destructive. Broad system pruning removes many unused resources; volume pruning can delete data from unused volumes; `compose down -v` can delete a project's named volumes, including database storage. Never run these as generic first-line troubleshooting commands.

---

# 19. Common end-to-end recipes

## Recipe A: Run Nginx on host port 8080

```bash
sudo docker pull nginx:stable
sudo docker run -d --name nginx-lab -p 8080:80 --restart unless-stopped nginx:stable
sudo docker ps
sudo docker logs --tail 50 nginx-lab
```

Downloads Nginx, starts it in the background with host 8080 mapped to container 80, then checks status and logs. Visit `http://localhost:8080` on the host.

## Recipe B: Build, run, update, replace

```bash
sudo docker build -t sample-app:1.0 .
sudo docker run -d --name sample-app -p 8080:80 sample-app:1.0
sudo docker build -t sample-app:1.1 .
sudo docker stop sample-app
sudo docker rm sample-app
sudo docker run -d --name sample-app -p 8080:80 sample-app:1.1
```

Builds and runs version 1.0, builds version 1.1, then replaces the container. Assumes the application listens on container port 80.

## Recipe C: Start a Compose web service

Create `compose.yaml`:

```yaml
services:
  web:
    image: nginx:stable
    ports:
      - "8080:80"
    restart: unless-stopped
```

Then:

```bash
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs -f web
```

Validates the YAML, starts the service detached, checks status, and follows logs.

## Recipe D: Change Compose port 8080 to 9090

Edit `compose.yaml` from `"8080:80"` to `"9090:80"`, then:

```bash
sudo docker compose config
sudo docker compose up -d
sudo docker compose ps
```

Applies and verifies the new host mapping. On EC2, update the security group and firewall if necessary.

## Recipe E: Push an image and run it on another machine

```bash
sudo docker login
sudo docker build -t YOUR_USERNAME/sample-app:1.0 .
sudo docker push YOUR_USERNAME/sample-app:1.0
```

Authenticates, builds a registry-tagged image, and pushes it. On the destination machine:

```bash
sudo docker pull YOUR_USERNAME/sample-app:1.0
sudo docker run -d --name sample-app -p 8080:80 YOUR_USERNAME/sample-app:1.0
```

Downloads and runs the image. Replace username, repository, tag, and port mapping as required.

## Recipe F: Investigate a database container without deleting data

```bash
sudo docker ps -a
sudo docker logs --tail 100 DB_CONTAINER_NAME
sudo docker inspect DB_CONTAINER_NAME
sudo docker volume ls
```

Checks database state, logs, mounts/configuration, and volumes. Diagnose and back up first; do not remove volumes as a first step.

## Recipe G: Investigate disk usage

```bash
df -h
sudo docker system df
sudo docker system df -v
sudo docker builder du
```

Checks host disk, Docker object usage, and build cache. Use the results to choose a specific cleanup target.

---

# 20. Quick command cheat sheet

| Task | Command |
|---|---|
| Version | `docker --version` |
| Engine information | `sudo docker info` |
| Running containers | `sudo docker ps` |
| All containers | `sudo docker ps -a` |
| Images | `sudo docker images` |
| Pull image | `sudo docker pull IMAGE:TAG` |
| Build image | `sudo docker build -t IMAGE:TAG .` |
| Run detached | `sudo docker run -d --name NAME IMAGE:TAG` |
| Publish port | `sudo docker run -d -p HOST:CONTAINER IMAGE:TAG` |
| Logs | `sudo docker logs NAME` |
| Follow logs | `sudo docker logs -f NAME` |
| Shell | `sudo docker exec -it NAME sh` |
| Stop/start/restart | `sudo docker stop NAME` / `sudo docker start NAME` / `sudo docker restart NAME` |
| Remove stopped container | `sudo docker rm NAME` |
| Inspect container | `sudo docker inspect NAME` |
| List networks | `sudo docker network ls` |
| List volumes | `sudo docker volume ls` |
| Validate Compose | `sudo docker compose config` |
| Start Compose | `sudo docker compose up -d` |
| Build and start | `sudo docker compose up -d --build` |
| Compose status | `sudo docker compose ps` |
| Compose logs | `sudo docker compose logs -f` |
| Stop Compose | `sudo docker compose stop` |
| Restart Compose | `sudo docker compose restart` |
| Remove Compose containers/network | `sudo docker compose down` |
| Docker disk usage | `sudo docker system df` |
| Host disk usage | `df -h` |

---

# 21. Key concepts and common mistakes

1. **Image versus container:** Image is the template; container is an instance.
2. **Build versus redeploy:** Rebuilding does not update running containers; recreate them to use the new image.
3. **Port order:** `-p 8080:80` means host 8080 forwards to container 80.
4. **`EXPOSE` does not publish:** Use `-p` or Compose `ports:`.
5. **`localhost` is relative:** Inside a container, it refers to that container.
6. **Compose service names:** A backend normally connects to database service `db:3306`, not `localhost:3306`.
7. **Persistence:** Container removal and volume deletion are different actions.
8. **`down` vs `down -v`:** The former normally preserves named volumes; the latter can delete them.
9. **Restart does not rebuild:** Use `build` and `up -d` after image changes.
10. **Pull versus build:** Pull downloads a registry image; build creates an image from a Dockerfile.
11. **Port conflicts:** Another host process/container may already be using the requested port.
12. **Environment files:** Compose interpolation and variables passed into a container are related but distinct; configure `environment:` or `env_file:` deliberately.
13. **Docker group permissions:** Membership can grant root-equivalent control of the host. This handbook uses `sudo docker`.
14. **Production:** Pin suitable image versions, protect secrets, back up persistent data, monitor logs/health, and configure firewalls deliberately.

## Official documentation

- Docker Engine: https://docs.docker.com/engine/
- Docker CLI reference: https://docs.docker.com/reference/cli/docker/
- Docker Compose: https://docs.docker.com/compose/
- Dockerfile reference: https://docs.docker.com/reference/dockerfile/
- Docker networking: https://docs.docker.com/engine/network/
- Docker volumes: https://docs.docker.com/engine/storage/volumes/
- Docker Hub: https://docs.docker.com/docker-hub/
- Docker installation on Ubuntu: https://docs.docker.com/engine/install/ubuntu/

---

**End of Docker Commands Handbook**
