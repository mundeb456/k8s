# StatefulSets

A **StatefulSet** is a workload API object used to manage **stateful applications**. Unlike a Deployment, a StatefulSet maintains a **sticky, stable identity** for each of its Pods. These Pods are created from the same spec but are **not interchangeable**: each has a persistent identifier that it retains across rescheduling.

Use StatefulSets for applications that require one or more of:

- **Stable, unique network identifiers** (predictable Pod names / DNS)
- **Stable, persistent storage** (each replica keeps its own volume)
- **Ordered, graceful deployment and scaling**
- **Ordered, automated rolling updates**

Typical workloads: databases (MySQL, PostgreSQL, MongoDB), distributed data stores (Cassandra, etcd), message brokers (Kafka, RabbitMQ), and clustered caches (Redis clusters).

---

## StatefulSet vs Deployment

| Aspect | Deployment | StatefulSet |
|--------|-----------|-------------|
| Pod identity | Random hash suffix, interchangeable | Stable ordinal index (`0..N-1`) |
| Pod name | `web-7d9f8c-abcde` | `web-0`, `web-1`, `web-2` |
| Storage | Usually shared / ephemeral | Per-Pod PVC via `volumeClaimTemplates` |
| Network identity | Via Service load-balancing only | Stable per-Pod DNS via headless Service |
| Scale up order | All at once (parallel) | Ordered `0 → N-1` (by default) |
| Scale down order | Any Pod | Reverse order `N-1 → 0` |
| Rescheduled Pod | New name, new identity | Same name, same identity, same volume |
| Use case | Stateless apps, web front-ends | Stateful apps, clustered data stores |

**Key idea:** In a Deployment the Pods are cattle; in a StatefulSet they are pets with names and dedicated storage.

---

## Stable Network Identity

Each Pod in a StatefulSet derives its hostname from the StatefulSet name and its ordinal index:

```
<statefulset-name>-<ordinal>
```

For a StatefulSet named `web` with 3 replicas, the Pods are `web-0`, `web-1`, `web-2`.

Combined with a **headless Service** (`clusterIP: None`), each Pod gets a stable DNS record:

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

Example:

```
web-0.nginx.default.svc.cluster.local
web-1.nginx.default.svc.cluster.local
web-2.nginx.default.svc.cluster.local
```

These records remain valid even if the Pod is rescheduled to a different node with a new IP. This lets peers discover each other by name (e.g., a Cassandra node contacting `cassandra-0.cassandra`).

---

## Headless Service Requirement

A StatefulSet **requires** a headless Service to control the network domain of its Pods. A headless Service is created by setting `clusterIP: None`. It does not load-balance; instead it returns the individual Pod A records for DNS lookups.

The StatefulSet references this Service via `spec.serviceName`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  clusterIP: None        # headless — no virtual IP, returns Pod records
  selector:
    app: nginx
  ports:
    - name: web
      port: 80
      targetPort: 80
```

> If you omit or misname the headless Service, Pods will not get their stable DNS names and the StatefulSet may not behave correctly.

---

## Ordered Deployment and Scaling

By default (`podManagementPolicy: OrderedReady`):

- **Scale up / create:** Pods are created **one at a time, in order** (`web-0`, then `web-1`, then `web-2`). Each Pod must be **Running and Ready** before the next is created.
- **Scale down / delete:** Pods are terminated in **reverse order** (`web-2`, then `web-1`, then `web-0`), one at a time, waiting for each to fully terminate.
- **Rolling update:** Pods are updated in reverse ordinal order.

Set `podManagementPolicy: Parallel` to launch or terminate all Pods at once (identity is still stable, but ordering guarantees are dropped). This is useful when the app does not need strict ordering during startup.

```yaml
spec:
  podManagementPolicy: Parallel   # or OrderedReady (default)
```

---

## volumeClaimTemplates

`volumeClaimTemplates` gives each Pod its **own** PersistentVolumeClaim. The controller creates a PVC per Pod, named:

```
<volumeClaimTemplate-name>-<statefulset-name>-<ordinal>
```

For a template named `www` and StatefulSet `web`:

```
www-web-0
www-web-1
www-web-2
```

**Important behavior:**

- Each PVC (and its bound PV) is **retained when a Pod is deleted or rescheduled**, so the Pod re-attaches to the same data.
- **Deleting the StatefulSet does NOT delete the PVCs** by default — you must delete them manually (or use `persistentVolumeClaimRetentionPolicy` in v1.27+).
- When a Pod is rescheduled, it re-mounts its existing PVC.

The `persistentVolumeClaimRetentionPolicy` (stable since v1.27) controls PVC lifecycle:

```yaml
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain    # Retain (default) or Delete
    whenScaled: Retain     # Retain (default) or Delete
```

---

## Full YAML Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  clusterIP: None          # headless service
  selector:
    app: nginx
  ports:
    - name: web
      port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"     # must match the headless Service name
  replicas: 3
  podManagementPolicy: OrderedReady   # default; Parallel to skip ordering
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0         # update all Pods with ordinal >= partition
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: nginx
          image: registry.k8s.io/nginx-slim:0.24
          ports:
            - name: web
              containerPort: 80
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "standard"
        resources:
          requests:
            storage: 1Gi
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Retain
```

---

## Update Strategies

`spec.updateStrategy.type` can be:

### RollingUpdate (default)

Updates Pods automatically in **reverse ordinal order** (`N-1 → 0`), one at a time, waiting for each to be Ready before proceeding.

### RollingUpdate with `partition`

A **partition** enables **staged / canary rollouts**. Only Pods with an ordinal **>= partition** are updated; Pods below the partition stay on the old version.

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2    # only web-2 (and higher) update; web-0, web-1 stay old
```

Lower the partition value gradually (`2 → 1 → 0`) to roll the change out across the fleet. When `partition: 0`, all Pods are updated.

### OnDelete

The controller does **not** automatically update Pods. You must **manually delete** each Pod for the new spec to take effect. Gives full manual control over update timing.

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

---

## Common Operations

```bash
# Create from manifest
kubectl apply -f statefulset.yaml

# List StatefulSets
kubectl get statefulset          # or: kubectl get sts

# Watch ordered Pod creation
kubectl get pods -l app=nginx -w

# Scale
kubectl scale statefulset web --replicas=5

# Inspect the per-Pod PVCs
kubectl get pvc

# Check DNS from another pod
kubectl run -it --rm dnsutils --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 \
  -- nslookup web-0.nginx

# Delete StatefulSet but keep Pods (orphan them)
kubectl delete statefulset web --cascade=orphan
```

---

## Exam Tips

- **Headless Service is mandatory.** `spec.serviceName` must match a Service with `clusterIP: None`. This is a frequent gotcha — a misconfigured serviceName leaves Pods without stable DNS.
- **Pod names are predictable:** `<name>-0`, `<name>-1`, ... Know this for questions about which Pod holds which data.
- **PVC naming:** `<claimTemplate>-<sts>-<ordinal>`. If asked to identify the volume for `web-1`, it is `www-web-1`.
- **Deleting a StatefulSet does not delete PVCs** by default — data survives. Deleting a Pod also keeps its PVC.
- **Scaling order:** up = ascending (`0→N`), down = descending (`N→0`), one at a time under `OrderedReady`.
- **`podManagementPolicy: Parallel`** removes ordering guarantees but keeps stable identity — useful when the app tolerates unordered startup.
- **Rolling update goes in reverse ordinal order.** Use `partition` for canary; `OnDelete` for manual control.
- If a Pod is stuck Pending during ordered rollout, later Pods will never start — check the blocking Pod first.
- `kubectl get sts` is the short form. `kubectl rollout status statefulset/web` shows update progress.

---

## Key Commands

```bash
kubectl apply -f sts.yaml                          # create/update
kubectl get sts                                     # list statefulsets
kubectl describe sts web                            # detail + events
kubectl scale sts web --replicas=5                  # scale
kubectl rollout status sts/web                      # update progress
kubectl rollout undo sts/web                         # roll back
kubectl get pvc                                     # per-pod volumes
kubectl delete sts web --cascade=orphan             # delete, keep pods
kubectl get pods -l app=nginx -w                    # watch ordering
```
