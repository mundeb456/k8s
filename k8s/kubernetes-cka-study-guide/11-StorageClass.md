# StorageClass

A **StorageClass** enables **dynamic provisioning**: instead of an administrator manually pre-creating PersistentVolumes, the cluster creates a PV on demand whenever a PersistentVolumeClaim requests it. This is the standard model in every cloud and most on-prem CSI setups.

When a PVC names a StorageClass (via `storageClassName`) and no matching static PV exists, the StorageClass's **provisioner** creates the backing storage and a matching PV, then binds it to the PVC automatically.

```
PVC (storageClassName: fast) ──▶ StorageClass "fast" ──▶ provisioner creates PV + real disk ──▶ PVC Bound
```

## StorageClass YAML

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: ebs.csi.aws.com          # the CSI driver / provisioner
parameters:                            # provisioner-specific options
  type: gp3
  encrypted: "true"
reclaimPolicy: Delete                  # Delete (default) or Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

StorageClass is **cluster-scoped** (not namespaced) and, notably, its fields are **immutable after creation** except for very limited cases — to change a provisioner or binding mode you generally delete and recreate the StorageClass.

## provisioner

The `provisioner` field names the volume plugin that creates the storage. Examples:

| Provisioner | Backend |
|-------------|---------|
| `ebs.csi.aws.com` | AWS EBS |
| `pd.csi.storage.gke.io` | GCE Persistent Disk |
| `disk.csi.azure.com` | Azure Disk |
| `rancher.io/local-path` | Local path (common in k3s/dev) |
| `kubernetes.io/no-provisioner` | No dynamic provisioning (used for local static volumes) |

`parameters` are entirely provisioner-specific (disk type, IOPS, filesystem, encryption, etc.) and are passed through to the driver.

## reclaimPolicy

Sets the `persistentVolumeReclaimPolicy` on PVs that this StorageClass dynamically creates.

- `Delete` (**default**) — when the PVC is deleted, the PV and the underlying storage asset are destroyed.
- `Retain` — the PV and data are kept after PVC deletion; an admin must reclaim manually.

Note this only applies to PVs created *by* the StorageClass; statically created PVs carry their own policy.

## volumeBindingMode

This is one of the most testable StorageClass fields.

| Mode | Behavior |
|------|----------|
| `Immediate` (default) | The PV is provisioned and bound as soon as the PVC is created, **before** any Pod schedules. Risk: the volume may be created in a zone/topology where the consuming Pod cannot be scheduled. |
| `WaitForFirstConsumer` | Provisioning and binding are **delayed** until a Pod that uses the PVC is scheduled. The scheduler's node choice (zone, node affinity, resource fit) informs where the volume is created. Preferred for topology-constrained storage like zonal cloud disks and local volumes. |

`WaitForFirstConsumer` keeps a PVC in `Pending` (with reason `waiting for first consumer to be created before binding`) until a Pod references it — this is expected, not an error.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer   # required pattern for local PVs
reclaimPolicy: Delete
```

## allowVolumeExpansion

When `true`, PVCs backed by this StorageClass can be **grown** by editing `spec.resources.requests.storage` to a larger value. The CSI driver must also support expansion.

```yaml
allowVolumeExpansion: true
```

Rules:
- You can only **increase** size, never shrink.
- The field must be set on the StorageClass **before** the PVC is created for expansion to work later; if it was `false` or absent, editing the PVC size is rejected.
- Some drivers require the Pod to be restarted to complete a filesystem resize (offline vs online expansion).

## Default StorageClass Annotation

Exactly one StorageClass may be marked as the cluster default. If a PVC omits `storageClassName` entirely (field absent), it uses the default class. The default is set with an annotation:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

Important distinctions:
- `storageClassName` **absent** → use default class (if one exists).
- `storageClassName: ""` (empty string) → explicitly **no** dynamic provisioning; only binds to a matching static PV.

Only one StorageClass should carry the default annotation set to `"true"`. If two do, the newest wins and the behavior is ambiguous — the exam may ask you to fix this by unsetting one.

```bash
# Mark a class as default
kubectl patch storageclass standard \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Unset the default on another class
kubectl patch storageclass old-default \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

## PVC Referencing storageClassName

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast      # triggers dynamic provisioning via the "fast" class
```

With `WaitForFirstConsumer`, this PVC stays `Pending` until a Pod mounts it:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-consumer
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: dynamic-pvc
```

## Exam Tips

- The default `volumeBindingMode` is `Immediate`. A PVC using a `WaitForFirstConsumer` class staying `Pending` until a Pod is created is **normal** — do not "fix" it by switching to `Immediate` unless the task asks.
- Default `reclaimPolicy` for a StorageClass is `Delete`. Dynamically provisioned PVs inherit it.
- `storageClassName: ""` disables dynamic provisioning; **missing** field uses the default class. Know the difference cold.
- To expand a PVC, the StorageClass needs `allowVolumeExpansion: true` set beforehand. You cannot retrofit expansion by editing the PVC alone.
- StorageClasses are cluster-scoped and effectively immutable — recreate rather than edit provisioner/bindingMode.
- Setting the default class is done with the `storageclass.kubernetes.io/is-default-class: "true"` annotation, applied with `kubectl patch`. Ensure only one class is default.
- `kubectl get storageclass` (alias `sc`) shows a `(default)` marker next to the default class — a quick way to answer "which class is default?".

## Key Commands

```bash
# List StorageClasses; the default is marked "(default)"
kubectl get storageclass
kubectl get sc
kubectl describe sc fast

# Create/apply
kubectl apply -f storageclass.yaml

# Set / unset the default class
kubectl patch sc standard \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl patch sc old \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Watch dynamic provisioning happen after applying a PVC
kubectl get pvc -w
kubectl get pv                      # a PV appears once bound

# Why is a PVC Pending?
kubectl describe pvc dynamic-pvc    # "waiting for first consumer" is expected for WFFC

# Field reference
kubectl explain storageclass
```
