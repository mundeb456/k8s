# Multiple Schedulers

Kubernetes ships with a default scheduler (`default-scheduler`), but you can run
**additional custom schedulers** alongside it and choose, per-pod, which one
schedules the pod. Useful when workloads need custom placement logic the default
scheduler doesn't provide.

## Why Multiple Schedulers

- Custom scheduling algorithms (e.g. gang scheduling, ML/GPU-aware placement).
- Isolate scheduling policy for a class of workloads without touching the
  default scheduler.
- Test a new scheduler in production without replacing the default one.

Multiple schedulers run **simultaneously**. Each pod is scheduled by exactly one
scheduler, selected via `.spec.schedulerName`. If omitted, `default-scheduler`
is used.

## How a Pod Picks a Scheduler

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-custom-sched
spec:
  schedulerName: my-scheduler   # must match the scheduler's profile name
  containers:
  - name: nginx
    image: nginx:1.27
```

- If `schedulerName` matches a running scheduler → that scheduler binds the pod.
- If **no** scheduler with that name is running → pod stays **Pending** forever
  (nothing claims it). Common exam gotcha.

## Approach 1 — Scheduler Profiles (preferred, single binary)

Since v1.18 the default `kube-scheduler` binary can run **multiple profiles**,
each with its own name — no need to deploy a second binary. One scheduler
process, multiple logical schedulers.

`KubeSchedulerConfiguration`:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
  - schedulerName: my-scheduler
    plugins:
      score:
        disabled:
          - name: PodTopologySpread
        enabled:
          - name: ImageLocality
            weight: 5
```

Pass to kube-scheduler with `--config=/etc/kubernetes/my-scheduler-config.yaml`.
A pod with `schedulerName: my-scheduler` gets scheduled by that profile.

## Approach 2 — Deploy a Second Scheduler (separate Deployment)

Run another instance of `kube-scheduler` (or a fully custom scheduler binary) as
a Deployment in `kube-system`, with its own name.

Config for the second scheduler:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: my-scheduler
leaderElection:
  leaderElect: true
  resourceNamespace: kube-system
  resourceName: my-scheduler        # unique — avoid clashing with default
```

Store as a ConfigMap and mount it:

```bash
kubectl create configmap my-scheduler-config \
  --from-file=my-scheduler-config.yaml -n kube-system
```

Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-scheduler
  namespace: kube-system
  labels:
    component: my-scheduler
spec:
  replicas: 1
  selector:
    matchLabels:
      component: my-scheduler
  template:
    metadata:
      labels:
        component: my-scheduler
    spec:
      serviceAccountName: my-scheduler
      containers:
      - name: kube-scheduler
        image: registry.k8s.io/kube-scheduler:v1.31.0
        command:
        - kube-scheduler
        - --config=/etc/kubernetes/my-scheduler/my-scheduler-config.yaml
        volumeMounts:
        - name: config
          mountPath: /etc/kubernetes/my-scheduler
      volumes:
      - name: config
        configMap:
          name: my-scheduler-config
```

## Approach 3 — Static Pod Scheduler

Mirror the default scheduler: drop a scheduler manifest into
`/etc/kubernetes/manifests/` with a distinct `--config` naming a different
`schedulerName`. Kubelet runs it as a static pod. See [26-StaticPods.md](26-StaticPods.md).

## RBAC for a Custom Scheduler

A separate scheduler needs permissions (bind pods, read nodes, events, leader
election leases). Easiest for the exam — reuse the built-in role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-scheduler-as-kube-scheduler
subjects:
- kind: ServiceAccount
  name: my-scheduler
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:kube-scheduler
  apiGroup: rbac.authorization.k8s.io
```

Also bind `system:volume-scheduler` and grant leader-election on the custom
lease name if `leaderElect: true`.

## Verify Which Scheduler Placed a Pod

```bash
# Scheduler is recorded in the pod's events
kubectl describe pod nginx-custom-sched | grep -i scheduled
kubectl get events -o wide | grep Scheduled

# Confirm the second scheduler is running
kubectl get pods -n kube-system | grep my-scheduler
kubectl logs -n kube-system deploy/my-scheduler
```

The `Scheduled` event `Source` / message shows which scheduler bound the pod.

## Exam Tips

- Pod stuck `Pending` + no scheduler errors in `describe`? Check
  `schedulerName` — likely names a scheduler that isn't running.
- `default-scheduler` is the name to match when `schedulerName` is omitted.
- Prefer **scheduler profiles** (Approach 1) over deploying a second binary —
  fewer moving parts, less RBAC.
- Two schedulers can both try to place the same pod only if misconfigured;
  normally one pod → one `schedulerName` → one scheduler. Race/conflicts show as
  pods bound twice — avoid overlapping profile names.
- If running a second scheduler with `leaderElect: true`, give it a **unique**
  `resourceName`/lease so it doesn't fight the default scheduler's lease.
- Scheduler image: `registry.k8s.io/kube-scheduler:<version>` (match cluster
  version).

## Key Commands

```bash
# Which scheduler placed the pod
kubectl describe pod <pod> | grep -iE 'scheduled|scheduler'

# Custom scheduler health
kubectl get pods -n kube-system | grep my-scheduler
kubectl logs -n kube-system deploy/my-scheduler

# Create scheduler config as ConfigMap
kubectl create configmap my-scheduler-config \
  --from-file=my-scheduler-config.yaml -n kube-system

# Watch pending pods (waiting on a scheduler)
kubectl get pods --field-selector status.phase=Pending -A
```

See also: [22-Scheduling.md](22-Scheduling.md), [26-StaticPods.md](26-StaticPods.md).
