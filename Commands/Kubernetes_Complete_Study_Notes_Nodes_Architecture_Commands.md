# Kubernetes Complete Study Notes
## Core Concepts, Cluster Architecture, Nodes, Pods, Networking, Storage, Deployments, and Practical Commands

**Purpose:** These notes explain Kubernetes from the fundamentals through practical commands, with special attention to what each node contains, how the control plane manages worker nodes, and how a three-tier Laravel application runs on a multi-node cluster.

**Lab context used in examples**
- Kubernetes: Minikube multi-node cluster
- Minikube profile: `laravel-three-tier`
- Namespace: `laravel-three-tier`
- Application tiers: Nginx, Laravel PHP-FPM, MySQL
- Application image examples: `sami1800/laravel-app`, `sami1800/laravel-web`

> Commands using `--context=laravel-three-tier` assume your Minikube context is named `laravel-three-tier`. Check your actual context with `kubectl config get-contexts` and substitute the correct name if needed. Commands are examples; they do not assume that a resource already exists.

---

## Table of contents

1. What Kubernetes is
2. Why Kubernetes is used
3. Kubernetes architecture at a glance
4. Control plane components
5. Worker node components
6. What each node contains
7. Nodes, pods, containers, and workloads
8. Kubernetes objects and YAML manifests
9. Three-tier application architecture
10. Pod scheduling and node placement
11. Services and networking
12. ConfigMaps, Secrets, and environment variables
13. Persistent storage and database data
14. Deployments, ReplicaSets, and rolling updates
15. Health checks and self-healing
16. Resource requests, limits, and scheduling
17. Install kubectl and inspect a cluster
18. Create a multi-node Minikube cluster
19. Deploy and inspect a three-tier example
20. Essential kubectl commands explained
21. Troubleshooting workflow
22. Jenkins and Kubernetes CI/CD
23. Common errors and misconceptions
24. Viva questions and answers
25. Quick command cheat sheet
26. Safe lab cleanup
27. Official documentation

---

## 1. What is Kubernetes?

**Kubernetes (K8s)** is an open-source container orchestration platform. It manages containerized applications across a group of machines called **nodes**.

Docker can build and run individual containers. Kubernetes manages applications made of containers across a cluster, including:

- Scheduling pods onto nodes.
- Restarting failed containers.
- Maintaining the desired number of replicas.
- Providing service discovery and load balancing.
- Connecting workloads to persistent storage.
- Performing rolling updates and rollbacks.
- Managing application configuration and Secrets.
- Monitoring pod health through probes.

### Simple example

Without Kubernetes, you might manually start two Laravel containers and one Nginx container. If one Laravel container crashes, you must detect and restart it yourself.

With Kubernetes, you define a desired state such as “run two Laravel replicas.” Kubernetes continually compares actual state with desired state and takes corrective action where possible.

### Declarative management

Kubernetes manifests normally describe **what you want**, rather than listing every action required to reach that state.

Example:

```yaml
spec:
  replicas: 2
```

For a Deployment, this says the desired number of pod replicas is two. Kubernetes controllers attempt to maintain that state.

---

## 2. Why is Kubernetes used?

| Capability | Meaning |
|---|---|
| Scheduling | Chooses a suitable node for each pod. |
| Self-healing | Recreates failed pods managed by controllers. |
| Scaling | Adjusts the number of replicas. |
| Service discovery | Gives workloads a stable way to reach other workloads. |
| Load balancing | Distributes Service traffic across eligible pod endpoints. |
| Rolling updates | Replaces old application replicas with new versions gradually. |
| Rollback | Can restore a prior Deployment revision. |
| Storage integration | Connects pods to volumes and persistent storage. |
| Configuration | Supplies settings through ConfigMaps, Secrets, and environment variables. |
| Resource controls | Uses CPU and memory requests and limits. |

Kubernetes is not a replacement for application design, database backups, secure credentials, monitoring, or testing. It orchestrates workloads; it cannot automatically fix every application-level defect.

---

## 3. Kubernetes architecture at a glance

A Kubernetes cluster contains a **control plane** and one or more **worker nodes**.

```text
                    Administrator / CI pipeline
                              |
                          kubectl
                              |
                              v
                  +-------------------------+
                  |       CONTROL PLANE     |
                  |                         |
                  | kube-apiserver          |
                  | etcd                    |
                  | kube-scheduler          |
                  | kube-controller-manager |
                  | (cloud controller,      |
                  |  when applicable)       |
                  +------------+------------+
                               |
                    Kubernetes API / state
                               |
             +-----------------+------------------+
             |                                    |
             v                                    v
    +----------------------+             +----------------------+
    | WORKER NODE 1        |             | WORKER NODE 2        |
    | kubelet              |             | kubelet              |
    | container runtime    |             | container runtime    |
    | kube-proxy*          |             | kube-proxy*          |
    |                      |             |                      |
    | Pod: Nginx           |             | Pod: Laravel         |
    | Pod: Laravel         |             | Pod: MySQL           |
    +----------------------+             +----------------------+

* kube-proxy is commonly used, but some cluster networking implementations
  replace its Service-routing role with another implementation.
```

The diagram is conceptual. Exact component placement depends on the cluster type. In a small Minikube cluster, the control-plane node can also run workloads unless scheduling is restricted. In managed or production clusters, control-plane components are commonly separated from application worker nodes.

---

## 4. Control plane components

The control plane makes cluster-wide decisions and exposes the Kubernetes API.

### 4.1 kube-apiserver

**Purpose:** The front door to the Kubernetes API.

It receives API requests from tools such as `kubectl`, authenticates and authorizes requests, validates objects, and provides the interface used by cluster components.

Example:

```bash
kubectl get nodes
```

What happens:
1. `kubectl` sends a request to the Kubernetes API server.
2. The API server processes the request and retrieves the relevant cluster state.
3. `kubectl` formats the response for the terminal.

### 4.2 etcd

**Purpose:** Stores Kubernetes cluster state.

It stores configuration and state for Kubernetes objects, such as Deployments, Services, Secrets, and node records. It is a critical control-plane data store.

Important:
- etcd is not the application's MySQL database.
- Do not manually edit etcd as a normal way to administer Kubernetes.
- Backups of etcd matter for self-managed production clusters.
- Managed Kubernetes providers generally manage the control plane and etcd for customers.

### 4.3 kube-scheduler

**Purpose:** Selects a suitable node for a newly created pod that has not yet been assigned to a node.

It considers factors such as:
- CPU and memory requests.
- Node readiness and available capacity.
- Node selectors, affinity, and anti-affinity.
- Taints and tolerations.
- Topology-spread constraints.
- Other scheduling policies.

The scheduler chooses a node; the kubelet on that node then works to start the pod.

### 4.4 kube-controller-manager

**Purpose:** Runs controllers that reconcile actual state with desired state.

Examples include:
- A Deployment requests three replicas.
- A pod is deleted or a node fails.
- A controller observes that fewer replicas exist than requested.
- The controller creates replacement pods, subject to scheduling and cluster constraints.

Controllers do not guarantee success if the cluster lacks resources, an image cannot be pulled, storage is unavailable, or the application cannot start.

### 4.5 cloud-controller-manager

**Purpose:** Integrates Kubernetes with a cloud provider when that component is used.

Depending on the provider and cluster setup, it can manage cloud-specific node information, routes, or load balancers. It is not necessarily present in a local Minikube cluster.

---

## 5. Worker node components

A **worker node** is a machine that runs application pods. It can be a physical machine, virtual machine, or a container-based node in a lab such as Minikube.

### 5.1 kubelet

The kubelet is the node agent. It communicates with the API server and ensures that the pods assigned to its node are running as specified.

Typical responsibilities:
- Receives pod specifications for its node.
- Requests container operations through the container runtime.
- Runs configured health probes.
- Reports node and pod status.
- Mounts volumes required by pods.

### 5.2 Container runtime

The container runtime runs containers. Common Kubernetes runtimes include `containerd` and CRI-O. Minikube's Docker driver uses Docker containers to represent nodes, but the Kubernetes node's container runtime arrangement depends on the Minikube configuration and version.

The runtime pulls images, creates containers, and manages their lifecycle through the Kubernetes container runtime interface (CRI).

### 5.3 kube-proxy or equivalent networking implementation

In many clusters, kube-proxy programs networking rules that implement Kubernetes Service routing. Some clusters use a networking implementation that replaces this function.

Do not assume that every cluster must have a visible kube-proxy pod.

### 5.4 CNI network plugin

The Container Network Interface (CNI) implementation provides pod networking. It helps assign network connectivity and enables pods to communicate across nodes.

Examples include Calico and Cilium; Minikube commonly uses its configured cluster networking implementation.

### 5.5 Pods and application containers

Worker nodes host pods. Each pod contains one or more containers that share certain resources, including the pod network namespace and any volumes mounted into those containers.

A node does **not** have to contain one particular application tier. Kubernetes schedules pods based on resource requirements and scheduling rules.

---

## 6. What does each node contain?

This is one of the most important Kubernetes concepts.

### 6.1 General answer

A node normally contains:
- The operating environment for that node.
- kubelet.
- A container runtime.
- Networking components or their equivalents.
- The pods scheduled onto that node.
- The containers inside those pods.
- Any volumes mounted for the pods, backed by the relevant storage system.

A node does **not** automatically contain a fixed tier such as “Nginx node,” “Laravel node,” or “MySQL node.” A pod can be scheduled onto any suitable node unless you configure placement rules.

### 6.2 Example three-node cluster

Suppose the cluster has:
- `laravel-three-tier` — control-plane node.
- `laravel-three-tier-m02` — worker node.
- `laravel-three-tier-m03` — worker node.

A possible pod placement might look like this:

| Node | Kubernetes components | Example application pods |
|---|---|---|
| Control-plane node | Control-plane components, kubelet, runtime, networking | Nginx pod |
| Worker node m02 | kubelet, runtime, networking | Laravel pod 1, MySQL pod |
| Worker node m03 | kubelet, runtime, networking | Nginx pod 2, Laravel pod 2 |

This is only an example, **not a guaranteed placement**. Kubernetes may place the pods differently depending on resource availability and scheduling rules.

### 6.3 Why might all pods run on one node?

Possible reasons:
- The cluster only has one schedulable node.
- Other nodes are NotReady or have taints the pods do not tolerate.
- Pods do not fit the other nodes' available resources.
- The scheduler's placement decisions happen to choose one node.
- The cluster has node selectors or affinity rules that constrain placement.

Check actual placement:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
```

The `NODE` column shows which node hosts each pod.

### 6.4 Does a control-plane node run application pods?

It can, depending on configuration. In many clusters, the control-plane node has a taint that prevents ordinary application pods from being scheduled there. Small Minikube clusters may allow workloads on the control-plane node.

Inspect node taints:

```bash
kubectl describe node NODE_NAME
```

Look for the `Taints` field. Do not remove control-plane taints in a production cluster just to force application placement.

### 6.5 Are nodes separate physical machines?

Not necessarily:
- In a cloud cluster, nodes are often separate virtual machines.
- In an on-premises cluster, nodes may be separate physical servers or VMs.
- In Minikube's Docker driver, multiple Kubernetes nodes are represented by containers on one host.

A three-node Minikube lab proves multi-node Kubernetes behavior, but it does not provide the same fault isolation as three independent machines.

---

## 7. Nodes, pods, containers, and workloads

These terms are related but not interchangeable.

| Term | Definition | Example |
|---|---|---|
| Cluster | All nodes and Kubernetes control-plane services managed as one cluster | A Minikube cluster |
| Node | A machine that runs pods | `laravel-three-tier-m02` |
| Pod | The smallest deployable Kubernetes unit | One Laravel replica |
| Container | A running image instance inside a pod | A PHP-FPM process container |
| Deployment | Manages a desired number of replicated pods and updates them | `deployment/laravel` |
| ReplicaSet | Maintains a requested number of pod replicas for a Deployment revision | Created by a Deployment |
| Service | Stable network endpoint for selected pods | `laravel-service` |
| Namespace | A logical grouping of Kubernetes resources | `laravel-three-tier` |
| PVC | A request for persistent storage | `mysql-pvc` |
| Secret | Stores sensitive configuration data in Kubernetes | `mysql-secret` |

### Relationship

```text
Cluster
  ├── Node 1
  │    ├── Pod A
  │    │    └── Container(s)
  │    └── Pod B
  │         └── Container(s)
  ├── Node 2
  │    └── Pod C
  │         └── Container(s)
  └── Node 3
       └── Pod D
            └── Container(s)
```

A pod is not a node. A Deployment is not a container. A Service is not a pod: it provides a stable way to reach matching pods.

---

## 8. Kubernetes objects and YAML manifests

Kubernetes objects are usually defined in YAML files and applied using `kubectl`.

Example minimal Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
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
          image: nginx:stable
          ports:
            - containerPort: 80
```

### Explanation of important fields

- `apiVersion`: API group and version used for the object.
- `kind`: object type, here `Deployment`.
- `metadata.name`: name of the object.
- `spec.replicas`: desired number of pod replicas.
- `selector.matchLabels`: how the Deployment identifies pods it manages.
- `template`: the pod template used to create replicas.
- `template.metadata.labels`: labels applied to created pods.
- `containers`: containers to run inside each pod.
- `image`: container image to pull and run.
- `containerPort`: documents the port the container is expected to listen on; it does not by itself expose that port outside the pod.

Apply and inspect:

```bash
kubectl apply -f deployment.yaml
kubectl get deployment web
kubectl describe deployment web
```

- `apply` creates or updates the declared object.
- `get` provides a concise status.
- `describe` shows detailed information and events useful for troubleshooting.

---

## 9. Three-tier application architecture in Kubernetes

A three-tier web application commonly has:

### Tier 1: Web / presentation tier — Nginx

Responsibilities:
- Accepts HTTP requests.
- Serves static assets.
- Forwards PHP requests to Laravel PHP-FPM.
- Is exposed through a Kubernetes Service.

Typical resources:
- `Deployment/web`
- `Service/web-service` (NodePort or LoadBalancer, depending on environment)

### Tier 2: Application tier — Laravel PHP-FPM

Responsibilities:
- Executes Laravel application logic.
- Handles routes, validation, business rules, and database queries.
- Connects to MySQL through an internal Service.

Typical resources:
- `Deployment/laravel`
- `Service/laravel-service` (ClusterIP)

### Tier 3: Database tier — MySQL

Responsibilities:
- Stores application data.
- Accepts database connections from Laravel.
- Uses persistent storage so pod replacement does not automatically erase database files.

Typical resources:
- `Deployment/mysql` or, where appropriate, a `StatefulSet`
- `Service/mysql-service` (ClusterIP)
- `PersistentVolumeClaim/mysql-pvc`
- `Secret/mysql-secret`

### Request flow

```text
Browser
  |
  v
web-service (NodePort or LoadBalancer)
  |
  v
Nginx pod
  |
  | FastCGI to laravel-service:9000
  v
Laravel PHP-FPM pod
  |
  | MySQL protocol to mysql-service:3306
  v
MySQL pod
  |
  v
Persistent volume
```

MySQL should generally remain internal to the cluster. Do not expose its port publicly for a normal web application.

---

## 10. Pod scheduling and node placement

The scheduler places pending pods on suitable nodes. It considers resources and scheduling rules; it does not promise to evenly distribute pods unless constraints are configured.

### See the nodes

```bash
kubectl get nodes -o wide
```

Shows node name, status, role, Kubernetes version, internal IP, and OS/image information.

### See pod placement

```bash
kubectl get pods -A -o wide
```

Shows pods across namespaces and the node each pod uses.

For a specific namespace:

```bash
kubectl -n laravel-three-tier get pods -o wide
```

### Inspect a node

```bash
kubectl describe node NODE_NAME
```

Shows labels, taints, capacity, allocatable resources, conditions, pods assigned to the node, and events.

### Request replicas

```bash
kubectl -n laravel-three-tier scale deployment/laravel --replicas=3
```

Asks Kubernetes to maintain three Laravel pod replicas. It does not mean that each replica will run on a different node.

### When placement must be controlled

Use the following features where needed:
- **Node selector:** selects nodes with particular labels.
- **Node affinity:** expresses more flexible node-selection preferences or requirements.
- **Pod affinity/anti-affinity:** influences whether pods run near or apart from other pods.
- **Topology spread constraints:** help spread replicas across nodes or zones.
- **Taints and tolerations:** restrict which pods can schedule onto tainted nodes.

Avoid hard-coding placement rules unless you have a clear requirement; strict rules can leave pods Pending if no node satisfies them.

---

## 11. Services and networking

Pod IP addresses are not stable identities. Pods may be recreated and receive different IPs. A Kubernetes Service provides a stable endpoint for a group of pods selected by labels.

### 11.1 ClusterIP

The default Service type. It exposes a service internally within the cluster.

Typical use:
- `laravel-service` for PHP-FPM.
- `mysql-service` for MySQL.

Example Laravel environment settings:

```text
DB_HOST=mysql-service
DB_PORT=3306
```

Kubernetes DNS resolves the Service name within the cluster namespace.

### 11.2 NodePort

Exposes a Service on a port across cluster nodes. A configured NodePort is commonly in the range 30000–32767, with the default range depending on cluster configuration.

Example:

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

The NodePort is not the same as the container port:
- `containerPort`: the port the container is expected to listen on.
- `targetPort`: the port to which Service traffic is sent.
- `port`: the Service's own port.
- `nodePort`: the port exposed on cluster nodes.

A NodePort in Minikube on a remote EC2 host may not be directly reachable from the internet when using the Docker driver. A port-forward or SSH tunnel may be required.

### 11.3 LoadBalancer

Requests a load-balancer integration from the environment. Managed cloud clusters can often provision a cloud load balancer. A local Minikube cluster does not automatically have a public cloud load balancer unless extra support is configured.

### 11.4 Inspect Services and endpoints

```bash
kubectl -n laravel-three-tier get services
kubectl -n laravel-three-tier get endpoints
kubectl -n laravel-three-tier get endpointslices
```

If a Service has no endpoints, inspect its selector and pod labels:

```bash
kubectl -n laravel-three-tier describe service laravel-service
kubectl -n laravel-three-tier get pods --show-labels
```

The Service selector must match labels on ready pods.

---

## 12. ConfigMaps, Secrets, and environment variables

### ConfigMap

A ConfigMap stores non-sensitive configuration, such as feature flags or application settings.

### Secret

A Kubernetes Secret stores sensitive configuration, such as database credentials or an application key. Secret values are base64-encoded in standard YAML representations; base64 is not encryption. Configure appropriate RBAC and cluster encryption at rest where required.

Create a Secret from literals (example values only):

```bash
kubectl -n laravel-three-tier create secret generic laravel-db-secret \
  --from-literal=DB_HOST=mysql-service \
  --from-literal=DB_PORT=3306 \
  --from-literal=DB_DATABASE=laravel \
  --from-literal=DB_USERNAME=laraveluser \
  --from-literal=DB_PASSWORD='REPLACE_WITH_YOUR_PASSWORD'
```

This command creates a Secret. It does not update an already existing Secret with the same name. To update, use a deliberate apply/replace workflow appropriate to your environment.

Reference a Secret in a pod template:

```yaml
envFrom:
  - secretRef:
      name: laravel-db-secret
```

Never commit real passwords, access tokens, kubeconfigs, or private keys to Git. Avoid printing Secret values in terminal screenshots and build logs.

---

## 13. Persistent storage and database data

Containers and pods can be replaced. If database files live only in a container's writable layer, replacing the pod can lose that data. Persistent storage is used for data that must survive pod replacement.

### Key storage terms

- **PersistentVolume (PV):** a cluster storage resource.
- **PersistentVolumeClaim (PVC):** a workload's request for storage.
- **StorageClass:** describes how storage can be provisioned.
- **Volume mount:** makes storage available at a path inside a container.

Example PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

The example omits a namespace so that it can be placed in a file applied to a chosen namespace. In the Laravel lab, resources should be created in `laravel-three-tier`.

Inspect storage:

```bash
kubectl -n laravel-three-tier get pvc
kubectl get pv
kubectl get storageclass
kubectl -n laravel-three-tier describe pvc mysql-pvc
```

- `Bound`: the claim has matched/provisioned storage.
- `Pending`: inspect StorageClass, provisioner, capacity, access mode, and events.

**Safety:** Do not delete a database PVC as a troubleshooting shortcut. Deleting a claim can result in data loss depending on the storage configuration and reclaim policy. Persistent storage is not a substitute for backups.

---

## 14. Deployments, ReplicaSets, and rolling updates

### Deployment

A Deployment manages stateless replicated pods and supports rolling updates.

Example:

```bash
kubectl -n laravel-three-tier get deployments
kubectl -n laravel-three-tier describe deployment laravel
```

### ReplicaSet

A ReplicaSet maintains a requested number of pod replicas. Deployments typically create and manage ReplicaSets for revisions.

```bash
kubectl -n laravel-three-tier get replicasets
```

### Scaling

```bash
kubectl -n laravel-three-tier scale deployment/laravel --replicas=3
```

The command changes the desired replica count to three. Kubernetes then schedules additional pods if resources and constraints allow.

### Rolling update

When a Deployment's pod template changes (for example, the image tag), Kubernetes replaces pods according to its update strategy.

```bash
kubectl -n laravel-three-tier set image deployment/laravel \
  laravel=sami1800/laravel-app:1.1

kubectl -n laravel-three-tier rollout status deployment/laravel
kubectl -n laravel-three-tier rollout history deployment/laravel
```

The example image tag must actually exist in the registry.

### Rollback

```bash
kubectl -n laravel-three-tier rollout undo deployment/laravel
```

This requests a rollback to the previous Deployment revision if one is available. It does not automatically roll back database schema migrations or external data changes.

---

## 15. Health checks and self-healing

Kubernetes supports probes to assess containers.

### Liveness probe

Checks whether a container should be restarted. Use it to detect a container that is stuck and cannot recover on its own.

### Readiness probe

Checks whether a container is ready to receive traffic. A pod that is not ready is generally excluded from Service endpoints.

### Startup probe

Gives slow-starting containers additional time to start before liveness/readiness checks take effect.

Example HTTP readiness probe:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

A probe must match the actual application. For example, a PHP-FPM process does not necessarily serve HTTP directly, so a TCP probe on port 9000 may be more appropriate than an HTTP probe for the Laravel container.

Inspect status and events:

```bash
kubectl -n laravel-three-tier get pods
kubectl -n laravel-three-tier describe pod POD_NAME
kubectl -n laravel-three-tier logs POD_NAME
kubectl -n laravel-three-tier logs POD_NAME --previous
```

`--previous` requests logs from the previous terminated container instance, if available.

---

## 16. Resource requests, limits, and scheduling

### Requests

Requests represent the resources Kubernetes uses when deciding whether a pod fits on a node. They are also relevant to resource allocation and scheduling.

### Limits

Limits cap resource consumption for supported resources. A container that exceeds its memory limit may be terminated with an OOM-related status. CPU limits can throttle CPU use.

Example:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

- `100m` CPU means one-tenth of one CPU core.
- `256Mi` means 256 mebibytes of memory.
- If requests are too high for all available nodes, pods can remain Pending.
- If limits are too low, workloads may be throttled or terminated.

Inspect resources:

```bash
kubectl top nodes
kubectl top pods -A
```

These commands require metrics support to be installed and available in the cluster. If metrics are unavailable, inspect `kubectl describe node` and workload events instead.

---

## 17. Install kubectl and inspect a cluster

`kubectl` is the command-line client used to interact with the Kubernetes API.

### Check client version

```bash
kubectl version --client
```

Shows the installed client version. It does not by itself prove the cluster is reachable.

### List contexts

```bash
kubectl config get-contexts
kubectl config current-context
```

A context selects a cluster, user credentials, and optionally a default namespace.

### List nodes

```bash
kubectl get nodes
kubectl get nodes -o wide
```

Shows node readiness and additional node information.

### List all pods in all namespaces

```bash
kubectl get pods -A -o wide
```

This is useful for identifying system pods, application pods, and the node where each pod runs.

### Inspect cluster information

```bash
kubectl cluster-info
```

Shows Kubernetes control-plane and core service endpoints known to the current context.

### Check node details

```bash
kubectl describe node NODE_NAME
```

Shows labels, taints, capacity, allocatable resources, conditions, assigned pods, and recent events.

---

## 18. Create a multi-node Minikube cluster

These commands are for a fresh lab environment. If you already have a cluster, inspect it first rather than creating another profile unintentionally.

### 18.1 Install Docker

Install Docker Engine using the official Docker Ubuntu installation guide. Then verify:

```bash
sudo systemctl enable --now docker
sudo docker version
sudo docker info
```

This guide uses `sudo docker`; it does not require Docker group membership changes.

### 18.2 Install Minikube and kubectl

Follow the official Minikube and Kubernetes installation instructions for your operating system and CPU architecture. Confirm the commands are available:

```bash
minikube version
kubectl version --client
```

### 18.3 Start three nodes

```bash
minikube start -p laravel-three-tier \
  --driver=docker \
  --nodes=3 \
  --cpus=2 \
  --memory=3072 \
  --disk-size=15g
```

What each flag means:
- `-p laravel-three-tier`: names the Minikube profile.
- `--driver=docker`: uses Docker to run the Minikube node containers.
- `--nodes=3`: requests a three-node cluster.
- `--cpus=2`: assigns the configured CPU count to each node, subject to driver behavior and host resources.
- `--memory=3072`: configures approximately 3 GiB of memory per node.
- `--disk-size=15g`: configures node disk size.

The EC2 host needs enough CPU, RAM, and disk for the nodes and the application. If startup fails, check:

```bash
nproc
free -h
df -h /
sudo docker system df
```

### 18.4 Verify nodes

```bash
minikube status -p laravel-three-tier
kubectl --context=laravel-three-tier get nodes -o wide
```

The node names and roles depend on the profile and Minikube version. Confirm the actual output rather than assuming a specific suffix or role label.

---

## 19. Deploy and inspect a three-tier example

This section shows the order and the inspection commands for a Laravel three-tier cluster. It assumes images and manifests have already been prepared and pushed to a registry.

### 19.1 Create the namespace

```bash
kubectl --context=laravel-three-tier create namespace laravel-three-tier
```

A namespace groups the resources for this lab. If it already exists, Kubernetes reports that; inspect it instead of recreating it.

### 19.2 Apply the database tier

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier \
  apply -f k8s/mysql.yaml
```

This creates/updates the MySQL Deployment, internal Service, and persistent volume claim defined in the file.

Check:

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier get pods,pvc,services
kubectl --context=laravel-three-tier -n laravel-three-tier rollout status deployment/mysql
```

Wait for MySQL to become ready before running application migrations.

### 19.3 Apply the Laravel tier

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier \
  apply -f k8s/laravel.yaml
```

This creates/updates the Laravel Deployment and internal PHP-FPM Service.

Check:

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier rollout status deployment/laravel
kubectl --context=laravel-three-tier -n laravel-three-tier get pods -o wide
```

### 19.4 Apply the web tier

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier \
  apply -f k8s/web.yaml
```

This creates/updates the Nginx Deployment and web Service.

Check:

```bash
kubectl --context=laravel-three-tier -n laravel-three-tier rollout status deployment/web
kubectl --context=laravel-three-tier -n laravel-three-tier get services
```

### 19.5 Run migrations

Run Laravel migrations from one application pod after the database is ready:

```bash
POD="$(kubectl --context=laravel-three-tier -n laravel-three-tier \
  get pods -l app=laravel -o jsonpath='{.items[0].metadata.name}')"

kubectl --context=laravel-three-tier -n laravel-three-tier \
  exec "$POD" -- php artisan migrate --force
```

This executes the migration command inside one running Laravel container. It does not require running the migration independently on every replica.

### 19.6 Verify all resources

```bash
kubectl --context=laravel-three-tier get nodes -o wide
kubectl --context=laravel-three-tier -n laravel-three-tier get deployments
kubectl --context=laravel-three-tier -n laravel-three-tier get pods -o wide
kubectl --context=laravel-three-tier -n laravel-three-tier get services
kubectl --context=laravel-three-tier -n laravel-three-tier get pvc
```

Expected high-level state:
- Nodes are `Ready`.
- Web and Laravel Deployments have their desired ready replicas.
- MySQL has one ready replica.
- MySQL PVC is `Bound`.
- Nginx is exposed through the intended Service.
- Laravel and MySQL Services are internal.

### 19.7 Access the app

For Minikube, obtain a service URL:

```bash
minikube -p laravel-three-tier service web-service \
  -n laravel-three-tier --url
```

On a remote EC2 host, the URL may not be directly reachable from your local browser. Use a port-forward on the EC2 host and an SSH tunnel from your local computer if needed.

---

## 20. Essential kubectl commands explained

### Resource listing

```bash
kubectl get nodes
kubectl get namespaces
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get replicasets
kubectl get pv
kubectl get pvc
```

`get` lists resources of the specified type in the current namespace unless `-A` is used or a namespace is specified.

### Describe a resource

```bash
kubectl describe pod POD_NAME
kubectl describe node NODE_NAME
kubectl describe deployment laravel
kubectl describe service web-service
```

`describe` gives detailed state, configuration, conditions, and recent events. It is often the best first step when a resource is not behaving as expected.

### Read logs

```bash
kubectl logs POD_NAME
kubectl logs POD_NAME -c CONTAINER_NAME
kubectl logs POD_NAME --previous
kubectl logs deployment/laravel --tail=100
```

Logs show application output from a container. Use `-c` when a pod has multiple containers.

### Execute a command inside a container

```bash
kubectl exec -it POD_NAME -- sh
kubectl exec POD_NAME -- php artisan --version
```

`exec` runs a command in a container. `-it` opens an interactive terminal; `--` separates kubectl options from the command being executed inside the container.

### Apply or delete a manifest

```bash
kubectl apply -f k8s/laravel.yaml
kubectl delete -f k8s/laravel.yaml
```

`apply` creates or updates resources from the file. `delete` removes the resources defined in the file. Do not use `delete` casually for database PVCs or persistent data.

### Watch resources

```bash
kubectl get pods -w
kubectl rollout status deployment/laravel
```

`-w` watches for changes. `rollout status` waits for the Deployment rollout to complete or report a failure.

### Use a namespace

```bash
kubectl -n laravel-three-tier get pods
```

`-n` is shorthand for `--namespace`; it scopes the command to that namespace.

### Use a context

```bash
kubectl --context=laravel-three-tier get nodes
```

`--context` selects the cluster/user context for the command. It is useful when several clusters are configured.

---

## 21. Troubleshooting workflow

When something fails, use this order rather than changing random configuration.

### Step 1: Check the high-level state

```bash
kubectl get nodes -o wide
kubectl -n laravel-three-tier get deployments,pods,services,pvc
```

### Step 2: Inspect the failed resource

```bash
kubectl -n laravel-three-tier describe pod POD_NAME
```

Read the **Events** section for image, scheduling, volume, probe, or resource errors.

### Step 3: Inspect logs

```bash
kubectl -n laravel-three-tier logs POD_NAME
kubectl -n laravel-three-tier logs POD_NAME --previous
```

### Step 4: Check the image and pull status

```bash
kubectl -n laravel-three-tier describe pod POD_NAME
```

`ImagePullBackOff` often means the image/tag is wrong, the repository is private without credentials, or the registry cannot be reached.

### Step 5: Check Service selectors and endpoints

```bash
kubectl -n laravel-three-tier get pods --show-labels
kubectl -n laravel-three-tier describe service laravel-service
kubectl -n laravel-three-tier get endpoints laravel-service
```

If there are no endpoints, the Service selector may not match the pod labels or no selected pods are ready.

### Step 6: Check storage

```bash
kubectl -n laravel-three-tier get pvc
kubectl -n laravel-three-tier describe pvc mysql-pvc
```

Do not delete the PVC to clear an unrelated pod error.

### Common statuses

| Status/error | Typical meaning | First action |
|---|---|---|
| `Pending` | Pod has not been scheduled or required resources/storage are unavailable | `describe pod` |
| `ImagePullBackOff` | Image cannot be pulled | Check image name, tag, credentials |
| `CrashLoopBackOff` | Container repeatedly exits | Inspect current and previous logs |
| `Running` but `0/1 Ready` | Container runs but has not passed readiness | Inspect probes and application logs |
| `Evicted` | Node pressure caused pod eviction | Inspect node resources and events |
| `OOMKilled` | Container was terminated due to memory pressure/limit | Review memory usage and limits |
| `PVC Pending` | Storage has not been bound | Inspect StorageClass and PVC events |
| `502 Bad Gateway` | Web tier cannot successfully reach the application tier | Check Nginx config, PHP-FPM, Service endpoints |
| MySQL `Access denied` | Credentials do not match the existing MySQL account | Check Secret consistency and database account |

---

## 22. Jenkins and Kubernetes CI/CD

Jenkins can automate building images and deploying them to Kubernetes.

Typical flow:

1. Checkout the GitHub repository.
2. Run tests and validate manifests.
3. Build Laravel and Nginx images.
4. Tag images with a commit identifier.
5. Push images to Docker Hub or another registry.
6. Authenticate to the Kubernetes API using a protected credential.
7. Apply manifests and update Deployment image tags.
8. Wait for rollouts.
9. Run database migrations safely.
10. Run an HTTP smoke test.

Jenkins does not automatically create a Kubernetes cluster just because a pipeline uses `kubectl`. The cluster must already exist, and Jenkins must be configured with working API connectivity and appropriate permissions.

### Example deployment commands used by a pipeline

```bash
kubectl -n laravel-three-tier apply -f k8s/mysql.yaml
kubectl -n laravel-three-tier apply -f k8s/laravel.yaml
kubectl -n laravel-three-tier apply -f k8s/web.yaml

kubectl -n laravel-three-tier set image deployment/laravel \
  laravel=sami1800/laravel-app:COMMIT_TAG

kubectl -n laravel-three-tier rollout status deployment/laravel
kubectl -n laravel-three-tier rollout status deployment/web
```

Replace `COMMIT_TAG` with a tag that exists in the image registry. Avoid using mutable tags without a release strategy. Do not store Docker Hub tokens or kubeconfig credentials in source control.

---

## 23. Common errors and misconceptions

### “A node is the same as a pod.”

Incorrect. A node is a machine; a pod is a workload scheduled onto a node.

### “Every node contains one tier.”

Incorrect. Kubernetes can schedule any eligible pod on any suitable node unless scheduling rules constrain placement.

### “Three Minikube nodes mean three separate servers.”

Not when using the Docker driver on one host. The nodes are represented by containers sharing the host's underlying resources.

### “A Deployment creates a Service automatically.”

Incorrect. A Deployment manages replicated pods. A Service is a separate resource and must be created if stable networking is needed.

### “containerPort exposes a port publicly.”

Incorrect. `containerPort` describes a port in the pod specification. Exposure requires a Service, Ingress, port-forward, or another network configuration.

### “A Pod IP is permanent.”

Incorrect. Pods can be replaced and their IPs can change. Use Services for stable access.

### “A Kubernetes Secret is encrypted because it is base64.”

Incorrect. Base64 is encoding, not encryption. Access control and encryption-at-rest configuration matter.

### “Deleting a pod deletes the database permanently.”

Not necessarily. A Deployment may recreate a pod, and a PVC can retain data independently of a pod. However, data survival depends on the storage configuration and deletion/reclaim policies.

### “Kubernetes automatically makes an application highly available.”

Not by itself. Replicas, node distribution, resource capacity, readiness checks, storage design, and application behavior all matter. Two replicas on the same host do not provide full host-level fault isolation.

---

## 24. Viva questions and answers

**Q1. What is Kubernetes?**  
Kubernetes is a container orchestration platform that schedules and manages containerized workloads across a cluster.

**Q2. What is a Kubernetes cluster?**  
A cluster is the collection of control-plane services and nodes that Kubernetes manages together.

**Q3. What is the difference between a control plane and a worker node?**  
The control plane manages the cluster's desired state and scheduling. Worker nodes run application pods.

**Q4. What does kube-apiserver do?**  
It exposes the Kubernetes API used by kubectl and other cluster components.

**Q5. What is etcd?**  
It is the key-value data store for Kubernetes cluster state; it is not the application's MySQL database.

**Q6. What does kube-scheduler do?**  
It chooses a suitable node for each unscheduled pod.

**Q7. What does kubelet do?**  
It runs on each node and helps ensure the pods assigned to that node are running as specified.

**Q8. What is a pod?**  
A pod is the smallest deployable Kubernetes unit and contains one or more closely related containers.

**Q9. What is a Deployment?**  
A Deployment manages replicated pods and supports rolling updates and rollbacks.

**Q10. What is a Service?**  
A Service provides a stable network endpoint for a group of pods selected by labels.

**Q11. What is the difference between ClusterIP and NodePort?**  
ClusterIP exposes a Service internally. NodePort exposes it through a port on cluster nodes.

**Q12. Why is a PVC used for MySQL?**  
It requests persistent storage so database files can survive ordinary pod replacement, depending on the storage configuration.

**Q13. How can you see which node hosts a pod?**  
Run `kubectl get pods -o wide`; inspect the `NODE` column.

**Q14. How can you see why a pod failed?**  
Run `kubectl describe pod POD_NAME` and inspect its events, then review logs.

**Q15. What does `CrashLoopBackOff` mean?**  
A container repeatedly exits and Kubernetes is delaying restart attempts. Check logs and configuration.

**Q16. What does `ImagePullBackOff` mean?**  
Kubernetes could not pull the specified container image. Check the registry, tag, credentials, and connectivity.

**Q17. Does scaling Laravel to three replicas guarantee one replica per node?**  
No. Scheduling rules must be configured if a specific distribution is required.

**Q18. Can Jenkins deploy to Kubernetes?**  
Yes. Jenkins can build and push images, then use a secured Kubernetes API credential to apply manifests and update Deployments.

**Q19. Does Minikube require separate physical servers for multiple nodes?**  
No. With the Docker driver, multiple Minikube nodes can run as containers on a single host.

**Q20. Why should MySQL normally not be exposed through a public NodePort?**  
It is an internal data service and should be accessible only to authorized workloads or administrators through a secured path.

---

## 25. Quick command cheat sheet

| Goal | Command |
|---|---|
| List nodes | `kubectl get nodes -o wide` |
| List all pods | `kubectl get pods -A -o wide` |
| List namespace pods | `kubectl -n laravel-three-tier get pods -o wide` |
| List Deployments | `kubectl -n laravel-three-tier get deployments` |
| List Services | `kubectl -n laravel-three-tier get services` |
| List PVCs | `kubectl -n laravel-three-tier get pvc` |
| Describe a pod | `kubectl -n laravel-three-tier describe pod POD_NAME` |
| Read pod logs | `kubectl -n laravel-three-tier logs POD_NAME` |
| Previous container logs | `kubectl -n laravel-three-tier logs POD_NAME --previous` |
| Run a command in a pod | `kubectl -n laravel-three-tier exec POD_NAME -- COMMAND` |
| Scale Laravel | `kubectl -n laravel-three-tier scale deployment/laravel --replicas=3` |
| Wait for rollout | `kubectl -n laravel-three-tier rollout status deployment/laravel` |
| View rollout history | `kubectl -n laravel-three-tier rollout history deployment/laravel` |
| Roll back Deployment | `kubectl -n laravel-three-tier rollout undo deployment/laravel` |
| Inspect Service endpoints | `kubectl -n laravel-three-tier get endpoints` |
| List contexts | `kubectl config get-contexts` |
| Check Minikube profiles | `minikube profile list` |
| Check Minikube status | `minikube status -p laravel-three-tier` |

---

## 26. Safe lab cleanup

To stop the Minikube profile but retain its state:

```bash
minikube stop -p laravel-three-tier
```

To start it again:

```bash
minikube start -p laravel-three-tier
kubectl --context=laravel-three-tier -n laravel-three-tier get pods
```

Do not run a cluster deletion command unless you intentionally want to remove that profile. Do not delete the MySQL PVC or prune Docker volumes as routine cleanup. Keep the existing `laravel-k8s` profile separate.

---

## 27. Official documentation

Use official documentation for current installation instructions and version-specific behavior:

- Kubernetes concepts: https://kubernetes.io/docs/concepts/
- Cluster architecture: https://kubernetes.io/docs/concepts/architecture/
- Nodes: https://kubernetes.io/docs/concepts/architecture/nodes/
- Pods: https://kubernetes.io/docs/concepts/workloads/pods/
- Deployments: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Services and networking: https://kubernetes.io/docs/concepts/services-networking/service/
- Persistent volumes: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- Secrets: https://kubernetes.io/docs/concepts/configuration/secret/
- kubectl reference: https://kubernetes.io/docs/reference/kubectl/
- Minikube: https://minikube.sigs.k8s.io/docs/
- Minikube multi-node: https://minikube.sigs.k8s.io/docs/tutorials/multi_node/

---

## Final revision checklist

- [ ] I can explain what Kubernetes does and why container orchestration is useful.
- [ ] I can distinguish the control plane from worker nodes.
- [ ] I can explain kube-apiserver, etcd, scheduler, controller manager, kubelet, and the container runtime.
- [ ] I know a node hosts pods and does not automatically correspond to one application tier.
- [ ] I can use `kubectl get pods -o wide` to identify pod placement.
- [ ] I can explain pods, containers, Deployments, ReplicaSets, Services, namespaces, Secrets, PVs, and PVCs.
- [ ] I can explain ClusterIP, NodePort, and LoadBalancer.
- [ ] I can deploy the Nginx, Laravel, and MySQL tiers in the correct order.
- [ ] I can inspect failures using `describe`, `logs`, and resource events.
- [ ] I can explain why a multi-node Minikube cluster on one host is not the same as independent servers.
- [ ] I understand that Kubernetes access credentials and database storage must be protected.
