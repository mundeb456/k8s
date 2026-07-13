# Kubernetes (K8s) Architecture

## Overview

**Kubernetes (K8s)** is an open-source container orchestration platform that automates the deployment, scaling, networking, and management of containerized applications.

A Kubernetes cluster consists of two main parts:

- **Control Plane** – Manages the cluster.
- **Worker Nodes** – Run application workloads.

```text
+----------------------------------+
|          Control Plane           |
|----------------------------------|
| API Server                       |
| Scheduler                        |
| Controller Manager               |
| ETCD                             |
+----------------------------------+
                |
                |
        +-------+-------+
        |               |
+---------------+ +---------------+
|    Worker     | |    Worker     |
|     Node      | |     Node      |
+---------------+ +---------------+
| kubelet       | | kubelet       |
| kube-proxy    | | kube-proxy    |
| Container     | | Container     |
| Runtime       | | Runtime       |
| Pods          | | Pods          |
+---------------+ +---------------+
```

---

# 1. Control Plane Components

The **Control Plane** is responsible for managing the cluster.

## API Server (`kube-apiserver`)

The API Server is the entry point to Kubernetes.

### Responsibilities

- Accepts REST API requests
- Authenticates and authorizes requests
- Validates requests
- Updates cluster state in ETCD
- Communicates with all Kubernetes components

Example:

```bash
kubectl create deployment nginx --image=nginx
```

Flow:

```text
kubectl
   ↓
API Server
   ↓
ETCD
```

---

## ETCD

ETCD is a distributed key-value database that stores the complete state of the Kubernetes cluster.

### Stores

- Pods
- Deployments
- Services
- Secrets
- ConfigMaps
- Nodes
- Persistent Volumes

Example:

```text
/registry/pods/default/nginx-pod
```

**ETCD is considered the source of truth for Kubernetes.**

---

## Scheduler (`kube-scheduler`)

The Scheduler decides **which worker node should run a newly created Pod**.

### Scheduling Factors

- CPU availability
- Memory availability
- Node affinity
- Taints and tolerations
- Resource requests
- Resource limits

Flow:

```text
Pod Created
     ↓
Scheduler
     ↓
Best Node Selected
```

---

## Controller Manager (`kube-controller-manager`)

Runs multiple controllers continuously to ensure the actual cluster state matches the desired state.

### Common Controllers

| Controller | Responsibility |
|------------|----------------|
| Deployment Controller | Maintains Deployments |
| ReplicaSet Controller | Maintains Pod replicas |
| Node Controller | Monitors node health |
| Job Controller | Executes batch jobs |
| Endpoint Controller | Updates service endpoints |

Example:

Desired replicas:

```text
3
```

Current replicas:

```text
2
```

Controller automatically creates one more Pod.

---

# 2. Worker Node Components

Worker nodes execute application workloads.

---

## Kubelet

Kubelet is an agent running on every worker node.

### Responsibilities

- Registers node with Kubernetes
- Receives Pod specifications
- Starts containers
- Monitors Pod health
- Reports node status

Flow:

```text
API Server
      ↓
Kubelet
      ↓
Container Runtime
```

---

## Container Runtime

Responsible for running containers.

Examples:

- containerd
- CRI-O

Earlier versions commonly used Docker through a CRI shim.

Example:

```text
Pod
 ├── Application Container
 └── Sidecar Container
```

The runtime launches and manages these containers.

---

## Kube Proxy

Responsible for networking inside the cluster.

### Responsibilities

- Service discovery
- Load balancing
- Packet forwarding
- Network routing

Example:

```text
Client
   ↓
Service
   ↓
Pod1
Pod2
Pod3
```

Traffic is distributed across available Pods.

---

# 3. Kubernetes Objects

## Pod

The smallest deployable unit in Kubernetes.

A Pod may contain one or more containers.

```text
Pod
 ├── Application Container
 └── Sidecar Container
```

Containers inside a Pod share:

- Network namespace
- Storage volumes

---

## ReplicaSet

Ensures a fixed number of Pod replicas are always running.

Example:

```text
Desired Pods = 3
Running Pods = 2
```

ReplicaSet creates one additional Pod automatically.

---

## Deployment

Deployment manages ReplicaSets and Pods.

Benefits:

- Rolling updates
- Rollbacks
- Scaling
- Self-healing

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

spec:
  replicas: 3
```

Deployment guarantees that three Pods remain available.

---

## Service

A Service provides a stable endpoint for accessing Pods.

Types:

- ClusterIP
- NodePort
- LoadBalancer
- ExternalName

Example:

```text
Service
   ↓
Pod A
Pod B
Pod C
```

Even if Pods restart, the Service IP remains unchanged.

---

## ConfigMap

Stores non-sensitive configuration.

Example:

```yaml
DATABASE_URL=mysql-service
LOG_LEVEL=INFO
```

---

## Secret

Stores sensitive information.

Example:

```yaml
PASSWORD=*****
TOKEN=*****
```

Secrets are Base64 encoded and can be encrypted at rest.

---

# 4. Request Lifecycle

Suppose a Deployment is created.

```bash
kubectl apply -f nginx.yaml
```

---

## Step 1

The request reaches the API Server.

```text
kubectl
   ↓
API Server
```

---

## Step 2

API Server stores the desired state inside ETCD.

```text
API Server
   ↓
ETCD
```

---

## Step 3

Deployment Controller notices a new Deployment.

```text
Deployment
      ↓
ReplicaSet
      ↓
Pod
```

---

## Step 4

Scheduler assigns the Pod to the best Worker Node.

```text
Pending Pod
      ↓
Scheduler
      ↓
Worker Node
```

---

## Step 5

Kubelet receives instructions.

```text
Kubelet
     ↓
Container Runtime
     ↓
Pod Started
```

---

## Step 6

Kube Proxy updates networking.

```text
Service
     ↓
Pod
```

Application becomes accessible.

---

# 5. Desired State Model

Kubernetes follows a declarative approach.

Example:

```yaml
replicas: 3
```

Desired state:

```text
3 Pods
```

Current state:

```text
Pod1 Running
Pod2 Running
Pod3 Failed
```

Controller detects the mismatch.

```text
Desired = 3
Actual = 2
```

A replacement Pod is created automatically.

This continuous process is known as the **reconciliation loop**.

---

# End-to-End Architecture

```text
                kubectl
                    |
                    v
           +----------------+
           | API Server     |
           +----------------+
                    |
                    v
           +----------------+
           | ETCD           |
           +----------------+
                    |
     +--------------+--------------+
     |                             |
     v                             v
+-----------+              +-------------+
| Scheduler |              | Controllers |
+-----------+              +-------------+
       |                           |
       +-------------+-------------+
                     |
                     v
          +----------------------+
          | Worker Nodes         |
          +----------------------+
          | kubelet              |
          | kube-proxy           |
          | container runtime    |
          | Pods                 |
          +----------------------+
```

---

# Component Summary

| Component | Purpose |
|-----------|---------|
| API Server | Entry point to the cluster |
| ETCD | Stores cluster state |
| Scheduler | Assigns Pods to Nodes |
| Controller Manager | Maintains desired state |
| Kubelet | Runs Pods on Worker Nodes |
| Container Runtime | Executes containers |
| Kube Proxy | Handles networking and load balancing |
| Pod | Smallest deployable unit |
| ReplicaSet | Maintains Pod replicas |
| Deployment | Manages ReplicaSets and rolling updates |
| Service | Provides stable network access to Pods |
| ConfigMap | Stores configuration |
| Secret | Stores sensitive data |

---

# Key Takeaways

- Kubernetes consists of a **Control Plane** and **Worker Nodes**.
- The **API Server** is the central communication hub.
- **ETCD** stores the desired and current cluster state.
- The **Scheduler** decides where Pods should run.
- **Controllers** continuously reconcile the cluster to match the desired state.
- **Kubelet** runs and monitors Pods on each node.
- **Kube Proxy** enables networking and load balancing.
- Applications are deployed as **Pods**, managed by **Deployments**, and exposed through **Services**.
