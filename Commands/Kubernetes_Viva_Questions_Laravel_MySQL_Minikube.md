# Kubernetes Viva Questions and Answers
## Laravel + MySQL on a Three-Node Minikube Cluster (AWS EC2)

> **Purpose:** A study guide for explaining the Kubernetes lab during a viva.
>
> **Project context:** An Ubuntu EC2 instance hosts a Minikube cluster named `laravel-k8s`, with one control-plane node and two worker nodes. Laravel runs as two replicas using `sami1800/laravel-app:1.0`; MySQL 8.0 runs with persistent storage. Kubernetes resources are deployed in the `laravel-k8s` namespace.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Kubernetes Fundamentals](#2-kubernetes-fundamentals)
3. [Cluster Nodes and Scheduling](#3-cluster-nodes-and-scheduling)
4. [Pods, Deployments, and ReplicaSets](#4-pods-deployments-and-replicasets)
5. [Services and Networking](#5-services-and-networking)
6. [ConfigMaps and Secrets](#6-configmaps-and-secrets)
7. [Persistent Storage and MySQL](#7-persistent-storage-and-mysql)
8. [Laravel and PHP-FPM](#8-laravel-and-php-fpm)
9. [Minikube on AWS EC2](#9-minikube-on-aws-ec2)
10. [Troubleshooting and Commands](#10-troubleshooting-and-commands)
11. [Architecture and Request Flow](#11-architecture-and-request-flow)
12. [Limitations and Important Distinctions](#12-limitations-and-important-distinctions)
13. [Rapid-Fire Viva Questions](#13-rapid-fire-viva-questions)
14. [Practical Demonstration Checklist](#14-practical-demonstration-checklist)

---

## 1. Project Overview

### Q1. What is the objective of this project?
**Answer:** To deploy a Laravel application and a MySQL database in Kubernetes on an AWS EC2 Ubuntu instance, using a three-node Minikube cluster. The lab demonstrates container orchestration, replicas, service discovery, configuration management, and persistent storage.

### Q2. What technologies did you use?
**Answer:** AWS EC2, Ubuntu, Docker, Minikube, Kubernetes, Laravel, PHP-FPM, MySQL 8.0, Kubernetes YAML manifests, ConfigMaps, Secrets, Services, Deployments, and a PersistentVolumeClaim.

### Q3. What is the cluster configuration?
**Answer:** The Minikube profile is `laravel-k8s`. It contains one control-plane node and two worker nodes: `laravel-k8s-m02` and `laravel-k8s-m03`.

### Q4. What is deployed in the cluster?
**Answer:** Two Laravel application Pods and one MySQL Pod, managed by Deployments. A Service exposes Laravel internally on port `9000`, and another exposes MySQL internally on port `3306`. MySQL uses a PVC for persistent storage.

### Q5. Why use Kubernetes instead of running containers manually?
**Answer:** Kubernetes provides declarative management, scheduling, service discovery, replica management, health monitoring, and recovery from certain container or Pod failures.

### Q6. What is the namespace used for?
**Answer:** The `laravel-k8s` namespace groups this lab's Kubernetes resources and helps separate them from resources in other namespaces.

---

## 2. Kubernetes Fundamentals

### Q7. What is Kubernetes?
**Answer:** Kubernetes is an open-source container orchestration platform. It automates deployment, scheduling, scaling, networking, and recovery of containerized applications.

### Q8. What is a Kubernetes cluster?
**Answer:** A cluster is a group of machines—physical or virtual—that run Kubernetes workloads. It includes a control plane and worker nodes.

### Q9. What is the control plane?
**Answer:** The control plane manages the cluster's desired state. Key components include the API server, scheduler, controller manager, and cluster data store (etcd).

### Q10. What does the API server do?
**Answer:** The API server is the main entry point for Kubernetes API requests. Tools such as `kubectl` communicate with the cluster through it.

### Q11. What does the scheduler do?
**Answer:** The scheduler selects a suitable node for a newly created Pod based on resource requests, constraints, and scheduling policies.

### Q12. What does the controller manager do?
**Answer:** It runs controllers that compare the current cluster state with the desired state and take action to reconcile differences.

### Q13. What is etcd?
**Answer:** etcd is the distributed key-value store used by Kubernetes to persist cluster state and configuration.

### Q14. What is a worker node?
**Answer:** A worker node runs application Pods. Node components include a kubelet, a container runtime, and networking components.

### Q15. What is a declarative configuration?
**Answer:** It describes the desired state—for example, two Laravel replicas—rather than listing every action needed to create and maintain that state. Kubernetes controllers work to maintain that state.

### Q16. What is a Kubernetes manifest?
**Answer:** A manifest is usually a YAML file that describes a Kubernetes resource, such as a Deployment, Service, ConfigMap, or PVC.

### Q17. What is `kubectl`?
**Answer:** `kubectl` is the command-line client used to inspect and manage Kubernetes resources.

---

## 3. Cluster Nodes and Scheduling

### Q18. Why did you configure three nodes?
**Answer:** The cluster has one control-plane node and two workers, which demonstrates scheduling workloads across multiple worker nodes.

### Q19. Which nodes ran the Laravel replicas?
**Answer:** In the observed deployment, one Laravel Pod ran on `laravel-k8s-m02` and the other on `laravel-k8s-m03`.

### Q20. Why were both Laravel Pods not on the same node?
**Answer:** Kubernetes scheduled the Pods on separate workers in the observed state. This provides a useful demonstration of multi-node placement, but without explicit anti-affinity or topology-spread rules, Kubernetes does not guarantee that replicas will always be on different nodes.

### Q21. Does a worker's `ROLES` column showing `<none>` mean it is not a worker?
**Answer:** No. The `ROLES` column is a label-based display. A node can be a worker even if no role label is displayed. Check its `STATUS`, labels, and workloads.

### Q22. How can you list the nodes?
**Answer:**
```bash
sudo kubectl get nodes -o wide
```

### Q23. How do you see which node is running each Pod?
**Answer:**
```bash
sudo kubectl get pods -n laravel-k8s -o wide
```

### Q24. What happens if a worker node becomes unavailable?
**Answer:** Kubernetes can attempt to recreate or reschedule managed Pods on healthy eligible nodes, provided resources and storage constraints allow it. A single-replica database with a local or node-constrained volume may need additional recovery considerations.

---

## 4. Pods, Deployments, and ReplicaSets

### Q25. What is a Pod?
**Answer:** A Pod is Kubernetes' smallest deployable unit. It contains one or more closely related containers that share network identity and can share storage.

### Q26. What is a Deployment?
**Answer:** A Deployment manages stateless application replicas through ReplicaSets and supports updates and rollbacks.

### Q27. What is a ReplicaSet?
**Answer:** A ReplicaSet ensures that the requested number of matching Pod replicas are running. Deployments normally manage ReplicaSets for application rollouts.

### Q28. Why did you use two Laravel replicas?
**Answer:** Two replicas provide multiple application instances and allow Kubernetes to maintain the requested replica count. They can improve availability and capacity, although they do not guarantee high availability by themselves.

### Q29. What does `READY 2/2` mean for the Laravel Deployment?
**Answer:** Two desired Pods are ready out of two desired replicas.

### Q30. What does `Running` mean for a Pod?
**Answer:** The Pod has been bound to a node and its containers have been created. `Running` alone does not always mean the application is ready to serve traffic; readiness checks and application tests are also important.

### Q31. What is a rolling update?
**Answer:** A rolling update gradually replaces old application Pods with new ones while aiming to keep the application available.

### Q32. What does this command do?
```bash
sudo kubectl rollout restart deployment/laravel -n laravel-k8s
```
**Answer:** It triggers a restart of the Deployment's Pods by updating the Pod template. It does not rebuild the Docker image.

### Q33. How do you check rollout completion?
**Answer:**
```bash
sudo kubectl rollout status deployment/laravel -n laravel-k8s --timeout=120s
```

### Q34. How do you inspect Pod logs?
**Answer:**
```bash
sudo kubectl logs -n laravel-k8s deployment/laravel
```
For a specific Pod, use its name. If it has multiple containers, specify the container with `-c`.

---

## 5. Services and Networking

### Q35. What is a Kubernetes Service?
**Answer:** A Service provides a stable network endpoint for a group of Pods selected by labels. This means clients do not need to track changing Pod IP addresses.

### Q36. Why do Pods need a Service?
**Answer:** Pod IP addresses can change when Pods are recreated. A Service provides a stable name and virtual IP to reach matching Pods.

### Q37. What is `laravel-service`?
**Answer:** It is a ClusterIP Service that exposes the Laravel PHP-FPM Pods on port `9000` inside the cluster.

### Q38. What is `mysql-service`?
**Answer:** It is a ClusterIP Service that exposes the MySQL Pod on port `3306` inside the cluster.

### Q39. What is a ClusterIP Service?
**Answer:** ClusterIP is the default Service type. It makes a Service reachable from within the cluster, rather than directly exposing it to the public internet.

### Q40. What is the difference between `port`, `targetPort`, and `nodePort`?
**Answer:**
- `port`: The port exposed by the Service.
- `targetPort`: The port on the selected Pod/container to which traffic is forwarded.
- `nodePort`: An optional port exposed on each node when using a NodePort Service.

### Q41. How does Kubernetes discover the MySQL database?
**Answer:** The Laravel configuration uses `DB_HOST=mysql-service`. Kubernetes DNS resolves that Service name within the namespace.

### Q42. What are Service endpoints?
**Answer:** Endpoints identify the Pod IPs and ports selected by a Service. In this lab, `laravel-service` had two endpoints on port `9000`, and `mysql-service` had one endpoint on port `3306`.

### Q43. How do you inspect Services and endpoints?
**Answer:**
```bash
sudo kubectl get services -n laravel-k8s
sudo kubectl get endpoints -n laravel-k8s
```
On newer Kubernetes versions, EndpointSlices are preferred:
```bash
sudo kubectl get endpointslices -n laravel-k8s
```

### Q44. Does a ClusterIP Service make Laravel publicly accessible?
**Answer:** No. ClusterIP is internal to the cluster. External access would require an additional entry point, such as an Ingress with a suitable controller, a LoadBalancer Service supported by the environment, or a NodePort, along with any required network and firewall configuration.

### Q45. Why is Nginx commonly used with PHP-FPM?
**Answer:** Nginx handles HTTP requests and serves static files, while PHP-FPM executes PHP scripts. Nginx forwards PHP requests to PHP-FPM using FastCGI.

---

## 6. ConfigMaps and Secrets

### Q46. What is a ConfigMap?
**Answer:** A ConfigMap stores non-confidential configuration separately from an application image, such as the database host, database name, or connection port.

### Q47. What is a Secret?
**Answer:** A Secret stores sensitive configuration, such as passwords or application keys, separately from ordinary configuration. Base64 encoding in a Secret manifest is not encryption by itself.

### Q48. Why separate configuration from the image?
**Answer:** It allows the same image to be deployed in different environments with different configuration values, without rebuilding the image for each environment.

### Q49. Which database configuration values does Laravel need?
**Answer:** Typically, Laravel needs the connection driver, database host, port, database name, username, and password. In this lab, the host is `mysql-service`, the port is `3306`, and the database name is `laravel`.

### Q50. Why can a Secret update fail to fix a database login?
**Answer:** Updating a Kubernetes Secret changes the value injected into newly created or restarted Pods, but it does not automatically change the password of a MySQL user already stored in an initialized database. The Secret and the actual MySQL account must agree.

### Q51. Should credentials be committed to GitHub?
**Answer:** No. Keep real credentials out of source control. Use protected secret-management methods and restrict access to Kubernetes Secrets.

### Q52. How do you inspect configuration resources without displaying secret values?
**Answer:**
```bash
sudo kubectl get configmap -n laravel-k8s
sudo kubectl get secrets -n laravel-k8s
```
Avoid printing decoded Secret values in terminal output or screenshots.

---

## 7. Persistent Storage and MySQL

### Q53. Why does MySQL need persistent storage?
**Answer:** A container's writable layer is not suitable as the only place for important database data because replacing the container can lose that data. A persistent volume provides storage that exists independently of the container lifecycle.

### Q54. What is a PersistentVolumeClaim (PVC)?
**Answer:** A PVC is a request for storage with specified requirements, such as capacity and access mode. Kubernetes binds it to a suitable PersistentVolume.

### Q55. What does `Bound` mean for `mysql-pvc`?
**Answer:** The claim has successfully bound to a PersistentVolume that satisfies its request.

### Q56. What storage did this lab request?
**Answer:** The observed `mysql-pvc` requested `5Gi` of storage with the `ReadWriteOnce` access mode and was bound using the `standard` StorageClass.

### Q57. What does `ReadWriteOnce` mean?
**Answer:** It allows a volume to be mounted read-write by a single node at a time. It does not necessarily mean that only one Pod can use the volume, because multiple Pods on the same node may be able to mount it depending on the volume implementation.

### Q58. Will the database survive deletion of the MySQL Pod?
**Answer:** The PVC is intended to preserve data when the Pod is recreated, as long as the underlying volume and its lifecycle remain intact. Actual durability depends on the storage provisioner, volume type, and retention configuration.

### Q59. Is Minikube's standard storage equivalent to production-grade managed database storage?
**Answer:** No. Minikube storage is useful for learning and testing. Production deployments should carefully consider durable storage, backups, recovery, availability, security, and database operations.

### Q60. Why is MySQL running as one replica?
**Answer:** The current lab uses one MySQL Pod and one PVC to keep the setup straightforward. This is not a highly available MySQL architecture. Simply increasing the MySQL replica count would not create a correct database cluster.

### Q61. How do you check the PVC?
**Answer:**
```bash
sudo kubectl get pvc -n laravel-k8s
sudo kubectl describe pvc mysql-pvc -n laravel-k8s
```

---

## 8. Laravel and PHP-FPM

### Q62. What is Laravel?
**Answer:** Laravel is a PHP web application framework. It provides features such as routing, middleware, database access, migrations, validation, and application structure.

### Q63. What is PHP-FPM?
**Answer:** PHP FastCGI Process Manager manages PHP worker processes and serves PHP scripts over FastCGI. It is commonly used with Nginx.

### Q64. Why does `laravel-service` use port `9000`?
**Answer:** Port `9000` is the conventional port used by PHP-FPM in many configurations. It is a FastCGI endpoint, not an ordinary HTTP website endpoint.

### Q65. Can I open the PHP-FPM Service directly in a browser?
**Answer:** No. A browser sends HTTP requests, while PHP-FPM expects FastCGI requests. An HTTP web server such as Nginx is normally placed in front of PHP-FPM.

### Q66. What is a Laravel migration?
**Answer:** A migration is a version-controlled PHP file that describes a database schema change, such as creating a table or adding a column.

### Q67. What does `php artisan migrate --force` do?
**Answer:** It runs pending Laravel database migrations without the interactive production confirmation. Use it only after verifying the database connection and confirming the migration is intended.

### Q68. What does an SQL `Access denied` error usually indicate?
**Answer:** It generally indicates an authentication or authorization problem: the username, password, allowed MySQL host, or grants may be incorrect. Network connectivity can still be working even when authentication fails.

### Q69. How do you run a Laravel command in a Pod?
**Answer:** First select a current running Pod, then use `kubectl exec`. For example:
```bash
sudo kubectl get pods -n laravel-k8s
sudo kubectl exec -n laravel-k8s <laravel-pod-name> -- php artisan migrate:status
```
Replace `<laravel-pod-name>` with an actual current Pod name. Do not run a migration until the database connection works and the migration is appropriate.

---

## 9. Minikube on AWS EC2

### Q70. What is Minikube?
**Answer:** Minikube runs a Kubernetes cluster for local development, learning, and testing. It can run a multi-node cluster using supported drivers.

### Q71. Why use Minikube on EC2?
**Answer:** EC2 provides a remote Ubuntu virtual machine on which the lab can be run and demonstrated. Minikube supplies the Kubernetes cluster inside that environment.

### Q72. Is Minikube the same as Amazon EKS?
**Answer:** No. Minikube is designed primarily for development and learning. Amazon EKS is AWS's managed Kubernetes service, where AWS manages important parts of the control plane.

### Q73. What does the Minikube profile name mean?
**Answer:** A profile identifies a particular Minikube cluster configuration. This lab uses the profile `laravel-k8s`.

### Q74. How do you check the Minikube cluster status?
**Answer:**
```bash
sudo minikube --profile=laravel-k8s status
```

### Q75. Why might commands use `sudo` in this lab?
**Answer:** The environment was configured to use Docker and Kubernetes commands with elevated privileges. The commands shown match that setup; they do not mean every Kubernetes installation must use `sudo`.

### Q76. Does three-node Minikube automatically provide production high availability?
**Answer:** No. A three-node learning cluster demonstrates multi-node scheduling, but it does not automatically provide production-grade control-plane redundancy, durable shared storage, secure external access, or disaster recovery.

---

## 10. Troubleshooting and Commands

### Q77. How do you get an overview of resources?
**Answer:**
```bash
sudo kubectl get all -n laravel-k8s -o wide
```

### Q78. How do you inspect a Deployment?
**Answer:**
```bash
sudo kubectl describe deployment laravel -n laravel-k8s
```

### Q79. How do you inspect a Pod when it is failing?
**Answer:**
```bash
sudo kubectl describe pod <pod-name> -n laravel-k8s
sudo kubectl logs <pod-name> -n laravel-k8s
```
Use the actual Pod name from `kubectl get pods`.

### Q80. What is the difference between `logs` and `describe`?
**Answer:** `logs` displays output from a container. `describe` displays resource details and events, such as scheduling problems, failed mounts, image-pull errors, and probe failures.

### Q81. What does `ImagePullBackOff` mean?
**Answer:** Kubernetes cannot successfully pull the required image and is retrying with a delay. Possible causes include a wrong image name or tag, registry authentication, network problems, or a missing image.

### Q82. What does `CrashLoopBackOff` mean?
**Answer:** A container repeatedly exits and Kubernetes is delaying subsequent restart attempts. Check its logs, startup command, configuration, dependencies, and probes.

### Q83. What does `Pending` mean for a Pod?
**Answer:** The Pod has not reached the running state. It may be waiting for scheduling, sufficient resources, storage, or another required condition. Use `describe pod` to find the reason.

### Q84. What if a Service has no endpoints?
**Answer:** Check whether its selector matches Pod labels, whether the Pods exist, and whether the Pods are ready.

### Q85. How do you check recent events?
**Answer:**
```bash
sudo kubectl get events -n laravel-k8s --sort-by=.lastTimestamp
```

### Q86. How do you validate YAML before applying it?
**Answer:** Review the YAML structure and resource fields, then use a server-side dry run when appropriate:
```bash
sudo kubectl apply --dry-run=server -f k8s/
```
This validates supported resources against the API server without persisting the changes.

### Q87. What does `kubectl apply` do?
**Answer:** It creates or updates resources to match the configuration in the supplied manifests.

### Q88. What is the difference between `kubectl delete pod` and deleting a Deployment?
**Answer:** Deleting a Pod managed by a Deployment usually causes the Deployment's ReplicaSet to create a replacement. Deleting the Deployment removes that controller and its managed Pods, unless other resources or policies affect deletion.

---

## 11. Architecture and Request Flow

### Q89. Explain the intended request flow.
**Answer:** A typical Laravel deployment uses this flow:

1. A client sends an HTTP request to an HTTP entry point such as Nginx.
2. Nginx serves static assets where configured and forwards PHP requests using FastCGI.
3. Nginx reaches the `laravel-service` Service on port `9000`.
4. The Service forwards the request to one of the ready Laravel PHP-FPM Pods.
5. Laravel connects to `mysql-service` on port `3306` when it needs database access.
6. MySQL reads or writes database data on its persistent volume.

**Important:** In the observed cluster output, the Laravel and MySQL Services existed, but no Kubernetes Nginx Deployment or web Service was shown. The Nginx portion describes the intended architecture if Nginx is deployed into Kubernetes; it should not be claimed as an already-running Kubernetes component without verifying it.

### Q90. Why is database access internal?
**Answer:** The MySQL Service is a ClusterIP Service, so it is intended to be reached from within the cluster. Exposing the database publicly would increase the attack surface and is generally unnecessary for this architecture.

### Q91. Why use a Service instead of putting a Pod IP in Laravel configuration?
**Answer:** A Service provides a stable name and endpoint even when the backing Pod is recreated and receives a different IP.

### Q92. What is the difference between application replicas and database replication?
**Answer:** Laravel replicas are multiple application instances serving requests. Database replication is a separate database-level mechanism for synchronizing data between database instances. Two Laravel Pods do not imply two synchronized MySQL databases.

---

## 12. Limitations and Important Distinctions

### Q93. Is the current setup production-ready?
**Answer:** It is a useful learning lab, not automatically a production-ready platform. Production hardening would include a suitable external ingress/load-balancing design, TLS, resource requests and limits, health probes, security policies, secret management, backups, monitoring, logging, and a robust database availability strategy.

### Q94. Does running two Laravel replicas guarantee no downtime?
**Answer:** No. Multiple replicas can improve resilience, but availability also depends on node health, readiness checks, rollout strategy, network entry points, application behavior, and database availability.

### Q95. Does a PVC itself guarantee backups?
**Answer:** No. Persistent storage and backups are different. A PVC provides a storage claim; a backup strategy must separately capture data and support restoration.

### Q96. Does base64 encoding protect a Kubernetes Secret?
**Answer:** No. Base64 is an encoding, not encryption. Access controls, encryption at rest where configured, secure secret delivery, and avoiding credential exposure are still necessary.

### Q97. Why should the existing Docker Compose deployment be kept separate from the Kubernetes deployment?
**Answer:** They are different orchestration setups. Changing or stopping Compose services could disrupt the existing deployment. Kubernetes manifests should be managed and tested independently unless a deliberate migration or cutover is planned.

### Q98. What is the difference between Docker and Kubernetes?
**Answer:** Docker provides tools to build and run containers. Kubernetes orchestrates containerized workloads across cluster nodes, managing scheduling, replicas, service discovery, and desired state. Kubernetes can use container runtimes without requiring the Docker Engine on every node.

---

## 13. Rapid-Fire Viva Questions

| Question | Short answer |
|---|---|
| Smallest deployable Kubernetes unit? | Pod |
| Resource that manages stateless replicas? | Deployment |
| Resource that maintains a replica count? | ReplicaSet |
| Default Service type? | ClusterIP |
| Laravel Service port in this lab? | `9000` |
| MySQL Service port? | `3306` |
| Database Service name? | `mysql-service` |
| Laravel Service name? | `laravel-service` |
| Namespace? | `laravel-k8s` |
| Minikube profile? | `laravel-k8s` |
| Laravel replicas? | 2 |
| MySQL replicas? | 1 |
| MySQL image? | `mysql:8.0` |
| Laravel image? | `sami1800/laravel-app:1.0` |
| PVC name? | `mysql-pvc` |
| PVC requested capacity? | `5Gi` |
| What stores cluster state? | etcd |
| What selects nodes for Pods? | Scheduler |
| What is a ConfigMap for? | Non-sensitive configuration |
| What is a Secret for? | Sensitive configuration |
| What does `Bound` mean for a PVC? | Bound to a PersistentVolume |
| Does PHP-FPM speak ordinary HTTP? | No, it uses FastCGI |
| Does a Secret update change MySQL's stored password? | No |
| Does three-node Minikube equal production HA? | No |
| Does a PVC replace backups? | No |

---

## 14. Practical Demonstration Checklist

Use these commands during your viva to demonstrate the actual cluster state.

### A. Show the nodes
```bash
sudo kubectl get nodes -o wide
```

### B. Show application and database Pods
```bash
sudo kubectl get pods -n laravel-k8s -o wide
```

### C. Show Deployments and Services
```bash
sudo kubectl get deployments,services -n laravel-k8s
```

### D. Show the MySQL PVC
```bash
sudo kubectl get pvc -n laravel-k8s
```

### E. Show Service endpoints
```bash
sudo kubectl get endpoints -n laravel-k8s
```

### F. Show rollout status
```bash
sudo kubectl rollout status deployment/laravel -n laravel-k8s
```

### G. Show recent events
```bash
sudo kubectl get events -n laravel-k8s --sort-by=.lastTimestamp
```

### H. Show Laravel logs
```bash
sudo kubectl logs deployment/laravel -n laravel-k8s
```

### I. Show MySQL logs
```bash
sudo kubectl logs deployment/mysql -n laravel-k8s
```

> **Viva tip:** Explain what each command proves. For example, `get pods -o wide` shows both Pod status and node placement, while `get endpoints` shows whether a Service has matching backend Pods. Avoid displaying Secret values or passwords during a screen-share.

---

## Final Revision Strategy

Prioritize these topics if preparation time is short:

1. **Cluster architecture:** control plane, worker nodes, scheduler, and etcd.
2. **Workload resources:** Pod, Deployment, and ReplicaSet.
3. **Networking:** ClusterIP, Service discovery, endpoints, and ports `9000` and `3306`.
4. **Storage:** PVC, PV, `ReadWriteOnce`, and database persistence.
5. **Configuration:** ConfigMaps versus Secrets, and why a Secret update does not rotate an existing MySQL account password.
6. **Troubleshooting:** `get`, `describe`, `logs`, events, rollout status, and common Pod errors.
7. **Architecture limits:** one MySQL replica, Minikube's learning purpose, and the distinction between the existing Docker Compose setup and Kubernetes.

Be ready to explain not only *what* each resource is, but also *why* it is used in this particular lab.
