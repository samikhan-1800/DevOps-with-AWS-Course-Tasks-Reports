# Jenkins CI/CD Pipeline for Kubernetes Multi-Node Deployment
## Laravel Three-Tier Application · Ubuntu EC2 · Docker Hub · Minikube · Kubernetes

This guide extends the existing Laravel CI/CD work by having **Jenkins build and publish container images, then deploy the three-tier application to Kubernetes**.

## 1. Goal and selected architecture

This guide uses one **fresh Ubuntu EC2 instance** to run Jenkins and a three-node Minikube cluster. This is the simplest lab arrangement because Jenkins can reach the Kubernetes API locally without exposing the API server to the internet.

Recommended starting EC2 size: **8 vCPUs, 32 GiB RAM, and at least 50 GiB EBS storage**. Running Jenkins, Docker builds, and three Kubernetes nodes together is resource-intensive. A smaller instance may work with lower node resources and fewer replicas, but can become slow or run out of memory/disk.

```text
GitHub: samikhan5055/laravel-project
                |
                v
             Jenkins
       1. Checkout source
       2. Validate/build
       3. Build Docker images
       4. Push images to Docker Hub
       5. Apply Kubernetes manifests
       6. Wait for rollout and test
                |
                v
      Minikube profile: laravel-three-tier
       +-------------------------------+
       | Control-plane node            |
       | Worker node m02               |
       | Worker node m03               |
       +-------------------------------+
                |
                v
      Kubernetes namespace: laravel-three-tier
       Nginx web Deployment (2 replicas)
                |
          laravel-service:9000
                |
       Laravel PHP-FPM Deployment (2 replicas)
                |
           mysql-service:3306
                |
       MySQL Deployment (1 replica) + PVC
```

The three tiers are:
- **Web:** Nginx and a Kubernetes `NodePort` Service.
- **Application:** Laravel running PHP-FPM.
- **Database:** MySQL with persistent storage.

> A three-node Minikube cluster on one EC2 instance demonstrates Kubernetes multi-node scheduling, but the nodes share the same EC2 host. It is not the same as three separate EC2 machines.

### Existing lab protection

- Repository: `https://github.com/samikhan5055/laravel-project.git`
- Branch: `main`
- Docker Hub namespace: `sami1800`
- Application image: `sami1800/laravel-app`
- Web image: `sami1800/laravel-web`
- Kubernetes profile used by this guide: `laravel-three-tier`
- Kubernetes namespace used by this guide: `laravel-three-tier`

The existing Kubernetes lab uses profile/namespace `laravel-k8s`. **Do not delete, rename, or overwrite that existing lab.** This guide deliberately uses a different profile and namespace.

---

## 2. EC2 security group

In the EC2 Security Group, allow:

| Port | Source | Purpose |
|---|---|---|
| TCP 22 | Your current public IP only | SSH |
| TCP 8080 | Your current public IP only | Jenkins web UI |
| TCP 30080 | Your current public IP only, only if direct NodePort access works | Application test |

Do not expose MySQL port 3306 or the Kubernetes API port publicly. If you use the SSH tunnel in Section 13, you do not need to open port 30080 to the internet.

Connect to the fresh instance:

```bash
ssh -i /path/to/your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Replace the key path and IP with your own values.

Check resources before installing anything:

```bash
nproc
free -h
df -h /
```

---

## 3. Install Docker Engine

Use Docker Engine from Docker's official Ubuntu repository:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y ca-certificates curl wget gnupg apt-transport-https \
  conntrack socat ebtables iptables git nano jq

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

sudo systemctl enable --now docker
sudo systemctl status docker --no-pager
sudo docker version
```

This guide uses `sudo docker ...`; it does not require adding your Linux user to the Docker group.

If Docker fails:

```bash
sudo systemctl restart docker
sudo journalctl -u docker --no-pager -n 100
```

---

## 4. Install Java and Jenkins

Install Java 21:

```bash
sudo apt install -y fontconfig openjdk-21-jre
java -version
```

Add the official Jenkins repository and install Jenkins LTS:

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins --no-pager
```

Allow Jenkins to invoke Docker using a narrow sudoers rule. Docker access is effectively root-level access, so only use this on a lab instance you control.

```bash
sudo visudo -f /etc/sudoers.d/jenkins-docker
```

Add this one line:

```text
jenkins ALL=(root) NOPASSWD: /usr/bin/docker
```

Save and exit. Validate the file:

```bash
sudo visudo -cf /etc/sudoers.d/jenkins-docker
sudo -u jenkins sudo /usr/bin/docker version
```

If the Docker executable is not at `/usr/bin/docker`, check it with `command -v docker` and use the actual path in the sudoers file.

Read the initial Jenkins password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open:

```text
http://YOUR_EC2_PUBLIC_IP:8080
```

Complete the Jenkins wizard and install suggested plugins. Confirm the **Pipeline**, **Git**, and **Credentials Binding** plugins are installed.

---

## 5. Install kubectl and Minikube

Install `kubectl`:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm -f kubectl
kubectl version --client
```

Install Minikube:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm -f minikube-linux-amd64
minikube version
```

Start a separate three-node profile. Start with these resources only if the EC2 instance has enough capacity:

```bash
minikube start -p laravel-three-tier \
  --driver=docker \
  --nodes=3 \
  --cpus=2 \
  --memory=3072 \
  --disk-size=15g \
  --kubernetes-version=stable
```

If the command fails for insufficient CPU/RAM, resize the EC2 instance or reduce per-node resources. Do not repeatedly retry the same configuration.

Verify nodes and profile:

```bash
minikube status -p laravel-three-tier
minikube profile list
kubectl --context=laravel-three-tier get nodes -o wide
```

All three nodes should be `Ready`. Do not change the old `laravel-k8s` profile.

---

## 6. Prepare the Kubernetes manifests in the repository

Clone the project on the EC2 instance as the `ubuntu` user:

```bash
cd ~
git clone https://github.com/samikhan5055/laravel-project.git
cd ~/laravel-project
git checkout main
git pull --ff-only
```

Inspect the Dockerfile and Nginx config before making assumptions about stage names or paths:

```bash
sed -n '1,260p' Dockerfile
find . -type f \( -name 'default.conf' -o -name 'nginx.conf' \) -print
```

The existing Docker build is expected to provide an `app` target for PHP-FPM. Confirm that the Dockerfile actually defines it. If not, use its real target name in the Jenkinsfile.

### 6.1 Create a Kubernetes-specific Nginx config

The Docker Compose configuration may use `fastcgi_pass app:9000;`. In Kubernetes, Nginx must use the Kubernetes Service DNS name `laravel-service:9000`.

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

### 6.2 Create a Kubernetes web Dockerfile

This standalone Nginx image copies Laravel's `public/` files into the web container. The PHP-FPM image must use the matching `/var/www/html` application path.

```bash
cat > Dockerfile.k8s-web <<'EOF'
FROM nginx:1.28-alpine
WORKDIR /var/www/html
COPY public/ /var/www/html/public/
COPY k8s/default.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
EOF
```

If the app image uses a different working directory, update both the Nginx `root` and `SCRIPT_FILENAME` configuration so they match the PHP-FPM container's path. If the Laravel frontend build creates public assets in a separate build stage, ensure those generated assets are present in `public/` before building the Nginx image.

Commit these files to the repository so Jenkins can check them out:

```bash
git status --short
git add k8s/default.conf Dockerfile.k8s-web
git commit -m "Add Kubernetes Nginx configuration"
git push origin main
```

If Git reports that your identity is not configured, set `user.name` and `user.email` to your own Git identity, then repeat the commit.

---

## 7. Add Kubernetes manifests

These files define the MySQL, Laravel, and Nginx tiers. Commit them with the Nginx config.

### 7.1 MySQL Deployment, Service, and persistent storage

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
    - port: 3306
      targetPort: 3306
EOF
```

### 7.2 Laravel Deployment and Service

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
          image: sami1800/laravel-app:initial
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

### 7.3 Nginx Deployment and Service

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
          image: sami1800/laravel-web:initial
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
    - port: 80
      targetPort: 80
      nodePort: 30080
EOF
```

### 7.4 Namespace and secrets

Do **not** commit credentials in YAML. Create the namespace and secrets on the cluster before Jenkins runs the pipeline.

Generate a separate random MySQL root password and save it with owner-only permissions:

```bash
MYSQL_ROOT_PASSWORD="$(openssl rand -hex 24)"
printf '%s' "$MYSQL_ROOT_PASSWORD" > "$HOME/.mysql-root-password-laravel-three-tier"
chmod 600 "$HOME/.mysql-root-password-laravel-three-tier"
```

Create namespace and secrets:

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

`APP_KEY` is normally required by Laravel. Generate one using the same app version and add it to `laravel-secret` as `APP_KEY`. For example, in a trusted checkout where dependencies are installed:

```bash
php artisan key:generate --show
```

Copy the generated value privately, then add it to the Secret without committing it:

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier create secret generic laravel-app-key \
  --from-literal=APP_KEY='PASTE_GENERATED_KEY_HERE'
```

Alternatively, add `APP_KEY` directly to `laravel-secret` at initial creation. If using a separate `laravel-app-key` secret, update the Laravel Deployment to include:

```yaml
          envFrom:
            - secretRef:
                name: laravel-secret
            - secretRef:
                name: laravel-app-key
```

The application key must be identical across all replicas. Do not paste it into a report, public repository, or screenshot.

Check secrets by name only; do not print their values:

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier get secrets
```

### 7.5 Commit the manifests

```bash
git add Dockerfile.k8s-web k8s/
git commit -m "Add Kubernetes three-tier manifests"
git push origin main
```

If you already committed some files in Section 6, Git will only commit the remaining changes.

---

## 8. Build and validate images manually once

Before Jenkins owns the deployment, verify the Docker builds work on the EC2 host.

```bash
cd ~/laravel-project

sudo docker build --target app -t sami1800/laravel-app:initial .
sudo docker build -f Dockerfile.k8s-web -t sami1800/laravel-web:initial .
sudo docker run --rm sami1800/laravel-web:initial nginx -t
```

If `--target app` fails, inspect `Dockerfile` for the actual PHP-FPM stage name. If `Dockerfile.k8s-web` fails, verify the `public/` directory exists and the Nginx config path is correct.

Push the initial images so the first Kubernetes deployment can pull them:

```bash
sudo docker login --username sami1800
sudo docker push sami1800/laravel-app:initial
sudo docker push sami1800/laravel-web:initial
```

Use a Docker Hub access token when possible. Enter it at the interactive prompt; do not include it in shell history.

---

## 9. Give Jenkins access to the Kubernetes API

Jenkins runs as the Linux user `jenkins`; the Minikube kubeconfig belongs to `ubuntu`. Do not give Jenkins unrestricted access to the whole host's home directory. Instead, create a flattened kubeconfig and add it to Jenkins as a **Secret file** credential.

Generate a kubeconfig for the new profile:

```bash
kubectl config view --raw --flatten --minify \
  --context=laravel-three-tier > "$HOME/kubeconfig-laravel-three-tier"
chmod 600 "$HOME/kubeconfig-laravel-three-tier"
```

Confirm the generated file references the expected cluster and context without displaying private key material:

```bash
KUBECONFIG="$HOME/kubeconfig-laravel-three-tier" kubectl config get-contexts
KUBECONFIG="$HOME/kubeconfig-laravel-three-tier" kubectl get nodes
```

**Important:** the API server address in this file must be reachable from the EC2 host and the Jenkins process. If `kubectl get nodes` fails using this flattened file, fix the kubeconfig/API reachability before continuing.

In Jenkins:
1. Open **Manage Jenkins → Credentials**.
2. Select the global (or appropriate folder) credentials domain.
3. Click **Add Credentials**.
4. Kind: **Secret file**.
5. Upload `kubeconfig-laravel-three-tier`.
6. ID: `kubeconfig-laravel-k8s`.
7. Description: `Kubeconfig for laravel-three-tier Minikube profile`.
8. Save.

The kubeconfig contains credentials that grant cluster access. Restrict who can read/use this Jenkins credential. For a more secure setup, create a Kubernetes ServiceAccount and narrowly scoped RBAC permissions rather than using a broad admin kubeconfig.

When finished adding the credential, securely remove the temporary local file if it is no longer needed:

```bash
rm -f "$HOME/kubeconfig-laravel-three-tier"
```

Do not delete the original Minikube files in `~/.minikube`.

---

## 10. Configure Docker Hub credentials in Jenkins

In Jenkins, open **Manage Jenkins → Credentials → Global → Add Credentials**:

- Kind: **Username with password**
- Username: `sami1800`
- Password: Docker Hub access token
- ID: `dockerhub-creds`
- Description: `Docker Hub push access for Laravel images`

The Jenkins pipeline will use this credential to push the two images. Never hard-code the Docker Hub token in the Jenkinsfile.

---

## 11. Add the Jenkins pipeline file

Create `Jenkinsfile.k8s` in the repository root. It uses a commit-specific tag, builds both images, pushes them, applies manifests, updates image tags, waits for rollout, runs Laravel migrations once, and performs a basic HTTP check.

Before using it, verify that the Dockerfile's PHP-FPM stage is named `app`. If not, replace `--target app` with the actual stage name. The smoke test is an HTTP response check; it does not replace application-specific automated tests.

```groovy
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        DOCKERHUB_USER = 'sami1800'
        APP_IMAGE = 'sami1800/laravel-app'
        WEB_IMAGE = 'sami1800/laravel-web'
        K8S_NAMESPACE = 'laravel-three-tier'
        K8S_CONTEXT = 'laravel-three-tier'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git status --short'
            }
        }

        stage('Validate Kubernetes manifests') {
            steps {
                sh '''
                    set -eu
                    test -f k8s/mysql.yaml
                    test -f k8s/laravel.yaml
                    test -f k8s/web.yaml
                    test -f Dockerfile.k8s-web
                    test -f k8s/default.conf
                    grep -q 'fastcgi_pass laravel-service:9000;' k8s/default.conf
                '''
            }
        }

        stage('Build Docker images') {
            steps {
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()
                }
                sh '''
                    set -eu
                    sudo docker build --target app \
                      -t "$APP_IMAGE:$IMAGE_TAG" .
                    sudo docker build -f Dockerfile.k8s-web \
                      -t "$WEB_IMAGE:$IMAGE_TAG" .
                    sudo docker run --rm "$WEB_IMAGE:$IMAGE_TAG" nginx -t
                '''
            }
        }

        stage('Push images to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_TOKEN'
                )]) {
                    sh '''
                        set +x
                        printf '%s' "$DH_TOKEN" | sudo docker login \
                          --username "$DH_USER" --password-stdin
                        set -e
                        sudo docker push "$APP_IMAGE:$IMAGE_TAG"
                        sudo docker push "$WEB_IMAGE:$IMAGE_TAG"
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig-laravel-k8s',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        kubectl --context="$K8S_CONTEXT" get nodes
                        kubectl --context="$K8S_CONTEXT" get namespace "$K8S_NAMESPACE"

                        kubectl --context="$K8S_CONTEXT" -n "$K8S_NAMESPACE" \
                          apply -f k8s/mysql.yaml
                        kubectl --context="$K8S_CONTEXT" -n "$K8S_NAMESPACE" \
                          apply -f k8s/laravel.yaml
                        kubectl --context="$K8S_CONTEXT" -n "$K8S_NAMESPACE" \
                          apply -f k8s/web.yaml

                        kubectl --context="$K8S_CONTEXT" -n "$K8S_NAMESPACE" \
                          set image deployment/laravel \
                          laravel="$APP_IMAGE:$IMAGE_TAG"
                        kubectl --context="$K8S_CONTEXT" -n "$K8S_NAMESPACE" \
                          set image deployment/web \
                          nginx="$WEB_IMAGE:$IMAGE_TAG"

                        kubectl --context="$K8S_CONTEXT" -n "$K8S_NAMESPACE" \
                          rollout status deployment/mysql --timeout=240s
                        kubectl --context="$K8S_CONTEXT" -n "$K8S_NAMESPACE" \
                          rollout status deployment/laravel --timeout=240s
                        kubectl --context="$K8S_CONTEXT" -n "$K8S_NAMESPACE" \
                          rollout status deployment/web --timeout=240s
                    '''
                }
            }
        }

        stage('Run Laravel migrations') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig-laravel-k8s',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        POD="$(kubectl --context="$K8S_CONTEXT" -n "$K8S_NAMESPACE" \
                          get pods -l app=laravel \
                          -o jsonpath='{.items[0].metadata.name}')"

                        test -n "$POD"
                        kubectl --context="$K8S_CONTEXT" -n "$K8S_NAMESPACE" \
                          exec "$POD" -- php artisan migrate --force
                    '''
                }
            }
        }

        stage('Verify deployment') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig-laravel-k8s',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        kubectl --context="$K8S_CONTEXT" -n "$K8S_NAMESPACE" \
                          get deployments,pods,services -o wide
                        kubectl --context="$K8S_CONTEXT" -n "$K8S_NAMESPACE" \
                          get pvc

                        # Test from the Jenkins/EC2 host through kubectl port-forward.
                        # Run a bounded background port-forward and always stop it afterward.
                        PORT=18088
                        kubectl --context="$K8S_CONTEXT" -n "$K8S_NAMESPACE" \
                          port-forward service/web-service "$PORT:80" \
                          --address 127.0.0.1 > port-forward.log 2>&1 &
                        PF_PID=$!
                        trap 'kill "$PF_PID" 2>/dev/null || true' EXIT

                        ATTEMPT=0
                        until curl --fail --silent --show-error \
                          "http://127.0.0.1:$PORT/" -o /tmp/laravel-k8s-response.html; do
                          ATTEMPT=$((ATTEMPT + 1))
                          if [ "$ATTEMPT" -ge 20 ]; then
                            cat port-forward.log
                            exit 1
                          fi
                          sleep 2
                        done

                        head -c 500 /tmp/laravel-k8s-response.html || true
                        rm -f /tmp/laravel-k8s-response.html
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                sudo docker logout >/dev/null 2>&1 || true
                rm -f port-forward.log
            '''
        }
    }
}
```

### Notes about this Jenkinsfile

- The pipeline uses the short Git commit hash as the image tag, so every build publishes a versioned image.
- The Kubernetes Secrets and namespace must exist before the first deployment.
- The `kubectl` context in the flattened kubeconfig must be named `laravel-three-tier`, as created above.
- The pipeline does not create the Minikube cluster. Cluster creation is a one-time infrastructure task.
- The database uses a persistent volume. A normal application redeployment should not delete or recreate the PVC.
- The migration step runs after the Laravel Deployment becomes ready and runs against one pod only.
- For production, add automated tests and a more specific health endpoint; the HTTP check here is a basic deployment smoke test.
- If a migration is not backward-compatible, use a migration/release strategy that avoids running incompatible application and schema versions concurrently.

Commit and push the Jenkinsfile:

```bash
git add Jenkinsfile.k8s
git commit -m "Add Jenkins Kubernetes deployment pipeline"
git push origin main
```

If you created the file on the EC2 host, push it to the repository before configuring the job.

---

## 12. Create the Jenkins Pipeline job

1. Open Jenkins at `http://YOUR_EC2_PUBLIC_IP:8080`.
2. Click **New Item**.
3. Name: `Laravel-Kubernetes-CI-CD`.
4. Select **Pipeline**, then click **OK**.
5. Under **Pipeline**, choose **Pipeline script from SCM**.
6. SCM: **Git**.
7. Repository URL: `https://github.com/samikhan5055/laravel-project.git`.
8. Branch specifier: `*/main`.
9. Script Path: `Jenkinsfile.k8s`.
10. Save.

If the repository is private, configure a Jenkins Git credential with read access. Do not place a GitHub personal access token directly in the repository URL.

Click **Build Now** and inspect **Console Output**.

To poll GitHub periodically, enable **Poll SCM** under Build Triggers with:

```text
H/5 * * * *
```

This checks roughly every five minutes. A GitHub webhook is more immediate but requires configuring the webhook endpoint and network access correctly.

---

## 13. Access the application

The Kubernetes Service exposes Nginx through NodePort 30080. With the Docker driver on a remote EC2 host, direct NodePort routing from the internet may not work in every setup.

### Recommended: SSH tunnel

On the EC2 host, you can manually test the Service using:

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier \
  port-forward service/web-service 8088:80 --address 127.0.0.1
```

Keep that process running. On your **local computer**, open a second terminal:

```bash
ssh -i /path/to/your-key.pem -L 8088:127.0.0.1:8088 ubuntu@YOUR_EC2_PUBLIC_IP
```

Open this URL in your local browser:

```text
http://127.0.0.1:8088/
```

If direct NodePort access is confirmed to work, you can instead try:

```text
http://YOUR_EC2_PUBLIC_IP:30080/
```

Only open port 30080 to your own IP in the Security Group if using direct access.

---

## 14. Verify and troubleshoot

### Check the cluster and workload

```bash
kubectl --context=laravel-three-tier get nodes -o wide
kubectl --context=laravel-three-tier -n laravel-three-tier get deployments
kubectl --context=laravel-three-tier -n laravel-three-tier get pods -o wide
kubectl --context=laravel-three-tier -n laravel-three-tier get services
kubectl --context=laravel-three-tier -n laravel-three-tier get pvc
```

Expected:
- Three Kubernetes nodes are Ready.
- `mysql` has one ready replica.
- `laravel` has two ready replicas.
- `web` has two ready replicas.
- `mysql-pvc` is Bound.
- `mysql-service` and `laravel-service` are internal ClusterIP Services.
- `web-service` exposes NodePort 30080.

### Common problems

**Jenkins cannot execute Docker**

```bash
sudo visudo -cf /etc/sudoers.d/jenkins-docker
sudo -u jenkins sudo /usr/bin/docker version
sudo systemctl status docker --no-pager
```

Check the Docker path in the sudoers rule. Do not solve this by changing Docker socket permissions to world-writable.

**Jenkins cannot access Kubernetes**

Test the exact kubeconfig file before uploading it to Jenkins:

```bash
KUBECONFIG="$HOME/kubeconfig-laravel-three-tier" kubectl get nodes
```

If the API server address is unreachable from the EC2 host, fix the kubeconfig/API reachability. Do not open the Kubernetes API to the public internet as a shortcut.

**`ImagePullBackOff`**

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier describe pod POD_NAME
```

Verify the commit-tagged images were pushed to Docker Hub and that the image repositories are public or the cluster has a valid image pull secret for private images.

**Nginx returns 502**

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier logs deployment/web --tail=100
kubectl --context=laravel-three-tier -n laravel-three-tier logs deployment/laravel --tail=100
kubectl --context=laravel-three-tier -n laravel-three-tier get endpoints laravel-service
```

Common causes include the wrong `fastcgi_pass` service name, mismatched document roots, PHP-FPM not listening on port 9000, or Laravel pods not ready.

**MySQL access denied**

The MySQL user's actual password must match `DB_PASSWORD` in the Laravel Secret. Updating a Kubernetes Secret does not change an existing MySQL user's password in a previously initialized data directory. Do not delete the PVC to work around this. Inspect MySQL logs and correct the database account password deliberately.

**Laravel does not start**

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier describe pod POD_NAME
kubectl --context=laravel-three-tier -n laravel-three-tier logs POD_NAME --previous
```

Check for missing `vendor/`, invalid `APP_KEY`, file permissions, incorrect environment variables, and database connectivity.

**EC2 runs out of disk space**

```bash
df -h /
sudo docker system df
```

Increase EBS storage or remove only known, disposable build artifacts. Do not run `docker system prune --volumes`, `docker volume prune`, or commands that may delete database data.

---

## 15. Evidence to capture for the lab report

Capture these screenshots, hiding credentials and tokens:

1. Jenkins job configuration showing Git repository, branch `main`, and script path `Jenkinsfile.k8s`.
2. Successful Jenkins console output for checkout, image build, Docker Hub push, deployment, and verification.
3. `kubectl get nodes -o wide` showing three Ready nodes.
4. `kubectl get deployments,pods -o wide` showing the three application tiers and pod placement.
5. `kubectl get services` showing the internal Services and Nginx NodePort.
6. `kubectl get pvc` showing the database PVC Bound.
7. Successful Laravel migration output.
8. The running application in a browser via SSH tunnel or the confirmed NodePort URL.

Write a short explanation before each screenshot in the report: the command or Jenkins stage, what it does, and what the output proves.

---

## 16. Quick viva commands

```bash
# Jenkins
sudo systemctl status jenkins --no-pager
sudo journalctl -u jenkins --no-pager -n 100

# Docker images
sudo docker image ls | grep 'sami1800/laravel'

# Minikube and nodes
minikube profile list
minikube status -p laravel-three-tier
kubectl --context=laravel-three-tier get nodes -o wide

# Deployments and Services
kubectl --context=laravel-three-tier -n laravel-three-tier get deployments
kubectl --context=laravel-three-tier -n laravel-three-tier get pods -o wide
kubectl --context=laravel-three-tier -n laravel-three-tier get services
kubectl --context=laravel-three-tier -n laravel-three-tier get pvc

# Rollout status
kubectl --context=laravel-three-tier -n laravel-three-tier rollout status deployment/laravel
kubectl --context=laravel-three-tier -n laravel-three-tier rollout history deployment/laravel

# Scale Laravel
kubectl --context=laravel-three-tier -n laravel-three-tier scale deployment/laravel --replicas=3

# Logs
kubectl --context=laravel-three-tier -n laravel-three-tier logs deployment/laravel --tail=100
kubectl --context=laravel-three-tier -n laravel-three-tier logs deployment/web --tail=100
```

---

## 17. Cleanup and safety

Stop the Minikube profile while retaining its state:

```bash
minikube stop -p laravel-three-tier
```

Start it again later:

```bash
minikube start -p laravel-three-tier
kubectl --context=laravel-three-tier -n laravel-three-tier get pods
```

Do not delete the profile or PVC unless you intentionally want to remove the lab and its database data. Do not disturb the separate `laravel-k8s` profile.

---

## 18. Official documentation

- Docker Engine on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Jenkins installation: https://www.jenkins.io/doc/book/installing/linux/
- Minikube start: https://minikube.sigs.k8s.io/docs/start/
- Minikube multi-node: https://minikube.sigs.k8s.io/docs/tutorials/multi_node/
- Kubernetes Deployments: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Kubernetes Services: https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes Secrets: https://kubernetes.io/docs/concepts/configuration/secret/
- Kubernetes Persistent Volumes: https://kubernetes.io/docs/concepts/storage/persistent-volumes/

## Final checklist

- [ ] Jenkins and Docker are running.
- [ ] Minikube profile `laravel-three-tier` has three Ready nodes.
- [ ] Jenkins has `dockerhub-creds` and `kubeconfig-laravel-k8s` credentials.
- [ ] `Jenkinsfile.k8s` is committed on branch `main`.
- [ ] Kubernetes-specific Nginx config uses `laravel-service:9000`.
- [ ] Docker images build and push with commit-specific tags.
- [ ] MySQL, Laravel, and Nginx Deployments become Ready.
- [ ] MySQL PVC is Bound and survives application redeployments.
- [ ] Laravel migrations complete successfully.
- [ ] Jenkins HTTP smoke test passes.
- [ ] The application is reachable through the SSH tunnel or confirmed NodePort.
