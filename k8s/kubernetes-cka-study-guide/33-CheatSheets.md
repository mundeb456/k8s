# 33 - CKA Cheat Sheets (Quick Reference)

Condensed, exam-day quick reference. Set up your shell first, then lean on
imperative generators for speed.

---

## Exam-day speed setup (do this first)

```bash
# Alias + autocomplete
alias k=kubectl
source <(kubectl completion bash)
complete -o default -F __start_kubectl k       # complete for the alias too

# Fast dry-run YAML generator
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"          # fast delete

# Now: k run nginx --image=nginx $do > pod.yaml

# vim: 2-space indent, expand tabs (put in ~/.vimrc)
# set expandtab tabstop=2 shiftwidth=2
```

Set / switch namespace context so you stop typing `-n`:

```bash
kubectl config set-context --current --namespace=<ns>
kubectl config view --minify | grep namespace
```

---

## Essential kubectl commands

```bash
k get pods -A -o wide                 # all pods, all namespaces, node + IP
k get all                             # common resources in current ns
k describe pod <p>                    # events, state, reasons
k logs <p> [-c <c>] [--previous] [-f] # container logs
k exec -it <p> -- sh                  # shell into a pod
k get events --sort-by=.metadata.creationTimestamp
k explain pod.spec.containers         # field docs
k api-resources                       # list resource kinds + short names
k apply -f file.yaml                  # declarative create/update
k delete pod <p> --force --grace-period=0
k edit deploy <d>                     # live edit
k label pod <p> tier=web              # add label
k annotate pod <p> note=hi
k top pod ; k top node                # metrics (needs metrics-server)
```

---

## Imperative generators (generate fast, then edit)

```bash
# Pod
k run nginx --image=nginx $do
k run nginx --image=nginx --port=80 --labels="app=web"
k run busybox --image=busybox --restart=Never -it --rm -- sh   # throwaway

# Deployment
k create deploy web --image=nginx --replicas=3 $do
k set image deploy/web nginx=nginx:1.27
k scale deploy web --replicas=5
k autoscale deploy web --min=2 --max=10 --cpu-percent=70

# Service (expose existing) / expose imperatively
k expose deploy web --port=80 --target-port=8080 --type=NodePort --name=web-svc
k create svc clusterip web --tcp=80:80 $do

# ConfigMap / Secret
k create configmap cm --from-literal=k=v --from-file=path/
k create secret generic s --from-literal=PASSWORD=pw

# Namespace / ServiceAccount
k create ns dev
k create sa builder -n dev

# RBAC
k create role pod-reader --verb=get,list,watch --resource=pods -n dev $do
k create rolebinding rb --role=pod-reader --user=jane -n dev $do
k create clusterrole cr --verb=get,list --resource=pods $do
k create clusterrolebinding crb --clusterrole=view --serviceaccount=dev:builder $do

# Job / CronJob
k create job hello --image=busybox -- echo hi
k create cronjob hello --image=busybox --schedule="*/5 * * * *" -- echo hi
```

---

## Common YAML skeletons

### Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  labels:
    app: web
spec:
  containers:
  - name: c
    image: nginx:1.27
    ports:
    - containerPort: 80
    resources:
      requests: {cpu: "100m", memory: "64Mi"}
      limits:   {cpu: "250m", memory: "128Mi"}
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels: {app: web}
  template:
    metadata:
      labels: {app: web}
    spec:
      containers:
      - name: web
        image: nginx:1.27
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  type: ClusterIP          # ClusterIP | NodePort | LoadBalancer
  selector: {app: web}
  ports:
  - port: 80
    targetPort: 80
    # nodePort: 30080      # only for NodePort
```

### PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: {storage: 1Gi}
  # storageClassName: standard
```

### NetworkPolicy (default deny + allow)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: prod
spec:
  podSelector: {matchLabels: {app: db}}
  policyTypes: ["Ingress"]
  ingress:
  - from:
    - podSelector: {matchLabels: {app: api}}
    ports:
    - {protocol: TCP, port: 5432}
```

---

## Taints, tolerations, affinity

```bash
# Taint / remove taint (trailing -)
k taint nodes node01 key=value:NoSchedule
k taint nodes node01 key=value:NoSchedule-
```

```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
nodeSelector:
  disktype: ssd
```

---

## Node maintenance: drain / cordon / uncordon

```bash
k drain <node> --ignore-daemonsets                       # evict + cordon
k drain <node> --ignore-daemonsets --delete-emptydir-data --force
k cordon <node>                                          # mark unschedulable only
k uncordon <node>                                        # schedulable again
```

---

## etcd backup one-liner (memorize)

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/snapshot.db

# Restore into a NEW data dir, then repoint etcd.yaml hostPath -> that dir
ETCDCTL_API=3 etcdctl snapshot restore /opt/snapshot.db \
  --data-dir=/var/lib/etcd-from-backup
```

---

## Cluster upgrade (kubeadm) short form

```bash
# control plane
sudo apt-get install -y kubeadm=1.31.1-1.1
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.31.1
k drain controlplane --ignore-daemonsets
sudo apt-get install -y kubelet=1.31.1-1.1 kubectl=1.31.1-1.1
sudo systemctl daemon-reload && sudo systemctl restart kubelet
k uncordon controlplane
# workers: drain -> kubeadm upgrade node -> upgrade kubelet -> restart -> uncordon
```

---

## jsonpath & custom-columns examples

```bash
# All pod names
k get pods -o jsonpath='{.items[*].metadata.name}'

# Node internal IPs
k get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# Container images across pods
k get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'

# Sort pods by name
k get pods --sort-by=.metadata.name

# Custom columns
k get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# Single field of one object
k get pod mypod -o jsonpath='{.status.podIP}'

# Filter by label / field selector
k get pods -l app=web
k get pods --field-selector status.phase=Running
```

---

## Troubleshooting fast cycle

```bash
k get pods -A -o wide
k describe pod <p>                    # Events first
k logs <p> --previous
k get nodes ; k describe node <n>
sudo systemctl status kubelet ; sudo journalctl -u kubelet -f
sudo crictl ps -a ; sudo crictl logs <id>
k get svc,endpoints <svc>             # empty endpoints => selector mismatch
k run t --rm -it --image=busybox:1.28 --restart=Never -- nslookup kubernetes.default
```

---

## Important ports

| Component | Port(s) |
|-----------|---------|
| kube-apiserver | 6443 |
| etcd client / peer | 2379 / 2380 |
| kube-scheduler | 10259 |
| kube-controller-manager | 10257 |
| kubelet API | 10250 |
| kube-proxy (health) | 10256 |
| NodePort range | 30000–32767 |
| CoreDNS (kube-dns Service, typical ClusterIP) | 10.96.0.10:53 |

---

## Key file locations (kubeadm node)

```
/etc/kubernetes/manifests/           # static pod manifests (apiserver, etcd, ...)
/etc/kubernetes/pki/                 # cluster certificates
/etc/kubernetes/pki/etcd/            # etcd certificates
/etc/kubernetes/admin.conf           # cluster-admin kubeconfig
/var/lib/kubelet/config.yaml         # kubelet config (staticPodPath here)
/var/lib/etcd/                       # etcd data dir
/var/log/pods/ , /var/log/containers/  # container logs
/etc/cni/net.d/                      # CNI config
```

---

## DNS name patterns

```
<service>.<namespace>.svc.cluster.local          # Service
<pod-ip-with-dashes>.<namespace>.pod.cluster.local  # Pod
```
