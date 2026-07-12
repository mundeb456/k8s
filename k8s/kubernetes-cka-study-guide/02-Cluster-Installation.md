# Cluster Installation with kubeadm

`kubeadm` is the official tool for bootstrapping a best-practice Kubernetes cluster. On the CKA you are unlikely to build a cluster from scratch, but you **will** join nodes, upgrade clusters, and fix broken installs — so knowing the full flow cold is valuable.

This guide targets **Kubernetes v1.31+** with **containerd** as the runtime on Ubuntu/Debian-style nodes. Run the same prerequisite steps on **every** node (control plane and workers).

## Overview of the Process

1. Prepare every node (swap off, kernel modules, sysctl, container runtime).
2. Install `kubeadm`, `kubelet`, `kubectl`.
3. `kubeadm init` on the control plane node.
4. Configure `kubectl` for your user.
5. Install a **CNI** (pod network) plugin.
6. `kubeadm join` the worker nodes.

## Step 1 — Prerequisites (run on ALL nodes)

### Disable swap

kubelet requires swap to be off (or explicitly configured). Disable it now and permanently.

```bash
sudo swapoff -a
# Comment out any swap line in /etc/fstab so it stays off after reboot
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Load required kernel modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### Configure sysctl for bridged traffic

These settings let iptables see bridged traffic and enable IP forwarding.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply without reboot
sudo sysctl --system
```

### Install and configure the containerd runtime

```bash
sudo apt-get update
sudo apt-get install -y containerd

# Generate a default config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

# CRITICAL: use the systemd cgroup driver (must match kubelet)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

> The **cgroup driver must match** between the kubelet and the container runtime. With systemd-based distros use `systemd` for both. A mismatch is a common reason nodes fail to become `Ready`.

## Step 2 — Install kubeadm, kubelet, kubectl (run on ALL nodes)

```bash
# Dependencies
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Add the Kubernetes apt repo key (v1.31 stream)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Pin the versions so an accidental apt upgrade doesn't break the cluster
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

## Step 3 — Initialize the control plane (control plane node only)

Choose a Pod network CIDR that matches your CNI. Common defaults:
- **Calico:** `192.168.0.0/16`
- **Flannel:** `10.244.0.0/16`

```bash
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<CONTROL_PLANE_IP>
```

On success, `kubeadm` prints two important things:
1. The commands to set up your `kubeconfig`.
2. A **`kubeadm join`** command (with token + CA cert hash) to run on workers — **copy this**.

### Optional: use a config file instead of flags

For reproducibility you can drive `kubeadm init` from a file:

```yaml
# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.31.0
networking:
  podSubnet: 192.168.0.0/16
  serviceSubnet: 10.96.0.0/12
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.0.0.10
  bindPort: 6443
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

```bash
sudo kubeadm init --config kubeadm-config.yaml
```

## Step 4 — Configure kubectl for your user

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify
kubectl get nodes        # control plane will show NotReady until a CNI is installed
```

## Step 5 — Install a CNI (Pod network) plugin

The cluster stays `NotReady` and CoreDNS Pods stay `Pending` until a network plugin is applied. Pick **one**.

### Calico

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

### Flannel (requires `--pod-network-cidr=10.244.0.0/16`)

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### Weave Net

```bash
kubectl apply -f https://reweave.azurewebsites.net/k8s/v1.31/net.yaml
```

After a minute, the node should report `Ready`:

```bash
kubectl get nodes
kubectl get pods -n kube-system   # CoreDNS should move Pending -> Running
```

## Step 6 — Join worker nodes (run on EACH worker)

Use the join command printed by `kubeadm init`. It looks like:

```bash
sudo kubeadm join <CONTROL_PLANE_IP>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### Lost or expired the join command?

Tokens expire after 24 hours. Regenerate everything from the control plane:

```bash
# Create a fresh token AND print the full join command in one shot
kubeadm token create --print-join-command

# List existing tokens
kubeadm token list

# If you only have the token but need the CA cert hash:
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt \
  | openssl rsa -pubin -outform der 2>/dev/null \
  | openssl dgst -sha256 -hex | sed 's/^.* //'
```

### Verify the join

```bash
# From the control plane
kubectl get nodes -o wide
```

## Tearing Down / Resetting a Node

```bash
# On the node you want to remove
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d $HOME/.kube/config

# From the control plane, remove it from the cluster
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl delete node <node-name>
```

## Exam Tips

- **Read every kubeadm hint.** `kubeadm init` and `kubeadm join` print exactly what to run next — including how to set up kubeconfig and the join command. Don't reinvent it.
- **`kubeadm token create --print-join-command`** is the fastest way to get a working join command — memorize it. This is a very common task.
- If a node is `NotReady`, the two usual suspects are: **no CNI installed**, or a **cgroup driver mismatch** between kubelet and containerd. Check `journalctl -u kubelet`.
- CoreDNS Pods stuck in `Pending`/`ContainerCreating` almost always means the **CNI isn't installed** yet.
- Match your `--pod-network-cidr` to the CNI's expected default (Flannel *must* use `10.244.0.0/16` with its default manifest).
- Prerequisite failures show up in the `kubeadm init` **preflight** output — read those warnings/errors (swap on, missing modules, port in use). Re-run with `--ignore-preflight-errors=...` only when you understand the warning.
- On the exam you usually run one command as your user and one as root; remember `sudo` for `kubeadm`/`systemctl` and the `.kube/config` copy for `kubectl`.

## Key Commands

```bash
# Prep (all nodes)
sudo swapoff -a
sudo modprobe overlay br_netfilter
sudo sysctl --system

# Install tooling (all nodes)
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Bootstrap control plane
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# kubeconfig for your user
mkdir -p $HOME/.kube && sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# CNI
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml

# Join workers
kubeadm token create --print-join-command          # (on control plane)
sudo kubeadm join <IP>:6443 --token <t> --discovery-token-ca-cert-hash sha256:<h>   # (on worker)

# Verify
kubectl get nodes -o wide
kubectl get pods -n kube-system
```
