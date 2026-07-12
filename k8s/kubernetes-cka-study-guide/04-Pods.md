# Pods

A **Pod** is the smallest deployable unit in Kubernetes. It represents one or more containers that share a network namespace (same IP and port space), storage volumes, and a lifecycle. You almost never create bare Pods in production — you let a controller (Deployment, ReplicaSet, Job, DaemonSet) manage them — but the CKA expects you to understand and create them directly.

## The Pod Concept

- Every Pod gets its **own cluster-internal IP**. Containers in the same Pod reach each other over `localhost` and can share ports (so they must not use the same port).
- Containers in a Pod are **co-scheduled and co-located** on the same node and share the Pod's volumes.
- A Pod is generally treated as **ephemeral and disposable**. When it dies, it is not resurrected in place; a controller creates a new Pod with a new IP.

## Single vs. Multi-Container Pods

Most Pods run a single application container. Multi-container Pods exist when helper containers must share resources tightly with the main one. Common patterns:

- **Sidecar** — augments the main container (log shipper, proxy, sync agent).
- **Ambassador** — proxies the main container's network connections to the outside world.
- **Adapter** — normalizes/reshapes the main container's output for consumers.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-sidecar
spec:
  containers:
  - name: app
    image: nginx:1.27
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  - name: log-shipper          # sidecar reads the shared volume
    image: busybox:1.36
    command: ["sh", "-c", "tail -F /var/log/nginx/access.log"]
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
  volumes:
  - name: logs
    emptyDir: {}
```

## Pod Lifecycle and Phases

`status.phase` is a high-level summary:

| Phase | Meaning |
|-------|---------|
| `Pending` | Accepted by the API server; not all containers running yet (scheduling, image pulls, init containers). |
| `Running` | Bound to a node; at least one container is running or starting/restarting. |
| `Succeeded` | All containers terminated successfully and will not restart. |
| `Failed` | All containers terminated; at least one failed (non-zero exit or killed). |
| `Unknown` | State could not be obtained (usually node communication failure). |

Container-level status (`waiting`, `running`, `terminated`) is finer-grained; check `kubectl describe pod` for reasons like `CrashLoopBackOff`, `ImagePullBackOff`, `ErrImagePull`, `CreateContainerConfigError`.

## Restart Policy

Set at the Pod level via `spec.restartPolicy` and applies to all containers:

- `Always` (default) — restart containers regardless of exit code. Used by Deployments/ReplicaSets.
- `OnFailure` — restart only on non-zero exit. Used by Jobs.
- `Never` — never restart.

Restarts use an exponential back-off (`CrashLoopBackOff`), capped at 5 minutes.

## Init Containers

Init containers run **to completion, one at a time, in order, before** any app container starts. Use them for setup work: waiting on a dependency, cloning config, running migrations. If an init container fails, the Pod restarts it (respecting `restartPolicy`) until it succeeds.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z db 5432; do echo waiting; sleep 2; done']
  containers:
  - name: app
    image: myapp:1.0
```

## Sidecar Containers (Native)

Kubernetes 1.29+ supports **native sidecar containers**: an init container with `restartPolicy: Always`. Unlike ordinary init containers, it keeps running alongside the main containers and starts before them, which is ideal for log/metric agents that must be up first and remain up.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-native-sidecar
spec:
  initContainers:
  - name: logshipper
    image: busybox:1.36
    restartPolicy: Always          # <- makes this a native sidecar
    command: ['sh', '-c', 'while true; do echo shipping; sleep 5; done']
  containers:
  - name: app
    image: myapp:1.0
```

## Probes

Probes let the kubelet check container health. Each probe supports one handler: `httpGet`, `tcpSocket`, `exec`, or `grpc`.

- **livenessProbe** — if it fails, the kubelet **kills and restarts** the container. Detects deadlocks.
- **readinessProbe** — if it fails, the Pod is **removed from Service endpoints** (no traffic) but not restarted. Detects "not ready yet / temporarily busy."
- **startupProbe** — runs first for slow-starting apps; disables liveness/readiness until it succeeds. Prevents premature restarts.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probed
spec:
  containers:
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30       # up to 30 * 10s = 5 min to become healthy
      periodSeconds: 10
```

Key tuning fields: `initialDelaySeconds`, `periodSeconds`, `timeoutSeconds`, `successThreshold`, `failureThreshold`.

## Resource Requests and Limits

- **requests** — what the scheduler reserves; guaranteed minimum. Used for scheduling decisions.
- **limits** — hard ceiling. Exceeding a **memory** limit gets the container **OOMKilled**; exceeding a **CPU** limit gets it **throttled** (not killed).

CPU is measured in cores; `500m` = 0.5 core. Memory uses `Mi`/`Gi` (binary) or `M`/`G` (decimal).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resourced
spec:
  containers:
  - name: app
    image: myapp:1.0
    resources:
      requests:
        cpu: "250m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
```

The relationship between requests and limits sets the Pod's **QoS class**: `Guaranteed` (requests == limits for all resources), `Burstable` (requests set, below limits), `BestEffort` (nothing set). QoS affects eviction order under node pressure.

## Creating Pods Imperatively

```bash
# Run a pod
kubectl run nginx --image=nginx

# Generate YAML instead of creating it
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# With a port, labels, env, and a command
kubectl run web --image=nginx --port=80 --labels="app=web,tier=frontend" \
  --env="ENV=prod" --dry-run=client -o yaml

# Override the container command/args
kubectl run busybox --image=busybox --restart=Never -- sleep 3600

# One-off command, then delete
kubectl run tmp --image=busybox --rm -it --restart=Never -- sh
```

`--restart=Never` produces a Pod; `--restart=OnFailure` produces a Job; the default `--restart=Always` (with older kubectl) once produced a Deployment but modern `kubectl run` always creates a Pod.

## Full Pod YAML Reference

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: full-example
  namespace: default
  labels:
    app: web
    tier: frontend
  annotations:
    description: "reference pod"
spec:
  restartPolicy: Always
  serviceAccountName: default
  nodeSelector:
    disktype: ssd
  containers:
  - name: app
    image: nginx:1.27
    imagePullPolicy: IfNotPresent
    command: ["nginx"]
    args: ["-g", "daemon off;"]
    ports:
    - name: http
      containerPort: 80
    env:
    - name: LOG_LEVEL
      value: "info"
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    envFrom:
    - configMapRef:
        name: app-config
    resources:
      requests:
        cpu: "250m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
```

## Static vs. Managed Pods (Brief)

- **Managed Pods** are created via the API server, usually by controllers, and scheduled by the scheduler.
- **Static Pods** are managed directly by the **kubelet** on a node from manifest files in a directory (default `/etc/kubernetes/manifests`, set by `--pod-manifest-path` / `staticPodPath` in the kubelet config). The kubelet creates a read-only **mirror Pod** in the API server so you can see it, but you cannot control it via the API — edit the file on the node instead. The control-plane components (`kube-apiserver`, `etcd`, `controller-manager`, `scheduler`) run as static Pods on kubeadm clusters. Static pod names end with the node name (e.g. `kube-apiserver-node01`).

```bash
# Find the static pod directory the kubelet uses
grep staticPodPath /var/lib/kubelet/config.yaml
# Manifests live there, e.g. /etc/kubernetes/manifests/*.yaml
```

## Exam Tips

- Generate every Pod with `kubectl run <name> --image=<img> $do > pod.yaml`, then edit — do not hand-type the boilerplate.
- `kubectl describe pod` events (bottom of output) tell you *why* a Pod is Pending/CrashLooping — read them first.
- Distinguish the probe purposes: liveness **restarts**, readiness **removes from endpoints**, startup **protects slow starters**. This is a frequent question.
- Memory limit breach = OOMKilled (container restarts); CPU limit breach = throttled (no restart). Know the difference.
- To debug a crashed container use `kubectl logs <pod> --previous`.
- To change a static pod, edit the manifest file on the node — `kubectl edit`/`delete` won't work (kubelet recreates it).
- `--restart=Never` → Pod, `--restart=OnFailure` → Job. Remember this for `kubectl run`.
- A multi-container Pod's containers share `localhost` and volumes but must use different ports.

## Key Commands

```bash
kubectl run <name> --image=<img> --dry-run=client -o yaml > pod.yaml
kubectl run <name> --image=<img> --restart=Never -- <cmd> <args>
kubectl get pods -o wide
kubectl describe pod <name>
kubectl logs <pod> [-c <container>] [--previous]
kubectl exec -it <pod> -c <container> -- sh
kubectl get pod <name> -o jsonpath='{.status.phase}'
kubectl delete pod <name> --grace-period=0 --force
kubectl explain pod.spec.containers.livenessProbe
```
