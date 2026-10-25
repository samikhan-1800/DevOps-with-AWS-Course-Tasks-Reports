# Kubernetes Multi-Node Deployment of a Laravel Three-Tier Application
## Fresh Ubuntu EC2 → Docker → Minikube multi-node cluster → Kubernetes manifests → verification

This guide deploys the Laravel three-tier application from the existing repository:

- **Repository:** `https://github.com/samikhan5055/laravel-project.git`
- **Docker Hub namespace:** `sami1800`
- **Application image:** `sami1800/laravel-app:1.0`
- **Web image:** `sami1800/laravel-web:1.0`
- **Database:** MySQL 8.0
- **Cluster:** Minikube with one control-plane node and two worker nodes
- **Application access:** HTTP through a Kubernetes NodePort and `minikube service`/SSH tunnel where required

> **Important scope:** A three-node Minikube cluster on one EC2 instance is a *multi-node Kubernetes lab*, but all nodes still share that EC2 host's CPU, RAM, disk, and network. It is not equivalent to three separate EC2 machines. If the assignment accepts Minikube or a separate multi-node cluster, this is the simpler demonstration.

> **Keep the existing Kubernetes lab safe:** This guide uses a new Minikube profile named `laravel-three-tier` and namespace `laravel-three-tier`. Do not delete the existing `laravel-k8s` profile or namespace. Do not run `minikube delete` without specifying the new profile. Do not run `docker system prune`, `docker volume prune`, or any command with `--volumes`.

---

## 1. Architecture and request flow

```text
Browser
  |
  | HTTP :30080 (NodePort)
  v
Kubernetes Service: web-service
  |
  v
Nginx web tier (Deployment; 2 replicas)
  |
  | FastCGI :9000
  v
Laravel PHP-FPM tier (Deployment; 2 replicas)
  |
  | MySQL protocol :3306
  v
MySQL database tier (Stateful workload; 1 replica + PersistentVolumeClaim)
```

Nginx serves static files and forwards PHP requests to Laravel PHP-FPM. Laravel connects to MySQL through the internal `mysql-service` DNS name. MySQL is **not** exposed publicly.

### Important compatibility note

The manifests below expect the repository's Docker build to provide:

- a PHP-FPM application image compatible with the app's `Dockerfile` `app` target;
- an Nginx image compatible with the `web` target and its Nginx configuration;
- Nginx configured to use `fastcgi_pass laravel-service:9000;` (or the equivalent service name used below).

If the repository's current Nginx configuration still contains `fastcgi_pass app:9000;` from Docker Compose, **do not assume that `app` resolves in Kubernetes**. Kubernetes Service DNS names differ from Compose service names. Use the Kubernetes-specific config in Section 5, build the web image, and push it before deploying.

---

## 2. Recommended EC2 size and security group

For a three-node Minikube cluster plus Laravel, Nginx, and MySQL, use an EC2 instance with at least:

- **4 vCPUs**
- **16 GiB RAM recommended**
- **30 GiB or more EBS storage**

A smaller instance may work with reduced node resources and fewer replicas, but can become slow or run out of memory/disk. Check the instance's actual capacity before continuing:

```bash
nproc
free -h
df -h /
```

In the EC2 Security Group, allow:
- **SSH TCP 22** from *your current public IP only*.
- **TCP 30080** from your IP only if you intend to access the NodePort directly.
- Do not expose MySQL TCP 3306.
- Do not expose the Kubernetes API port publicly.

If you are reusing an existing EC2 instance, avoid changing its security group or installing a second cluster until you understand what is already running. This guide assumes a **fresh Ubuntu EC2 instance**.

Connect to the instance:

```bash
ssh -i /path/to/your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Replace the key path and IP with your own values.

---

## 3. Update Ubuntu and install required utilities

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y ca-certificates curl wget gnupg apt-transport-https \
  conntrack socat ebtables iptables git nano jq
```

Check disk space again after upgrades:

```bash
df -h /
```

If the root filesystem is nearly full, stop and increase the EBS volume / filesystem or free known, safe-to-remove files before downloading container images. Do not blindly prune Docker volumes.

---

## 4. Install Docker Engine

Use Docker Engine as Minikube's driver. The commands below install Docker from its official Ubuntu repository.

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

. /etc/os-release
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  ${UBUNTU_CODENAME:-$VERSION_CODENAME} stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

Enable Docker and verify it using `sudo`; this guide does **not** add your user to the Docker group.

```bash
sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
sudo docker version
sudo docker info
```

If `docker info` fails, check the service before continuing:

```bash
sudo systemctl restart docker
sudo journalctl -u docker --no-pager -n 100
```

---

## 5. Clone the Laravel repository and prepare Kubernetes-specific Nginx configuration

```bash
cd ~
git clone https://github.com/samikhan5055/laravel-project.git
cd ~/laravel-project
git branch --show-current
ls -la
```

Confirm the repository contains the expected build files:

```bash
find . -maxdepth 3 -type f \( -name 'Dockerfile' -o -name 'compose.yaml' \
  -o -name 'docker-compose.yml' -o -name 'default.conf' -o -name 'nginx.conf' \) -print
```

Inspect the Dockerfile and current Nginx config before building:

```bash
sed -n '1,240p' Dockerfile
find . -type f \( -name 'default.conf' -o -name 'nginx.conf' \) -print
```

### 5.1 Create a Kubernetes Nginx configuration

The Kubernetes web tier must forward PHP requests to the **Kubernetes Service** `laravel-service`, not the Docker Compose service name `app`.

First, check the repository's document root and existing Nginx config. The following standard Laravel config assumes the application is copied into `/var/www/html` in the Nginx image and its public directory is `/var/www/html/public`. If your Dockerfile uses a different path, adjust `root` to match it.

```bash
mkdir -p k8s
cat > k8s/default.conf <<'EOF'
server {
    listen 80;
    server_name _;
    root /var/www/html/public;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass laravel-service:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
EOF
```

**Do not build yet** if the existing `web` target does not copy the application files to `/var/www/html`. Inspect its `COPY` commands. The Nginx image needs the Laravel `public/` directory to serve static assets and resolve PHP scripts. If the web target copies the application from the `app` build stage, use that same source path while replacing its Nginx config with `k8s/default.conf`.

### 5.2 Add a separate Kubernetes web Dockerfile

This leaves the repository's existing Dockerfile and Compose workflow unchanged. The example below assumes the main Dockerfile has a build stage named `app` and that `/var/www/html` is the application path. If the existing Dockerfile's stage name/path differs, adjust the `COPY --from=...` line to match the actual Dockerfile.

```bash
cat > Dockerfile.k8s-web <<'EOF'
FROM nginx:1.28-alpine
WORKDIR /var/www/html
COPY --from=app /var/www/html /var/www/html
COPY k8s/default.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
EOF
```

This separate Dockerfile can only copy from another Dockerfile's stage when the source stage is defined in the *same build*. To make that reliable, use the multi-stage build pattern below only if the repository's Dockerfile exposes the `app` stage in a way that Docker can use. If the `app` stage is not accessible from a separate Dockerfile, the safer option is to add the Kubernetes Nginx config to the repository's existing web target and build that target, or create a combined Dockerfile that includes the original stages. **Verify the actual build with the command in Section 6 before deploying.**

---

## 6. Build and push the two images to Docker Hub

Log in to Docker Hub. Use a Docker Hub access token rather than your account password where possible:

```bash
sudo docker login --username sami1800
```

Enter the token/password when prompted; do not put it directly in a command or commit it to Git.

### 6.1 Build the Laravel PHP-FPM image

If the repository Dockerfile defines an `app` target:

```bash
sudo docker build --target app -t sami1800/laravel-app:1.0 .
```

If this fails with `target stage app could not be found`, inspect the Dockerfile and use its actual PHP-FPM stage name. Do not guess a stage name.

Run a quick check:

```bash
sudo docker image inspect sami1800/laravel-app:1.0 \
  --format '{{.Config.WorkingDir}} {{json .Config.ExposedPorts}}'
```

Push the image:

```bash
sudo docker push sami1800/laravel-app:1.0
```

### 6.2 Build the Kubernetes Nginx image

Use one of these approaches, depending on the repository's actual Dockerfile.

**Approach A — the existing Dockerfile has a `web` target and can be updated to use `k8s/default.conf`:**

Update the web target's Nginx config copy to use the Kubernetes config, then build:

```bash
sudo docker build --target web -t sami1800/laravel-web:1.0 .
```

If you need the Compose version unchanged, use a separate branch or create a dedicated Kubernetes Dockerfile that includes the same application-copy steps as the existing web stage.

**Approach B — the repository already has a standalone web Dockerfile that copies the app files correctly:**

Use that Dockerfile and ensure it copies `k8s/default.conf` into `/etc/nginx/conf.d/default.conf`:

```bash
sudo docker build -f Dockerfile.k8s-web -t sami1800/laravel-web:1.0 .
```

Do not continue until the build succeeds. Test Nginx syntax if the image includes `nginx`:

```bash
sudo docker run --rm sami1800/laravel-web:1.0 nginx -t
```

Push the web image:

```bash
sudo docker push sami1800/laravel-web:1.0
```

Verify both images exist locally:

```bash
sudo docker image ls | grep -E 'sami1800/laravel-(app|web)'
```

> For repeatable submissions, tag images with a Git commit SHA instead of relying only on `1.0`. Keep the app and web tags in the Kubernetes manifests synchronized.

---

## 7. Install kubectl and Minikube

### 7.1 Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm -f kubectl
kubectl version --client
```

### 7.2 Install Minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm -f minikube-linux-amd64
minikube version
```

### 7.3 Start a new three-node Minikube profile

The following uses one control-plane node and two worker nodes. Resource allocations must fit the EC2 instance. Start with 2 CPUs and 3 GiB RAM **per node only on a sufficiently large instance**.

```bash
minikube start -p laravel-three-tier \
  --driver=docker \
  --nodes=3 \
  --cpus=2 \
  --memory=3072 \
  --disk-size=15g \
  --kubernetes-version=stable
```

If Minikube reports insufficient CPU or memory, do not repeatedly retry the same configuration. Check `free -h` and `nproc`, then lower per-node resources (for example `--cpus=1 --memory=2048`) or resize the EC2 instance. Three nodes plus the host require more resources than one node.

Check the cluster:

```bash
minikube status -p laravel-three-tier
kubectl config get-contexts
kubectl --context=laravel-three-tier get nodes -o wide
```

If the context was not switched automatically, explicitly use the profile context with `--context=laravel-three-tier` in subsequent `kubectl` commands. Do not change or delete the old `laravel-k8s` profile.

---

## 8. Create namespace and database credentials

Use a Kubernetes Secret for credentials. The example app password is `sami1234`, matching the lab's established Laravel database user password. Use a separate strong password for MySQL root.

Generate a separate root password without printing it:

```bash
MYSQL_ROOT_PASSWORD="$(openssl rand -hex 24)"
printf '%s' "$MYSQL_ROOT_PASSWORD" > "$HOME/.mysql-root-password-laravel-three-tier"
chmod 600 "$HOME/.mysql-root-password-laravel-three-tier"
```

Create the namespace and secret. The app user and database are `laraveluser` and `laravel`:

```bash
kubectl --context=laravel-three-tier create namespace laravel-three-tier

kubectl --context=laravel-three-tier -n laravel-three-tier create secret generic mysql-secret \
  --from-literal=MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASSWORD" \
  --from-literal=MYSQL_DATABASE=laravel \
  --from-literal=MYSQL_USER=laraveluser \
  --from-literal=MYSQL_PASSWORD=sami1234

kubectl --context=laravel-three-tier -n laravel-three-tier create secret generic laravel-secret \
  --from-literal=DB_CONNECTION=mysql \
  --from-literal=DB_HOST=mysql-service \
  --from-literal=DB_PORT=3306 \
  --from-literal=DB_DATABASE=laravel \
  --from-literal=DB_USERNAME=laraveluser \
  --from-literal=DB_PASSWORD=sami1234
```

If the namespace or secret already exists, Kubernetes may report `AlreadyExists`. Inspect what exists before changing it. For a fresh profile/namespace, the commands should succeed.

Do not commit Secret YAML containing real passwords. In a production environment, use an external secret manager and rotate lab credentials after the exercise.

---

## 9. Create Kubernetes manifests

Create the manifests directory:

```bash
mkdir -p k8s
```

### 9.1 MySQL persistent storage and Deployment

Create `k8s/mysql.yaml`:

```bash
cat > k8s/mysql.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: laravel-three-tier
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: laravel-three-tier
spec:
  replicas: 1
  strategy:
    type: Recreate
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
          envFrom:
            - secretRef:
                name: mysql-secret
          readinessProbe:
            exec:
              command:
                - sh
                - -c
                - mysqladmin ping -h 127.0.0.1 -uroot -p"$MYSQL_ROOT_PASSWORD"
            initialDelaySeconds: 20
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 12
          livenessProbe:
            exec:
              command:
                - sh
                - -c
                - mysqladmin ping -h 127.0.0.1 -uroot -p"$MYSQL_ROOT_PASSWORD"
            initialDelaySeconds: 60
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 6
          resources:
            requests:
              cpu: 200m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: laravel-three-tier
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

Apply it:

```bash
kubectl --context=laravel-three-tier apply -f k8s/mysql.yaml
kubectl --context=laravel-three-tier -n laravel-three-tier get pvc
kubectl --context=laravel-three-tier -n laravel-three-tier get pods -w
```

Wait until the MySQL pod is `Running` and `Ready`. Press `Ctrl+C` to stop watching once ready.

If the PVC remains `Pending`, inspect the StorageClass:

```bash
kubectl --context=laravel-three-tier get storageclass
kubectl --context=laravel-three-tier -n laravel-three-tier describe pvc mysql-pvc
```

Minikube normally provides a default storage class. Do not remove a PVC to fix a password or readiness issue; a PVC may contain the database data.

### 9.2 Laravel PHP-FPM Deployment and internal Service

Create `k8s/laravel.yaml`:

```bash
cat > k8s/laravel.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel
  namespace: laravel-three-tier
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
          imagePullPolicy: Always
          ports:
            - name: php-fpm
              containerPort: 9000
          envFrom:
            - secretRef:
                name: laravel-secret
          env:
            - name: APP_ENV
              value: production
            - name: APP_DEBUG
              value: "false"
            - name: LOG_CHANNEL
              value: stderr
          readinessProbe:
            tcpSocket:
              port: 9000
            initialDelaySeconds: 10
            periodSeconds: 10
          livenessProbe:
            tcpSocket:
              port: 9000
            initialDelaySeconds: 30
            periodSeconds: 20
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 750m
              memory: 768Mi
EOF

cat >> k8s/laravel.yaml <<'EOF'
---
apiVersion: v1
kind: Service
metadata:
  name: laravel-service
  namespace: laravel-three-tier
spec:
  type: ClusterIP
  selector:
    app: laravel
  ports:
    - name: php-fpm
      port: 9000
      targetPort: 9000
EOF
```

Apply and check:

```bash
kubectl --context=laravel-three-tier apply -f k8s/laravel.yaml
kubectl --context=laravel-three-tier -n laravel-three-tier rollout status deployment/laravel --timeout=180s
kubectl --context=laravel-three-tier -n laravel-three-tier get pods -o wide
```

**Application key:** Laravel usually needs a valid `APP_KEY`. If the image does not already contain a configured key, generate one from the application source using the same Laravel version and set it as a Kubernetes Secret value. Do not publish it in Git. For example, in a trusted local checkout with Composer dependencies installed:

```bash
php artisan key:generate --show
```

Copy the output privately and add it to `laravel-secret` as `APP_KEY`. Avoid running `key:generate` independently on each pod: every replica must use the same application key. If the application uses encrypted cookies/sessions or other encryption, changing the key later may invalidate existing encrypted data.

If the app image has no `vendor/` directory, cannot run `php artisan`, or the PHP-FPM port does not listen on 9000, fix the image build before proceeding.

### 9.3 Nginx web Deployment and NodePort Service

Create `k8s/web.yaml`:

```bash
cat > k8s/web.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: laravel-three-tier
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: sami1800/laravel-web:1.0
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 20
            periodSeconds: 20
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 300m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: laravel-three-tier
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30080
EOF
```

Apply:

```bash
kubectl --context=laravel-three-tier apply -f k8s/web.yaml
kubectl --context=laravel-three-tier -n laravel-three-tier rollout status deployment/web --timeout=180s
```

### 9.4 Apply all manifests and review

If you created the files above:

```bash
kubectl --context=laravel-three-tier apply -f k8s/mysql.yaml
kubectl --context=laravel-three-tier apply -f k8s/laravel.yaml
kubectl --context=laravel-three-tier apply -f k8s/web.yaml

kubectl --context=laravel-three-tier -n laravel-three-tier get all
kubectl --context=laravel-three-tier -n laravel-three-tier get pvc
kubectl --context=laravel-three-tier -n laravel-three-tier get pods -o wide
```

Wait for the database first, then Laravel, then Nginx. If `mysql` is not ready, Laravel migrations and database requests will fail.

---

## 10. Run Laravel migrations

First check the actual pod names and logs:

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier get pods -l app=laravel
kubectl --context=laravel-three-tier -n laravel-three-tier logs deployment/laravel --tail=100
```

Run migrations from one Laravel pod only:

```bash
POD="$(kubectl --context=laravel-three-tier -n laravel-three-tier get pods \
  -l app=laravel -o jsonpath='{.items[0].metadata.name}')"

kubectl --context=laravel-three-tier -n laravel-three-tier exec "$POD" -- php artisan migrate --force
```

If this reports database access denied, compare the credentials in the two Kubernetes Secrets. The `MYSQL_PASSWORD` used to initialize the MySQL database and the Laravel `DB_PASSWORD` must match exactly. Updating a Secret does **not** automatically change an already-initialized MySQL user's password in the persistent database. Do not delete the PVC as a shortcut. Correct the user password in MySQL or deliberately reset the database only if you have confirmed there is no data to preserve and the assignment allows it.

If the app container exits before the command can run, inspect:

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier describe pod "$POD"
kubectl --context=laravel-three-tier -n laravel-three-tier logs "$POD" --previous
```

---

## 11. Verify the three-tier deployment

### 11.1 Check all Kubernetes resources

```bash
kubectl --context=laravel-three-tier get nodes -o wide
kubectl --context=laravel-three-tier -n laravel-three-tier get deployments
kubectl --context=laravel-three-tier -n laravel-three-tier get services
kubectl --context=laravel-three-tier -n laravel-three-tier get pods -o wide
kubectl --context=laravel-three-tier -n laravel-three-tier get pvc
```

Expected high-level state:
- 3 Minikube nodes are `Ready`.
- `web` has 2 ready replicas.
- `laravel` has 2 ready replicas.
- `mysql` has 1 ready replica.
- `mysql-pvc` is `Bound`.
- `web-service` is a NodePort service on port `30080`.
- `laravel-service` and `mysql-service` are internal ClusterIP services.

### 11.2 Confirm the pods are distributed across nodes

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier get pods -o wide
```

With only two replicas per app tier and three nodes, pods may be placed on any nodes according to Kubernetes scheduling. Kubernetes does not guarantee an even spread without topology-spread constraints or anti-affinity. For a lab, the `-o wide` output is useful evidence that pods can run on different nodes. If you need to guarantee spread, add `topologySpreadConstraints` or pod anti-affinity to the Deployments and verify scheduling afterward.

### 11.3 Access the web service

First try Minikube's service helper:

```bash
minikube -p laravel-three-tier service web-service -n laravel-three-tier --url
```

On a remote EC2 host, the URL returned by this command may be reachable only from the EC2 host. You can test from the EC2 shell:

```bash
curl -i http://127.0.0.1:30080/
```

If direct NodePort routing works for your Minikube Docker driver, access:

```text
http://YOUR_EC2_PUBLIC_IP:30080/
```

Only use that URL if you have allowed TCP 30080 from your own IP in the EC2 Security Group and the EC2 host actually routes the NodePort to the Minikube nodes. Docker-driver Minikube on a remote VM does not always expose node ports directly to the EC2 host. If direct access fails, use the SSH tunnel method below rather than opening more ports indiscriminately.

### 11.4 Reliable remote access with an SSH tunnel

On the EC2 instance, run:

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier port-forward \
  service/web-service 8088:80 --address 127.0.0.1
```

Keep that terminal running. On **your local computer**, open another terminal and run:

```bash
ssh -i /path/to/your-key.pem -L 8088:127.0.0.1:8088 ubuntu@YOUR_EC2_PUBLIC_IP
```

Then open this on your local browser:

```text
http://127.0.0.1:8088/
```

This avoids exposing the application port publicly. If port 8088 is already in use, choose another local/remote port pair.

---

## 12. Troubleshooting common failures

### A. Minikube fails due to RAM, CPU, or disk

```bash
free -h
nproc
df -h /
sudo docker system df
```

- Resize the EC2 instance or reduce node resources.
- Increase the EBS volume if needed.
- Remove only known, disposable build artifacts/images that you can rebuild.
- Do not remove named volumes or use `--volumes` prune commands.

### B. `ImagePullBackOff` or `ErrImagePull`

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier describe pod POD_NAME
sudo docker login --username sami1800
sudo docker pull sami1800/laravel-app:1.0
sudo docker pull sami1800/laravel-web:1.0
```

Ensure the repositories and tags exist and are public, or create an `imagePullSecret` for private images. A local Docker image on the EC2 host is not automatically available inside Minikube's Kubernetes nodes; push the image to a registry or load it into the correct Minikube profile.

### C. `CrashLoopBackOff`

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier describe pod POD_NAME
kubectl --context=laravel-three-tier -n laravel-three-tier logs POD_NAME --previous
```

Look for missing `vendor/`, invalid environment variables, missing application key, file permission errors, and database connection errors.

### D. Nginx returns `502 Bad Gateway`

Check the app Service endpoints and PHP-FPM pod logs:

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier get endpoints laravel-service
kubectl --context=laravel-three-tier -n laravel-three-tier get pods -l app=laravel
kubectl --context=laravel-three-tier -n laravel-three-tier logs deployment/laravel --tail=100
kubectl --context=laravel-three-tier -n laravel-three-tier logs deployment/web --tail=100
```

Typical causes:
- Nginx uses `fastcgi_pass app:9000;` instead of `laravel-service:9000;`.
- Laravel pods are not ready.
- PHP-FPM is not listening on port 9000.
- Nginx's `root` path does not match the app path in the image.
- The app image and web image were built from different versions of the code.

### E. MySQL pod is not ready

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier describe pod -l app=mysql
kubectl --context=laravel-three-tier -n laravel-three-tier logs deployment/mysql --tail=100
kubectl --context=laravel-three-tier -n laravel-three-tier get pvc
```

Check PVC binding, memory limits, and MySQL initialization logs. If the database already initialized its data directory, changing the Secret may not change the existing MySQL user's password.

### F. Laravel reports `SQLSTATE[HY000] [1045] Access denied`

Verify the app secret is using the same password as the MySQL user that exists inside the database. A Secret update alone does not update the stored MySQL account password. Correct the database account password in-place, then restart Laravel:

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier rollout restart deployment/laravel
kubectl --context=laravel-three-tier -n laravel-three-tier rollout status deployment/laravel
```

Do not paste real passwords into screenshots, reports, GitHub, or public logs.

### G. `kubectl` shows the wrong cluster

```bash
kubectl config get-contexts
kubectl --context=laravel-three-tier get nodes
minikube profile list
```

Always use `--context=laravel-three-tier` while working through this guide. Do not delete or overwrite the earlier `laravel-k8s` lab.

### H. NodePort works on the cluster but not from your browser

Use the SSH tunnel in Section 11.4. Also verify the EC2 security group only permits the intended access. Do not expose the Kubernetes API server or MySQL publicly.

---

## 13. Useful viva commands

```bash
# Cluster and nodes
minikube profile list
minikube status -p laravel-three-tier
kubectl --context=laravel-three-tier get nodes -o wide

# Workloads
kubectl --context=laravel-three-tier -n laravel-three-tier get deployments
kubectl --context=laravel-three-tier -n laravel-three-tier get pods -o wide
kubectl --context=laravel-three-tier -n laravel-three-tier get services
kubectl --context=laravel-three-tier -n laravel-three-tier get pvc

# Inspect and debug
kubectl --context=laravel-three-tier -n laravel-three-tier describe deployment laravel
kubectl --context=laravel-three-tier -n laravel-three-tier describe pod POD_NAME
kubectl --context=laravel-three-tier -n laravel-three-tier logs deployment/laravel
kubectl --context=laravel-three-tier -n laravel-three-tier exec -it POD_NAME -- sh

# Rollout and scaling
kubectl --context=laravel-three-tier -n laravel-three-tier rollout status deployment/laravel
kubectl --context=laravel-three-tier -n laravel-three-tier rollout history deployment/laravel
kubectl --context=laravel-three-tier -n laravel-three-tier scale deployment/laravel --replicas=3

# Undo a rollout if a Deployment update caused a problem
kubectl --context=laravel-three-tier -n laravel-three-tier rollout undo deployment/laravel
```

---

## 14. Evidence to capture for the lab report

Capture screenshots with commands and outputs visible, but hide passwords, tokens, private keys, and other credentials.

1. `minikube profile list` and `kubectl --context=laravel-three-tier get nodes -o wide` showing three Ready nodes.
2. `kubectl ... get deployments` showing the web, Laravel, and MySQL Deployments.
3. `kubectl ... get pods -o wide` showing pod status and node placement.
4. `kubectl ... get services` showing the internal Services and `web-service` NodePort.
5. `kubectl ... get pvc` showing `mysql-pvc` as `Bound`.
6. Successful Laravel migration command/output.
7. `curl -i http://127.0.0.1:30080/` or browser output through the SSH tunnel.
8. Logs showing the app is running without database connection errors.

Add a short explanation **before each screenshot** in the report: state the command, what it configures or verifies, and what the expected output proves.

---

## 15. Cleanup: stop the lab without deleting its database

To stop the new Minikube profile but retain its cluster state:

```bash
minikube stop -p laravel-three-tier
```

To resume later:

```bash
minikube start -p laravel-three-tier
kubectl --context=laravel-three-tier -n laravel-three-tier get pods
```

Do not run `minikube delete -p laravel-three-tier` unless you intentionally want to remove this lab's cluster. Do not delete the PVC or prune Docker volumes unless you have verified the data is disposable.

---

## 16. Official documentation

- Docker Engine installation on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Minikube start and multi-node clusters: https://minikube.sigs.k8s.io/docs/start/
- Minikube multi-node documentation: https://minikube.sigs.k8s.io/docs/tutorials/multi_node/
- Kubernetes Deployments: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Kubernetes Services: https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes Persistent Volumes: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- Kubernetes Secrets: https://kubernetes.io/docs/concepts/configuration/secret/

## Final checklist

- [ ] Fresh Ubuntu EC2 is sized appropriately and disk space is sufficient.
- [ ] Docker Engine, kubectl, and Minikube are installed.
- [ ] The `laravel-three-tier` Minikube profile has three Ready nodes.
- [ ] `mysql-pvc` is Bound and MySQL is Ready.
- [ ] Laravel app and Nginx images are published and pull successfully.
- [ ] Laravel and Nginx Deployments have Ready replicas.
- [ ] Laravel uses `DB_HOST=mysql-service` and matching credentials.
- [ ] Laravel migrations complete successfully.
- [ ] `web-service` serves the Laravel application through NodePort or SSH tunnel.
- [ ] Evidence screenshots contain no credentials.
