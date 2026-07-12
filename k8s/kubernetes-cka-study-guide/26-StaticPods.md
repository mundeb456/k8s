# Static Pods

A **static pod** is a Pod managed directly by the **kubelet** on a specific
Node, without the API server or scheduler being involved. The kubelet watches a
directory of manifest files and runs whatever Pod definitions it finds there.

Static pods are how kubeadm bootstraps the control plane (api-server, etcd,
controller-manager, scheduler all run as static pods).

---

## How static pods work

1. The kubelet is configured with a **staticPodPath** (a directory).
2. It periodically scans that directory for `*.yaml` / `*.json` Pod manifests.
3. For each manifest it creates the Pod directly via the container runtime.
4. If a manifest changes, the kubelet recreates the Pod. If a manifest is
   removed, the kubelet deletes the Pod.

Because the kubelet owns them, static pods:

- Run even if the API server / scheduler are down.
- Are tied to **one Node** (the one whose kubelet reads the file).
- Cannot be created/deleted with `kubectl` — you must edit files on the Node.

---

## staticPodPath

The directory is set in the kubelet configuration. Two ways it is configured:

### Via kubelet config file (kubeadm default)

The kubelet's config file (path shown in its systemd unit / `--config` flag,
commonly `/var/lib/kubelet/config.yaml`) contains:

```yaml
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
staticPodPath: /etc/kubernetes/manifests
```

### Via a command-line flag (older style)

```
--pod-manifest-path=/etc/kubernetes/manifests
```

The standard kubeadm location is **`/etc/kubernetes/manifests`**.

Find the path on an unknown cluster:

```bash
# Identify the kubelet config file
systemctl status kubelet
ps -ef | grep kubelet          # look for --config=...

# Read the staticPodPath from the config
grep -i staticPodPath /var/lib/kubelet/config.yaml
```

---

## Creating a static pod

On the Node, drop a Pod manifest into the staticPodPath. No `kubectl apply`
needed — the kubelet picks it up.

```bash
cat <<'EOF' > /etc/kubernetes/manifests/static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    role: static
spec:
  containers:
    - name: web
      image: nginx:1.27
      ports:
        - containerPort: 80
EOF

# Kubelet detects the file within seconds and starts the pod.
```

Generate a manifest quickly with `--dry-run`:

```bash
kubectl run static-web --image=nginx:1.27 --dry-run=client -o yaml \
  > /etc/kubernetes/manifests/static-web.yaml
```

---

## Removing a static pod

Delete the manifest file — do **not** `kubectl delete` (it will be recreated, or
for the mirror pod it just reappears):

```bash
rm /etc/kubernetes/manifests/static-web.yaml
# Kubelet stops and removes the pod.
```

---

## Mirror pods

When the API server *is* reachable, the kubelet creates a read-only **mirror
pod** in the API server for each static pod, so you can see it with `kubectl`.

- The mirror pod's name is the static pod name with the node name appended:
  `static-web-node01`.
- It is read-only via the API: `kubectl delete` removes the mirror object, but
  the kubelet immediately recreates it from the file.
- Mirror pods carry the annotation
  `kubernetes.io/config.source: file`.

```bash
kubectl get pods -o wide
# static-web-node01   1/1   Running   ...   node01

kubectl get pod static-web-node01 -o jsonpath='{.metadata.annotations}{"\n"}'
# includes kubernetes.io/config.source: file
```

Spotting a static pod via kubectl:

```bash
# Static pods have an ownerReference of kind Node (not ReplicaSet/DaemonSet)
kubectl get pod static-web-node01 -o jsonpath='{.metadata.ownerReferences}{"\n"}'
```

---

## Control plane runs as static pods

On a kubeadm cluster, look in `/etc/kubernetes/manifests` on the control-plane
node:

```bash
ls /etc/kubernetes/manifests
# etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

To change a control-plane component's flags, **edit its static manifest**; the
kubelet restarts it automatically:

```bash
vi /etc/kubernetes/manifests/kube-apiserver.yaml
# save -> kubelet detects change -> recreates the api-server pod
```

This is a very common CKA task (e.g. enabling an admission plugin, changing a
flag). If you make a typo the api-server may fail to come back — keep a backup.

---

## Static pod vs DaemonSet

| | Static Pod | DaemonSet |
|--|-----------|-----------|
| Managed by | kubelet (per node, from a file) | DaemonSet controller (API server) |
| Needs control plane? | No | Yes |
| Created via | manifest file on the node | `kubectl apply` |
| Scope | one specific node | one Pod per matching node, cluster-wide |
| Scheduler involved? | No | No (bypasses scheduler, sets nodeName) |
| Deletion | remove the file | `kubectl delete ds` |
| Typical use | control-plane components, bootstrap | node agents (log/metrics/CNI) |

Both bypass the scheduler, but a DaemonSet is a cluster-managed object that
guarantees one Pod per node; a static pod is a local, file-driven Pod on a
single node that survives without the control plane.

---

## Full example

`/etc/kubernetes/manifests/static-web.yaml` on `node01`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  namespace: default
  labels:
    role: static-web
spec:
  containers:
    - name: web
      image: nginx:1.27
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "100m"
          memory: "64Mi"
```

After the kubelet reads it, `kubectl get pods -o wide` shows
`static-web-node01` Running on `node01`.

---

## Exam Tips

- Static pods are created by **placing a file in the staticPodPath**, not with
  `kubectl`. Removing the pod means **deleting the file**.
- Default kubeadm staticPodPath is **`/etc/kubernetes/manifests`**; confirm via
  `staticPodPath` in `/var/lib/kubelet/config.yaml`.
- The **mirror pod name = `<pod-name>-<node-name>`**. Its ownerReference is the
  **Node**, and it has annotation `kubernetes.io/config.source: file`.
- Control-plane components run as static pods — **edit their manifests to change
  flags**; the kubelet restarts them. Back up before editing the api-server
  manifest.
- If you `kubectl delete` a static pod's mirror, it comes right back — you must
  remove the underlying file.
- Static pods run on whatever node holds the file. To place one on a worker,
  SSH to that worker and drop the file in its manifest dir.
- Restart the kubelet if it doesn't pick up changes:
  `systemctl restart kubelet`.

---

## Key Commands

```bash
# Find the static pod directory
ps -ef | grep kubelet
grep -i staticPodPath /var/lib/kubelet/config.yaml
systemctl status kubelet

# Create / remove a static pod (on the node)
kubectl run static-web --image=nginx:1.27 --dry-run=client -o yaml \
  > /etc/kubernetes/manifests/static-web.yaml
rm /etc/kubernetes/manifests/static-web.yaml

# See mirror pods and confirm they're static
kubectl get pods -o wide
kubectl get pod <name>-<node> -o jsonpath='{.metadata.ownerReferences}{"\n"}'
kubectl get pod <name>-<node> -o jsonpath='{.metadata.annotations}{"\n"}'

# List control-plane static manifests
ls /etc/kubernetes/manifests

# Apply a kubelet restart after editing config
systemctl restart kubelet
```
