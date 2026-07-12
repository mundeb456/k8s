# Taints and Tolerations

**Taints** are applied to **Nodes** to repel Pods. **Tolerations** are applied
to **Pods** to allow (but not require) them to be scheduled onto tainted Nodes.

Think of it as: a taint says *"keep Pods away unless they can tolerate me."*

Important: taints/tolerations **repel**, they do not **attract**. A toleration
lets a Pod land on a tainted Node, but it does not guarantee it goes there — for
attraction you need node affinity / nodeSelector.

---

## Taint syntax

A taint has the form:

```
key=value:effect
```

- **key** — required.
- **value** — optional.
- **effect** — one of `NoSchedule`, `PreferNoSchedule`, `NoExecute`.

```bash
# Add a taint
kubectl taint nodes node01 app=blue:NoSchedule

# Add a taint with no value
kubectl taint nodes node01 dedicated:NoSchedule

# Remove a taint (note the trailing minus)
kubectl taint nodes node01 app=blue:NoSchedule-
```

View a node's taints:

```bash
kubectl describe node node01 | grep -i taint
kubectl get node node01 -o jsonpath='{.spec.taints}{"\n"}'
```

---

## Taint effects

| Effect | Meaning |
|--------|---------|
| `NoSchedule` | New Pods that don't tolerate the taint are **not scheduled**. Existing Pods stay. |
| `PreferNoSchedule` | Scheduler **tries** to avoid placing intolerant Pods, but will if no other option. Soft. |
| `NoExecute` | Intolerant Pods are **not scheduled AND existing intolerant Pods are evicted**. |

`NoExecute` is the aggressive one: it can kick already-running Pods off the Node.

---

## Tolerations (Pod YAML)

A toleration must match the taint's `key`, `value` (depending on operator), and
`effect`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: blue-pod
spec:
  tolerations:
    - key: "app"
      operator: "Equal"
      value: "blue"
      effect: "NoSchedule"
  containers:
    - name: nginx
      image: nginx:1.27
```

### operator: Equal vs Exists

- **`Equal`** (default) — matches if `key`, `value`, and `effect` all match.
  Requires `value`.
- **`Exists`** — matches if the `key` exists (with the given `effect`), ignoring
  value. Do **not** set `value` with `Exists`.

```yaml
  tolerations:
    - key: "app"
      operator: "Exists"       # tolerates any value of key "app"
      effect: "NoSchedule"
```

### Tolerate everything

An empty `key` with `operator: Exists` and no `effect` tolerates **all taints**
(used by DaemonSets and system components):

```yaml
  tolerations:
    - operator: "Exists"       # tolerate every taint, every effect
```

### tolerationSeconds (only meaningful with NoExecute)

For `NoExecute`, `tolerationSeconds` says how long the Pod may stay bound to a
Node after the taint is added before it is evicted. If omitted, the Pod tolerates
indefinitely.

```yaml
  tolerations:
    - key: "node.kubernetes.io/not-ready"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 300    # evict 5 min after node goes not-ready
```

Kubernetes auto-adds `not-ready` and `unreachable` NoExecute tolerations with
`tolerationSeconds: 300` to Pods that don't specify them.

---

## Control-plane taint

Control-plane nodes are tainted so ordinary workloads don't land on them:

```
node-role.kubernetes.io/control-plane:NoSchedule
```

```bash
kubectl describe node controlplane | grep -i taint
# Taints: node-role.kubernetes.io/control-plane:NoSchedule
```

To allow scheduling on the control-plane node (e.g. single-node clusters),
remove the taint:

```bash
kubectl taint nodes controlplane node-role.kubernetes.io/control-plane:NoSchedule-
```

Or add the matching toleration to specific Pods:

```yaml
  tolerations:
    - key: "node-role.kubernetes.io/control-plane"
      operator: "Exists"
      effect: "NoSchedule"
```

---

## Well-known automatic taints

The node controller adds these taints on node conditions; matching NoExecute
tolerations control eviction timing:

- `node.kubernetes.io/not-ready`
- `node.kubernetes.io/unreachable`
- `node.kubernetes.io/memory-pressure`
- `node.kubernetes.io/disk-pressure`
- `node.kubernetes.io/pid-pressure`
- `node.kubernetes.io/unschedulable` (set by `kubectl cordon`)
- `node.kubernetes.io/network-unavailable`

---

## Use cases

- **Dedicated nodes** — taint nodes `dedicated=gpu:NoSchedule` and add the
  matching toleration plus node affinity so only GPU workloads run there.
- **Special hardware** — reserve nodes with GPUs/SSDs for eligible Pods.
- **Draining** — `kubectl drain` cordons and adds NoExecute effects to evict
  Pods for maintenance.
- **Control-plane isolation** — keep user workloads off control-plane nodes.

---

## Full example: dedicated GPU nodes

Taint the node:

```bash
kubectl taint nodes gpu-node-1 dedicated=gpu:NoSchedule
kubectl label  nodes gpu-node-1 hardware=gpu
```

Pod that both tolerates the taint and is attracted via affinity:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-job
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: hardware
                operator: In
                values: ["gpu"]
  containers:
    - name: trainer
      image: nvidia/cuda:12.4.0-base-ubuntu22.04
      command: ["sleep", "3600"]
```

The toleration alone would let it run on the GPU node but not force it there;
the affinity ensures it actually goes to GPU nodes and nowhere else.

---

## Exam Tips

- **Taints go on nodes, tolerations go on pods.** Don't mix them up.
- Remember to add the trailing `-` when **removing** a taint:
  `kubectl taint nodes node01 key=value:Effect-`.
- A toleration only permits scheduling; it does **not** attract. Pair with
  affinity/nodeSelector when you must pin Pods to specific nodes.
- `NoExecute` **evicts running pods**; `NoSchedule` only affects new ones.
- `operator: Exists` must **not** have a `value`. `operator: Equal` (default)
  requires the exact value.
- `tolerationSeconds` only applies to `NoExecute`.
- The control-plane taint is `node-role.kubernetes.io/control-plane:NoSchedule`.
- If a Pod won't schedule anywhere, check node taints with
  `kubectl describe node <n> | grep -i taint`.

---

## Key Commands

```bash
# Apply / remove taints
kubectl taint nodes node01 app=blue:NoSchedule
kubectl taint nodes node01 app=blue:NoSchedule-
kubectl taint nodes node01 dedicated=gpu:NoExecute

# Inspect node taints
kubectl describe node node01 | grep -i taint
kubectl get node node01 -o jsonpath='{.spec.taints}{"\n"}'

# See the control-plane taint
kubectl describe node controlplane | grep -i taint

# Verify a pod's tolerations
kubectl get pod <pod> -o jsonpath='{.spec.tolerations}{"\n"}'

# Explain the schema
kubectl explain pod.spec.tolerations
```
