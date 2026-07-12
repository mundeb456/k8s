# Volumes

Kubernetes volumes solve two problems that container filesystems cannot: **data persistence beyond a container restart** and **sharing files between containers in the same Pod**. A container's own filesystem is ephemeral — when the kubelet restarts a crashed container, it starts from the clean image and everything written to the container layer is lost. A volume is defined at the **Pod** level and mounted into one or more containers, so its lifetime is tied to the Pod (or, for persistent volumes, to something longer-lived) rather than to a single container.

## The Volume Concept

A volume is essentially a directory, possibly with data in it, that is accessible to the containers in a Pod. How that directory comes into being, the medium that backs it, and its lifetime are all determined by the **volume type**.

Two things must always be specified:

1. **`spec.volumes`** — declares the volume(s) available to the Pod (the source of the storage).
2. **`spec.containers[].volumeMounts`** — mounts a declared volume into a specific container at a specific path.

A volume that is declared but never mounted does nothing. A `volumeMount` that references a name not present in `spec.volumes` is a validation error.

## volumes vs volumeMounts

This distinction is a frequent point of confusion and a common exam trap.

| Field | Level | Purpose |
|-------|-------|---------|
| `spec.volumes` | Pod | Defines *what* the storage is (type + source) and gives it a `name`. |
| `spec.containers[].volumeMounts` | Container | Defines *where* a named volume is mounted inside that container. |

The `name` field is the glue: a `volumeMount.name` must match a `volumes.name`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: shared-data          # must match a volumes[].name below
          mountPath: /usr/share/nginx/html   # path inside THIS container
  volumes:
    - name: shared-data              # the volume definition
      emptyDir: {}                   # the volume TYPE / source
```

The same volume can be mounted into multiple containers at different paths, which is how sidecars share files.

## Ephemeral vs Persistent Volumes

- **Ephemeral volumes** live and die with the Pod. When the Pod is deleted (not merely restarted), the data is gone. `emptyDir` is the canonical example. Good for scratch space, caches, and inter-container sharing.
- **Persistent volumes** have a lifetime independent of any Pod. They are represented by the `PersistentVolume` (PV) / `PersistentVolumeClaim` (PVC) API objects. Data survives Pod deletion and can be re-attached to a new Pod. Covered in detail in `10-PersistentVolumes.md`.

Note the difference between a container **restart** and a Pod **deletion/reschedule**:

| Event | `emptyDir` data | `hostPath` data | PV data |
|-------|-----------------|-----------------|---------|
| Container crashes & restarts (same Pod) | Kept | Kept | Kept |
| Pod deleted / rescheduled to another node | Lost | Left on old node (orphaned) | Kept, re-attached |

## emptyDir

An `emptyDir` volume is created empty when a Pod is assigned to a node and exists as long as that Pod runs on that node. All containers in the Pod can read and write the same files. When the Pod is removed from the node, the data is deleted permanently.

Use cases: scratch space, a disk-based merge sort, checkpointing a long computation, sharing files between a content-producing container and a web-serving container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date >> /data/out.txt; sleep 5; done"]
      volumeMounts:
        - name: cache
          mountPath: /data
    - name: reader
      image: busybox:1.36
      command: ["sh", "-c", "tail -f /data/out.txt"]
      volumeMounts:
        - name: cache
          mountPath: /data
  volumes:
    - name: cache
      emptyDir: {}
```

You can back an `emptyDir` with RAM (a tmpfs) instead of the node disk, and cap its size:

```yaml
  volumes:
    - name: cache
      emptyDir:
        medium: Memory      # tmpfs, backed by RAM (counts against the container's memory limit)
        sizeLimit: 256Mi    # optional cap
```

`medium: Memory` gives fast I/O but the data counts against the Pod's memory usage and is still lost when the Pod is removed.

## hostPath

A `hostPath` volume mounts a file or directory from the **host node's filesystem** into the Pod. This ties the Pod's data to a specific node and is a well-known security and portability concern — it can expose the host and breaks if the Pod is rescheduled elsewhere.

Legitimate uses are mostly system-level: a container needing access to the container runtime socket, node logs, or `cgroups`. For application data, prefer a PV.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: host-vol
          mountPath: /host-data
  volumes:
    - name: host-vol
      hostPath:
        path: /var/log            # path ON THE NODE
        type: Directory           # optional but recommended
```

Common `hostPath` `type` values (they act as pre-mount safety checks):

| `type` | Meaning |
|--------|---------|
| `""` (empty) | No check performed (default). |
| `DirectoryOrCreate` | Create an empty directory (mode 0755) if it does not exist. |
| `Directory` | The path must already exist and be a directory. |
| `FileOrCreate` | Create an empty file (mode 0644) if it does not exist. |
| `File` | The path must already exist and be a file. |
| `Socket` | The path must be an existing UNIX socket. |

## Common Volume Types Overview

You are not expected to memorize every provisioner, but you should recognize these categories.

| Type | Backing | Lifetime | Typical use |
|------|---------|----------|-------------|
| `emptyDir` | Node disk or RAM | Pod | Scratch / sidecar sharing |
| `hostPath` | Node filesystem | Node (orphaned on reschedule) | System-level node access |
| `configMap` | ConfigMap object | Pod | Inject config files (see `12-ConfigMaps.md`) |
| `secret` | Secret object | Pod | Inject credentials/certs (see `13-Secrets.md`) |
| `downwardAPI` | Pod metadata | Pod | Expose Pod fields as files |
| `projected` | Multiple sources | Pod | Combine configMap/secret/downwardAPI/token into one dir |
| `persistentVolumeClaim` | A bound PV (any CSI backend) | Independent of Pod | Persistent app data |
| `csi` | Any CSI driver | Depends on driver | Cloud/enterprise storage (EBS, Ceph, NFS via CSI, etc.) |
| `nfs` | NFS server | External | Shared network storage (often ReadWriteMany) |

Modern cloud/enterprise storage integrates through the **Container Storage Interface (CSI)**. In-tree cloud volume types (e.g. `awsElasticBlockStore`, `gcePersistentDisk`, `azureDisk`) have been removed in favor of CSI drivers in recent Kubernetes versions.

### A projected volume example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
      volumeMounts:
        - name: all-in-one
          mountPath: /projected
          readOnly: true
  volumes:
    - name: all-in-one
      projected:
        sources:
          - configMap:
              name: app-config
          - secret:
              name: app-secret
          - downwardAPI:
              items:
                - path: "labels"
                  fieldRef:
                    fieldPath: metadata.labels
```

## Exam Tips

- Remember the two-part structure every single time: **`spec.volumes`** (source) + **`spec.containers[].volumeMounts`** (mount point). The linking key is `name`. A mismatch is a validation error that will keep the Pod from being created.
- `emptyDir` data survives a **container** restart but not a **Pod** deletion. This is the classic multiple-choice distractor.
- `readOnly: true` belongs on the `volumeMount`, not the volume definition.
- `hostPath` is node-local — if the scheduler moves the Pod, the data does not follow. Expect a question testing this.
- To share files between two containers in one Pod, mount the **same** `emptyDir` into both.
- `medium: Memory` on an `emptyDir` uses tmpfs (RAM) and counts against the container's memory limit.
- Use `kubectl explain pod.spec.volumes` and `kubectl explain pod.spec.containers.volumeMounts` in the exam to recall field names quickly — this is faster than guessing.

## Key Commands

```bash
# Discover volume/volumeMount field structure (invaluable in the exam)
kubectl explain pod.spec.volumes
kubectl explain pod.spec.containers.volumeMounts

# Apply and inspect a Pod that uses volumes
kubectl apply -f pod.yaml
kubectl describe pod volume-demo            # shows Volumes and Mounts sections

# Verify the mount from inside a container
kubectl exec -it volume-demo -c app -- ls -l /usr/share/nginx/html
kubectl exec -it emptydir-demo -c reader -- cat /data/out.txt

# See what volumes a running Pod actually has
kubectl get pod volume-demo -o jsonpath='{.spec.volumes}' | jq

# Generate a Pod skeleton to edit
kubectl run tmp --image=nginx --dry-run=client -o yaml > pod.yaml
```
