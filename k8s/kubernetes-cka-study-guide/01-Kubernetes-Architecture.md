# Kubernetes Architecture

A Kubernetes cluster is a set of machines (nodes) that run containerized workloads. Nodes fall into two roles:

- **Control plane nodes** — make global decisions about the cluster (scheduling, responding to events) and expose the API.
- **Worker nodes** — run the actual application Pods.

Understanding *which component does what* is essential for the CKA — especially for Troubleshooting (30% of the exam), where you diagnose failures by knowing where each responsibility lives.

## Control Plane Components

### kube-apiserver

- The front door of the cluster. Exposes the Kubernetes REST API over HTTPS on port **6443**.
- The **only** component that talks directly to `etcd`. Everything else talks to the API server.
- Handles authentication, authorization (RBAC), and admission control for every request.
- Stateless and horizontally scalable — you can run multiple replicas behind a load balancer for HA.

### etcd

- A consistent, highly-available **key-value store**; the single source of truth for all cluster state.
- Uses the Raft consensus algorithm; run an odd number of members (3 or 5) for quorum in HA.
- Client port **2379**, peer (member-to-member) port **2380**.
- **Back it up.** `etcdctl snapshot save` is a near-guaranteed exam task (see topic 27).

### kube-scheduler

- Watches for newly created Pods with no assigned node and selects a node for them to run on.
- Two phases: **filtering** (which nodes *can* run the Pod — resources, taints, affinity, nodeSelector) and **scoring** (which node is *best*).
- It does not start the Pod; it only writes the chosen node name into the Pod's `spec.nodeName` via the API server.
- Serves health/metrics on port **10259** (HTTPS).

### kube-controller-manager

- Runs the cluster's **control loops** as a single binary. Examples:
  - **Node controller** — notices and responds to nodes going down.
  - **ReplicaSet/Deployment controllers** — maintain the desired number of Pods.
  - **Job controller**, **EndpointSlice controller**, **ServiceAccount controller**, etc.
- Each controller continuously reconciles *actual state* toward *desired state*.
- Serves health/metrics on port **10257** (HTTPS).

### cloud-controller-manager

- Runs controllers that integrate with a specific cloud provider (AWS, GCP, Azure).
- Manages cloud-specific resources: **Node** lifecycle (via the cloud API), **Route** setup, and **Service** type `LoadBalancer` provisioning.
- Optional — absent on bare-metal/on-prem clusters.

## Node (Worker) Components

### kubelet

- The primary node agent; runs on **every** node (including control plane nodes running static pods).
- Registers the node with the API server, watches for PodSpecs assigned to its node, and ensures the described containers are running and healthy.
- Talks to the container runtime through the **CRI** (Container Runtime Interface).
- Runs liveness/readiness/startup probes.
- Serves its API on port **10250** (HTTPS).
- On kubeadm clusters, kubelet is a **systemd service** (not a pod) — use `systemctl status kubelet` and `journalctl -u kubelet` to debug it.

### kube-proxy

- Maintains network rules on each node that implement the **Service** abstraction (virtual IPs → backend Pods).
- Uses `iptables` (default) or `ipvs` mode to route/load-balance traffic to Service endpoints.
- Typically deployed as a DaemonSet.

### Container Runtime (CRI)

- The software that actually runs containers. Kubernetes talks to it via the **Container Runtime Interface (CRI)**.
- Common runtimes: **containerd** (most common), **CRI-O**. (Docker's `dockershim` was removed in v1.24.)
- Debug it with **`crictl`** — e.g. `crictl ps`, `crictl logs`, `crictl pods`. Configure via `/etc/crictl.yaml`.

## How a Pod Gets Scheduled (End to End)

1. A user runs `kubectl apply -f pod.yaml` (or a controller creates the Pod object).
2. `kubectl` sends the request to the **kube-apiserver**.
3. The API server authenticates, authorizes (RBAC), runs admission controllers, then persists the Pod object to **etcd**. At this point the Pod exists but has no `nodeName` — it is `Pending`.
4. The **kube-scheduler**, watching for unscheduled Pods, filters and scores nodes, picks one, and writes the binding (`spec.nodeName`) back through the API server.
5. The **kubelet** on the chosen node sees a Pod assigned to it (via its watch on the API server).
6. The kubelet calls the **container runtime** over the CRI to pull images and start containers, sets up networking via the **CNI** plugin, and mounts any volumes (via CSI).
7. The kubelet reports Pod status back to the API server; **kube-proxy** programs Service rules once the Pod's endpoints are ready.

## The Control Loop (Reconciliation) Concept

Kubernetes is **declarative**. You declare the *desired state* (e.g. "3 replicas of nginx"); controllers run non-terminating loops that:

```
observe (current state)  →  diff (against desired state)  →  act (converge)
```

This loop repeats forever. If a Pod dies, the ReplicaSet controller observes 2 running vs 3 desired and creates a replacement. This is why Kubernetes is **self-healing**: you never issue "create one more pod" commands imperatively for managed workloads — you change the desired state and let controllers converge.

## Core API Objects

- **Pod** — the smallest deployable unit; one or more co-located containers sharing network and storage.
- **ReplicaSet** — ensures N identical Pods are running.
- **Deployment** — declarative updates and rollbacks for ReplicaSets.
- **Service** — stable virtual IP + DNS name fronting a set of Pods.
- **ConfigMap / Secret** — configuration and sensitive data.
- **Namespace** — virtual cluster partition for isolation and quota.
- **Node** — a worker machine, represented as an API object.
- **PersistentVolume / PersistentVolumeClaim** — storage abstraction.

Everything is an object stored in etcd and manipulated through the API server. Objects share a common structure: `apiVersion`, `kind`, `metadata`, `spec` (desired), and `status` (observed, set by the system).

## ASCII Architecture Diagram

```
                                CONTROL PLANE NODE
        +---------------------------------------------------------------+
        |                                                               |
        |   +----------------+        +--------------------------+      |
        |   | kube-apiserver |<------>| etcd  (2379 client       |      |
        |   |    (6443)      |        |        2380 peer)        |      |
        |   +----------------+        +--------------------------+      |
        |     ^   ^      ^                                              |
        |     |   |      |                                              |
        |     |   |      +------------------+                           |
        |     |   |                         |                           |
        |     |   +--------------+          |                           |
        |     |                  |          |                           |
        |  +-----------------+  +-----------------------+  +----------+ |
        |  | kube-scheduler  |  | kube-controller-mgr   |  | cloud-   | |
        |  |    (10259)      |  |      (10257)          |  | ctrl-mgr | |
        |  +-----------------+  +-----------------------+  +----------+ |
        |                                                               |
        +----------------------------|----------------------------------+
                                      | (API over HTTPS :6443)
              +-----------------------+------------------------+
              |                                                |
    WORKER NODE 1                                    WORKER NODE 2
  +-------------------------+                    +-------------------------+
  |  +-------------------+  |                    |  +-------------------+  |
  |  | kubelet (10250)   |  |                    |  | kubelet (10250)   |  |
  |  +---------+---------+  |                    |  +---------+---------+  |
  |            | CRI        |                    |            | CRI        |
  |  +---------v---------+  |                    |  +---------v---------+  |
  |  | container runtime |  |                    |  | container runtime |  |
  |  | (containerd)      |  |                    |  | (containerd)      |  |
  |  +-------------------+  |                    |  +-------------------+  |
  |  +-------------------+  |                    |  +-------------------+  |
  |  | kube-proxy        |  |                    |  | kube-proxy        |  |
  |  +-------------------+  |                    |  +-------------------+  |
  |  [ Pods ]              |                    |  [ Pods ]              |
  +-------------------------+                    +-------------------------+
```

## Key Ports

| Port | Component | Purpose |
|------|-----------|---------|
| **6443** | kube-apiserver | Kubernetes API (HTTPS) |
| **2379** | etcd | Client requests (from API server) |
| **2380** | etcd | Peer / member-to-member communication |
| **10250** | kubelet | Kubelet API (metrics, exec, logs) |
| **10259** | kube-scheduler | Health/metrics (HTTPS) |
| **10257** | kube-controller-manager | Health/metrics (HTTPS) |
| 10256 | kube-proxy | Health check |
| 30000–32767 | all nodes | Default NodePort range |

## Exam Tips

- **Know that the API server is the only thing that talks to etcd.** Many troubleshooting questions hinge on this.
- On **kubeadm** clusters the control plane components (apiserver, scheduler, controller-manager, etcd) run as **static pods**. Their manifests live in `/etc/kubernetes/manifests/`. Editing a file there restarts the pod automatically — this is how you fix a misconfigured component.
- **kubelet is NOT a pod** on kubeadm clusters — it is a systemd service. If `kubectl get nodes` shows `NotReady`, check `systemctl status kubelet` and `journalctl -u kubelet -f` on that node.
- To inspect what's actually running when the API server is down, use **`crictl ps`** and `crictl logs <id>` (kubelet + runtime still work without the API server).
- Remember the scheduler's job stops at writing `nodeName` — it never starts containers. The **kubelet** starts containers.
- If a Pod is stuck `Pending`, suspect the **scheduler** (no suitable node) or resources/taints; if a Pod is scheduled but not running, suspect the **kubelet/runtime** on that node.

## Key Commands

```bash
# View cluster component health
kubectl get componentstatuses            # (deprecated but sometimes present)
kubectl get --raw='/readyz?verbose'      # API server readiness detail

# See control plane static pods (kubeadm)
kubectl get pods -n kube-system
ls /etc/kubernetes/manifests/

# Node status and details
kubectl get nodes -o wide
kubectl describe node <node-name>

# Kubelet (systemd service, run on the node)
systemctl status kubelet
journalctl -u kubelet -f

# Container runtime (run on the node)
crictl ps
crictl pods
crictl logs <container-id>

# Cluster-wide overview
kubectl cluster-info
kubectl get all -A
```
