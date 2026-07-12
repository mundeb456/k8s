# CronJobs

A **CronJob** creates **Jobs on a repeating schedule**, written in standard cron format. It is the Kubernetes equivalent of a Unix crontab entry: use it for periodic and recurring tasks such as backups, report generation, cleanup, and sending emails.

A CronJob manages Jobs; each Job in turn manages Pods. So the hierarchy is:

```
CronJob  ->  Job  ->  Pod(s)
```

API group: **`batch/v1`** (stable since v1.21).

---

## Schedule (Cron Syntax)

`spec.schedule` uses the standard 5-field cron format, interpreted in the controller's time zone (UTC by default):

```
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of week (0 - 6) (Sunday = 0 or 7)
# │ │ │ │ │
# * * * * *
```

Common examples:

| Schedule | Meaning |
|----------|---------|
| `* * * * *` | Every minute |
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour, on the hour |
| `0 0 * * *` | Every day at midnight |
| `0 3 * * 0` | 03:00 every Sunday |
| `0 0 1 * *` | Midnight on the 1st of every month |
| `15 14 1 * *` | 14:15 on the 1st of every month |

Shorthand macros are also supported: `@yearly`, `@monthly`, `@weekly`, `@daily`, `@hourly`.

### timeZone

`spec.timeZone` (stable v1.27+) sets the time zone for the schedule, e.g. `"America/New_York"`. Without it, the schedule uses the time zone of the kube-controller-manager (usually UTC).

---

## jobTemplate

`spec.jobTemplate` is a full Job template. Everything you can put in a Job spec (completions, parallelism, backoffLimit, the Pod template with `restartPolicy: Never|OnFailure`) goes here.

```yaml
spec:
  jobTemplate:
    spec:
      completions: 1
      backoffLimit: 3
      template:
        spec:
          restartPolicy: OnFailure
          containers: [...]
```

---

## concurrencyPolicy

`spec.concurrencyPolicy` controls what happens when a new scheduled run is due but the **previous run is still running**:

- **`Allow`** (default) — allow concurrent Jobs to run simultaneously.
- **`Forbid`** — skip the new run if the previous is still active. The new run is **not queued**; it is missed.
- **`Replace`** — cancel the currently running Job and replace it with the new one.

```yaml
spec:
  concurrencyPolicy: Forbid
```

---

## startingDeadlineSeconds

`spec.startingDeadlineSeconds` — if a scheduled run misses its start time (e.g., the controller was down, or too many Jobs were suspended), this is the deadline (in seconds) within which the Job may still be started. If the deadline passes, the run is counted as **missed** and skipped.

If not set, there is no deadline and missed schedules will still attempt to run when the controller comes back (subject to the 100-missed-schedules limit — if more than 100 schedules are missed, the CronJob stops scheduling and logs an error).

```yaml
spec:
  startingDeadlineSeconds: 200
```

---

## History Limits

- **`successfulJobsHistoryLimit`** — how many **completed** Jobs to retain. Default `3`.
- **`failedJobsHistoryLimit`** — how many **failed** Jobs to retain. Default `1`.

Setting either to `0` keeps none (Jobs are cleaned up immediately after finishing).

```yaml
spec:
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
```

---

## suspend

`spec.suspend` — if `true`, the controller does **not** schedule new Jobs. Already-running Jobs continue. Use it to temporarily pause a CronJob without deleting it. Default `false`.

```yaml
spec:
  suspend: true
```

```bash
# Suspend / resume imperatively
kubectl patch cronjob backup -p '{"spec":{"suspend":true}}'
kubectl patch cronjob backup -p '{"spec":{"suspend":false}}'
```

---

## Full YAML Example

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
spec:
  schedule: "0 2 * * *"              # 02:00 daily
  timeZone: "Etc/UTC"
  concurrencyPolicy: Forbid          # do not overlap runs
  startingDeadlineSeconds: 300       # allow up to 5 min late start
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  suspend: false
  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 600
      template:
        spec:
          restartPolicy: OnFailure   # Never or OnFailure
          containers:
            - name: backup
              image: bitnami/postgresql:16
              command:
                - "/bin/sh"
                - "-c"
                - "pg_dump -h db -U admin app > /backup/dump-$(date +%F).sql"
              volumeMounts:
                - name: backup-vol
                  mountPath: /backup
          volumes:
            - name: backup-vol
              persistentVolumeClaim:
                claimName: backup-pvc
```

---

## Common Operations

```bash
# Create declaratively
kubectl apply -f cronjob.yaml

# Imperative create (great for the exam)
kubectl create cronjob backup --image=busybox --schedule="*/5 * * * *" -- /bin/sh -c "date"

# Scaffold YAML without applying
kubectl create cronjob backup --image=busybox --schedule="*/5 * * * *" \
  --dry-run=client -o yaml -- /bin/sh -c "date" > cronjob.yaml

# List and inspect
kubectl get cronjob                 # or: kubectl get cj
kubectl describe cronjob backup

# See the Jobs it has spawned
kubectl get jobs --watch

# Trigger a manual run from a CronJob (create a one-off Job)
kubectl create job --from=cronjob/backup backup-manual-001

# Suspend / resume
kubectl patch cronjob backup -p '{"spec":{"suspend":true}}'

# Delete
kubectl delete cronjob backup
```

---

## Exam Tips

- **API group is `batch/v1`** and kind is `CronJob` (capital C, capital J).
- **Cron format has 5 fields.** Memorize `*/5 * * * *` (every 5 min), `0 0 * * *` (daily midnight), `0 3 * * 0` (Sunday 3am).
- **`concurrencyPolicy`:** `Allow` (default, overlap), `Forbid` (skip if still running), `Replace` (kill old, start new). Very commonly tested.
- **`startingDeadlineSeconds`** decides whether a missed run is still allowed to start late. If more than 100 runs are missed with no deadline, scheduling halts.
- **History limits default:** `successfulJobsHistoryLimit: 3`, `failedJobsHistoryLimit: 1`. Set to `0` to keep none.
- **`suspend: true`** pauses scheduling. Use `kubectl patch` to toggle quickly.
- **Manually trigger** a CronJob's Job with `kubectl create job --from=cronjob/<name> <new-job-name>` — a favorite exam task.
- The Pod template still needs **`restartPolicy: Never` or `OnFailure`** (same rule as Jobs).
- Use `--dry-run=client -o yaml` to scaffold; the `--` separates the container command.

---

## Key Commands

```bash
kubectl apply -f cronjob.yaml                                            # create
kubectl create cronjob backup --image=busybox --schedule="*/5 * * * *" -- date   # imperative
kubectl create cronjob backup --image=busybox --schedule="* * * * *" \
  --dry-run=client -o yaml -- date                                       # scaffold
kubectl get cj                                                           # list cronjobs
kubectl describe cronjob backup                                          # detail
kubectl create job --from=cronjob/backup manual-run                      # trigger now
kubectl patch cronjob backup -p '{"spec":{"suspend":true}}'              # suspend
kubectl delete cronjob backup                                            # delete
```
