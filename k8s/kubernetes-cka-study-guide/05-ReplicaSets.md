# ReplicaSets

A **ReplicaSet (RS)** ensures that a specified number of identical Pod replicas are running at any given time. If Pods die or are deleted, the ReplicaSet creates replacements; if there are too many, it deletes the excess. It is the reconciliation engine behind Deployments.

## Purpose

- Maintain a stable **replica count** (self-healing).
- Guarantee availability of a set of identical Pods.
- Replaced the older **ReplicationController**; the key difference is that a ReplicaSet supports **set-based selectors** (`matchExpressions`), not just equality.

You rarely create a ReplicaSet directly. In practice a **Deployment** manages ReplicaSets for you, adding rollout and rollback. Still, the CKA expects you to read, edit, scale, and troubleshoot ReplicaSets.

## Selectors

The `spec.selector` tells the ReplicaSet which Pods it owns. It **must match** the labels in the Pod template (`spec.template.metadata.labels`) or the API server rejects the object.

### matchLabels (equality-based)

```yaml
selector:
  matchLabels:
    app: web
    tier: frontend
```

### matchExpressions (set-based)

Operators: `In`, `NotIn`, `Exists`, `DoesNotExist`.

```yaml
selector:
  matchExpressions:
  - key: app
    operator: In
    values: [web, api]
  - key: tier
    operator: Exists
```

`matchLabels` and `matchExpressions` can be combined; all conditions are ANDed.

## Full ReplicaSet YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:                    # this is a Pod template
    metadata:
      labels:
        app: web               # MUST match spec.selector.matchLabels
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```

Note: `apiVersion` is `apps/v1` (not `v1`). A common exam mistake is using the wrong apiVersion or mismatching the selector and template labels.

## Relationship to Deployments

```
Deployment  --manages-->  ReplicaSet  --manages-->  Pods
```

- A **Deployment** creates and owns one **ReplicaSet** per revision (per template hash).
- During a rolling update, the Deployment spins up a new ReplicaSet and scales the old one down — this is how rollbacks work (scale the old RS back up).
- `revisionHistoryLimit` on the Deployment controls how many old (scaled-to-zero) ReplicaSets are retained.

Because Deployments give you rollouts and rollbacks for free, **prefer Deployments** over bare ReplicaSets for stateless workloads.

## Scaling

```bash
kubectl scale replicaset web-rs --replicas=5
kubectl scale rs web-rs --replicas=5              # 'rs' shortname
kubectl scale --replicas=5 -f web-rs.yaml

# Or edit the live object
kubectl edit rs web-rs                            # change spec.replicas, save
```

Scaling just changes `spec.replicas`; the controller reconciles toward it. If you scale a ReplicaSet that is owned by a Deployment, the Deployment controller will fight you and reset it — scale the Deployment instead.

## ownerReferences

When a ReplicaSet creates a Pod, it stamps the Pod's `metadata.ownerReferences` with a back-pointer to the ReplicaSet. This is how the RS knows which Pods are "its own," and it drives **garbage collection**: delete the ReplicaSet and its Pods are cascade-deleted. Likewise, a Deployment sets the `ownerReferences` on each ReplicaSet it owns.

```bash
kubectl get pod <pod> -o jsonpath='{.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].name}'
# -> ReplicaSet/web-rs-6c9f...

# See which controller owns a pod
kubectl get pod <pod> -o yaml | grep -A5 ownerReferences
```

**Adoption**: a ReplicaSet will *adopt* any bare Pod whose labels match its selector and that has no controller owner — even one you created manually. If those adopted Pods push the count above `replicas`, the RS deletes some. This is why selector hygiene matters.

Delete behavior:

```bash
kubectl delete rs web-rs                 # cascade: deletes RS + its Pods (foreground/background)
kubectl delete rs web-rs --cascade=orphan   # delete only the RS, leave Pods running
```

## Common Operations

```bash
kubectl get rs
kubectl get rs -o wide                    # shows CONTAINERS, IMAGES, SELECTOR
kubectl describe rs web-rs                # events + owned pods
kubectl get rs web-rs -o yaml

# Which pods does this RS own?
kubectl get pods -l app=web

# There is no imperative 'create replicaset'; generate from a deployment then
# change kind to ReplicaSet, or write the YAML directly.
kubectl create deployment web --image=nginx --replicas=3 --dry-run=client -o yaml
```

There is no `kubectl create replicaset` generator — create a Deployment manifest with `--dry-run=client -o yaml`, or write the RS YAML by hand.

## Troubleshooting

- **Pods not being created**: check `kubectl describe rs` events and confirm `selector.matchLabels` equals `template.metadata.labels`.
- **RS keeps recreating a Pod you deleted**: expected — that is its job. Scale to 0 or delete the RS to stop it.
- **Too many Pods**: a bare Pod with matching labels got adopted, or another controller owns overlapping labels. Tighten selectors.
- **Selector immutable**: `spec.selector` cannot be changed after creation. To change it you must delete and recreate.

## Exam Tips

- `apiVersion: apps/v1`, `kind: ReplicaSet`. Getting the apiVersion wrong is a classic error.
- `spec.selector.matchLabels` **must** match `spec.template.metadata.labels`, or creation fails.
- Do not scale a ReplicaSet owned by a Deployment — scale the Deployment; the Deployment controller overrides direct RS edits.
- Use `--cascade=orphan` to delete a controller but keep its Pods (rare, but sometimes asked).
- `matchExpressions` operators: `In`, `NotIn`, `Exists`, `DoesNotExist`.
- Check ownership fast with `kubectl get pod <pod> -o yaml | grep -A5 ownerReferences`.
- No imperative generator exists — generate a Deployment manifest and switch the `kind`, or write it by hand.

## Key Commands

```bash
kubectl get rs
kubectl get rs -o wide
kubectl describe rs <name>
kubectl scale rs <name> --replicas=<n>
kubectl edit rs <name>
kubectl delete rs <name>                     # cascade delete pods
kubectl delete rs <name> --cascade=orphan    # keep pods
kubectl get pods -l <selector>
kubectl get pod <pod> -o yaml | grep -A5 ownerReferences
```
