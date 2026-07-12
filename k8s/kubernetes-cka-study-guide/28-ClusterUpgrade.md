# 28 - Cluster Upgrade with kubeadm

Upgrading a Kubernetes cluster with `kubeadm` is a guaranteed exam task. The core
idea: **upgrade the control plane first, then the worker nodes, one node at a
time, draining each before you touch it.**

---

## Version skew policy (know this cold)

Kubernetes supports a limited version difference between components:

- **kube-apiserver** is the reference. In an HA cluster, apiservers may differ by
  at most one minor version from each other.
- **kubelet** may be **up to 3 minor versions older** than the apiserver
  (`kube-apiserver: 1.31` -> `kubelet` may be `1.28`–`1.31`). kubelet must
  **never** be newer than the apiserver.
- **kube-controller-manager, kube-scheduler, cloud-controller-manager** may be
  up to 1 minor version older than the apiserver.
- **kubectl** may be within 1 minor version of the apiserver (one newer or older).

**Critical rule: you upgrade ONE minor version at a time.** You cannot jump from
1.30 to 1.32. Go 1.30 -> 1.31 -> 1.32.

---

## Upgrade order

1. **Control plane node(s) first.**
   - First control plane node: `kubeadm upgrade apply`.
   - Additional control plane nodes: `kubeadm upgrade node`.
2. **Worker nodes second**, one at a time, each drained first.

Within any node the component order is:
1. Upgrade the `kubeadm` binary.
2. Run `kubeadm upgrade ...` (plan/apply on first CP, node on others/workers).
3. Upgrade the `kubelet` and `kubectl` binaries.
4. Restart the kubelet.
5. Uncordon the node.

---

## Step 0: Update the package repository

Starting with v1.28+, each Kubernetes minor version has its **own** apt/yum repo.
To go to 1.31 you must switch the repo URL to the 1.31 line.

### Debian / Ubuntu

```bash
# Point the repo at the target minor version (example: v1.31)
sudo sed -i 's|v1.30|v1.31|' /etc/apt/sources.list.d/kubernetes.list
# Or rewrite it explicitly:
# echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
#   https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" \
#   | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
```

### RHEL / CentOS

```bash
sudo sed -i 's|v1.30|v1.31|' /etc/yum.repos.d/kubernetes.repo
```

Find the exact patch version available:

```bash
apt-cache madison kubeadm
# or
sudo yum list --showduplicates kubeadm
```

---

## Control plane node — first (primary) control plane

### 1. Upgrade the kubeadm binary

```bash
sudo apt-get update
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.31.1-1.1
sudo apt-mark hold kubeadm

kubeadm version   # verify it now reports 1.31.x
```

### 2. Plan the upgrade

`kubeadm upgrade plan` shows the current versions, the target version, and which
components will be upgraded. It changes nothing.

```bash
sudo kubeadm upgrade plan
```

### 3. Apply the upgrade to the control plane

This upgrades the control plane static pods (apiserver, controller-manager,
scheduler) and etcd, and renews certs as needed.

```bash
sudo kubeadm upgrade apply v1.31.1
```

You will see `[upgrade/successful] SUCCESS! Your cluster was upgraded ...`.

### 4. Drain the control plane node

```bash
kubectl drain <controlplane-node> --ignore-daemonsets
```

### 5. Upgrade kubelet and kubectl, then restart kubelet

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.31.1-1.1 kubectl=1.31.1-1.1
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### 6. Uncordon the control plane node

```bash
kubectl uncordon <controlplane-node>
kubectl get nodes   # control plane should be Ready and on v1.31.1
```

---

## Additional control plane nodes (HA clusters)

Same as above, but instead of `kubeadm upgrade apply`, use:

```bash
sudo kubeadm upgrade node
```

`kubeadm upgrade node` upgrades the local component configs without re-running
the cluster-wide upgrade. Everything else (drain, kubelet/kubectl, restart,
uncordon) is identical.

---

## Worker nodes — one at a time

Do this for each worker node in turn. Run the `drain`/`uncordon` from a machine
with kubectl access (usually the control plane), and the binary upgrades on the
worker itself.

### 1. Drain the worker (from control plane)

```bash
kubectl drain <worker-node> --ignore-daemonsets
# Add --delete-emptydir-data if pods use emptyDir volumes
# Add --force if there are unmanaged (bare) pods
kubectl drain <worker-node> --ignore-daemonsets --delete-emptydir-data --force
```

### 2. On the worker: upgrade kubeadm

```bash
# (update the repo to v1.31 first, as shown above)
sudo apt-get update
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.31.1-1.1
sudo apt-mark hold kubeadm
```

### 3. On the worker: upgrade the local kubelet config

```bash
sudo kubeadm upgrade node
```

### 4. On the worker: upgrade kubelet and kubectl, restart kubelet

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.31.1-1.1 kubectl=1.31.1-1.1
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### 5. Uncordon the worker (from control plane)

```bash
kubectl uncordon <worker-node>
kubectl get nodes
```

---

## drain / cordon / uncordon quick reference

```bash
# Mark unschedulable AND evict existing pods (respecting PodDisruptionBudgets)
kubectl drain <node> --ignore-daemonsets

# Common extra flags:
#   --ignore-daemonsets      required; DaemonSet pods can't be evicted
#   --delete-emptydir-data   proceed even though pods use emptyDir
#   --force                  evict bare pods not managed by a controller
#   --grace-period=<sec>     override pod termination grace period

# Mark unschedulable only, do NOT evict running pods
kubectl cordon <node>

# Make the node schedulable again
kubectl uncordon <node>
```

- `drain` = `cordon` + evict pods.
- DaemonSet-managed pods are not moved by drain (they live on every node), hence
  `--ignore-daemonsets` is mandatory.

---

## Full command sequence (single control plane + workers)

```bash
### ----- CONTROL PLANE -----
# 1. Switch repo to v1.31, then:
sudo apt-get update
sudo apt-mark unhold kubeadm && \
  sudo apt-get install -y kubeadm=1.31.1-1.1 && sudo apt-mark hold kubeadm

sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.31.1

kubectl drain controlplane --ignore-daemonsets

sudo apt-mark unhold kubelet kubectl && \
  sudo apt-get install -y kubelet=1.31.1-1.1 kubectl=1.31.1-1.1 && \
  sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

kubectl uncordon controlplane

### ----- EACH WORKER (repeat) -----
kubectl drain node01 --ignore-daemonsets --delete-emptydir-data --force
# --- ssh to node01 ---
#   switch repo to v1.31, then:
sudo apt-get update
sudo apt-mark unhold kubeadm && \
  sudo apt-get install -y kubeadm=1.31.1-1.1 && sudo apt-mark hold kubeadm
sudo kubeadm upgrade node
sudo apt-mark unhold kubelet kubectl && \
  sudo apt-get install -y kubelet=1.31.1-1.1 kubectl=1.31.1-1.1 && \
  sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet
# --- back on control plane ---
kubectl uncordon node01
```

---

## Verification

```bash
kubectl get nodes -o wide          # every node Ready, VERSION column = v1.31.1
kubectl version                    # client + server versions
kubectl get pods -n kube-system    # control plane pods Running
```

---

## Common gotchas

- **Forgetting `--ignore-daemonsets`** — drain fails immediately.
- **Not switching the apt/yum repo** to the target minor version — the install
  finds no newer package (v1.28+ split repos by minor version).
- **Skipping a minor version** — you must go one minor version at a time.
- **`apt-mark hold`** — Kubernetes packages are held to prevent accidental
  upgrades; you must `unhold` before install and `hold` after.
- **Wrong command on the right node** — `upgrade apply` only on the first control
  plane; `upgrade node` on all other control plane and worker nodes.
- **Forgetting `daemon-reload` + `restart kubelet`** after installing the new
  kubelet binary.
