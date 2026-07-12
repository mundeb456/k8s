# Deployments

A **Deployment** provides declarative updates for Pods and ReplicaSets. You describe the desired state, and the Deployment controller changes the actual state at a controlled rate — creating a new ReplicaSet for each revision, scaling it up while scaling the old one down. Deployments are the standard way to run **stateless** workloads and are heavily tested on the CKA.

## What a Deployment Gives You

- **Rollouts** — roll out a new version gradually.
- **Rollbacks** — revert to a previous revision if something breaks.
- **Scaling** — change replica count (manually or via HPA).
- **Self-healing** — via the ReplicaSets it manages.

The ownership chain is: `Deployment → ReplicaSet (one per revision) → Pods`.

## Full Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 3
  revisionHistoryLimit: 10          # how many old ReplicaSets to keep
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  template:
    metadata:
      labels:
        app: web                    # MUST match spec.selector.matchLabels
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
```

## Rollout Strategies

Set in `spec.strategy.type`:

### RollingUpdate (default)

Gradually replaces old Pods with new ones — zero downtime if readiness probes are set. Controlled by:

- **maxSurge** — how many Pods *above* `replicas` may exist during the update (extra new Pods spun up before old ones are removed). Absolute number or percent. Default `25%`.
- **maxUnavailable** — how many Pods *below* `replicas` may be unavailable during the update. Absolute number or percent. Default `25%`.

`maxSurge` and `maxUnavailable` cannot both be 0. With `maxSurge: 1, maxUnavailable: 0`, updates add one new Pod before removing an old one (safest, needs spare capacity).

### Recreate

Terminates **all** old Pods, then creates new ones. Causes downtime but guarantees no two versions run simultaneously — use when the app cannot tolerate mixed versions.

```yaml
spec:
  strategy:
    type: Recreate
```

## Creating and Updating

```bash
# Create imperatively
kubectl create deployment web --image=nginx --replicas=3

# Generate YAML to edit
kubectl create deployment web --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml
kubectl apply -f deploy.yaml

# Update the image (triggers a rollout)
kubectl set image deployment/web nginx=nginx:1.27
# 'nginx' above is the CONTAINER name, not the image name

# Edit live (triggers a rollout on template change)
kubectl edit deployment web
```

Only changes to the **Pod template** (`spec.template`) trigger a new rollout/ReplicaSet. Changing `replicas` scales but does not create a revision.

## Rollout Management

```bash
kubectl rollout status deployment/web          # watch progress; blocks until done or fails
kubectl rollout history deployment/web         # list revisions
kubectl rollout history deployment/web --revision=3   # detail of one revision
kubectl rollout undo deployment/web            # roll back to previous revision
kubectl rollout undo deployment/web --to-revision=2
kubectl rollout restart deployment/web         # restart all pods (fresh rollout, no spec change)
kubectl rollout pause deployment/web           # stop rollout to batch multiple changes
kubectl rollout resume deployment/web          # resume, applying batched changes at once
```

To capture the command in `CHANGE-CAUSE` for history, add an annotation:

```bash
kubectl annotate deployment/web kubernetes.io/change-cause="update to nginx 1.27"
```

(The old `--record` flag is deprecated; use the annotation instead.)

### Pausing/Resuming Pattern

Pause, make several edits (image, resources, env), then resume so all changes go out in **one** rollout instead of several:

```bash
kubectl rollout pause deployment/web
kubectl set image deployment/web nginx=nginx:1.27
kubectl set resources deployment/web -c nginx --limits=cpu=500m,memory=256Mi
kubectl rollout resume deployment/web
```

## revisionHistoryLimit

`spec.revisionHistoryLimit` (default **10**) caps how many old, scaled-to-zero ReplicaSets are retained for rollback. Set it to `0` to keep none (you lose rollback ability). Old ReplicaSets beyond the limit are garbage-collected.

```bash
kubectl get rs -l app=web        # see current + retained old ReplicaSets (0 desired)
```

## Scaling

```bash
kubectl scale deployment/web --replicas=5
kubectl scale --replicas=5 -f deploy.yaml

# Autoscale (needs metrics-server)
kubectl autoscale deployment/web --min=2 --max=10 --cpu-percent=80
kubectl get hpa
```

## Inspecting

```bash
kubectl get deploy
kubectl get deploy web -o wide           # READY, UP-TO-DATE, AVAILABLE, CONTAINERS, IMAGES
kubectl describe deployment web          # strategy, events, conditions
kubectl get deploy web -o yaml
```

`READY` = ready/desired; `UP-TO-DATE` = replicas updated to the latest template; `AVAILABLE` = replicas available to users.

## Troubleshooting Rollouts

- **Rollout stuck (`kubectl rollout status` hangs)**: new Pods failing readiness probes, `ImagePullBackOff`, or insufficient resources. Check `kubectl describe deploy` and `kubectl get pods`.
- **`ProgressDeadlineExceeded`**: the rollout did not progress within `spec.progressDeadlineSeconds` (default 600s). Fix the underlying Pod failure and it retries.
- **Bad image rolled out**: `kubectl rollout undo deployment/web` restores the last good revision immediately.
- Selector is **immutable** after creation, like ReplicaSets.

## Exam Tips

- Prefer Deployments over bare ReplicaSets/Pods for stateless apps — they are what most questions expect.
- `kubectl set image deployment/<name> <CONTAINER>=<image>` — the token before `=` is the **container name**, a frequent trip-up.
- Roll back fast with `kubectl rollout undo deployment/<name>` and target a specific point with `--to-revision=N`.
- `kubectl rollout restart` bounces all Pods without changing the spec — handy to reload updated ConfigMaps/Secrets.
- Only Pod-template changes create a new revision; scaling does not.
- Know `maxSurge` (extra Pods allowed) vs `maxUnavailable` (missing Pods allowed); they cannot both be 0.
- Use `--dry-run=client -o yaml` to generate, then edit `strategy`, `resources`, probes.
- Set readiness probes so RollingUpdate is truly zero-downtime.

## Key Commands

```bash
kubectl create deployment <name> --image=<img> --replicas=<n> [--dry-run=client -o yaml]
kubectl set image deployment/<name> <container>=<image>
kubectl scale deployment/<name> --replicas=<n>
kubectl rollout status  deployment/<name>
kubectl rollout history deployment/<name> [--revision=N]
kubectl rollout undo    deployment/<name> [--to-revision=N]
kubectl rollout restart deployment/<name>
kubectl rollout pause   deployment/<name>
kubectl rollout resume  deployment/<name>
kubectl annotate deployment/<name> kubernetes.io/change-cause="..."
kubectl autoscale deployment/<name> --min=2 --max=10 --cpu-percent=80
kubectl get deploy <name> -o wide
kubectl describe deployment <name>
```
