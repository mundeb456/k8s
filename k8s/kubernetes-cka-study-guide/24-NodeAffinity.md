# Node Affinity

**Node affinity** constrains which Nodes a Pod can be scheduled onto, based on
Node **labels**. It is a more expressive successor to `nodeSelector`: it
supports set-based operators, soft preferences, and weights.

Node affinity **attracts** Pods to Nodes (unlike taints, which repel).

---

## Where it lives

Node affinity is defined under `spec.affinity.nodeAffinity`:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution: ...
      preferredDuringSchedulingIgnoredDuringExecution: ...
```

---

## The two affinity types

### requiredDuringSchedulingIgnoredDuringExecution (hard)

The rule **must** be satisfied for the Pod to be scheduled. If no Node matches,
the Pod stays `Pending`.

### preferredDuringSchedulingIgnoredDuringExecution (soft)

The scheduler **prefers** matching Nodes and gives them a higher score, but will
schedule the Pod elsewhere if no matching Node is available.

### What "IgnoredDuringExecution" means

Both types are only evaluated **at scheduling time**. If a Node's labels change
*after* the Pod is already running, the Pod is **not** evicted — the rule is
ignored during execution. (A `requiredDuringSchedulingRequiredDuringExecution`
variant is planned but does not exist yet — do not use it on the exam.)

---

## required: nodeSelectorTerms and matchExpressions

`required` uses `nodeSelectorTerms` (a list). The Pod schedules if **any one
term matches** (terms are ORed). Within a term, all `matchExpressions` must
match (they are ANDed).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: required-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values:
                  - ssd
                  - nvme
              - key: zone
                operator: In
                values:
                  - us-east-1a
  containers:
    - name: app
      image: nginx:1.27
```

This requires a Node with `disktype in (ssd, nvme)` AND `zone=us-east-1a`.

---

## matchExpressions operators

| Operator | Meaning | Needs values? |
|----------|---------|---------------|
| `In` | Label value is in the given set | Yes |
| `NotIn` | Label value is NOT in the set (anti-affinity) | Yes |
| `Exists` | Label key exists (any value) | No |
| `DoesNotExist` | Label key does not exist | No |
| `Gt` | Label value (integer) is greater than | Yes (single) |
| `Lt` | Label value (integer) is less than | Yes (single) |

- `Exists` / `DoesNotExist` must **not** include `values`.
- `Gt` / `Lt` compare integers and take exactly one value; the Node label value
  must parse as an integer.

```yaml
        nodeSelectorTerms:
          - matchExpressions:
              - key: gpu
                operator: Exists
              - key: cores
                operator: Gt
                values:
                  - "8"
```

You can also use `matchFields` (e.g. to match `metadata.name`), but
`matchExpressions` on labels is what the exam expects.

---

## preferred: weights

`preferred` is a list of `{weight, preference}` entries. `weight` is 1–100 and
is added to a Node's score when its `preference` (a matchExpressions block)
matches. Higher total = more preferred.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: preferred-affinity
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
        - weight: 20
          preference:
            matchExpressions:
              - key: zone
                operator: In
                values: ["us-east-1a"]
  containers:
    - name: app
      image: nginx:1.27
```

A Node with an SSD scores +80; one also in `us-east-1a` scores +100. If none
match, the Pod still schedules somewhere.

---

## Combining required + preferred

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values: ["linux"]
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
```

Must be Linux (hard); prefer SSD nodes (soft).

---

## Node labels

Affinity matches on Node labels. Set them with `kubectl label`:

```bash
kubectl label node node01 disktype=ssd
kubectl label node node01 zone=us-east-1a
kubectl label node node01 cores=16

kubectl get nodes --show-labels
kubectl get nodes -L disktype -L zone
```

Useful built-in labels: `kubernetes.io/hostname`,
`kubernetes.io/os`, `kubernetes.io/arch`,
`topology.kubernetes.io/zone`, `topology.kubernetes.io/region`,
`node.kubernetes.io/instance-type`.

---

## nodeAffinity vs nodeSelector

| | nodeSelector | nodeAffinity |
|--|--------------|--------------|
| Location | `spec.nodeSelector` | `spec.affinity.nodeAffinity` |
| Match type | exact key=value only | `In/NotIn/Exists/DoesNotExist/Gt/Lt` |
| Logic | AND of all keys | AND within a term, OR across terms |
| Soft/preferred | No | Yes (`preferred...` + weights) |
| Verbosity | very simple | more verbose |

Use `nodeSelector` for a simple exact match. Use `nodeAffinity` when you need
operators, OR logic, or soft preferences.

Note: node affinity does **not** repel other Pods from a Node. To keep other
Pods *off* a node, combine with **taints and tolerations**.

---

## Exam Tips

- `required...` failing to match → Pod stays **Pending**. Check node labels.
- `IgnoredDuringExecution`: changing a Node label after scheduling never evicts
  a running Pod.
- `nodeSelectorTerms` are **ORed**; `matchExpressions` inside one term are
  **ANDed**. Know this distinction.
- `Exists` and `DoesNotExist` take **no** `values`. `Gt`/`Lt` take exactly one
  integer value (quote it).
- `preferred` entries use `weight` (1–100) + `preference`; note it is a plain
  list, not `nodeSelectorTerms`.
- To both attract eligible Pods and keep others away, pair node affinity with
  taints/tolerations.
- Set/inspect node labels quickly: `kubectl label node <n> k=v` and
  `kubectl get nodes --show-labels`.

---

## Key Commands

```bash
# Label nodes for affinity to match
kubectl label node node01 disktype=ssd
kubectl label node node01 disktype-        # remove

kubectl get nodes --show-labels
kubectl get nodes -l disktype=ssd
kubectl get nodes -L disktype,zone         # show as columns

# Explain the schema
kubectl explain pod.spec.affinity.nodeAffinity
kubectl explain pod.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution

# Diagnose a pending pod
kubectl describe pod <pod>   # Events show "node(s) didn't match node affinity"
```
