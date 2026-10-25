# Updated Laravel Three-Tier Docker Deployment — Ubuntu 26.04 EC2

This guide is updated for the actual EC2 instance shown in the terminal:

- OS: Ubuntu 26.04.1 LTS (Resolute Raccoon)
- User: `ubuntu`
- Docker was **not installed** when checked.
- Target architecture: Nginx web tier, Laravel/PHP-FPM application tier, and MySQL database tier.
- Public access: HTTP on port 80 only. Do not expose MySQL 3306 or PHP-FPM 9000 publicly.

The steps use Docker's official APT repository, which currently lists Ubuntu 26.04 as supported. They include checks after important stages. No guide can guarantee zero errors on every AWS account or repository, so stop at the first failed command and use the exact error to diagnose it.

---

## Part 0 — Confirm the EC2 instance

Run on the EC2 terminal:

```bash
cat /etc/os-release
uname -m
whoami
df -h /
```

Expected user: `ubuntu`. `uname -m` will commonly show `x86_64` or `aarch64`. Keep at least a few GB of free disk space for Docker images and builds. If the root disk is nearly full, increase the EBS volume before building images.

## Part 1 — Install Docker Engine and Compose plugin

These commands are for the EC2 instance. They use Docker's official repository rather than relying on the Ubuntu `docker.io` package.

### 1.1 Install prerequisites

```bash
sudo apt update
sudo apt install -y ca-certificates curl git
```

### 1.2 Add Docker's official signing key

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Check that the key file exists:

```bash
test -s /etc/apt/keyrings/docker.asc && echo "Docker signing key downloaded"
```

### 1.3 Add the repository for the detected Ubuntu release and architecture

```bash
sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Update package metadata:

```bash
sudo apt update
```

If this fails with a repository or release error, stop and copy the complete error. Do not substitute a different Ubuntu codename manually.

### 1.4 Install Docker and Compose

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Enable and start Docker:

```bash
sudo systemctl enable --now docker
```

All Docker Engine and Docker Compose commands in this guide use `sudo docker` so you do not need to create or join a Docker group. Verify the installation with:

```bash
sudo docker --version
sudo docker compose version
sudo systemctl is-active docker
sudo docker run --rm hello-world
sudo docker ps
```

Expected: Docker and Compose print version information, `is-active` prints `active`, `hello-world` prints a success message, and `sudo docker ps` runs successfully. Continue to use `sudo docker ...` for every Docker command below. The `sudo` prefix is only for Docker commands; ordinary project commands such as `nano`, `git`, and `openssl` do not need it.

## Part 2 — Prepare the Laravel repository on your development computer

Do these steps on the computer where you edit and push the GitHub project, **not in the EC2 terminal**.

### 2.1 Clone or open the project

Repository:

```text
https://github.com/samikhan5055/laravel-project.git
```

If you have not cloned it:

```bash
git clone https://github.com/samikhan5055/laravel-project.git
cd laravel-project
```

If the repository already exists locally, enter its directory instead. Confirm the Laravel project root:

```bash
test -f artisan && test -f composer.json && echo "Laravel project root confirmed"
ls
```

Check whether the project uses Vite:

```bash
test -f package.json && echo "package.json exists"
test -f package-lock.json && echo "package-lock.json exists"
```

The Dockerfile below expects a normal Laravel project with `package.json` and an npm `build` script. If either is missing, do not continue unchanged; inspect `package.json` and adjust the asset-build stage to the project actually present.

### 2.2 Create the Nginx configuration directory

```bash
mkdir -p docker/nginx
```

From the Laravel project root, create/edit `docker/nginx/default.conf`. Run these commands first, then paste the Nginx content shown below into nano and save it:

```bash
cd ~/laravel-project
mkdir -p docker/nginx
nano docker/nginx/default.conf
```

Paste this content into the file:

```nginx
server {
    listen 80;
    server_name _;
    root /var/www/html/public;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri =404;
        include fastcgi_params;
        fastcgi_pass app:9000;
        fastcgi_param SCRIPT_FILENAME /var/www/html/public$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT /var/www/html/public;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### 2.3 Create the Dockerfile

From the Laravel project root, create/edit the `Dockerfile` using nano:

```bash
cd ~/laravel-project
nano Dockerfile
```

Paste this content into the file:

```dockerfile
# Build frontend assets.
FROM node:22-alpine AS assets
WORKDIR /src
COPY package*.json ./
RUN if [ -f package-lock.json ]; then npm ci; else npm install; fi
COPY . .
RUN npm run build

# Build/run Laravel with PHP-FPM.
FROM php:8.4-fpm AS app

RUN apt-get update && apt-get install -y --no-install-recommends \
    git unzip libzip-dev libicu-dev libonig-dev \
    && docker-php-ext-install -j"$(nproc)" pdo_mysql mbstring bcmath pcntl zip intl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer

WORKDIR /var/www/html
COPY . .

# Install locked production dependencies when composer.lock is available.
RUN if [ -f composer.lock ]; then \
        composer install --no-dev --prefer-dist --no-interaction --optimize-autoloader; \
    else \
        composer update --no-dev --prefer-dist --no-interaction --optimize-autoloader; \
    fi

# Copy the compiled Vite assets from the frontend build.
COPY --from=assets /src/public/build/ /var/www/html/public/build/

RUN mkdir -p storage/framework/cache storage/framework/sessions \
    storage/framework/views storage/logs bootstrap/cache \
    && chown -R www-data:www-data storage bootstrap/cache

EXPOSE 9000
CMD ["php-fpm"]

# Nginx image with Laravel's public directory and custom configuration.
FROM nginx:alpine AS web
COPY docker/nginx/default.conf /etc/nginx/conf.d/default.conf
COPY --from=app /var/www/html/public/ /var/www/html/public/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Important: `composer update` is only a fallback when the repository has no `composer.lock`, because it resolves dependency versions rather than installing a locked set. For reproducible builds, commit a valid `composer.lock` and use `composer install`.

### 2.4 Create `.dockerignore`

From the Laravel project root, create/edit `.dockerignore` using nano:

```bash
cd ~/laravel-project
nano .dockerignore
```

Paste this content into the file:

```text
.git
.github
.env
.env.*
!.env.example
vendor
node_modules
storage/logs/*
storage/framework/cache/*
storage/framework/sessions/*
storage/framework/views/*
```

This prevents local secrets and unnecessary directories from being copied into Docker build contexts.

### 2.5 Create `compose.yaml`

From the Laravel project root, create/edit `compose.yaml` using nano:

```bash
cd ~/laravel-project
nano compose.yaml
```

Paste this content into the file:

```yaml
services:
  db:
    image: mysql:8.0
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${DB_DATABASE:?Set DB_DATABASE in .env}
      MYSQL_USER: ${DB_USERNAME:?Set DB_USERNAME in .env}
      MYSQL_PASSWORD: ${DB_PASSWORD:?Set DB_PASSWORD in .env}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:?Set MYSQL_ROOT_PASSWORD in .env}
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p\"$$MYSQL_ROOT_PASSWORD\" --silent"]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 30s

  app:
    image: ${DOCKERHUB_USERNAME:?Set DOCKERHUB_USERNAME in .env}/laravel-app:1.0
    build:
      context: .
      dockerfile: Dockerfile
      target: app
    restart: unless-stopped
    environment:
      APP_NAME: Laravel
      APP_ENV: production
      APP_DEBUG: "false"
      APP_URL: ${APP_URL:-http://localhost}
      APP_KEY: ${APP_KEY:?Set APP_KEY in .env}
      LOG_CHANNEL: stderr
      DB_CONNECTION: mysql
      DB_HOST: db
      DB_PORT: 3306
      DB_DATABASE: ${DB_DATABASE}
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
      SESSION_DRIVER: file
      CACHE_STORE: file
      QUEUE_CONNECTION: sync
    volumes:
      - laravel_storage:/var/www/html/storage
    networks:
      - frontend
      - backend
    depends_on:
      db:
        condition: service_healthy

  web:
    image: ${DOCKERHUB_USERNAME:?Set DOCKERHUB_USERNAME in .env}/laravel-web:1.0
    build:
      context: .
      dockerfile: Dockerfile
      target: web
    restart: unless-stopped
    ports:
      - "80:80"
    networks:
      - frontend
    depends_on:
      - app

networks:
  frontend:
  backend:
    internal: true

volumes:
  mysql_data:
  laravel_storage:
```

The database is only on the internal backend network. Nginx is the only service publishing a host port. The app joins both networks so it can receive FastCGI requests from Nginx and connect to MySQL.

### 2.6 Create the local `.env` used by Docker Compose

Do not commit this file to Git. The Docker Hub username used throughout this guide is `sami1800`. For this lab, use the password `sami12345` consistently for both the Laravel database user and MySQL root account in the local and EC2 `.env` files.

```bash
cd ~/laravel-project
nano .env
```

Enter these lines in nano, then save with `Ctrl+O`, press Enter, and exit with `Ctrl+X`:

```dotenv
DOCKERHUB_USERNAME=sami1800
APP_URL=http://localhost
APP_KEY=base64:PASTE_YOUR_REAL_LARAVEL_KEY_HERE
DB_DATABASE=laravel
DB_USERNAME=laraveluser
DB_PASSWORD=sami12345
MYSQL_ROOT_PASSWORD=sami12345
```

Keep `.env` private and do not commit it to GitHub. The database user password and MySQL root password are both `sami12345` in this lab guide. Keep the same values in local and EC2 `.env` files. For a real production deployment, use separate strong secrets instead.

Validate Compose syntax before building:

```bash
sudo docker compose config --quiet
```

Do not run validation while `APP_KEY` still contains the example placeholder. Generate a real key in the next step, replace the placeholder in `.env`, and then run `sudo docker compose config --quiet`. If any variable is missing, fix `.env` first.

### 2.7 Generate the Laravel application key

The `APP_KEY` in `.env` must be a real key, not the example placeholder. From the Laravel project root, use **one** of these methods.

**Method A — PHP and Laravel are installed locally:**

```bash
cd ~/laravel-project
php artisan key:generate --show
```

Copy the complete output (it begins with `base64:`) and replace the example `APP_KEY` value in `.env` using:

```bash
cd ~/laravel-project
nano .env
```

**Method B — Laravel/PHP is not installed locally:**

Generate a Laravel-compatible key using OpenSSL:

```bash
openssl rand -base64 32
```

Copy its output and set `APP_KEY=base64:PASTE_OUTPUT_HERE` in `.env` with nano. The `base64:` prefix is required; without it, Laravel treats the 44-character OpenSSL output as a raw key and can return HTTP 500 with “Unsupported cipher or incorrect key length.” For example, if OpenSSL prints `ABC...`, put the full output after `base64:`. Do not leave the example placeholder in place. Keep this key private and use the same key on EC2 if you want the same application encryption/session continuity.

After updating `.env`, verify the Compose configuration:

```bash
sudo docker compose config --quiet
```

### 2.8 Build and test the images locally

Before building, confirm the required files exist:

```bash
test -f Dockerfile
test -f compose.yaml
test -f docker/nginx/default.conf
test -f artisan
test -f composer.json
test -f package.json
```

Build:

```bash
sudo docker compose build app web
```

Check the images:

```bash
sudo docker image ls
```

If the build fails, inspect the **first** failing build step and its error. Do not keep running later deployment steps until the build passes.

### 2.9 Push images to Docker Hub

Log in interactively; do not put your Docker Hub password in a command or file:

```bash
sudo docker login
```

Push both images:

```bash
sudo docker compose push app web
```

If Docker Compose says the image does not exist or was not built, rebuild first. Confirm both repositories exist in Docker Hub under your username.

### 2.10 Commit deployment files to GitHub

Check the changes and ensure `.env` is not staged:

```bash
git status --short
git check-ignore .env
```

If `.env` appears as an untracked or staged file, do not commit it. Then commit the Docker deployment files:

```bash
git add Dockerfile .dockerignore compose.yaml docker/nginx/default.conf
git commit -m "Add three-tier Docker deployment"
git push origin main
```

If your branch is not `main`, use the actual branch shown by `git branch --show-current`.

---

## Part 3 — Configure AWS Security Group

In the AWS EC2 console, open the security group attached to this instance and configure inbound rules:

| Type | Port | Source |
|---|---:|---|
| SSH | 22 | My IP |
| HTTP | 80 | `0.0.0.0/0` |
| HTTP IPv6 (only if using IPv6) | 80 | `::/0` |

Do **not** add public inbound rules for ports 3306 or 9000. The database and PHP-FPM should not be exposed directly.

Make sure the instance has an outbound route to the internet so it can pull Docker Hub images. If using a private Docker Hub repository, the EC2 instance must log in to Docker Hub.

---

## Part 4 — Deploy on the EC2 instance

Run the following on the EC2 terminal.

### 4.1 Clone or update the repository

If the project directory does not exist:

```bash
git clone https://github.com/samikhan5055/laravel-project.git
cd laravel-project
```

If it already exists:

```bash
cd ~/laravel-project
git pull
```

Confirm deployment files:

```bash
test -f compose.yaml && test -f Dockerfile && test -f docker/nginx/default.conf
```

### 4.2 Create the EC2 `.env`

Create a production `.env` file with the **same** Docker Hub username, database name, database username/password, and root password used when the images were built. Generate a new production `APP_KEY` if the local key is not the one you intend to use in production. Do not paste secrets into chat.

```bash
cd ~/laravel-project
nano .env
```

Enter these values. Replace `YOUR_EC2_PUBLIC_IPV4` with the instance public IPv4 address and paste your real Laravel APP_KEY:

```dotenv
DOCKERHUB_USERNAME=sami1800
APP_URL=http://YOUR_EC2_PUBLIC_IPV4
APP_KEY=base64:PASTE_YOUR_REAL_LARAVEL_KEY_HERE
DB_DATABASE=laravel
DB_USERNAME=laraveluser
DB_PASSWORD=sami12345
MYSQL_ROOT_PASSWORD=sami12345
```

Save in nano: `Ctrl+O`, Enter, then `Ctrl+X`.

Protect the file:

```bash
chmod 600 .env
```

Do not create or use an `.env` committed to GitHub. Compose reads this file for variable substitution.

### 4.3 Log in to Docker Hub

```bash
sudo docker login -u sami1800
```

Enter your Docker Hub password or access token at the prompt. Never include it directly in the command.

### 4.4 Validate Compose and pull the images

```bash
sudo docker compose config --quiet
sudo docker compose pull
```

If pull reports `denied`, `unauthorized`, or `manifest unknown`, verify the Docker Hub username, repository names, tag `1.0`, and login. Do not proceed until both app and web images pull successfully.

### 4.5 Start the three services

```bash
sudo docker compose up -d --no-build
```

Check status:

```bash
sudo docker compose ps
```

Expected services: `db`, `app`, and `web`. Wait for MySQL to become healthy:

```bash
sudo docker compose ps
sudo docker compose logs --tail=100 db
```

If the database is not healthy, inspect its logs before proceeding.

### 4.6 Run Laravel migrations

Only after `db` is healthy and the app container is running:

```bash
sudo docker compose exec app php artisan config:clear
sudo docker compose exec app php artisan migrate --force
```

If migrations fail, check `DB_HOST=db`, DB credentials, and the database logs. Do not use `localhost` as `DB_HOST` inside the Laravel container; the service name `db` is the correct Docker-network hostname.

### 4.7 Verify the app locally on EC2

```bash
curl -I http://127.0.0.1/
sudo docker compose ps
sudo docker compose logs --tail=100 app web
```

Then open this in your computer's browser:

```text
http://YOUR_EC2_PUBLIC_IPV4
```

Use the instance's **current Public IPv4 address** from the EC2 console. Do not use its private `172.31.x.x` address in your external browser.

---

## Part 5 — Verify the three-tier architecture

### 5.1 Confirm all containers are running

```bash
sudo docker compose ps
```

### 5.2 Confirm Nginx can reach PHP-FPM

The Nginx configuration uses `fastcgi_pass app:9000`. Both `web` and `app` share the `frontend` network, so Nginx resolves the service name `app` through Docker DNS.

Inspect the Nginx configuration:

```bash
sudo docker compose exec web nginx -t
```

Expected: `syntax is ok` and `test is successful`.

### 5.3 Confirm the app can resolve MySQL

```bash
sudo docker compose exec app php -r 'echo gethostbyname("db"), PHP_EOL;'
```

This should print a container-network IP address for the `db` service.

### 5.4 Confirm only HTTP is published

```bash
sudo docker compose ps
```

Only `web` should show a host mapping like `0.0.0.0:80->80/tcp`. Neither `db` nor `app` should publish a host port.

---

## Part 6 — Verify volume persistence

First, list volumes:

```bash
sudo docker volume ls
```

Inspect the Compose volumes:

```bash
sudo docker volume inspect laravel-project_mysql_data
sudo docker volume inspect laravel-project_laravel_storage
```

Compose may choose a different prefix if the directory/project name differs. Use the exact names shown by `sudo docker volume ls`.

Check that the database volume is mounted:

```bash
sudo docker inspect "$(docker compose ps -q db)" --format '{{json .Mounts}}'
```

Restart services without deleting volumes:

```bash
sudo docker compose down
sudo docker compose up -d --no-build
sudo docker compose ps
```

The named volumes should remain. **Do not run `sudo docker compose down -v`** during a persistence test, because `-v` deletes the named volumes and can erase database data.

---

## Part 7 — Troubleshooting checks

Run these only when needed.

### Docker service

```bash
sudo systemctl status docker --no-pager
sudo journalctl -u docker --no-pager -n 100
```

### Container status and logs

```bash
sudo docker compose ps
sudo docker compose logs --tail=150
```

### Disk space

```bash
df -h
sudo docker system df
```

Do not prune volumes if they contain data you need.

### Nginx configuration

```bash
sudo docker compose exec web nginx -t
```

### Laravel logs

```bash
sudo docker compose exec app sh -lc 'tail -n 100 storage/logs/laravel.log 2>/dev/null || true'
```

### Database health/logs

```bash
sudo docker compose ps
sudo docker compose logs --tail=150 db
```

### Browser does not open the application

Check, in this order:

1. The EC2 instance is running and has a current Public IPv4 address.
2. The security group allows inbound TCP 80 from your IP or the intended audience.
3. `sudo docker compose ps` shows `web`, `app`, and `db` running.
4. `curl -I http://127.0.0.1/` works on EC2.
5. `sudo docker compose logs --tail=100 web app db` shows no startup errors.

---

## Important notes

- Run image builds and `sudo docker compose push` on the development computer. Run `sudo docker compose pull` and `sudo docker compose up -d --no-build` on EC2.
- If Docker Hub repositories are private, log in on EC2 before pulling.
- Do not expose MySQL 3306 or PHP-FPM 9000 publicly.
- Keep `.env`, `APP_KEY`, database passwords, and Docker Hub credentials private.
- Never delete named volumes during a persistence test.
- A project-specific dependency, missing `package.json`, missing npm `build` script, or incompatible Composer requirement may require adapting the Dockerfile. Use the first error output to identify that exact issue.
