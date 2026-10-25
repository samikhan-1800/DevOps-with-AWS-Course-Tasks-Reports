# Jenkins Freestyle and Pipeline CI/CD for Laravel Student Management System on AWS EC2

**Repository:** `https://github.com/samikhan5055/laravel-project.git`\
**Branch:** `main`\
**Operating system used in the session:** Ubuntu 26.04 LTS on AWS EC2\
**Application:** Laravel Student Management CRUD application\
**Production path:** `/var/www/student-management`\
**Jenkins Pipeline job:** `Laravel-Student-Pipeline`\
**Freestyle jobs:** `JOB1-Laravel-Build`, `JOB2-Laravel-Test`,
`JOB3-Laravel-Deploy`

> **Purpose:** Use this as a repeatable implementation runbook when
> setting up the task again without an AI assistant. Commands are
> grouped by where they run. Jenkins UI instructions specify what to
> select and what to paste. Replace placeholders with your own values.
> Never copy a production secret into GitHub or this document.

> **Status note:** The previous session confirmed that
> `curl -I http://127.0.0.1/students` returned `HTTP/1.1 200 OK`, Nginx
> listened on port 80, and the latest reported Pipeline test stage
> passed three tests. The Jenkinsfile push, full successful deployment
> stage, and external website access were not confirmed. This runbook is
> intended for a clean, repeatable setup and includes verification
> gates.

------------------------------------------------------------------------

## Contents

1.  Architecture and workflow
2.  AWS EC2 instance and Security Group
3.  Connect to EC2 and check the OS
4.  Install system packages
5.  Install and start Jenkins
6.  Clone the Laravel repository
7.  Prepare the production database and `.env`
8.  Configure the Laravel production directory and permissions
9.  Configure Nginx and PHP-FPM
10. Verify the website locally before Jenkins deployment
11. Create Jenkins Freestyle Build job
12. Create Jenkins Freestyle Test job
13. Create Jenkins Freestyle Deploy job
14. Create and push the `Jenkinsfile`
15. Create the Jenkins Pipeline job
16. Run and verify the complete Pipeline
17. Troubleshooting: errors and resolutions
18. Security checklist
19. Final verification checklist

------------------------------------------------------------------------

## 1. Architecture and workflow

The intended request path is:

``` text
Browser
  |
  | HTTP :80
  v
AWS Security Group
  |
  v
Nginx
  |
  v
PHP-FPM / Laravel
  |
  v
MySQL database
```

Jenkins runs on the same EC2 instance in this lab and listens on port
`8080`.

### 1.1 Freestyle CI/CD workflow

``` text
JOB1-Laravel-Build
        |
        | runs after success
        v
JOB2-Laravel-Test
        |
        | runs after success
        v
JOB3-Laravel-Deploy
```

### 1.2 Pipeline workflow

``` text
Checkout -> Build -> Test -> Deploy to EC2
```

The Freestyle jobs and the Pipeline job are two ways to demonstrate the
same broad CI/CD stages. They are separate Jenkins jobs; do not assume
they share a workspace.

### 1.3 Repository and application structure

The Laravel application in this task includes the Student CRUD feature.
Files discussed in the project include:

``` text
app/Models/Student.php
app/Http/Controllers/StudentController.php
database/migrations/2026_10_20_070330_create_students_table.php
routes/web.php
resources/views/layout.blade.php
resources/views/students/index.blade.php
resources/views/students/create.blade.php
resources/views/students/show.blade.php
resources/views/students/edit.blade.php
tests/Feature/ExampleTest.php
```

The Student model fields include `student_id`, `name`, `email`,
`department`, and `semester`. If you are repeating this on a new
checkout, confirm these files exist rather than assuming the repository
has not changed.

------------------------------------------------------------------------

## 2. AWS EC2 instance and Security Group

### 2.1 Create or select the EC2 instance

In AWS Console:

1.  Open **EC2 → Instances**.
2.  Launch an Ubuntu EC2 instance, or select the intended existing
    instance.
3.  Choose an instance size appropriate for the lab.
4.  Select or create a key pair and save the private key securely.
5.  Ensure the instance is in a subnet with a route to an Internet
    Gateway if it must be accessed publicly.
6.  Assign a public IPv4 address or an Elastic IP.
7.  Record the current public IPv4 address. Do not rely on an old IP
    from a previous instance.

The earlier session used a public IP of `13.51.206.252` and private
hostname `ip-172-31-33-121`; these are historical values, **not
guaranteed to be current**.

### 2.2 Configure the instance Security Group

In **EC2 → Instances → select instance → Security tab → Security groups
→ Edit inbound rules**, add only the rules needed:

  -------------------------------------------------------------------------
  Type          Protocol                   Port Source        Purpose
  ------------- ------------- ----------------- ------------- -------------
  SSH           TCP                          22 Your trusted  Admin access
                                                public IP     
                                                `/32`         

  HTTP          TCP                          80 `0.0.0.0/0`   Public
                                                              website over
                                                              HTTP

  Custom TCP    TCP                        8080 Your trusted  Jenkins UI
                                                public IP     for lab/admin
                                                `/32`         

  HTTPS         TCP                         443 `0.0.0.0/0`   HTTPS website
                                                only if HTTPS 
                                                is configured 
  -------------------------------------------------------------------------

For IPv6 access, add corresponding IPv6 rules only if the
instance/network is configured for it.

**Do not add a public inbound rule for MySQL port 3306 or PHP-FPM port
9000.** Do not expose Jenkins port 8080 to `0.0.0.0/0` unless there is a
deliberate, secured reason. For a public production site, configure
HTTPS rather than sending user traffic over plain HTTP.

Security Group rules are stateful, but the instance still needs a valid
public IP, subnet route, and functioning service.

### 2.3 Connect to EC2

From your local Linux terminal, use the path to your downloaded key:

``` bash
chmod 400 /path/to/your-ec2-key.pem
ssh -i /path/to/your-ec2-key.pem ubuntu@CURRENT_EC2_PUBLIC_IP
```

Replace both placeholders. Do not upload the private key to GitHub or
paste it into chat.

------------------------------------------------------------------------

## 3. Connect to EC2 and check the OS

Run these commands **on EC2**:

``` bash
whoami
cat /etc/os-release
uname -m
df -h /
free -h
ip -brief address
```

Expected SSH user is usually `ubuntu`. Check disk space before
installing packages or building software.

Update package lists:

``` bash
sudo apt update
sudo apt upgrade -y
```

If `apt update` or `apt upgrade` reports an error, stop and resolve it
before continuing.

------------------------------------------------------------------------

## 4. Install system packages

The previous server used PHP 8.5 and its PHP-FPM socket was
`/run/php/php8.5-fpm.sock`. Package availability depends on the Ubuntu
release and enabled repositories. Check before installing:

``` bash
apt-cache policy nginx mysql-server php8.5-cli php8.5-fpm
```

Install core services and tools:

``` bash
sudo apt install -y nginx mysql-server git curl unzip rsync ca-certificates
```

Install PHP and the Laravel extensions. On the Ubuntu 26.04 environment
used in the session, PHP 8.5 packages were present:

``` bash
sudo apt install -y \
  php8.5-cli \
  php8.5-fpm \
  php8.5-mysql \
  php8.5-xml \
  php8.5-mbstring \
  php8.5-curl \
  php8.5-zip \
  php8.5-bcmath \
  php8.5-sqlite3
```

If `apt` cannot locate a PHP package, do not blindly copy an unrelated
package version: inspect `apt-cache policy`, identify the installed PHP
version, and install the matching packages. The PHP-FPM socket in Nginx
must match that installed version.

Verify:

``` bash
php -v
php -m
nginx -v
mysql --version
rsync --version
```

Enable and start services:

``` bash
sudo systemctl enable --now nginx
sudo systemctl enable --now mysql
sudo systemctl enable --now php8.5-fpm
```

Verify:

``` bash
sudo systemctl is-active nginx
sudo systemctl is-active mysql
sudo systemctl is-active php8.5-fpm
```

Each should print `active`.

### 4.1 Install Composer

Check whether Composer is already available:

``` bash
composer --version
```

If it is not installed, download and verify the Composer installer using
the official Composer installation instructions. Do not run an installer
copied from an untrusted website. Then ensure the `composer` executable
is available to both the `ubuntu` user and the Jenkins build
environment, and verify:

``` bash
composer --version
sudo -u jenkins composer --version
```

The second command will work only after Jenkins is installed and the
executable is in the Jenkins user's `PATH`.

------------------------------------------------------------------------

## 5. Install and start Jenkins

### 5.1 Install Java

Jenkins requires a supported Java runtime. Install a Java version
supported by the Jenkins release you install; Java 21 was used as a
suitable baseline for this lab:

``` bash
sudo apt install -y fontconfig openjdk-21-jre
java -version
```

If the Jenkins repository or current Jenkins release specifies a
different minimum Java version, follow the official Jenkins installation
documentation for that release.

### 5.2 Add the official Jenkins repository

Use the official Jenkins Linux installation instructions for the current
stable release and Ubuntu version. The following is the common
Debian/Ubuntu package-repository pattern; if Jenkins has changed its
repository key or package instructions, use the current official
commands instead:

``` bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key \
  -o /etc/apt/keyrings/jenkins-keyring.asc
sudo chmod 644 /etc/apt/keyrings/jenkins-keyring.asc

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list >/dev/null

sudo apt update
sudo apt install -y jenkins
```

If repository verification fails, stop and consult Jenkins' current
official installation page rather than disabling package-signature
checks.

### 5.3 Enable Jenkins

``` bash
sudo systemctl enable --now jenkins
sudo systemctl status jenkins --no-pager
sudo ss -ltnp | grep ':8080'
```

Get the initial administrator password if the setup wizard requests it:

``` bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open this in a browser, using the current EC2 public IP:

``` text
http://CURRENT_EC2_PUBLIC_IP:8080
```

Complete the setup wizard, install suggested plugins, create an
administrator account, and sign in. Jenkins must be reachable through
the Security Group rule for TCP 8080 from your trusted IP.

### 5.4 Install/check Jenkins plugins

In **Manage Jenkins → Plugins**, confirm the following are available:

-   Git
-   Pipeline
-   Pipeline: Stage View (optional but useful)
-   Credentials Binding (optional; useful when using credentials in
    shell steps)

Restart Jenkins if the UI asks you to do so.

------------------------------------------------------------------------

## 6. Clone the Laravel repository

The repository used in this task is:

``` text
https://github.com/samikhan5055/laravel-project.git
```

Clone it on EC2 for initial inspection and production setup:

``` bash
cd /home/ubuntu
git clone https://github.com/samikhan5055/laravel-project.git
cd /home/ubuntu/laravel-project
```

If the folder already exists, do not clone over it. Inspect it:

``` bash
cd /home/ubuntu/laravel-project
git status
git branch --show-current
git remote -v
ls -la
test -f artisan && echo "artisan found"
test -f composer.json && echo "composer.json found"
```

If the repository is private, use an appropriate Jenkins credential for
checkout. Do not embed a personal access token in the repository URL,
`Jenkinsfile`, or shell command history.

Check the source routes and tests:

``` bash
php artisan route:list --path=students
```

If dependencies are not installed in this checkout, install them:

``` bash
composer install --no-interaction --prefer-dist --optimize-autoloader
```

This is a developer/inspection checkout. Jenkins jobs have their own
workspace unless configured otherwise.

------------------------------------------------------------------------

## 7. Prepare the production database and `.env`

### 7.1 Create the MySQL database and application user

Open the MySQL shell:

``` bash
sudo mysql
```

At the `mysql>` prompt, execute the following SQL, replacing the
password placeholder with a unique, strong password. Do not use the
placeholder literally:

``` sql
CREATE DATABASE student_management
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'student_app'@'localhost'
  IDENTIFIED BY 'REPLACE_WITH_A_UNIQUE_STRONG_PASSWORD';

GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, ALTER, INDEX, DROP
  ON student_management.* TO 'student_app'@'localhost';

FLUSH PRIVILEGES;
EXIT;
```

Use a secure secret manager or restricted server-side file to store the
password. If the application requires additional permissions for
migrations, grant them only to the account that performs migrations;
avoid using the MySQL root account in Laravel.

Test the login:

``` bash
mysql -u student_app -p -h 127.0.0.1 student_management
```

Enter the password at the prompt. Exit MySQL with:

``` sql
EXIT;
```

### 7.2 Create the production directory

``` bash
sudo mkdir -p /var/www/student-management
```

The production `.env` must live at:

``` text
/var/www/student-management/.env
```

Create it from `.env.example` if it does not exist. Do not overwrite an
existing production file:

``` bash
if [ ! -f /var/www/student-management/.env ]; then
  sudo cp /home/ubuntu/laravel-project/.env.example /var/www/student-management/.env
  sudo chown root:www-data /var/www/student-management/.env
  sudo chmod 640 /var/www/student-management/.env
fi
```

Edit the file securely:

``` bash
sudo nano /var/www/student-management/.env
```

Set the production values appropriately. Example shape (replace every
placeholder and use the real generated application key):

``` dotenv
APP_NAME="Student Management"
APP_ENV=production
APP_KEY=
APP_DEBUG=false
APP_URL=http://CURRENT_EC2_PUBLIC_IP

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=student_management
DB_USERNAME=student_app
DB_PASSWORD=REPLACE_WITH_THE_DATABASE_PASSWORD

SESSION_DRIVER=file
CACHE_STORE=file
QUEUE_CONNECTION=sync
```

Retain any other environment variables required by the actual project.
Do not paste secrets into GitHub, a public issue, or this Markdown file.
Set `APP_KEY` after dependencies have been installed, as described
below.

**Do not run `migrate:fresh` against the production database.** It can
drop tables and destroy data.

------------------------------------------------------------------------

## 8. Configure the Laravel production directory and permissions

### 8.1 Copy the initial application source

For the initial deployment only, copy the project files into the
production directory while preserving the production `.env`:

``` bash
sudo rsync -rlpt \
  --no-owner --no-group \
  --exclude='/.env' \
  --exclude='/.git/' \
  --exclude='/storage/' \
  --exclude='/bootstrap/cache/' \
  /home/ubuntu/laravel-project/ /var/www/student-management/
```

Create runtime directories:

``` bash
sudo mkdir -p /var/www/student-management/storage/{app,framework/{cache,sessions,testing,views},logs}
sudo mkdir -p /var/www/student-management/bootstrap/cache
```

Install Composer dependencies in the production directory:

``` bash
cd /var/www/student-management
sudo composer install --no-dev --no-interaction --prefer-dist --optimize-autoloader
```

If running Composer as root displays a warning, prefer configuring the
directory for a dedicated deploy user and run Composer as that user.
Avoid routinely running application build tools as root.

Generate the production key only if it is not already set. First inspect
whether `APP_KEY` is configured without printing the secret:

``` bash
sudo grep -q '^APP_KEY=base64:' /var/www/student-management/.env \
  && echo "APP_KEY is configured" \
  || echo "APP_KEY must be generated"
```

If missing:

``` bash
cd /var/www/student-management
sudo php artisan key:generate --force
```

Check that Laravel can load its environment:

``` bash
sudo php artisan --version
```

### 8.2 Set ownership and permissions

The earlier lab used Jenkins as the deploy account, `www-data` as the
web group, and a protected production `.env`. A dedicated deployment
user with only the necessary write access is preferable for production.
For the lab, these commands reflect the permissions discussed:

``` bash
sudo chown -R jenkins:www-data /var/www/student-management
sudo chown -R www-data:www-data /var/www/student-management/storage
sudo chown -R jenkins:www-data /var/www/student-management/bootstrap/cache

sudo find /var/www/student-management -type d -exec chmod 755 {} \;
sudo find /var/www/student-management -type f -exec chmod 644 {} \;

sudo chmod -R 2775 /var/www/student-management/storage
sudo chmod -R 2775 /var/www/student-management/bootstrap/cache

sudo chown root:www-data /var/www/student-management/.env
sudo chmod 640 /var/www/student-management/.env
```

Ensure Jenkins is in the `www-data` group only if your deployment
permission design requires it:

``` bash
sudo usermod -aG www-data jenkins
sudo systemctl restart jenkins
id jenkins
```

**Do not use `chmod -R 777`.** Re-check `.env` ownership and mode after
any recursive `chown` command.

### 8.3 Run database migrations

Before running production migrations, take a backup if the database
contains data. Then run:

``` bash
cd /var/www/student-management
sudo php artisan config:clear
sudo php artisan migrate --force
```

This runs outstanding migrations without dropping all tables. If
migration fails, inspect the error and database permissions before
retrying.

------------------------------------------------------------------------

## 9. Configure Nginx and PHP-FPM

### 9.1 Confirm the PHP-FPM socket

``` bash
ls -l /run/php/
```

The previous environment used:

``` text
/run/php/php8.5-fpm.sock
```

If your actual socket has a different version, use the actual path in
the Nginx configuration.

### 9.2 Create the Nginx site

``` bash
sudo nano /etc/nginx/sites-available/student-management
```

Paste:

``` nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    root /var/www/student-management/public;
    index index.php index.html;
    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico {
        access_log off;
        log_not_found off;
    }

    location = /robots.txt {
        access_log off;
        log_not_found off;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.5-fpm.sock;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

If PHP-FPM is not version 8.5, update the socket line to match
`ls -l /run/php/`.

### 9.3 Enable the site

First inspect existing sites:

``` bash
ls -l /etc/nginx/sites-enabled/
```

For a single-site lab where this Laravel app should be the default
website, run:

``` bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -sfn /etc/nginx/sites-available/student-management \
  /etc/nginx/sites-enabled/student-management
sudo nginx -t
sudo systemctl reload nginx
```

If another website is already hosted on this instance, do not remove its
default site without reviewing the existing configuration.

### 9.4 Configure Laravel caches

Run from the production directory:

``` bash
cd /var/www/student-management
sudo php artisan optimize:clear
sudo php artisan config:cache
sudo php artisan route:cache
sudo php artisan view:cache
```

If these commands fail due to permission errors, correct the ownership
of `bootstrap/cache` and `storage`; do not work around the issue with
`777`.

------------------------------------------------------------------------

## 10. Verify the website locally before Jenkins deployment

Run on EC2:

``` bash
sudo nginx -t
sudo systemctl is-active nginx
sudo systemctl is-active php8.5-fpm
sudo ss -ltnp | grep -E ':80|:8080'
curl -I --max-time 5 http://127.0.0.1/students
```

Expected: Nginx syntax succeeds, Nginx and PHP-FPM are active, Nginx
listens on port 80, and `/students` responds with HTTP 200 or an
expected redirect.

Now test the public URL in a browser:

``` text
http://CURRENT_EC2_PUBLIC_IP/students
```

If local `curl` succeeds but the browser cannot connect, go to **Section
17.4**. A local HTTP 200 alone does not prove public network access or
full CRUD/database functionality.

------------------------------------------------------------------------

## 11. Create Jenkins Freestyle Build job

### 11.1 Create the job

In Jenkins UI:

1.  Click **New Item**.
2.  Name: `JOB1-Laravel-Build`.
3.  Select **Freestyle project**.
4.  Click **OK**.

### 11.2 Configure Git checkout

Under **Source Code Management**:

-   Select **Git**.
-   Repository URL:
    `https://github.com/samikhan5055/laravel-project.git`
-   Branch Specifier: `*/main`
-   Credentials: select a Jenkins credential if the repository is
    private.

Do not place a personal access token directly in the URL or a build
shell script.

### 11.3 Add the build shell commands

Under **Build Steps → Add build step → Execute shell**, paste:

``` bash
#!/bin/bash
set -eu

echo "PHP version:"
php -v

echo "Composer version:"
composer --version

echo "Installing Laravel dependencies..."
composer install --no-interaction --prefer-dist --optimize-autoloader

echo "Checking Laravel project..."
php artisan --version
php artisan route:list --path=students

echo "Build job completed."
```

Save and click **Build Now**. Open **Console Output** and confirm the
build succeeds.

**Important:** The job agent must have PHP, Composer, and required
extensions. If the Jenkins agent does not have them, install them or
configure a dedicated build agent. Do not assume packages installed for
your interactive `ubuntu` shell are automatically available to Jenkins.

------------------------------------------------------------------------

## 12. Create Jenkins Freestyle Test job

### 12.1 Create the job

1.  Click **New Item**.
2.  Name: `JOB2-Laravel-Test`.
3.  Select **Freestyle project**.
4.  Click **OK**.

Configure Git SCM with the same repository and `*/main` branch. This
makes the test job's workspace independent and repeatable.

### 12.2 Configure test database

The project tests were run using SQLite in the Jenkins workspace. Ensure
PHP SQLite support is installed:

``` bash
php -m | grep -i sqlite
```

Under **Build Steps → Execute shell**, paste:

``` bash
#!/bin/bash
set -eu

composer install --no-interaction --prefer-dist --optimize-autoloader

if [ ! -f .env ]; then
    cp .env.example .env
fi

mkdir -p database
touch database/database.sqlite

export DB_CONNECTION=sqlite
export DB_DATABASE="$WORKSPACE/database/database.sqlite"
export SESSION_DRIVER=file
export CACHE_STORE=file
export QUEUE_CONNECTION=sync

if ! grep -q '^APP_KEY=base64:' .env; then
    php artisan key:generate --force
fi

php artisan config:clear
php artisan migrate --force
php artisan test
```

If the project has a different test configuration, adjust it to the
repository's actual `phpunit.xml` and test setup. Do not point CI tests
at the production database.

### 12.3 Trigger after successful Build job

In `JOB2-Laravel-Test`:

1.  Open **Configure**.
2.  Find **Build Triggers**.
3.  Enable **Build after other projects are built**.
4.  Enter `JOB1-Laravel-Build`.
5.  Select **Trigger only if build is stable**.
6.  Save.

Run `JOB1-Laravel-Build`, then verify `JOB2-Laravel-Test` runs and
passes. The prior session reported three Laravel tests passing.

------------------------------------------------------------------------

## 13. Create Jenkins Freestyle Deploy job

### 13.1 Create the job

1.  Click **New Item**.
2.  Name: `JOB3-Laravel-Deploy`.
3.  Select **Freestyle project**.
4.  Click **OK**.

Configure Git SCM with the repository URL and branch `*/main`.

### 13.2 Configure deploy trigger

Under **Build Triggers**:

1.  Enable **Build after other projects are built**.
2.  Enter `JOB2-Laravel-Test`.
3.  Select **Trigger only if build is stable**.
4.  Save.

### 13.3 Add a deployment shell step

This is a simple lab deployment step. It deliberately preserves
production `.env`, `storage`, and `bootstrap/cache`. The production
directory must already exist and be writable by the deployment account.

Under **Build Steps → Execute shell**, paste:

``` bash
#!/bin/bash
set -eu

DEPLOY_DIR="/var/www/student-management"

test -f "$DEPLOY_DIR/.env"
test -d "$DEPLOY_DIR/storage"
test -d "$DEPLOY_DIR/bootstrap/cache"

echo "Reviewing deployment changes (dry run)..."
rsync -rlptn \
  --no-owner --no-group \
  --delete \
  --exclude='/.env' \
  --exclude='/storage/' \
  --exclude='/bootstrap/cache/' \
  --exclude='/.git/' \
  --exclude='/database/database.sqlite' \
  "$WORKSPACE/" "$DEPLOY_DIR/"

echo "Applying deployment..."
rsync -rlpt \
  --no-owner --no-group \
  --delete \
  --exclude='/.env' \
  --exclude='/storage/' \
  --exclude='/bootstrap/cache/' \
  --exclude='/.git/' \
  --exclude='/database/database.sqlite' \
  "$WORKSPACE/" "$DEPLOY_DIR/"

cd "$DEPLOY_DIR"

php artisan optimize:clear
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan --version
php artisan route:list --path=students

echo "Freestyle deployment commands completed."
```

**Review the dry-run output before relying on `--delete`.**
`rsync --delete` can remove destination files that are not in the
source. The exclusions protect the paths listed above, not every
possible local file. For a production setup, consider immutable release
directories and a controlled symlink switch instead.

This job can fail if Composer dependencies are missing from the
deployment workspace. Each Freestyle job has its own workspace; if the
deployment stage requires `vendor/`, install Composer dependencies in
this job or deliberately transfer a tested build artifact. Do not assume
another job's `vendor/` directory is shared.

Run the Test job and verify that the Deploy job is triggered only after
the tests pass.

------------------------------------------------------------------------

## 14. Create and push the `Jenkinsfile`

This section is performed in the **Git working copy** that you will
commit and push. The Jenkins Pipeline job will fetch this file from the
repository.

### 14.1 Open the repository

On your development computer, or in the existing Git checkout on EC2:

``` bash
cd /path/to/laravel-project
git status
git branch --show-current
```

Replace `/path/to/laravel-project` with the real project path. If using
the EC2 checkout from this guide, use:

``` bash
cd /home/ubuntu/laravel-project
```

### 14.2 Create/edit the Jenkinsfile

``` bash
nano Jenkinsfile
```

Paste the following complete file:

``` groovy
pipeline {
    agent any

    environment {
        DEPLOY_DIR = '/var/www/student-management'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh '''
                    set -eu

                    echo "Installing Composer dependencies..."
                    composer install \
                        --no-interaction \
                        --prefer-dist \
                        --optimize-autoloader

                    if [ ! -f .env ]; then
                        cp .env.example .env
                    fi

                    mkdir -p database
                    touch database/database.sqlite

                    export DB_CONNECTION=sqlite
                    export DB_DATABASE="$WORKSPACE/database/database.sqlite"
                    export SESSION_DRIVER=file
                    export CACHE_STORE=file
                    export QUEUE_CONNECTION=sync

                    if ! grep -q '^APP_KEY=base64:' .env; then
                        php artisan key:generate --force
                    fi

                    php artisan config:clear
                    echo "Build completed."
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    set -eu

                    export DB_CONNECTION=sqlite
                    export DB_DATABASE="$WORKSPACE/database/database.sqlite"
                    export SESSION_DRIVER=file
                    export CACHE_STORE=file
                    export QUEUE_CONNECTION=sync

                    echo "Preparing CI test database..."
                    php artisan migrate --force

                    echo "Running Laravel tests..."
                    php artisan test

                    echo "All tests passed."
                '''
            }
        }

        stage('Deploy to EC2') {
            steps {
                sh '''
                    set -eu

                    echo "Checking production paths..."
                    test -f "$DEPLOY_DIR/.env"
                    test -d "$DEPLOY_DIR/storage"
                    test -d "$DEPLOY_DIR/bootstrap/cache"

                    echo "Previewing file synchronization..."
                    rsync -rlptn \
                        --no-owner \
                        --no-group \
                        --delete \
                        --exclude='/.env' \
                        --exclude='/storage/' \
                        --exclude='/bootstrap/cache/' \
                        --exclude='/.git/' \
                        --exclude='/database/database.sqlite' \
                        "$WORKSPACE/" "$DEPLOY_DIR/"

                    echo "Synchronizing application files..."
                    rsync -rlpt \
                        --no-owner \
                        --no-group \
                        --delete \
                        --exclude='/.env' \
                        --exclude='/storage/' \
                        --exclude='/bootstrap/cache/' \
                        --exclude='/.git/' \
                        --exclude='/database/database.sqlite' \
                        "$WORKSPACE/" "$DEPLOY_DIR/"

                    cd "$DEPLOY_DIR"

                    echo "Refreshing Laravel caches..."
                    php artisan optimize:clear
                    php artisan config:cache
                    php artisan route:cache
                    php artisan view:cache

                    echo "Checking deployed application..."
                    php artisan --version
                    php artisan route:list --path=students

                    echo "Deployment commands completed."
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Build, tests, and deployment commands completed.'
        }

        failure {
            echo 'FAILURE: Review the Console Output for the failed stage.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
```

Save in nano with `Ctrl+O`, press `Enter`, then exit with `Ctrl+X`.

Verify that the file is in the repository root and has the exact name
`Jenkinsfile` (capital J, no extension):

``` bash
test -f Jenkinsfile && echo "Jenkinsfile exists"
sed -n '1,260p' Jenkinsfile
git status --short
```

### 14.3 Important Jenkinsfile assumptions

-   The Jenkins agent needs PHP, Composer, required PHP extensions, and
    `rsync`.
-   The production `.env`, `storage`, and `bootstrap/cache` must already
    exist.
-   The deploy user must have only the necessary access to update
    application files and Laravel caches.
-   The tests use SQLite in the Jenkins workspace; the production app
    uses MySQL.
-   The `rsync --delete` operation must be reviewed. Do not deploy
    blindly to a directory containing files that are not tracked by Git.
-   This Pipeline deploys files to a directory on the **same EC2
    instance** where Jenkins runs. It is not an SSH deployment to a
    different server.
-   The file's presence in GitHub and a green Pipeline build must both
    be verified; creating the file locally alone does not update
    Jenkins' SCM checkout.

### 14.4 Commit and push the Jenkinsfile

Before staging, inspect status carefully:

``` bash
git status
git diff -- Jenkinsfile
```

Stage only the intended file:

``` bash
git add Jenkinsfile
git status
git commit -m "Add Jenkins CI/CD pipeline for EC2 deployment"
git push origin main
```

If application code changes also need to be committed, review and stage
those files individually. Do not use `git add .` blindly; it can
accidentally stage `.env`, database files, or secrets.

Verify on GitHub that `Jenkinsfile` appears at the root of the `main`
branch:

``` text
https://github.com/samikhan5055/laravel-project
```

------------------------------------------------------------------------

## 15. Create the Jenkins Pipeline job

### 15.1 Create the job

In Jenkins UI:

1.  Click **New Item**.
2.  Name: `Laravel-Student-Pipeline`.
3.  Select **Pipeline**.
4.  Click **OK**.

### 15.2 Configure Pipeline from SCM

Scroll to the **Pipeline** section and set:

-   **Definition:** `Pipeline script from SCM`
-   **SCM:** `Git`
-   **Repository URL:**
    `https://github.com/samikhan5055/laravel-project.git`
-   **Credentials:** select the correct Jenkins credential if the
    repository is private
-   **Branch Specifier:** `*/main`
-   **Script Path:** `Jenkinsfile`

Click **Save**.

**Do not choose "Pipeline script from SCM" and leave the repository or
script path blank.** The earlier error,
`No flow definition, cannot run`, occurred when the job did not have a
usable Pipeline definition. The job must fetch the `Jenkinsfile` from
the selected repository/branch.

### 15.3 Ensure Jenkins can deploy

The Pipeline's Deploy stage runs on the Jenkins agent and writes to
`/var/www/student-management`. In this lab that is the same EC2
instance. Ensure the Jenkins user can read the workspace and has the
required, limited write access to the deployment directory and cache
directories.

Check:

``` bash
id jenkins
sudo -u jenkins test -r /var/www/student-management/.env \
  && echo "Jenkins can read production .env" \
  || echo "Jenkins cannot read production .env"
sudo -u jenkins test -w /var/www/student-management \
  && echo "Jenkins can write deployment directory" \
  || echo "Jenkins cannot write deployment directory"
```

The Pipeline's `test -f "$DEPLOY_DIR/.env"` checks existence; it does
not require Jenkins to print or reveal the file contents. Keep the file
private.

If permissions are wrong, adjust them narrowly and re-check `.env`
access. Do not solve permission errors with `chmod -R 777`.

------------------------------------------------------------------------

## 16. Run and verify the complete Pipeline

1.  Open `Laravel-Student-Pipeline`.
2.  Click **Build Now**.
3.  Open the new build.
4.  Click **Console Output**.
5.  Review each stage: `Checkout`, `Build`, `Test`, and `Deploy to EC2`.

The Test stage should show Laravel's test summary and a successful exit
code. The Deploy stage must complete its synchronization and Artisan
checks. A successful Test stage alone does not prove deployment
succeeded.

### 16.1 Verify the deployed site locally

On EC2:

``` bash
sudo nginx -t
sudo systemctl is-active nginx
sudo systemctl is-active php8.5-fpm
curl -I --max-time 5 http://127.0.0.1/students
```

### 16.2 Verify the public website

In an external browser, use the **current** public IP shown in the EC2
Console:

``` text
http://CURRENT_EC2_PUBLIC_IP/students
```

The root URL should redirect to the students page if the root route is
still configured that way:

``` text
http://CURRENT_EC2_PUBLIC_IP/
```

Jenkins UI:

``` text
http://CURRENT_EC2_PUBLIC_IP:8080
```

Only expose port 8080 to trusted IP addresses.

### 16.3 Verify CRUD and database functionality

Open the application and test creating, viewing, editing, and deleting a
test student record. Confirm the production app uses the intended MySQL
database, not the SQLite test database. Do not use destructive test
operations on real student data.

------------------------------------------------------------------------

## 17. Troubleshooting: errors and resolutions

### 17.1 `No flow definition, cannot run`

**Likely cause:** Jenkins job is not configured with a valid Pipeline
definition, or Jenkins cannot find the `Jenkinsfile`.

**Resolution:**

1.  Open `Laravel-Student-Pipeline → Configure`.
2.  Set Definition to **Pipeline script from SCM**.
3.  Select Git and configure the repository URL.
4.  Set Branch Specifier to `*/main`.
5.  Set Script Path to `Jenkinsfile`.
6.  Confirm `Jenkinsfile` is committed and pushed to the `main` branch.
7.  Save and run the job again.

### 17.2 `composer: command not found`

Check:

``` bash
command -v composer
sudo -u jenkins sh -lc 'command -v composer; composer --version'
```

The Jenkins service may have a different `PATH` from your interactive
terminal. Install Composer in a system-wide location and verify it as
the Jenkins user. Do not rely on a Composer executable available only
inside another user's home directory.

### 17.3 Laravel dependencies missing in a downstream Freestyle job

Each Freestyle job can have a separate workspace. If
`vendor/autoload.php` is missing, install dependencies in the job that
requires them or pass a build artifact explicitly:

``` bash
composer install --no-interaction --prefer-dist --optimize-autoloader
```

Do not assume a dependency folder created by `JOB1-Laravel-Build`
automatically appears in `JOB2-Laravel-Test` or `JOB3-Laravel-Deploy`.

### 17.4 Website shows "Connection refused" or cannot be reached

Run on EC2:

``` bash
sudo nginx -t
sudo systemctl is-active nginx
sudo systemctl status nginx --no-pager
sudo ss -ltnp | grep ':80'
curl -I --max-time 5 http://127.0.0.1/students
```

Interpret the results:

-   If local `curl` fails, investigate Nginx, PHP-FPM, Laravel logs, and
    the document root.
-   If local `curl` returns `200 OK` but the public URL fails, check the
    current EC2 public IPv4 address, Security Group inbound TCP 80,
    subnet route/Internet Gateway, and host firewall.
-   Do not use an old public IP from a previous EC2 session without
    checking the Console.

Inspect logs:

``` bash
sudo tail -n 100 /var/log/nginx/error.log
sudo tail -n 100 /var/log/nginx/access.log
sudo tail -n 100 /var/www/student-management/storage/logs/laravel.log
```

If the Laravel log does not exist, inspect the directory:

``` bash
sudo ls -la /var/www/student-management/storage/logs
```

### 17.5 `nginx -t` fails

Run:

``` bash
sudo nginx -t
```

Use the exact file and line number shown in the error. Inspect the site
file:

``` bash
sudo sed -n '1,220p' /etc/nginx/sites-available/student-management
ls -l /etc/nginx/sites-enabled/
ls -l /run/php/
```

Confirm the PHP-FPM socket matches the installed PHP version. After
correcting the configuration:

``` bash
sudo nginx -t && sudo systemctl reload nginx
```

### 17.6 `rsync` error code 23 / permission denied

Earlier output included messages such as:

``` text
chgrp .../bootstrap failed
mkstemp .../bootstrap/.app.php: Permission denied
rsync error: some files/attrs were not transferred (code 23)
```

**Cause:** Destination permissions were insufficient and/or `rsync`
attempted to set ownership/group metadata the deployment user could not
set.

The `rsync` flags used in the revised Pipeline include:

``` bash
--no-owner --no-group
```

These stop `rsync` from attempting to preserve source ownership/group,
but the destination must still be writable. Check:

``` bash
id jenkins
namei -l /var/www/student-management/bootstrap/cache
namei -l /var/www/student-management/storage
sudo -u jenkins test -w /var/www/student-management/bootstrap/cache
sudo -u jenkins test -w /var/www/student-management
```

Correct ownership and permissions narrowly. Do not use `chmod -R 777`.

### 17.7 Laravel cache command fails due to permissions

Run:

``` bash
namei -l /var/www/student-management/bootstrap/cache
namei -l /var/www/student-management/storage
ls -ld /var/www/student-management/bootstrap/cache
ls -ld /var/www/student-management/storage
```

Ensure the account executing Artisan can write to `bootstrap/cache` and
`storage`, while `.env` remains restricted. Then retry the failed cache
command and inspect its exact error.

### 17.8 MySQL access denied or migrations fail

Test the application account without showing the password in the command
line:

``` bash
mysql -u student_app -p -h 127.0.0.1 student_management
```

Check that the production `.env` has the correct database name,
username, password, host, and port. After changing `.env`, refresh
cached configuration:

``` bash
cd /var/www/student-management
sudo php artisan config:clear
sudo php artisan config:cache
```

Do not run `migrate:fresh` against the production database. If a
migration fails, read the error before making database changes.

### 17.9 Public Jenkins page does not open

Check on EC2:

``` bash
sudo systemctl is-active jenkins
sudo ss -ltnp | grep ':8080'
```

Then verify the Security Group has inbound TCP `8080` from your current
trusted public IP `/32`. If your home/public IP changed, update the
source rule. Avoid opening Jenkins to all internet addresses as a quick
fix.

------------------------------------------------------------------------

## 18. Security checklist

-   [ ] Use a strong EC2 SSH key and restrict SSH to a trusted IP where
    practical.
-   [ ] Restrict Jenkins port 8080 to trusted IPs.
-   [ ] Keep MySQL port 3306 and PHP-FPM port 9000 private.
-   [ ] Never commit `.env`, passwords, private keys, or tokens.
-   [ ] Keep production `.env` mode restrictive; in this lab it was set
    to `640` with owner `root` and group `www-data`.
-   [ ] Do not use `chmod -R 777`.
-   [ ] Do not run `php artisan migrate:fresh` on the production
    database.
-   [ ] Review the `rsync --delete` dry run before deploying.
-   [ ] Preserve production `.env`, uploads/runtime storage, and the
    production database.
-   [ ] Do not print secret values in Jenkins logs.
-   [ ] Use HTTPS for real public production traffic.
-   [ ] Back up the database before risky schema changes.
-   [ ] Confirm the actual deploy user and permissions rather than
    copying recursive `chown` commands blindly.
-   [ ] Use Jenkins credentials for private repository authentication.
-   [ ] Rotate any weak or reused credentials before production use.

------------------------------------------------------------------------

## 19. Final verification checklist

### AWS and server

-   [ ] EC2 instance is running.
-   [ ] Current public IPv4 address is confirmed.
-   [ ] Security Group has TCP 80 for the public website.
-   [ ] SSH and Jenkins access are restricted to trusted sources.
-   [ ] Nginx, PHP-FPM, MySQL, and Jenkins services are active.
-   [ ] Composer and PHP extensions are available to Jenkins.

### Laravel

-   [ ] Repository checkout succeeds.
-   [ ] `artisan` and `composer.json` exist.
-   [ ] Student routes are registered.
-   [ ] Production `.env` exists and is not committed.
-   [ ] Production database connection works.
-   [ ] Migrations run safely.
-   [ ] Nginx routes to Laravel and local `/students` responds.
-   [ ] CRUD operations have been tested against the intended database.

### Jenkins Freestyle

-   [ ] `JOB1-Laravel-Build` checks out source and installs
    dependencies.
-   [ ] `JOB2-Laravel-Test` runs after a stable Build and tests pass.
-   [ ] `JOB3-Laravel-Deploy` runs after stable tests and preserves
    production data/configuration.

### Jenkins Pipeline

-   [ ] `Jenkinsfile` is at the repository root.
-   [ ] `Jenkinsfile` is committed and pushed to `main`.
-   [ ] `Laravel-Student-Pipeline` uses **Pipeline script from SCM**.
-   [ ] SCM URL, branch `*/main`, and Script Path `Jenkinsfile` are
    correct.
-   [ ] Checkout, Build, and Test stages pass.
-   [ ] Deploy stage completes successfully.
-   [ ] Public website opens from an external browser.
-   [ ] The final application and database behavior have been verified.

------------------------------------------------------------------------

## Conclusion

This runbook provides a command-first path to install the server stack,
configure the AWS Security Group, prepare Laravel and MySQL, configure
Nginx, create all three Jenkins Freestyle jobs, add and push the
complete `Jenkinsfile`, and configure/run a Pipeline job from SCM.

Treat each verification command as a gate. If a command fails, stop at
that point and resolve the specific error before moving to the next
section. A passing local HTTP request, passing tests, or a configured
Jenkins job does not individually prove that the full public CI/CD
deployment is complete.
