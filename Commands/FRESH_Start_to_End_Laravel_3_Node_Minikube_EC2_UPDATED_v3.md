# Fresh Kubernetes Practice on AWS EC2: Install Minikube, Create Three Nodes, and Deploy the Laravel Repository

**Execution location:** the AWS EC2 Ubuntu SSH terminal only  
**Repository:** `https://github.com/samikhan5055/laravel-project.git`  
**Kubernetes tool:** Minikube with its Docker driver  
**Cluster size:** 3 Minikube nodes (one control plane and two workers)  
**Application:** the Laravel three-tier app: Nginx → Laravel/PHP-FPM → MySQL  
**Existing deployment:** keep the already-running Docker Compose deployment untouched.

> This guide starts the Kubernetes learning exercise from the beginning. It does **not** reuse old Kubernetes Pods, old Minikube profiles, or old application containers as the Kubernetes deployment. We clone the repository into a new directory, create a new Minikube profile, build/load fresh lab images, and create new Kubernetes resources. The existing Docker Compose deployment must remain running.

---

# Part 1 — Understand the separation

## 1.1 What is already installed and what we will install?

You have said Docker Engine is installed on EC2 and the existing Docker deployment must keep running.

| Component | Starting position for this lab | Action |
|---|---|---|
| Docker Engine | Already installed; used by the existing app | Verify it; do not remove/reinstall it |
| Existing Docker Compose app | Must keep running | Leave its containers, volumes, ports and network alone |
| Minikube | Treat as not installed for this exercise | Install and verify |
| `kubectl` | Treat as not installed for this exercise | Install and verify |
| Kubernetes cluster | New, dedicated cluster | Create using Minikube with 3 nodes |
| Laravel source | Existing public Git repository | Clone a fresh copy into a new lab directory |
| Kubernetes workloads | None created for this lab yet | Build new resources in a dedicated namespace |

**Important terminology:** Minikube creates and manages a Kubernetes cluster; we do not install a separate Kubernetes server package by hand. `kubectl` is the command-line client used to control that cluster.

## 1.2 Target layout

```text
AWS EC2 Ubuntu instance
|
|-- Existing Docker Compose deployment (leave running)
|     |-- existing Nginx/web container
|     |-- existing Laravel/PHP container
|     `-- existing MySQL container and volume
|
`-- New Minikube profile: laravel-k8s
      |-- Node 1: control plane
      |-- Node 2: worker
      `-- Node 3: worker
             |
             `-- Kubernetes namespace: laravel-k8s
                   |-- Nginx Deployment + Service
                   |-- Laravel/PHP-FPM Deployment + Service
                   `-- MySQL Deployment + Service + PVC
```

The three Minikube nodes are containers on the same EC2 VM. This is a learning cluster, not three separate EC2 machines or production-grade high availability.

## 1.3 Known lab configuration values

Use the values already provided for this lab instead of generic placeholders:

- Docker Hub username: `sami1800`
- Laravel `APP_KEY`: `base64:xKoXsGqYAF/3BhvHDQ37paFiKdIZ9sg78v6mZYYdSfY=`
- Database name: `laravel`
- Database username: `laraveluser`
- Database password: `sami1234`
- MySQL root password: configure separately from the application password. Do not reuse `sami1234` for root. On a genuinely fresh setup, choose and securely retain a separate root password before creating the MySQL Secret. If MySQL has already initialized its PVC, do not change the root Secret casually; changing a Secret does not rotate an existing MySQL account.

Do not commit `.env` or Kubernetes Secret values to Git. The APP_KEY is sensitive and must remain consistent across Laravel replicas. These values are for this lab only, not production.

---

# Part 2 — AWS EC2 preparation

## 2. Check EC2 Security Group before starting

In AWS Console:

1. Open **EC2 → Instances**.
2. Select the Ubuntu instance where the Docker app is currently running.
3. Open **Security → Security groups**.
4. Select the attached Security Group and choose **Edit inbound rules**.

Use the minimum rules needed:

| Type | Port | Source | Why |
|---|---:|---|---|
| SSH | TCP 22 | Your current public IP `/32` | Connect to EC2 |
| Custom TCP | TCP 8085 | Your current public IP `/32` | Optional temporary Kubernetes browser test |
| HTTP 80 | TCP 80 | Keep only if the existing Docker app requires it | Existing application |
| HTTPS 443 | TCP 443 | Keep only if already required | Existing HTTPS application |

Do **not** open MySQL port 3306, Kubernetes API port 8443, or the full NodePort range 30000–32767 to the internet. We will use a temporary `kubectl port-forward` for browser testing. If the EC2 public IP or your home public IP changes, update the relevant rule.

AWS references:
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-rules-reference.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-security-group.html

## 3. Open EC2 terminal and record the current Docker deployment

All commands in this guide run in the EC2 SSH terminal unless explicitly labelled AWS Console.

```bash
whoami
hostname
cat /etc/os-release | head
uname -m
nproc
free -h
df -h /
sudo systemctl is-active docker
sudo docker version
sudo docker ps
sudo docker compose ls
```

Save or note the output from `sudo docker ps` and `sudo docker compose ls`. These are our baseline checks to prove the existing Docker application remains running.

**Do not run any of these against the existing project:** `docker compose down`, `docker rm`, `docker volume rm`, `docker system prune`, or `minikube delete` without a profile. They are not required for this lab and can damage unrelated work.

Check the CPU architecture:
- `x86_64` means use the AMD64 installation commands below.
- `aarch64` means use the ARM64 commands noted in the installation section.

Check free disk space and memory. Minikube's documented minimum is 2 CPUs, 2 GB available memory and 20 GB free disk; a three-node cluster plus the existing application needs more practical headroom. If the instance cannot accommodate both deployments, resize it before continuing instead of stopping the existing app.

References:
- https://minikube.sigs.k8s.io/docs/start/
- https://minikube.sigs.k8s.io/docs/tutorials/running_minikube_on_aws_ec2/

---

# Part 3 — Install Minikube and kubectl from scratch

## 4. Install prerequisite utilities

Docker is already installed, so do not reinstall Docker or remove any Docker configuration. Install only utilities needed for downloading the binaries and cloning the repository:

```bash
sudo apt-get update
sudo apt-get install -y curl ca-certificates git
```

Verify:

```bash
git --version
curl --version | head -n 1
sudo systemctl is-active docker
```

If Docker is not `active`, stop and investigate before proceeding; do not uninstall or reset Docker.

## 5. Install Minikube

### 5.1 AMD64 EC2 instance (`x86_64`)

```bash
cd /tmp
curl -fSLO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm -f minikube-linux-amd64
minikube version
```

### 5.2 ARM64 EC2 instance (`aarch64`)

```bash
cd /tmp
curl -fSLO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-arm64
sudo install minikube-linux-arm64 /usr/local/bin/minikube
rm -f minikube-linux-arm64
minikube version
```

Do only the command set matching the output of `uname -m`.

Official installation documentation: https://minikube.sigs.k8s.io/docs/start/

## 6. Install kubectl from scratch

Use the architecture matching `uname -m`.

### 6.1 AMD64

```bash
cd /tmp
KUBECTL_VERSION="$(curl -fsSL https://dl.k8s.io/release/stable.txt)"
curl -fSLO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
curl -fSLO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm -f kubectl kubectl.sha256
kubectl version --client
```

### 6.2 ARM64

```bash
cd /tmp
KUBECTL_VERSION="$(curl -fsSL https://dl.k8s.io/release/stable.txt)"
curl -fSLO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/arm64/kubectl"
curl -fSLO "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/arm64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm -f kubectl kubectl.sha256
kubectl version --client
```

Official Kubernetes tools page: https://kubernetes.io/docs/tasks/tools/

## 7. Run Minikube consistently with Docker permissions

This lab avoids adding your account to the Docker group. Therefore, run Minikube and kubectl consistently with `sudo`, so the Docker driver can access Docker and Minikube uses a consistent root-owned kubeconfig.

Use these command patterns throughout this guide:

```bash
sudo minikube ...
sudo kubectl ...
sudo docker ...
```

Do not mix root-owned Minikube configuration with a non-root Minikube profile. Do not use `sudo docker compose down` on the existing app.

---

# Part 4 — Create a brand-new three-node Kubernetes cluster

## 8. Check profile names without deleting anything

```bash
sudo minikube profile list
sudo docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

We will create a dedicated profile called `laravel-k8s`. This is a new profile for this exercise. If `laravel-k8s` already exists because you previously attempted this exact guide, inspect it before deciding whether to reuse it or intentionally remove only that named lab profile. Do not delete another profile.

## 9. Start Minikube with three nodes

Use the Docker driver:

```bash
sudo minikube start \
  --profile=laravel-k8s \
  --driver=docker \
  --nodes=3 \
  --cpus=2 \
  --memory=2048
```

The requested CPU/memory values are per node and must fit alongside the existing Docker deployment. If Minikube reports insufficient memory/CPU, do not keep increasing allocations. Re-check `free -h`, `nproc`, and the EC2 instance size. A cluster on one EC2 instance shares the same host resources.

Wait for startup to finish before continuing.

## 10. Verify all three nodes

```bash
sudo minikube --profile=laravel-k8s status
sudo minikube --profile=laravel-k8s node list
sudo kubectl config get-contexts
sudo kubectl get nodes -o wide
sudo kubectl get pods -A -o wide
```

**Checkpoint 1:** the new cluster must show three nodes in `Ready` state. Their names will be similar to `laravel-k8s`, `laravel-k8s-m02`, and `laravel-k8s-m03`, but always use the actual names from your output.

If a node is not Ready:

```bash
sudo kubectl describe nodes
sudo kubectl get events -A --sort-by=.lastTimestamp
sudo minikube --profile=laravel-k8s logs
sudo docker ps -a
free -h
df -h /
```

Stop here if the cluster is unhealthy. Do not start deploying the application until all nodes are Ready.

## 11. Confirm the correct kubectl context

```bash
sudo kubectl config get-contexts
sudo kubectl config current-context
```

If the context named `laravel-k8s` is shown, select it:

```bash
sudo kubectl config use-context laravel-k8s
sudo kubectl get nodes -o wide
```

If the displayed context name differs, select the exact context that belongs to the `laravel-k8s` profile. Do not guess a context name.

---

# Part 5 — Clone the repository into a separate lab directory

## 12. Create a fresh checkout

The existing Docker deployment stays in its current directory. This practice uses a separate directory so we don't overwrite or modify its files.

```bash
mkdir -p ~/kubernetes-practice
cd ~/kubernetes-practice
git clone https://github.com/samikhan5055/laravel-project.git laravel-project
cd ~/kubernetes-practice/laravel-project
pwd
git status --short
git log -1 --oneline
ls -la
```

Expected directory:

```text
/home/ubuntu/kubernetes-practice/laravel-project/
```

Your username or home directory might differ; `pwd` shows the actual path.

**Do not clone over `~/laravel-project` if that is the directory used by the existing Docker deployment.** The new checkout is separate by design.

## 13. Inspect the repository before building anything

Do not assume filenames, ports, or image names. First list its deployment files:

```bash
cd ~/kubernetes-practice/laravel-project
find . -maxdepth 3 -type f \
  \( -name 'Dockerfile' -o -name 'compose.yaml' -o -name 'docker-compose.yml' \
     -o -iname '*nginx*.conf' -o -name '.dockerignore' \) -print
```

Check which Compose file exists:

```bash
ls -l compose.yaml docker-compose.yml 2>/dev/null || true
```

### 13.1 Create the missing `.env` in the fresh practice checkout

The current error says `APP_KEY`, `DOCKERHUB_USERNAME`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`, and `MYSQL_ROOT_PASSWORD` are required but missing. The `.env.example` file exists, but `.env` does not. Create the local environment file for this checkout only:

```bash
cd ~/kubernetes-practice/laravel-project
cp .env.example .env
chmod 600 .env
nano .env
```

Keep the other Laravel settings from `.env.example`, and ensure the following entries exist exactly once. Replace matching lines if they already exist; do not append duplicate definitions. These values use the lab details already provided:

```dotenv
APP_KEY=base64:xKoXsGqYAF/3BhvHDQ37paFiKdIZ9sg78v6mZYYdSfY=
DOCKERHUB_USERNAME=sami1800
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=laraveluser
DB_PASSWORD=sami1234
MYSQL_ROOT_PASSWORD=REPLACE_THIS_WITH_YOUR_SEPARATE_ROOT_PASSWORD
```

`DB_HOST=db` is for Docker Compose only. In Kubernetes, the database host is `mysql-service`. `DB_PASSWORD` must be `sami1234`. Set `MYSQL_ROOT_PASSWORD` to a separate root password and keep it private; this placeholder must be replaced before using this `.env` for Compose. Do not reuse the application password for root or use these lab credentials in production.

Save in Nano with `Ctrl+O`, press `Enter`, then `Ctrl+X`. Validate without starting containers:

```bash
sudo docker compose -f compose.yaml config --quiet
sudo docker compose -f compose.yaml config --services
sudo docker compose -f compose.yaml config --images
```

If Compose reports another missing variable, read the error and add that specific required variable to this practice copy's `.env`. Do not print or share the fully rendered Compose configuration because it can expose secrets.

If the repository uses `docker-compose.yml` instead, substitute that filename in all commands. The current checkout's observed file is `compose.yaml`.

**Do not run `docker compose up`** for this Kubernetes lab. This step only prepares and validates configuration; it must not replace or stop the existing deployment.

### What we must identify

Before writing Kubernetes manifests, identify from the repository:
1. Which service is the Laravel/PHP-FPM application.
2. Which service is Nginx/web, if the repository has a separate web tier.
3. Which service is MySQL.
4. Which Dockerfile builds each custom image.
5. Which Nginx config and document-root paths are used.
6. Which host ports the existing Compose stack uses.
7. Which environment variables Laravel expects.

If the repository only has an app and database service and does not include a separate Nginx service, we must add a separate Kubernetes-compatible Nginx tier for the requested three-tier architecture. Do not invent image names or assume that a PHP CLI image runs PHP-FPM.

## 14. Inspect the Dockerfiles and Nginx configuration

```bash
cd ~/kubernetes-practice/laravel-project
find . -maxdepth 3 -type f -name 'Dockerfile' -print
find . -maxdepth 4 -type f \( -iname '*nginx*.conf' -o -path '*/nginx/*' \) -print
```

For each listed file, inspect it with `sed` or `nano`, for example:

```bash
nano Dockerfile
```

Use the actual path if the Dockerfile is in a subdirectory. If there are multiple Dockerfiles, inspect each one.

A valid Laravel application container for this architecture must run **PHP-FPM** and listen on port 9000. If the current Dockerfile runs `php artisan serve` or a CLI command instead, it is not automatically suitable as a PHP-FPM backend; build a suitable image/configuration before continuing.

The Nginx image must have a Laravel-compatible `server` configuration and the expected public files. The FastCGI upstream in Kubernetes must point to `laravel-service:9000`, not a Compose-only hostname.

---

# Part 6 — Build fresh images for this Kubernetes lab


### 14.1 Image availability decision guide (updated after the image-load error)

The current EC2 checks confirmed that these images exist in the **host Docker daemon**:

- `sami1800/laravel-app:1.0`
- `sami1800/laravel-web:1.0`

That does **not** prove that both images have been loaded into the `laravel-k8s` Minikube profile. The earlier `image load` commands were interrupted with `Ctrl+C`, so verify the destination before proceeding. Do not use literal placeholders such as `sami1800/laravel-app:1.0`; they are examples, not valid image names.

#### A. Check the source images on EC2

Run on EC2:

```bash
sudo docker image inspect sami1800/laravel-app:1.0 --format='{{.RepoTags}}'
sudo docker image inspect sami1800/laravel-web:1.0 --format='{{.RepoTags}}'
```

- If both commands print the expected tag, the images exist locally. Continue to Section B.
- If either reports `No such image`, build the image from the repository's Dockerfile targets as described in Section D, then check again.
- Do not rebuild an image that already exists unless you intentionally changed its source or need to refresh the build.

#### B. Check whether Minikube has the images

```bash
sudo minikube --profile=laravel-k8s status
sudo minikube --profile=laravel-k8s image ls
```

Look for both `docker.io/sami1800/laravel-app:1.0` and `docker.io/sami1800/laravel-web:1.0`.

- If both are listed, image transfer is complete; proceed to the Kubernetes manifest section.
- If an image is missing, load only the missing image. Do not interrupt the command while it is running.

#### C. Load only the missing image

Try the normal Minikube image-load command first, with the exact image reference:

```bash
sudo minikube --profile=laravel-k8s image load sami1800/laravel-app:1.0
```

Wait for it to finish. Then load the web image:

```bash
sudo minikube --profile=laravel-k8s image load sami1800/laravel-web:1.0
```

Verify again:

```bash
sudo minikube --profile=laravel-k8s image ls
```

If the normal load repeatedly hangs or fails, do not repeatedly press `Ctrl+C`. Try the daemon-based loading option, one image at a time:

```bash
sudo minikube --profile=laravel-k8s image load --daemon sami1800/laravel-app:1.0
```

```bash
sudo minikube --profile=laravel-k8s image load --daemon sami1800/laravel-web:1.0
```

If the installed Minikube version rejects `--daemon`, use the normal command and share the complete error. If either method fails, stop there and diagnose the actual error rather than continuing with manifests that refer to unavailable images.

#### D. If a source image does not exist, build it

The checked-out `compose.yaml` defines both `app` and `web` from `Dockerfile`, using targets `app` and `web`. First confirm the file still has those targets:

```bash
cd ~/kubernetes-practice/laravel-project
grep -nE '^(FROM|FROM .* AS|FROM .* as)' Dockerfile
```

Build only the missing target:

```bash
sudo docker build --target app -t sami1800/laravel-app:1.0 .
```

```bash
sudo docker build --target web -t sami1800/laravel-web:1.0 .
```

These commands build images only; they do not start the Compose services. Verify with `sudo docker image inspect ...`, then follow Section B and C to load images into Minikube. If the Dockerfile does not define `app` and `web` targets, do not run these commands blindly; inspect the Dockerfile and adjust to its actual targets.

#### E. Keep image references consistent in Kubernetes

Kubernetes manifests must use the same exact references:

```yaml
image: sami1800/laravel-app:1.0
```

and:

```yaml
image: sami1800/laravel-web:1.0
```

For deployments using locally loaded images, set `imagePullPolicy: IfNotPresent` (or `Never` only when you are certain the image is loaded into every node that may run the Pod). With `IfNotPresent`, the kubelet can use a local image when available and otherwise attempt to pull it. Do not set `Always` for this offline/local-image workflow.

After applying manifests, if a Pod shows `ErrImagePull` or `ImagePullBackOff`, inspect it:

```bash
sudo kubectl --context=laravel-k8s -n laravel-k8s get pods
sudo kubectl --context=laravel-k8s -n laravel-k8s describe pod POD_NAME
```

Replace `POD_NAME` with the real Pod name. Read the Events section to distinguish an image-name typo, an image that is absent from the node, and a registry authentication/network error.

**Important:** A host Docker image existing on EC2 and an image being available to a Minikube node are two separate conditions. Always verify both sides before deploying.

## 15. Build from the fresh checkout

Build only from the new clone under `~/kubernetes-practice/laravel-project`. This does not stop the old Docker Compose app.

**Continue here after the `.env` fix:** the immediate next commands are the three validation commands in Section 13.1. Only after `config --quiet` succeeds should you inspect image/build definitions below. Do not start a Compose stack.

First see how the repository defines its images:

```bash
cd ~/kubernetes-practice/laravel-project
sudo docker compose -f compose.yaml config --services
sudo docker compose -f compose.yaml config --images
```

If the Compose file is named differently, replace `compose.yaml` with its real name. If the repository uses `build:` entries, build those images:

```bash
sudo docker compose -f compose.yaml build
```

This command builds the images declared by the new checkout; it does not start or stop the Compose stack. **Do not run `docker compose up` for this lab** because we want the Kubernetes deployment to be managed by Kubernetes, not a second Compose stack.

Inspect the images after building:

```bash
sudo docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.ID}}'
```

If the Compose file does not define `build:` for a custom image, use its Dockerfile directly and give the lab image a new, distinct tag. Example pattern only—adjust the Dockerfile path and build context to match the repository:

```bash
sudo docker build -t laravel-k8s-app:1.0 -f PATH_TO_APP_DOCKERFILE BUILD_CONTEXT
```

Do not type `PATH_TO_APP_DOCKERFILE` or `BUILD_CONTEXT` literally. Replace them with actual paths from Section 14. If you need a separate Nginx image, build it with its actual Dockerfile and context, for example using a unique tag `laravel-k8s-nginx:1.0`.

## 16. Load the fresh images into the new Minikube cluster

Minikube's Docker driver runs the Kubernetes nodes in Docker containers. Load each locally built custom image into the **new** Minikube profile so the Pods use those images:

```bash
sudo minikube --profile=laravel-k8s image load sami1800/laravel-app:1.0
sudo minikube --profile=laravel-k8s image load sami1800/laravel-web:1.0
```

Replace the image references with the actual image names and tags from the preceding build. If the application uses a public official MySQL image, Kubernetes can pull it from the registry; alternatively, pre-pull it and load it if desired.

Verify the image names:

```bash
sudo minikube --profile=laravel-k8s image ls
```

Use unique lab tags so there is no ambiguity between images used by the existing Compose application and this Kubernetes exercise.

---

# Part 7 — Create a new Kubernetes namespace and configuration

## 17. Create the manifests directory

```bash
mkdir -p ~/kubernetes-practice/laravel-project/k8s
cd ~/kubernetes-practice/laravel-project/k8s
pwd
```

All Kubernetes YAML files for this exercise go here. This is separate from the Compose configuration.

## 18. Create a namespace

```bash
cat > namespace.yaml <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: laravel-k8s
EOF
```

Apply:

```bash
sudo kubectl apply -f namespace.yaml
sudo kubectl get namespace laravel-k8s
```

## 19. Create the MySQL Secret

For the Laravel application database account, the requested password is `sami1234`. Keep the MySQL root password separate. On a fresh install only, set a separate root password in your shell (choose and securely retain it) before creating the Secret:

```bash
read -rsp 'Enter the separate MySQL root password: ' MYSQL_ROOT_PASSWORD
echo
sudo kubectl create secret generic mysql-secret \
  --namespace=laravel-k8s \
  --from-literal=MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASSWORD" \
  --from-literal=MYSQL_DATABASE='laravel' \
  --from-literal=MYSQL_USER='laraveluser' \
  --from-literal=MYSQL_PASSWORD='sami1234' \
  --dry-run=client -o yaml | sudo kubectl apply -f -
unset MYSQL_ROOT_PASSWORD
```

**Important:** This Secret creation step is for the initial setup. If MySQL has already initialized the `mysql-pvc` volume, applying a changed Secret does not change passwords inside MySQL. Do not rerun this section as a password-reset procedure or delete the PVC.

If the Secret already exists from an earlier partial attempt, the command updates it using `apply`. Verify only its name:

```bash
sudo kubectl get secrets -n laravel-k8s
```

Do not print Secret data or commit it to Git. In a real application, use a strong unique password and a proper secret-management process.

## 20. Create the Laravel Secret using the fresh checkout's APP_KEY

First check whether `.env` exists without printing its contents:

```bash
cd ~/kubernetes-practice/laravel-project
if [ -f .env ]; then
  echo ".env exists"
  grep -q '^APP_KEY=base64:' .env && echo "APP_KEY has a base64 prefix" || echo "Check APP_KEY format before proceeding"
else
  echo ".env does not exist"
fi
```

If the repository does not contain a valid `APP_KEY`, inspect the repository README and application setup instructions. Do not invent a key or print it to the terminal. For a fresh lab database, you may generate a Laravel key in a controlled environment, but the same key must be used by all Laravel replicas and must be retained for this application.

If `.env` exists and contains the correct `APP_KEY`, create the Secret without displaying the value:

```bash
cd ~/kubernetes-practice/laravel-project
APP_KEY_VALUE="$(grep '^APP_KEY=' .env | head -n 1 | cut -d '=' -f 2-)"
if [ -z "$APP_KEY_VALUE" ]; then
  echo "ERROR: APP_KEY missing; stop and resolve it before continuing."
else
  sudo kubectl create secret generic laravel-secret \
    --namespace=laravel-k8s \
    --from-literal=APP_KEY="$APP_KEY_VALUE" \
    --from-literal=DB_PASSWORD='sami1234' \
    --dry-run=client -o yaml | sudo kubectl apply -f -
  echo "Laravel Secret created/updated; APP_KEY was not displayed."
fi
unset APP_KEY_VALUE
```

Do not continue until a valid `APP_KEY` exists. A Laravel key must be consistent across replicas. Never commit `.env` to Git.

## 21. Create a ConfigMap for non-secret Laravel settings

```bash
cd ~/kubernetes-practice/laravel-project/k8s

cat > configmap.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: laravel-config
  namespace: laravel-k8s
data:
  APP_ENV: "production"
  APP_DEBUG: "false"
  LOG_CHANNEL: "stderr"
  DB_CONNECTION: "mysql"
  DB_HOST: "mysql-service"
  DB_PORT: "3306"
  DB_DATABASE: "laravel"
  DB_USERNAME: "laraveluser"
EOF
```

Apply:

```bash
sudo kubectl apply -f configmap.yaml
sudo kubectl get configmap laravel-config -n laravel-k8s
```

Compare these variables with the actual repository's Laravel configuration and update them if the app expects different names.

---

# Part 8 — Deploy the database tier

## 22. Create a PersistentVolumeClaim

```bash
cd ~/kubernetes-practice/laravel-project/k8s

cat > mysql-pvc.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: laravel-k8s
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
EOF
```

Apply and inspect:

```bash
sudo kubectl apply -f mysql-pvc.yaml
sudo kubectl get pvc -n laravel-k8s
sudo kubectl get storageclass
```

**Multi-node storage note:** Minikube's default host-path storage behavior can be limited for multi-node clusters. If the claim remains `Pending`, do not continue with MySQL. Read the PVC events and use the current Minikube documentation to choose a supported storage provisioner for this cluster:
https://minikube.sigs.k8s.io/docs/tutorials/multi_node/

Diagnose:

```bash
sudo kubectl describe pvc mysql-pvc -n laravel-k8s
sudo kubectl get events -n laravel-k8s --sort-by=.lastTimestamp
```

Do not delete a PVC containing data as a troubleshooting shortcut.

## 23. Create MySQL Deployment

```bash
cat > mysql-deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: laravel-k8s
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: mysql
              containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_DATABASE
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
          readinessProbe:
            exec:
              command:
                - sh
                - -c
                - 'mysqladmin ping -h 127.0.0.1 -uroot -p"$MYSQL_ROOT_PASSWORD"'
            initialDelaySeconds: 20
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 12
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-pvc
EOF
```

## 24. Create an internal MySQL Service

```bash
cat > mysql-service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: laravel-k8s
spec:
  type: ClusterIP
  selector:
    app: mysql
  ports:
    - name: mysql
      port: 3306
      targetPort: 3306
EOF
```

Apply:

```bash
sudo kubectl apply -f mysql-deployment.yaml
sudo kubectl apply -f mysql-service.yaml
sudo kubectl rollout status deployment/mysql -n laravel-k8s --timeout=180s
sudo kubectl get pods,services,pvc -n laravel-k8s -o wide
sudo kubectl logs deployment/mysql -n laravel-k8s --tail=100
```

**Checkpoint 2:** MySQL Pod must be Ready and `mysql-pvc` must be `Bound` before moving to Laravel.

MySQL is intentionally not exposed to the internet. Laravel reaches it through `mysql-service:3306`.

---

# Part 9 — Deploy Laravel/PHP-FPM tier

## 25. Confirm the actual Laravel image and runtime

From the image list, identify the image built from the application's PHP-FPM Dockerfile. It must run PHP-FPM and listen on port 9000. Do not use an arbitrary PHP CLI image as the backend.

```bash
sudo docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.ID}}'
sudo minikube --profile=laravel-k8s image ls
```

If the existing repository uses a PHP CLI image, create a separate Kubernetes-compatible PHP-FPM image based on the actual project Dockerfile and required PHP extensions. The exact Dockerfile changes depend on the repository contents; do not replace a working Dockerfile blindly.

## 26. Create Laravel Deployment manifest

Create the file with a placeholder, then replace it with the exact image built in Part 6:

```bash
cd ~/kubernetes-practice/laravel-project/k8s

cat > laravel-deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel
  namespace: laravel-k8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: laravel
  template:
    metadata:
      labels:
        app: laravel
    spec:
      containers:
        - name: laravel
          image: sami1800/laravel-app:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: fastcgi
              containerPort: 9000
          envFrom:
            - configMapRef:
                name: laravel-config
          env:
            - name: APP_KEY
              valueFrom:
                secretKeyRef:
                  name: laravel-secret
                  key: APP_KEY
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: laravel-secret
                  key: DB_PASSWORD
          readinessProbe:
            tcpSocket:
              port: 9000
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
EOF
```

Edit and replace the placeholder:

```bash
nano laravel-deployment.yaml
```

This lab uses `sami1800/laravel-app:1.0`. Before applying the Deployment, verify it is loaded in this Minikube profile with `sudo minikube --profile=laravel-k8s image ls | grep laravel-app`.

## 27. Create Laravel's internal Service

```bash
cat > laravel-service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: laravel-service
  namespace: laravel-k8s
spec:
  type: ClusterIP
  selector:
    app: laravel
  ports:
    - name: fastcgi
      port: 9000
      targetPort: 9000
EOF
```

Apply:

```bash
sudo kubectl apply -f laravel-deployment.yaml
sudo kubectl apply -f laravel-service.yaml
sudo kubectl get pods -n laravel-k8s -o wide
sudo kubectl logs deployment/laravel -n laravel-k8s --tail=100
```

If the deployment fails:

```bash
sudo kubectl describe deployment laravel -n laravel-k8s
sudo kubectl get pods -n laravel-k8s -o wide
sudo kubectl get events -n laravel-k8s --sort-by=.lastTimestamp
```

Only continue when the Laravel Pods are running and the logs do not show configuration errors.

## 28. Verify database credentials and run Laravel migrations

Do not run migrations until MySQL is Ready and the Laravel database password matches the intended MySQL application-user password. The app password specified for this lab is `sami1234`.

First check the actual resources and readiness:

```bash
sudo kubectl get pods,services,pvc -n laravel-k8s -o wide
sudo kubectl rollout status deployment/mysql -n laravel-k8s --timeout=180s
sudo kubectl rollout status deployment/laravel -n laravel-k8s --timeout=180s
```

Compare the two Secret values without printing either password:

```bash
MYSQL_PASS="$(sudo kubectl get secret mysql-secret -n laravel-k8s -o jsonpath='{.data.MYSQL_PASSWORD}' | base64 -d)"
APP_PASS="$(sudo kubectl get secret laravel-secret -n laravel-k8s -o jsonpath='{.data.DB_PASSWORD}' | base64 -d)"
if [ "$MYSQL_PASS" = "$APP_PASS" ]; then
  echo "Secret passwords match"
else
  echo "ERROR: Secret passwords differ; stop before running migrations"
fi
unset MYSQL_PASS APP_PASS
```

For a **fresh database initialization**, the intended application password is `sami1234` in both Secrets. If the passwords differ and MySQL has already initialized its PVC, do not assume changing a Secret fixes the existing MySQL account. A Secret update changes Kubernetes configuration, not the password stored inside MySQL. Follow the troubleshooting section to verify/repair the MySQL account without deleting the PVC.

Check the ConfigMap values without printing Secret values:

```bash
sudo kubectl get configmap laravel-config -n laravel-k8s -o yaml
```

Confirm it sets `DB_HOST=mysql-service`, `DB_PORT="3306"`, `DB_DATABASE=laravel`, `DB_USERNAME=laraveluser`, and `DB_CONNECTION=mysql`.

Only after the checks above pass, automatically select one ready Laravel pod and run the migration once:

```bash
POD="$(sudo kubectl get pods -n laravel-k8s -l app=laravel \
  --field-selector=status.phase=Running \
  -o jsonpath='{.items[0].metadata.name}')"
if [ -z "$POD" ]; then
  echo "ERROR: No running Laravel pod found; stop and inspect pods."
else
  echo "Using Laravel pod: $POD"
  sudo kubectl exec -n laravel-k8s "$POD" -- php artisan migrate --force
fi
unset POD
```

If the migration reports `SQLSTATE[HY000] [1045] Access denied`, stop. Do not repeatedly retry the migration, delete the PVC, or recreate MySQL. Follow the recovery section in Part 14 to reconcile the actual MySQL account password with the Kubernetes Secret safely.

---

# Part 10 — Deploy Nginx/web tier

## 29. Confirm Nginx's configuration

Nginx must:
- listen on port 80;
- serve the correct Laravel public directory;
- have the needed Laravel public files available;
- forward PHP requests to `laravel-service:9000`;
- use the Kubernetes Service name rather than a Docker Compose service name.

The important FastCGI lines in a suitable `server` block look like:

```nginx
location ~ \.php$ {
    include fastcgi_params;
    fastcgi_pass laravel-service:9000;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param DOCUMENT_ROOT $document_root;
}
```

This is only a PHP location block, not a full Nginx configuration. Preserve the application's correct `root`, `index`, `try_files`, and static file rules. The exact document root and application file paths must match the image built from your repository.

If the repository's Nginx image points to `app:9000`, update the Kubernetes-specific configuration to use `laravel-service:9000` and build a fresh lab Nginx image. Do not modify the Nginx configuration used by the existing Compose deployment.

## 30. Create Nginx Deployment

Replace the image placeholder with the actual compatible Nginx image built and loaded in Part 6.

```bash
cd ~/kubernetes-practice/laravel-project/k8s

cat > nginx-deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: laravel-k8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: sami1800/laravel-web:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
EOF
```

Replace the image placeholder:

```bash
nano nginx-deployment.yaml
```

## 31. Create the external-facing Nginx Service

```bash
cat > nginx-service.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: laravel-k8s
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30080
EOF
```

Apply the Nginx tier:

```bash
sudo kubectl apply -f nginx-deployment.yaml
sudo kubectl apply -f nginx-service.yaml
sudo kubectl rollout status deployment/nginx -n laravel-k8s --timeout=180s
sudo kubectl get pods,services -n laravel-k8s -o wide
```

We will use port-forward for access from EC2 and the browser. Do not expose NodePort `30080` in the public Security Group for this lab.

---

# Part 11 — Validate the complete application

## 32. Check Pods, Services, endpoints and storage

```bash
sudo kubectl get all -n laravel-k8s -o wide
sudo kubectl get pvc -n laravel-k8s
sudo kubectl get endpoints -n laravel-k8s
sudo kubectl get events -n laravel-k8s --sort-by=.lastTimestamp
```

Expected:
- one Ready MySQL Pod;
- two Ready Laravel Pods;
- two Ready Nginx Pods;
- the PVC is Bound;
- all three Services exist and have endpoints.

If a Service has no endpoints, check that its selector matches the corresponding Pod labels:

```bash
sudo kubectl describe service mysql-service -n laravel-k8s
sudo kubectl describe service laravel-service -n laravel-k8s
sudo kubectl describe service nginx-service -n laravel-k8s
```

Check logs:

```bash
sudo kubectl logs deployment/mysql -n laravel-k8s --tail=100
sudo kubectl logs deployment/laravel -n laravel-k8s --all-pods=true --tail=100
sudo kubectl logs deployment/nginx -n laravel-k8s --all-pods=true --tail=100
```

## 33. Test the web service locally on EC2

Run this in one EC2 SSH terminal and keep it running:

```bash
sudo kubectl port-forward \
  --address 127.0.0.1 \
  service/nginx-service \
  8085:80 \
  -n laravel-k8s
```

Open a second EC2 SSH terminal and run:

```bash
curl -I http://127.0.0.1:8085/
curl -sS http://127.0.0.1:8085/ | head
```

If you get HTTP 500, 502 or 504, inspect logs before changing configuration:

```bash
sudo kubectl logs deployment/nginx -n laravel-k8s --tail=100
sudo kubectl logs deployment/laravel -n laravel-k8s --all-pods=true --tail=100
sudo kubectl get endpoints -n laravel-k8s
```

Stop the port-forward using `Ctrl+C` in the terminal where it is running.

## 34. Open the app from your browser

After the local EC2 `curl` test works, add the Security Group rule TCP 8085 **only from your own public IP `/32`**.

Run this in the EC2 terminal and keep it running:

```bash
sudo kubectl port-forward \
  --address 0.0.0.0 \
  service/nginx-service \
  8085:80 \
  -n laravel-k8s
```

Then browse to:

```text
http://EC2_PUBLIC_IP:8085
```

Replace `EC2_PUBLIC_IP` with the instance's current public IPv4 address. The port-forward stops when its terminal is closed or the process is stopped. Because it listens on all EC2 interfaces, the Security Group must remain restricted to your IP. Stop it with `Ctrl+C` when finished.

Safer alternative: keep the port-forward bound to `127.0.0.1` and use an SSH tunnel from your laptop. Do not open the entire NodePort range publicly.

---

# Part 12 — Demonstrate that the three nodes are being used

## 35. Get exact node names

```bash
sudo kubectl get nodes -o wide
sudo minikube --profile=laravel-k8s node list
sudo kubectl get pods -n laravel-k8s -o wide
```

The `NODE` column shows where each Pod runs. A three-node cluster does not automatically mean each application tier will land on a particular node.

## 36. Label the two workers

Use exact worker node names from the previous command. For example only:

```bash
sudo kubectl label node laravel-k8s-m02 role=worker-2 --overwrite
sudo kubectl label node laravel-k8s-m03 role=worker-3 --overwrite
sudo kubectl get nodes --show-labels
```

Do not type the placeholders literally. Do not label the control plane as a worker.

## 37. Schedule workloads deliberately

Kubernetes uses scheduler rules, not the names of Deployments, to decide Pod placement. For a lab, we can use `nodeSelector` or node affinity, but strict placement can prevent rescheduling if the selected worker fails.

For example, adding this under `spec.template.spec` in a Deployment schedules its Pods only on a node labelled `role=worker-2`:

```yaml
nodeSelector:
  role: worker-2
```

Use this only after the node has the label and you understand that it restricts placement. To distribute replicas across both workers, use appropriate pod anti-affinity or topology-spread constraints instead of placing every replica on one worker.

For the first run, observe default placement with `kubectl get pods -o wide`. Then add scheduling rules and demonstrate the difference.

---

# Part 13 — Scaling, self-healing and database persistence

## 38. Scale Laravel

```bash
sudo kubectl scale deployment/laravel --replicas=3 -n laravel-k8s
sudo kubectl rollout status deployment/laravel -n laravel-k8s
sudo kubectl get pods -n laravel-k8s -o wide
```

Return to two replicas:

```bash
sudo kubectl scale deployment/laravel --replicas=2 -n laravel-k8s
```

## 39. Demonstrate self-healing

Get a real Laravel Pod name:

```bash
sudo kubectl get pods -n laravel-k8s -l app=laravel
```

Delete only one Laravel Pod:

```bash
POD="$(sudo kubectl get pods -n laravel-k8s -l app=laravel -o jsonpath='{.items[0].metadata.name}')"
[ -n "$POD" ] && sudo kubectl delete pod "$POD" -n laravel-k8s
unset POD
sudo kubectl get pods -n laravel-k8s -o wide --watch
```

Replace the placeholder with the actual Pod name. Stop watching with `Ctrl+C`. The Deployment should create a replacement Pod. Use an application Pod for this first demonstration, not MySQL.

## 40. Test database persistence

First verify that the PVC is Bound and the MySQL Pod is Ready:

```bash
sudo kubectl get pvc,pods -n laravel-k8s -o wide
```

Insert a test record using a table that actually exists in the Laravel schema. To verify Pod replacement, only delete the MySQL Pod after confirming that the PVC is Bound and the data is disposable or backed up:

```bash
sudo kubectl delete pod -l app=mysql -n laravel-k8s
sudo kubectl rollout status deployment/mysql -n laravel-k8s --timeout=180s
sudo kubectl get pods,pvc -n laravel-k8s -o wide
```

Check the test record again. A PVC is not a backup or database replication system.

---

# Part 14 — Troubleshooting

## 41. Node is not Ready

```bash
sudo kubectl describe nodes
sudo kubectl get events -A --sort-by=.lastTimestamp
sudo minikube --profile=laravel-k8s logs
sudo docker ps -a
free -h
df -h /
```

Do not restart Docker or delete the cluster until you understand the impact on the existing Compose containers.

## 42. Pod is Pending

```bash
for POD in $(sudo kubectl get pods -n laravel-k8s -o name); do sudo kubectl describe -n laravel-k8s "$POD"; done
sudo kubectl get events -n laravel-k8s --sort-by=.lastTimestamp
sudo kubectl describe nodes
```

Common causes include insufficient resources, an unbound PVC, or scheduling rules that match no node.

## 43. ImagePullBackOff / ErrImagePull

```bash
for POD in $(sudo kubectl get pods -n laravel-k8s -o name); do sudo kubectl describe -n laravel-k8s "$POD"; done
sudo minikube --profile=laravel-k8s image ls
sudo docker images
```

Check the image name/tag and confirm the image was loaded into the `laravel-k8s` profile. If using a private registry, configure an image pull Secret.

## 44. Laravel cannot connect to MySQL

Collect safe diagnostics first:

```bash
sudo kubectl get pods,services,endpoints,pvc -n laravel-k8s -o wide
sudo kubectl logs deployment/laravel -n laravel-k8s --all-pods=true --tail=80
sudo kubectl logs deployment/mysql -n laravel-k8s --tail=80
sudo kubectl describe service mysql-service -n laravel-k8s
```

Check the app's non-secret DB settings:

```bash
sudo kubectl get configmap laravel-config -n laravel-k8s -o yaml
```

Compare Secret passwords without displaying them:

```bash
MYSQL_PASS="$(sudo kubectl get secret mysql-secret -n laravel-k8s -o jsonpath='{.data.MYSQL_PASSWORD}' | base64 -d)"
APP_PASS="$(sudo kubectl get secret laravel-secret -n laravel-k8s -o jsonpath='{.data.DB_PASSWORD}' | base64 -d)"
if [ "$MYSQL_PASS" = "$APP_PASS" ]; then echo "Secret passwords match"; else echo "Secret passwords differ"; fi
unset MYSQL_PASS APP_PASS
```

If the error is `SQLSTATE[HY000] [1045] Access denied`, this is an authentication failure, not a service-discovery failure. On an already-initialized MySQL PVC, changing `MYSQL_PASSWORD` in a Secret does not rotate the existing MySQL account password. Do not delete the PVC or reinitialize MySQL. Confirm which password was used when the database was first initialized, then log in using the configured MySQL root credential and explicitly update the `laraveluser` account to the intended application password. After that, align `laravel-secret` with the same application password and restart the Laravel deployment so it reloads the Secret:

```bash
sudo kubectl rollout restart deployment/laravel -n laravel-k8s
sudo kubectl rollout status deployment/laravel -n laravel-k8s --timeout=120s
```

Do not run a password-reset command until the actual MySQL root login is confirmed; never expose passwords in terminal output or commit Secret values to Git.

## 45. Nginx gives 502/504

```bash
sudo kubectl logs deployment/nginx -n laravel-k8s --all-pods=true --tail=100
sudo kubectl logs deployment/laravel -n laravel-k8s --all-pods=true --tail=100
sudo kubectl get endpoints laravel-service -n laravel-k8s
```

Check that PHP-FPM listens on port 9000, Nginx points to `laravel-service:9000`, and both images use compatible Laravel file paths.

## 46. PVC remains Pending

```bash
sudo kubectl get storageclass
sudo kubectl describe pvc mysql-pvc -n laravel-k8s
sudo kubectl get events -n laravel-k8s --sort-by=.lastTimestamp
```

Follow the Minikube multi-node storage documentation before installing a different provisioner or deleting a PVC:
https://minikube.sigs.k8s.io/docs/tutorials/multi_node/

## 47. Port 8085 is busy

```bash
sudo ss -ltnp | grep ':8085'
```

Choose a free host port such as 8086 if necessary, then update the port-forward command, Security Group rule, and browser URL. Do not stop the existing Docker app to free the port.

---

# Part 15 — Final verification and preserving both deployments

## 48. Verify the Kubernetes lab

```bash
sudo minikube --profile=laravel-k8s status
sudo minikube --profile=laravel-k8s node list
sudo kubectl get nodes -o wide
sudo kubectl get all -n laravel-k8s -o wide
sudo kubectl get pvc -n laravel-k8s
sudo kubectl get endpoints -n laravel-k8s
```

Checklist:

- [ ] Minikube and kubectl installed and verified.
- [ ] New `laravel-k8s` cluster has three Ready nodes.
- [ ] Repository cloned to `~/kubernetes-practice/laravel-project`.
- [ ] Fresh application images built and loaded into this profile.
- [ ] Kubernetes namespace and Secrets created.
- [ ] MySQL Pod Ready and PVC Bound.
- [ ] Laravel Pods Ready and able to reach MySQL.
- [ ] Nginx Pods Ready and Service endpoints populated.
- [ ] Browser access works through port 8085.
- [ ] Pod scheduling shown across the three nodes.
- [ ] Scaling and self-healing demonstrated.
- [ ] Database test data survives MySQL Pod replacement.
- [ ] MySQL/API ports are not public.
- [ ] Existing Docker deployment is still running.

## 49. Verify the existing Docker deployment was not changed

```bash
sudo docker ps
sudo docker compose ls
```

Compare with the baseline output from Section 3. The original Docker Compose application should still be running. This lab did not require `docker compose down` or removal of its volumes.

## 50. Stop and resume the Kubernetes lab

To stop the dedicated Minikube cluster while preserving its state:

```bash
sudo minikube --profile=laravel-k8s stop
```

To resume:

```bash
sudo minikube --profile=laravel-k8s start
sudo kubectl get nodes -o wide
sudo kubectl get all -n laravel-k8s -o wide
```

Do not use `minikube delete` unless you deliberately intend to destroy this lab cluster. Never delete an unspecified profile or Docker volumes for cleanup.

---

# Official documentation

- Minikube install: https://minikube.sigs.k8s.io/docs/start/
- Minikube Docker driver: https://minikube.sigs.k8s.io/docs/drivers/docker/
- Minikube multi-node: https://minikube.sigs.k8s.io/docs/tutorials/multi_node/
- Minikube on EC2: https://minikube.sigs.k8s.io/docs/tutorials/running_minikube_on_aws_ec2/
- Kubernetes install tools / kubectl: https://kubernetes.io/docs/tasks/tools/
- AWS EC2 Security Groups: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-rules-reference.html

---

# Work method

Follow the guide in order and stop at each checkpoint if an error occurs. The repository's actual Dockerfiles, Compose services, and Nginx configuration determine the correct image names and file paths; inspect them rather than guessing. The only existing service we preserve is the already-running Docker Compose deployment. The Kubernetes application is built from a fresh checkout, in a new Minikube profile, with a separate namespace and new Kubernetes resources.
