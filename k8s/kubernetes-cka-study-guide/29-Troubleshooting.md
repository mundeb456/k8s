# 29 - Troubleshooting

Troubleshooting is the **largest CKA domain (~30% of the exam)**. You must be
fast and systematic. This guide covers application failures, control plane
failures, worker node failures, networking/DNS, and the diagnostic tooling
(`kubectl`, `crictl`, `journalctl`, logs).

---

## A systematic approach

Work top-down, from the workload to the platform:

1. **Is the object created and healthy?** `kubectl get`, `kubectl describe`.
2. **What do the events say?** `kubectl get events`, the `Events:` block in
   `describe`.
3. **What do the logs say?** `kubectl logs`, `--previous` for crashed containers.
4. **Is the node healthy?** `kubectl get nodes`, `kubectl describe node`.
5. **Is the control plane healthy?** static pods, kubelet, `journalctl`.
6. **Is networking/DNS working?** Services, endpoints, CoreDNS, kube-proxy.

Always narrow the blast radius: one pod vs. one node vs. whole cluster.

---

## Core diagnostic commands

```bash
kubectl get pods -o wide                       # status, node, IP
kubectl get pods -A                            # across all namespaces
kubectl describe pod <pod>                     # events, state, reasons
kubectl logs <pod>                             # current container logs
kubectl logs <pod> -c <container>              # specific container
kubectl logs <pod> --previous                  # logs from the crashed instance
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get events -A --field-selector type=Warning
kubectl top pods; kubectl top nodes            # requires metrics-server
```

`kubectl describe` and `kubectl get events` are your fastest wins — read the
`Events:` section first, it usually names the exact problem.

---

## 1. Application (Pod) failures

### Reading pod status

| STATUS | Likely cause | Where to look |
|--------|--------------|---------------|
| `Pending` | Can't be scheduled | `describe pod` events: resources, taints, affinity, no PVC bound |
| `ImagePullBackOff` / `ErrImagePull` | Bad image name/tag, no registry auth | `describe pod` events |
| `CrashLoopBackOff` | Container starts then exits | `logs --previous`, exit code |
| `CreateContainerConfigError` | Missing ConfigMap/Secret referenced | `describe pod` events |
| `RunContainerError` | Runtime/command issue | `logs`, `describe` |
| `Error` / `Completed` | Process exited (non-zero / zero) | `logs`, exit code |
| `Init:*` | Init container failing | `logs <pod> -c <init-container>` |
| `Terminating` (stuck) | Finalizer / node gone | `describe`, finalizers |

### Common application problems and checks

```bash
# Pending due to scheduling — read the message
kubectl describe pod <pod> | sed -n '/Events/,$p'
# e.g. "0/3 nodes are available: 3 Insufficient cpu"
#      "node(s) had untolerated taint {...}"
#      "pod has unbound immediate PersistentVolumeClaims"

# CrashLoopBackOff — look at why it exited
kubectl logs <pod> --previous
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].lastState}'

# ImagePullBackOff — verify the image string and registry access
kubectl describe pod <pod> | grep -A3 -i "failed"

# Config/Secret missing
kubectl get configmap,secret -n <ns>
```

### Probes causing restarts

A misconfigured `livenessProbe` restarts a perfectly healthy container. Check the
probe path/port and initialDelaySeconds:

```bash
kubectl describe pod <pod> | grep -A5 -i "liveness\|readiness"
```

### Debugging inside a pod

```bash
kubectl exec -it <pod> -- sh                    # shell into container
kubectl exec <pod> -- env                       # check env vars
kubectl exec <pod> -- cat /etc/config/app.conf  # check mounted files

# Ephemeral debug container (image has no shell/tools)
kubectl debug -it <pod> --image=busybox --target=<container>
```

---

## 2. Control plane failures

In a kubeadm cluster the control plane components (apiserver, scheduler,
controller-manager, etcd) run as **static pods**. Static pods are defined by
manifest files, NOT by the API, and are managed directly by the kubelet.

```bash
ls /etc/kubernetes/manifests/
# etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

### Symptom: kubectl commands hang / "connection refused"

The API server itself is down. Since kubectl can't reach it, drop to the node and
use the container runtime + kubelet logs.

```bash
# 1. Is the kubelet running?
sudo systemctl status kubelet
sudo journalctl -u kubelet -f

# 2. What containers exist at the runtime level?
sudo crictl ps -a
sudo crictl ps -a | grep apiserver

# 3. Inspect the failed apiserver container's logs
sudo crictl logs <apiserver-container-id>

# 4. If the container is missing/restarting, the manifest is likely bad
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

### Common control plane problems

- **Typo in a static pod manifest** (bad flag, wrong image tag, wrong port). The
  kubelet won't start the pod. Fix the YAML; the kubelet re-reads it automatically.
- **Wrong `--etcd-servers` / cert paths in apiserver.** apiserver crashes on
  start.
- **Scheduler down** -> new pods stay `Pending` forever even though nodes are
  fine. Check `kube-scheduler` static pod.
- **Controller-manager down** -> Deployments don't create ReplicaSets/Pods,
  endpoints don't populate. Check `kube-controller-manager` static pod.

```bash
# View control plane pod health (when apiserver is up)
kubectl get pods -n kube-system
kubectl logs -n kube-system kube-scheduler-<node>
kubectl logs -n kube-system kube-controller-manager-<node>
kubectl logs -n kube-system kube-apiserver-<node>
```

### Static pod logs when apiserver is up but the component crashed

```bash
kubectl -n kube-system logs kube-controller-manager-controlplane
kubectl -n kube-system logs kube-controller-manager-controlplane --previous
```

### Where logs live on disk

```bash
/var/log/pods/                     # per-pod container logs (symlinked to containers)
/var/log/containers/               # container log symlinks
/var/log/syslog  or  /var/log/messages   # host/kubelet in some distros
journalctl -u kubelet              # kubelet service logs (systemd)
```

---

## 3. Node not Ready

```bash
kubectl get nodes
kubectl describe node <node>       # read Conditions and Events
```

Look at the node `Conditions`:
- `Ready: False/Unknown` -> kubelet not reporting.
- `MemoryPressure`, `DiskPressure`, `PIDPressure` -> resource exhaustion.

### Investigate on the node itself

```bash
# Is kubelet running?
sudo systemctl status kubelet
sudo systemctl restart kubelet          # common fix if it's stopped/dead

# Why did it fail? (most valuable command for node issues)
sudo journalctl -u kubelet -f
sudo journalctl -u kubelet --no-pager | tail -50

# Is the container runtime up?
sudo systemctl status containerd
sudo crictl info
```

### Frequent root causes of NotReady

- **kubelet service stopped or crashed** — restart it, read journalctl.
- **Wrong kubelet config** — bad `--config`, wrong cert paths, wrong
  `/var/lib/kubelet/config.yaml` or `/etc/kubernetes/kubelet.conf`.
- **Expired certs / clock skew** — kubelet can't authenticate to apiserver.
- **Container runtime down** — `containerd`/`crio` stopped; kubelet reports
  NotReady.
- **Disk/memory pressure** — free space, evictions.
- **CNI not installed / broken** — node stays NotReady with
  `NetworkPluginNotReady`.

```bash
# Check kubelet kubeconfig points to the right apiserver
sudo cat /etc/kubernetes/kubelet.conf | grep server
# Check disk
df -h ; sudo journalctl -u kubelet | grep -i "disk\|evict"
```

---

## 4. Service / networking / DNS

### Service returns nothing / connection refused

The usual culprit: the Service selector does not match Pod labels, so no
endpoints are populated.

```bash
kubectl get svc <svc> -o wide
kubectl get endpoints <svc>          # EMPTY endpoints => selector mismatch
kubectl get endpointslices -l kubernetes.io/service-name=<svc>

# Compare selector vs. pod labels
kubectl describe svc <svc> | grep -i selector
kubectl get pods --show-labels
```

Checklist:
- Service `selector` matches pod labels? (empty endpoints = mismatch)
- `targetPort` matches the container's listening port?
- Pods are `Running` and passing readiness probes? (not-ready pods are excluded
  from endpoints)
- Correct Service type (`ClusterIP`, `NodePort`, `LoadBalancer`)?

### Test connectivity

```bash
# From a debug pod, hit the service by name and by ClusterIP
kubectl run tmp --rm -it --image=busybox --restart=Never -- sh
  # inside:
  wget -qO- http://<svc>.<ns>.svc.cluster.local:<port>
  nc -zv <clusterIP> <port>
```

### DNS (CoreDNS) troubleshooting

```bash
# Is CoreDNS running?
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# Is the kube-dns Service present with endpoints?
kubectl get svc -n kube-system kube-dns
kubectl get endpoints -n kube-system kube-dns

# Resolve a name from inside a pod
kubectl run dnsutils --rm -it --image=busybox:1.28 --restart=Never -- \
  nslookup kubernetes.default

# Check a pod's DNS config
kubectl exec <pod> -- cat /etc/resolv.conf
# nameserver should be the kube-dns ClusterIP (usually 10.96.0.10)
```

DNS names to remember:
- `<service>.<namespace>.svc.cluster.local`
- `<pod-ip-dashes>.<namespace>.pod.cluster.local`

### kube-proxy

kube-proxy programs iptables/ipvs rules for Services. If Services are unreachable
cluster-wide, check it:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

### CNI / pod networking

If pods can't reach each other across nodes, the CNI plugin is often the problem:

```bash
kubectl get pods -n kube-system            # look for the CNI pods (calico/flannel/weave)
ls /etc/cni/net.d/                         # CNI config present?
kubectl describe node <node> | grep -i "NetworkPluginNotReady\|cni"
```

---

## 5. crictl — talking to the container runtime

When the API server is down, `crictl` is how you see and debug containers on a
node. It talks directly to containerd/cri-o.

```bash
# May need: export CONTAINER_RUNTIME_ENDPOINT=unix:///run/containerd/containerd.sock
sudo crictl ps                 # running containers
sudo crictl ps -a              # include exited containers
sudo crictl pods               # pod sandboxes
sudo crictl logs <container-id>
sudo crictl inspect <container-id>
sudo crictl images
sudo crictl rm <container-id>  # remove a container
```

Use `crictl` (not `docker`) on modern nodes — Docker was removed as a runtime.

---

## 6. RBAC / auth failures

```bash
# "Forbidden" errors — check what a user/serviceaccount can do
kubectl auth can-i create pods --as=jane -n dev
kubectl auth can-i '*' '*' --as=system:serviceaccount:dev:builder

kubectl describe rolebinding,clusterrolebinding -A | grep -i <subject>
```

---

## Troubleshooting cheat cycle (fast recall)

```bash
kubectl get pods -A -o wide                       # what's broken?
kubectl describe pod <pod>                         # why? (Events)
kubectl logs <pod> --previous                      # what did it say before dying?
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl get nodes; kubectl describe node <node>    # node healthy?
sudo systemctl status kubelet; sudo journalctl -u kubelet -f
sudo crictl ps -a; sudo crictl logs <id>           # runtime-level view
kubectl get svc,endpoints <svc>                    # service wired up?
```

---

## Common gotchas

- **Empty `endpoints`** almost always means a selector/label mismatch — fix the
  Service selector or the Pod labels.
- **A static pod manifest error** silently keeps the component down; the kubelet
  reloads on save, so just fix the YAML.
- **`--previous` logs** are the only way to see why a CrashLoopBackOff container
  died — the current instance may have no useful output yet.
- **Restart the kubelet** (`systemctl restart kubelet`) after config changes;
  it is the most common node-recovery fix.
- **Use `crictl`, not `docker`** on exam nodes.
- **CoreDNS in CrashLoop** breaks name resolution cluster-wide — check its logs
  and the `kube-dns` Service/endpoints.
