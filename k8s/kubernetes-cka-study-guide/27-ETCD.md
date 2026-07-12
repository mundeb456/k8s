# 27 - etcd: Backup and Restore

etcd is the primary datastore of a Kubernetes cluster. Backing up and restoring
etcd is one of the most reliably-tested topics on the CKA exam. Master the
`etcdctl snapshot save` / `snapshot restore` workflow until it is muscle memory.

---

## What etcd is

- etcd is a **consistent, distributed, highly-available key-value store**.
- The Kubernetes API server is the **only** component that talks to etcd directly.
- Everything about cluster state lives in etcd: Pods, Deployments, Secrets,
  ConfigMaps, Roles, ServiceAccounts, node registrations, etc.
- If you lose etcd and have no backup, you lose the entire cluster state.
- In a `kubeadm` cluster, etcd usually runs as a **static Pod** on each control
  plane node, managed by the kubelet from a manifest in
  `/etc/kubernetes/manifests/etcd.yaml`.

```bash
# See the etcd static pod
kubectl get pods -n kube-system -l component=etcd

# Confirm etcd is running as a static pod managed by kubelet
ls /etc/kubernetes/manifests/
# etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

---

## etcdctl and the v3 API

Always set the API version to 3. On modern etcd (3.4+) v3 is the default, but the
exam graders and older tooling expect you to be explicit.

```bash
export ETCDCTL_API=3
etcdctl version
```

### The four connection flags you must know

Because etcd is protected by mutual TLS, every read/write/snapshot command needs
these flags (values come from the etcd static pod manifest):

| Flag | Meaning | Typical value |
|------|---------|---------------|
| `--endpoints` | etcd listen address | `https://127.0.0.1:2379` |
| `--cacert` | CA cert to verify the etcd server | `/etc/kubernetes/pki/etcd/ca.crt` |
| `--cert` | client cert for authentication | `/etc/kubernetes/pki/etcd/server.crt` |
| `--key` | client key for authentication | `/etc/kubernetes/pki/etcd/server.key` |

### Finding the correct flag values

Never guess. Read them straight from the manifest:

```bash
cat /etc/kubernetes/manifests/etcd.yaml | grep -E "listen-client-urls|cert-file|key-file|trusted-ca-file|advertise-client-urls"
```

You will see something like:

```yaml
    - --advertise-client-urls=https://192.168.0.10:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://192.168.0.10:2379
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

### Verifying you can talk to etcd

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

You should see one row per etcd member with `started` status.

---

## Also learn the etcd version

The exam sometimes asks for the etcd version. Read it from the manifest image tag:

```bash
grep image: /etc/kubernetes/manifests/etcd.yaml
# image: registry.k8s.io/etcd:3.5.15-0
```

---

## Backup: `snapshot save`

This creates a point-in-time snapshot file of the entire etcd datastore.

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db
```

You should see `Snapshot saved at /opt/etcd-backup.db`.

### Verifying a snapshot

```bash
ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db --write-out=table
```

Example output:

```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 3e3f8c2a |    23481 |       1372 |     4.8 MB |
+----------+----------+------------+------------+
```

> Note: on newer etcdctl, `snapshot status` prints a deprecation warning about
> requiring `--endpoints` — it still works and does **not** need the TLS flags
> because it reads a local file.

---

## Restore: `snapshot restore`

Restore extracts a snapshot into a **new data directory**. It does NOT overwrite
the live etcd data directory in place, and you must repoint etcd at the new dir.

```bash
ETCDCTL_API=3 etcdctl \
  snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-from-backup
```

Key points:

- `--data-dir` is the **most important** restore flag. Restore into a fresh
  directory (e.g. `/var/lib/etcd-from-backup`), never the running one.
- Restore is a **local operation** on the snapshot file — the TLS/endpoint flags
  are not required for `restore` (only for `save`, `member list`, live queries).
- Optional flags for multi-member clusters: `--name`, `--initial-cluster`,
  `--initial-cluster-token`, `--initial-advertise-peer-urls`. For the single-node
  exam scenario you usually only need `--data-dir`.

---

## Full backup + restore procedure with kubeadm

This is the exact sequence to practice for the exam.

### 1. Take the backup

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db
```

### 2. Restore the snapshot into a new data directory

```bash
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-from-backup
```

### 3. Point the etcd static pod at the new data directory

Edit `/etc/kubernetes/manifests/etcd.yaml`. Change the **hostPath volume** that
backs `/var/lib/etcd` so it maps to the restored directory:

```yaml
  volumes:
  - hostPath:
      path: /var/lib/etcd-from-backup   # <-- changed from /var/lib/etcd
      type: DirectoryOrCreate
    name: etcd-data
```

Do NOT change the `mountPath` inside the container (`/var/lib/etcd`) — that stays
the same. You only change the host path.

> Alternative approach some scenarios use: keep the volume path the same and pass
> `--data-dir=/var/lib/etcd-from-backup` as an etcd command arg. Whichever you do,
> the container's `--data-dir` and the mounted volume must agree. Changing the
> hostPath volume is the cleaner, most common answer.

### 4. Let kubelet restart etcd

Because etcd is a static pod, the kubelet detects the manifest change and
recreates the pod automatically. If it seems stuck, restart the kubelet:

```bash
systemctl restart kubelet
```

### 5. Verify the cluster recovered

```bash
kubectl get pods -n kube-system
kubectl get nodes
kubectl get all -A
# The objects that existed at snapshot time should be back.
```

Watch etcd come back specifically:

```bash
watch crictl ps        # container runtime view
kubectl get pod -n kube-system -l component=etcd
```

---

## Setting flags once with environment variables

To type less under exam time pressure, export the common flags:

```bash
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key

# Now this is enough:
etcdctl snapshot save /opt/etcd-backup.db
etcdctl member list
```

---

## Exam-critical commands (memorize these)

```bash
# BACKUP
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/snapshot.db

# STATUS (verify a snapshot file)
ETCDCTL_API=3 etcdctl snapshot status /opt/snapshot.db --write-out=table

# RESTORE (into a fresh data dir)
ETCDCTL_API=3 etcdctl snapshot restore /opt/snapshot.db \
  --data-dir=/var/lib/etcd-from-backup

# THEN edit /etc/kubernetes/manifests/etcd.yaml hostPath -> /var/lib/etcd-from-backup

# Read the correct flag values from the manifest
grep -E "cert-file|key-file|trusted-ca-file|listen-client-urls" \
  /etc/kubernetes/manifests/etcd.yaml
```

---

## Common gotchas

- **Wrong cert paths.** `--cacert` uses the etcd CA (`pki/etcd/ca.crt`), not the
  apiserver CA (`pki/ca.crt`). When in doubt, read the manifest.
- **Restoring into the live data dir.** Always use a new `--data-dir` and then
  repoint the volume; do not overwrite `/var/lib/etcd` directly.
- **Forgetting `ETCDCTL_API=3`.** Some environments still default to v2 for
  historical commands; set it explicitly.
- **Editing the container mountPath instead of the hostPath.** Only the host path
  changes on restore.
- **Not waiting.** After editing the static pod manifest, give kubelet a moment
  (or restart it) before checking `kubectl get nodes`.
