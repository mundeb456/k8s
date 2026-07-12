# Persistent Volumes

Persistent storage in Kubernetes decouples *how storage is provisioned* from *how it is consumed*, using two API objects:

- **PersistentVolume (PV)** — a piece of storage in the cluster, provisioned by an administrator (static) or dynamically by a StorageClass. It is a **cluster-scoped** resource with a lifecycle independent of any Pod.
- **PersistentVolumeClaim (PVC)** — a **namespaced** request for storage by a user. It specifies size and access mode, and Kubernetes binds it to a matching PV.

A Pod never references a PV directly; it references a **PVC**, and the PVC is bound to a PV. This indirection lets developers request storage without knowing the underlying infrastructure.

```
Pod ── uses ──▶ PVC ── binds to ──▶ PV ── backed by ──▶ real storage (EBS, NFS, Ceph, ...)
```

## PersistentVolume YAML (static provisioning)

Static provisioning means the administrator creates PVs ahead of time. A PVC then binds to one of them.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data-01
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem          # Filesystem (default) or Block
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual        # PVC must request the same class (or "" ) to bind
  hostPath:                       # demo backend; production uses csi/nfs/etc.
    path: /mnt/data
```

## PersistentVolumeClaim YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 3Gi        # asks for 3Gi; can bind to the 5Gi PV above
  storageClassName: manual
```

Binding succeeds when a PV satisfies **all** of the PVC's requirements: `accessModes` (the PV must support them), `storageClassName` (must match exactly), `volumeMode`, and `capacity` ≥ requested `storage`. The bind is **1:1** — once a PV is bound to a PVC, no other PVC can use it, even if there is spare capacity.

## Access Modes

Access modes describe how nodes may mount the volume. Support depends on the backend; not every backend supports every mode.

| Mode | Short | Meaning |
|------|-------|---------|
| `ReadWriteOnce` | RWO | Mounted read-write by a **single node**. Multiple Pods on that same node may use it. |
| `ReadOnlyMany` | ROX | Mounted read-only by **many nodes**. |
| `ReadWriteMany` | RWX | Mounted read-write by **many nodes**. |
| `ReadWriteOncePool` | RWOP | Mounted read-write by a **single Pod** (enforced). Requires CSI + feature support. |

Key nuance: `ReadWriteOnce` is enforced at the **node** level, not the Pod level. Two Pods scheduled to the *same* node can both write to an RWO volume. `ReadWriteOncePool` (RWOP) is the mode that truly restricts to a single Pod.

Block storage (EBS, GCE PD, Azure Disk) typically supports only RWO. Network filesystems (NFS, CephFS) support RWX.

## Reclaim Policies

The reclaim policy (`spec.persistentVolumeReclaimPolicy` on the PV) determines what happens to the PV — and its underlying storage — when its bound PVC is deleted.

| Policy | Behavior on PVC deletion |
|--------|--------------------------|
| `Retain` | PV enters **Released** state; data is kept. An admin must manually reclaim it (delete/recreate the PV or clean the data). Safest; no accidental data loss. |
| `Delete` | The PV **and** the underlying storage asset are deleted automatically. Default for dynamically provisioned PVs. |
| `Recycle` | **Deprecated.** Performed a basic scrub (`rm -rf`) and made the PV available again. Do not use; replaced by dynamic provisioning. |

A `Retain` PV that has been Released cannot be re-bound automatically — its `claimRef` still points at the old PVC. You must clear the `claimRef` (edit the PV) or recreate the PV to make it Available again.

## Capacity and volumeMode

- **`capacity.storage`** — the size of the PV. A PVC binds to a PV only if PV capacity ≥ PVC request. Excess capacity is not reclaimed or shared.
- **`volumeMode`**:
  - `Filesystem` (default) — the volume is mounted into the Pod as a directory. If the volume is a raw block device, Kubernetes creates a filesystem on it first.
  - `Block` — the volume is presented to the container as a raw block device (no filesystem). The container is responsible for how to use it. Use `volumeDevices` (not `volumeMounts`) in the Pod for block mode.

PV and PVC `volumeMode` must match for binding.

## Binding Lifecycle & PV Phases

A PV moves through these phases (`kubectl get pv` shows the `STATUS`):

| Phase | Meaning |
|-------|---------|
| `Available` | Free, not yet bound to any claim. |
| `Bound` | Bound to a PVC. |
| `Released` | The PVC was deleted, but the resource has not yet been reclaimed by the cluster (typical with `Retain`). |
| `Failed` | Automatic reclamation failed. |

A PVC is `Pending` until it binds, then `Bound`. If no matching PV exists and no dynamic provisioning is available, the PVC stays `Pending` and any Pod using it stays `Pending` too (`kubectl describe pod` shows `unbound immediate PersistentVolumeClaims`).

## Using a PVC in a Pod

The Pod references the PVC by name via a `persistentVolumeClaim` volume source.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-consumer
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc-data      # references the PVC, never the PV directly
```

For **block** mode, use `volumeDevices` instead:

```yaml
    spec:
      containers:
        - name: app
          image: busybox:1.36
          volumeDevices:
            - name: data
              devicePath: /dev/xvda      # raw device path in the container
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: pvc-block
```

## Full End-to-End Example

```yaml
# 1. PV (static, admin-created)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-app
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/app-data
---
# 2. PVC (developer request)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: manual
---
# 3. Pod consuming the PVC
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: app-storage
          mountPath: /usr/share/nginx/html
  volumes:
    - name: app-storage
      persistentVolumeClaim:
        claimName: pvc-app
```

## Exam Tips

- A Pod uses a **PVC**, never a PV directly. If a task says "mount this storage into a Pod," you reference `persistentVolumeClaim.claimName`.
- PVC stuck `Pending`: check `storageClassName` matches, requested size ≤ PV capacity, and access modes are compatible. `kubectl describe pvc <name>` gives the reason.
- `storageClassName: ""` (empty string) explicitly disables dynamic provisioning and forces binding to a matching static PV. Omitting the field entirely may fall back to the **default** StorageClass.
- `ReadWriteOnce` = one **node**, not one Pod. `ReadWriteOncePool` (RWOP) = one **Pod**. This is a favorite trick question.
- Default reclaim policy for **dynamic** PVs is `Delete`; for **static** PVs you set it explicitly (`Retain` is safest for exam data-protection scenarios).
- Binding is 1:1 and consumes the whole PV regardless of size difference.
- `Recycle` is deprecated — never the correct answer for a modern setup.
- To expand a PVC you edit `spec.resources.requests.storage` upward — but only if the StorageClass has `allowVolumeExpansion: true` (see `11-StorageClass.md`). You cannot shrink a PVC.
- Deleting a PVC bound to a `Retain` PV leaves the PV `Released`; it will not auto-rebind until you clear its `claimRef`.

## Key Commands

```bash
# List and inspect
kubectl get pv
kubectl get pvc
kubectl get pv,pvc -o wide
kubectl describe pv pv-app
kubectl describe pvc pvc-app          # shows binding status and events

# See why a PVC is Pending
kubectl describe pvc pvc-app | grep -A5 Events

# Apply resources
kubectl apply -f pv.yaml
kubectl apply -f pvc.yaml

# Expand a PVC (requires allowVolumeExpansion on the StorageClass)
kubectl edit pvc pvc-app              # increase spec.resources.requests.storage
# or patch:
kubectl patch pvc pvc-app -p '{"spec":{"resources":{"requests":{"storage":"5Gi"}}}}'

# Make a Retain PV bindable again after Release: clear the claimRef
kubectl patch pv pv-app -p '{"spec":{"claimRef": null}}'

# Delete (order matters: delete the Pod, then PVC, then PV)
kubectl delete pod app-pod
kubectl delete pvc pvc-app
kubectl delete pv pv-app

# Field reference
kubectl explain pv.spec
kubectl explain pvc.spec
```
