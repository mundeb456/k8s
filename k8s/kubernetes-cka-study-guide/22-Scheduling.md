# Scheduling

The **kube-scheduler** is the control-plane component that assigns Pods to
Nodes. A Pod with no `nodeName` set sits in `Pending` until the scheduler binds
it to a suitable Node.

---

## Scheduler overview

The scheduler watches for newly created Pods that have an empty
`spec.nodeName`. For each such Pod it:

1. Finds the set of **feasible** Nodes (filtering).
2. **Scores** the feasible Nodes and picks the best one.
3. **Binds** the Pod by writing `nodeName` into the Pod object.

The kubelet on that Node then sees the Pod is bound to it and starts the
containers.

If no Node is feasible, the Pod stays `Pending` and the scheduler records an
event explaining why (visible via `kubectl describe pod`).

---

## The scheduling process: filtering and scoring

### 1. Filtering (predicates)

Eliminates Nodes that cannot run the Pod. Common filters:

- **Resource fit** — Node has enough allocatable CPU/memory for the Pod's
  requests.
- **Node selectors / affinity** — Node matches `nodeSelector` / required
  `nodeAffinity`.
- **Taints / tolerations** — Pod tolerates the Node's taints.
- **Volume constraints** — required volumes can be attached (zone, count).
- **Port availability** — hostPort not already taken.

### 2. Scoring (priorities)

Ranks the remaining feasible Nodes 0–100. Factors include spreading Pods across
Nodes, `preferred` affinity/anti-affinity, image locality (Node already has the
image), and least/most requested resources. The highest-scoring Node wins (ties
broken randomly).

### 3. Binding

The scheduler creates a Binding object; the API server sets `spec.nodeName`.

---

## Manual scheduling with nodeName

You can bypass the scheduler entirely by setting `spec.nodeName` yourself. The
kubelet on that node will run the Pod directly — no filtering or scoring, and it
runs even if there is no scheduler at all.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: manual-pod
spec:
  nodeName: node01        # bind directly to node01
  containers:
    - name: nginx
      image: nginx:1.27
```

Caveats:

- If the named Node lacks resources, the Pod may still be placed but fail to
  run (kubelet rejects → `OutOfcpu`/`OutOfmemory`).
- You **cannot change `nodeName` on a running Pod** — delete and recreate.
- Useful for the exam when the scheduler is broken and you must place a Pod
  manually.

### Manually binding an already-Pending Pod

If a Pod is already created and Pending, create a Binding object:

```bash
cat <<EOF | curl -s --data-binary @- \
  -H "Content-Type:application/json" \
  http://$SERVER/api/v1/namespaces/default/pods/manual-pod/binding/
{
  "apiVersion": "v1",
  "kind": "Binding",
  "metadata": { "name": "manual-pod" },
  "target": { "apiVersion": "v1", "kind": "Node", "name": "node01" }
}
EOF
```

(In practice on the exam it is faster to delete and recreate with `nodeName`.)

---

## nodeSelector

The simplest way to constrain a Pod to Nodes with specific **labels**. It is an
exact key=value match; all listed labels must be present on the Node.

Label a node first:

```bash
kubectl label node node01 disktype=ssd
```

Then require it:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  nodeSelector:
    disktype: ssd
  containers:
    - name: app
      image: nginx:1.27
```

Limitations: `nodeSelector` only supports equality and ANDs all keys. For
`In/NotIn/Exists`, ranges, or preferences, use **nodeAffinity** (see
`24-NodeAffinity.md`).

---

## Resource requests and scheduling

Scheduling decisions are based on a Pod's **requests**, not its limits. The
scheduler places a Pod only where `sum(requests of pods on node) + pod.requests
<= node.allocatable`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sized-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "500m"
          memory: "256Mi"
        limits:
          cpu: "1"
          memory: "512Mi"
```

- **requests** = guaranteed reservation used for scheduling.
- **limits** = hard cap enforced at runtime (CPU throttled, memory → OOMKill).
- A Pod requesting more than any Node's allocatable will stay `Pending` with a
  `FailedScheduling` / `Insufficient cpu/memory` event.

---

## Capacity vs Allocatable (kubectl describe node)

```bash
kubectl describe node node01
```

Key sections:

- **Capacity** — total hardware resources on the Node.
- **Allocatable** — Capacity minus resources reserved for the kubelet, OS, and
  eviction thresholds (`kube-reserved`, `system-reserved`, `eviction-hard`).
  The scheduler uses **Allocatable**.
- **Allocated resources** — sum of requests/limits of Pods already on the Node,
  shown as a percentage of allocatable. This tells you how much room is left.

```bash
# Quick view of requests/limits per node
kubectl describe node node01 | grep -A15 "Allocated resources"

# Actual live usage (needs metrics-server)
kubectl top node
kubectl top pod
```

---

## Multiple schedulers (brief)

You can run more than one scheduler and choose which one places a Pod via
`spec.schedulerName`. Pods without a `schedulerName` use the default
`default-scheduler`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-sched-pod
spec:
  schedulerName: my-custom-scheduler
  containers:
    - name: app
      image: nginx:1.27
```

If you name a scheduler that is not running, the Pod stays `Pending` (no
scheduler picks it up). Check which scheduler acted on a Pod via its events:

```bash
kubectl get events -o wide | grep <pod-name>
# The Source column shows the scheduler name
```

---

## Full example: constrained, sized pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  nodeSelector:
    disktype: ssd
  schedulerName: default-scheduler
  containers:
    - name: nginx
      image: nginx:1.27
      resources:
        requests:
          cpu: "250m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      ports:
        - containerPort: 80
```

---

## Exam Tips

- A `Pending` Pod almost always means a scheduling problem — run
  `kubectl describe pod <name>` and read the **Events** at the bottom
  (`FailedScheduling`, `Insufficient cpu`, taint messages, etc.).
- `nodeName` bypasses the scheduler completely — use it when the scheduler is
  down or when the task says "place this Pod on node X" quickly.
- Scheduling uses **requests**, not limits. Learn to read `describe node` for
  Capacity vs Allocatable vs Allocated.
- `nodeSelector` = exact label match only. Reach for `nodeAffinity` for
  operators/preferences.
- If a Pod names a `schedulerName` that isn't running, it will hang in
  `Pending` — a classic troubleshooting trap.
- Check if the default scheduler is even running (it's often a static pod):
  `kubectl get pods -n kube-system | grep scheduler`.

---

## Key Commands

```bash
# Label / unlabel nodes
kubectl label node node01 disktype=ssd
kubectl label node node01 disktype-

kubectl get nodes --show-labels
kubectl get nodes -l disktype=ssd

# Inspect node capacity/allocatable and current load
kubectl describe node node01
kubectl get node node01 -o jsonpath='{.status.allocatable}{"\n"}'
kubectl top node

# Diagnose a Pending pod
kubectl describe pod <pod>
kubectl get events --sort-by=.metadata.creationTimestamp

# Check the scheduler itself
kubectl get pods -n kube-system | grep scheduler
kubectl logs -n kube-system kube-scheduler-<controlplane>
```
