# Jobs

A **Job** creates one or more Pods and ensures that a specified number of them **successfully terminate**. Unlike Deployments (which keep Pods running indefinitely), a Job runs Pods to **completion**. When the required number of successful completions is reached, the Job is complete.

Jobs are for **run-to-completion / batch** work: database migrations, backups, report generation, batch processing, one-off scripts.

---

## Job Concept

- A Job tracks the successful completion of Pods.
- When a Pod fails or is deleted, the Job may start a new Pod (subject to `backoffLimit` and `restartPolicy`).
- When the target number of successes is reached, the Job stops creating Pods and is marked **Complete**.
- The Pods (and their logs) remain after completion for inspection, until you delete the Job or a TTL expires.

---

## Key Fields

### completions

`spec.completions` — the total number of Pods that must **succeed** for the Job to be considered complete. Default is `1`.

### parallelism

`spec.parallelism` — the maximum number of Pods that may run **concurrently**. Default is `1`.

Combining these gives three common patterns:

| Pattern | completions | parallelism | Behavior |
|---------|-------------|-------------|----------|
| Single run | 1 (default) | 1 (default) | One Pod, must succeed once |
| Fixed count, sequential | N | 1 | Run N Pods one after another |
| Fixed count, parallel | N | M | Run up to M at once until N succeed |
| Work queue | unset | M | M Pods run in parallel; each exits when the queue is empty |

### backoffLimit

`spec.backoffLimit` — the number of **retries** before the Job is marked **Failed**. Default is `6`. Failed Pods are retried with an exponential back-off delay (10s, 20s, 40s ... capped at 6 minutes).

### activeDeadlineSeconds

`spec.activeDeadlineSeconds` — a wall-clock time limit for the entire Job. Once exceeded, the Job and all its running Pods are terminated and the Job is marked Failed with reason `DeadlineExceeded`. This takes **precedence over `backoffLimit`**.

### restartPolicy

The **Pod template** `restartPolicy` for a Job must be `Never` or `OnFailure` (it **cannot** be `Always`).

- **`Never`** — a failed container is not restarted in place; the Pod is marked failed and the Job creates a **new Pod**. You will see multiple Pod objects for retries.
- **`OnFailure`** — the kubelet **restarts the failed container in the same Pod**; you see fewer Pod objects but the container restart count rises.

### ttlSecondsAfterFinished

`spec.ttlSecondsAfterFinished` — automatically delete the Job (and its Pods) this many seconds after it finishes. Great for cleanup.

---

## completionMode: Indexed

`spec.completionMode` can be `NonIndexed` (default) or `Indexed`.

- **NonIndexed** — the Job is complete when `completions` Pods succeed; Pods are interchangeable.
- **Indexed** — each Pod gets a unique **completion index** from `0` to `completions-1`. The index is exposed to the Pod via:
  - the annotation `batch.kubernetes.io/job-completion-index`
  - the environment variable `JOB_COMPLETION_INDEX`
  - the Pod hostname suffix (`<job>-<index>`)

Indexed Jobs are useful for **static work partitioning** — each index processes a fixed shard of the workload (e.g., index 0 handles file 0, index 1 handles file 1).

```yaml
spec:
  completions: 5
  parallelism: 3
  completionMode: Indexed
```

---

## Full YAML Examples

### Simple single-completion Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 4
  activeDeadlineSeconds: 100
  ttlSecondsAfterFinished: 600     # auto-clean 10 min after finish
  template:
    spec:
      restartPolicy: Never          # or OnFailure
      containers:
        - name: pi
          image: perl:5.34.0
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
```

### Parallel Indexed Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: indexed-batch
spec:
  completions: 5
  parallelism: 3
  completionMode: Indexed
  backoffLimit: 6
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: worker
          image: busybox:1.36
          command:
            - "sh"
            - "-c"
            - "echo Processing shard $JOB_COMPLETION_INDEX; sleep 5"
```

---

## Common Operations

```bash
# Create
kubectl apply -f job.yaml

# Imperative create
kubectl create job pi --image=perl:5.34.0 -- perl -Mbignum=bpi -wle 'print bpi(2000)'

# Generate YAML without applying (very useful in the exam)
kubectl create job pi --image=busybox --dry-run=client -o yaml -- echo hello > job.yaml

# Watch status
kubectl get jobs
kubectl get pods --selector=job-name=pi

# View logs from the Job's Pod
kubectl logs job/pi

# Describe (see completions, failures, events)
kubectl describe job pi

# Delete Job and its Pods
kubectl delete job pi
```

---

## Exam Tips

- **`restartPolicy` must be `Never` or `OnFailure`** for Jobs — `Always` is rejected. This is a common trap when converting a Deployment/Pod spec into a Job.
- **`completions` = required successes**, **`parallelism` = concurrency**. Know the difference cold.
- **`backoffLimit` default is 6.** After that many failures the Job is marked Failed.
- **`activeDeadlineSeconds` overrides `backoffLimit`** — it is a hard time cap on the whole Job.
- **`Never` creates a new Pod per retry** (multiple Pod objects); **`OnFailure` restarts the container in place** (rising restart count). If asked why there are many Pods for a failing Job, the answer is `restartPolicy: Never`.
- Use `kubectl create job ... --dry-run=client -o yaml` to scaffold quickly, then edit.
- **Indexed mode** gives each Pod `JOB_COMPLETION_INDEX` (0..N-1) for static partitioning.
- API group is **`batch/v1`**.
- Logs from a Job: `kubectl logs job/<name>` (grabs one of its Pods). Use `-l job-name=<name>` to select all its Pods.
- `ttlSecondsAfterFinished` auto-cleans finished Jobs — handy but requires the TTL controller (on by default).

---

## Key Commands

```bash
kubectl apply -f job.yaml                                          # create
kubectl create job pi --image=busybox -- echo hi                   # imperative
kubectl create job pi --image=busybox --dry-run=client -o yaml     # scaffold YAML
kubectl get jobs                                                   # list
kubectl get pods -l job-name=pi                                    # job's pods
kubectl logs job/pi                                                # job logs
kubectl describe job pi                                            # detail + events
kubectl delete job pi                                              # delete
```
