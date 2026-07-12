# Pod Affinity and Anti-Affinity

While **node affinity** places Pods relative to *Node labels*, **pod affinity**
and **pod anti-affinity** place Pods relative to *other Pods* already running.

- **podAffinity** — attract this Pod to Nodes where matching Pods already run
  (co-location).
- **podAntiAffinity** — repel this Pod from Nodes where matching Pods run
  (spreading).

Both live under `spec.affinity`:

```yaml
spec:
  affinity:
    podAffinity: ...
    podAntiAffinity: ...
```

---

## Key concepts

### labelSelector

Selects the **other Pods** you want to be near (or away from), by their labels.

### topologyKey (mandatory)

A Node label key that defines the **topology domain** in which the rule is
evaluated. It answers "same what?" — same Node, same zone, same region.

Common values:

- `kubernetes.io/hostname` — per-Node (each node is its own domain).
- `topology.kubernetes.io/zone` — per-availability-zone.
- `topology.kubernetes.io/region` — per-region.

The scheduler groups Nodes by the value of `topologyKey`; the affinity rule is
satisfied/violated **within each such group**.

`topologyKey` is required and cannot be empty.

### required vs preferred

Same semantics as node affinity:

- `requiredDuringSchedulingIgnoredDuringExecution` — hard; Pod stays `Pending`
  if unsatisfiable.
- `preferredDuringSchedulingIgnoredDuringExecution` — soft; weighted preference.

For `preferred`, the entry wraps the term in `podAffinityTerm` with a `weight`.

---

## podAffinity (co-location)

Schedule this Pod onto a Node in the same topology domain as Pods labeled
`app=cache`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: cache
          topologyKey: kubernetes.io/hostname
  containers:
    - name: nginx
      image: nginx:1.27
```

Use case: keep a web frontend on the same node/zone as its cache to reduce
latency.

---

## podAntiAffinity (spreading)

Ensure no two Pods of the same app land on the same Node — a classic HA pattern:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: web
              topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.27
```

With `topologyKey: kubernetes.io/hostname` and hard anti-affinity, each replica
must land on a distinct Node. If replicas > nodes, the extras stay `Pending`.

Soft version (prefer to spread, but pack if necessary):

```yaml
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: web
                topologyKey: kubernetes.io/hostname
```

---

## labelSelector operators

`labelSelector` supports `matchLabels` and `matchExpressions` (with
`In / NotIn / Exists / DoesNotExist`):

```yaml
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values: ["web", "api"]
          topologyKey: topology.kubernetes.io/zone
```

There is an optional `namespaces` field (and `namespaceSelector`) to match Pods
in other namespaces; by default the selector applies to the Pod's own namespace.

---

## Full example: co-locate with cache, spread own replicas

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 80
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: cache
                topologyKey: kubernetes.io/hostname
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: web
              topologyKey: kubernetes.io/hostname
      containers:
        - name: nginx
          image: nginx:1.27
```

Prefer nodes that already run the cache; require that no two `web` Pods share a
node.

---

## topologySpreadConstraints (brief)

`topologySpreadConstraints` is a newer, simpler mechanism for even distribution
of Pods across topology domains. It is often preferred over `podAntiAffinity`
for spreading because it is less expensive to compute and lets you set how much
skew is acceptable.

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule   # or ScheduleAnyway
      labelSelector:
        matchLabels:
          app: web
  containers:
    - name: nginx
      image: nginx:1.27
```

- `maxSkew` — max allowed difference in Pod count between the most and least
  populated domains.
- `whenUnsatisfiable` — `DoNotSchedule` (hard) or `ScheduleAnyway` (soft).
- `topologyKey` — the domain to balance across.

---

## Exam Tips

- **`topologyKey` is required** and drives the meaning of the rule. `hostname`
  = per-node; `zone` = per-zone. Forgetting it is a common mistake.
- `podAffinity` = together; `podAntiAffinity` = apart.
- Hard anti-affinity with `topologyKey: kubernetes.io/hostname` and more
  replicas than nodes leaves extra Pods **Pending** — expect this.
- For `preferred`, remember the extra `podAffinityTerm` wrapper and `weight`.
- The `labelSelector` matches **other Pods' labels**; don't confuse it with node
  labels (that's node affinity).
- By default selectors are scoped to the Pod's own namespace; use `namespaces`
  to look across namespaces.
- Pod affinity is computationally heavier than node affinity; for pure
  "spread evenly" tasks, `topologySpreadConstraints` is often the cleaner
  answer.

---

## Key Commands

```bash
# See which node each pod landed on (verify affinity worked)
kubectl get pods -o wide
kubectl get pods -o wide --sort-by=.spec.nodeName

# Pod labels drive the selectors
kubectl get pods --show-labels

# Node labels back topologyKey (zone/region/hostname)
kubectl get nodes -L topology.kubernetes.io/zone -L kubernetes.io/hostname

# Diagnose pending pods (events mention affinity/anti-affinity)
kubectl describe pod <pod>

# Explain the schema
kubectl explain pod.spec.affinity.podAffinity
kubectl explain pod.spec.affinity.podAntiAffinity
kubectl explain pod.spec.topologySpreadConstraints
```
