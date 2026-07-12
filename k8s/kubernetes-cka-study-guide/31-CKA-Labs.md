# 31 - CKA Hands-On Practice Labs

Fifteen practice labs spanning the whole CKA curriculum. Each has a **Task** and
a **Solution** (kubectl commands + YAML). Try each task first with the terminal
before opening the solution. Everything below is verified for CKA v1.31+.

> Exam tip: prefer imperative commands (`kubectl create ...`, `--dry-run=client
> -o yaml`) to generate YAML fast, then edit.

---

## Lab 1 — Create a Pod imperatively

**Task:** Create a Pod named `nginx-pod` running the `nginx:1.27` image in the
`default` namespace. Then output its YAML without creating it.

<details>
<summary>Solution</summary>

```bash
# Create it
kubectl run nginx-pod --image=nginx:1.27

# Generate YAML only (dry run)
kubectl run nginx-pod --image=nginx:1.27 --dry-run=client -o yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx-pod
    image: nginx:1.27
```
</details>

---

## Lab 2 — Deployment with 3 replicas and a rollout

**Task:** Create a Deployment `web` using `nginx:1.26` with 3 replicas. Then
update the image to `nginx:1.27` and record the rollout. Roll it back.

<details>
<summary>Solution</summary>

```bash
kubectl create deployment web --image=nginx:1.26 --replicas=3

# Update image (triggers a rolling update)
kubectl set image deployment/web nginx=nginx:1.27

# Watch rollout
kubectl rollout status deployment/web
kubectl rollout history deployment/web

# Roll back to the previous revision
kubectl rollout undo deployment/web
```
</details>

---

## Lab 3 — Expose a Deployment as a Service

**Task:** Expose the `web` Deployment on port 80 as a `NodePort` Service named
`web-svc`. Verify the endpoints are populated.

<details>
<summary>Solution</summary>

```bash
kubectl expose deployment web --name=web-svc --port=80 --target-port=80 --type=NodePort

kubectl get svc web-svc
kubectl get endpoints web-svc      # must list pod IPs — else selector mismatch
```

Equivalent YAML:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```
</details>

---

## Lab 4 — Multi-container Pod with resource limits and env

**Task:** Create a Pod `app` with one container `main` (`busybox`, runs
`sleep 3600`), CPU request 100m / limit 250m, memory request 64Mi / limit 128Mi,
and env var `MODE=prod`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: main
    image: busybox
    command: ["sh", "-c", "sleep 3600"]
    env:
    - name: MODE
      value: "prod"
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "250m"
        memory: "128Mi"
```

```bash
kubectl apply -f app.yaml
kubectl exec app -- env | grep MODE
```
</details>

---

## Lab 5 — ConfigMap and Secret consumption

**Task:** Create a ConfigMap `app-config` with `APP_COLOR=blue` and a Secret
`app-secret` with `PASSWORD=s3cr3t`. Mount both as environment variables into a
Pod `configured`.

<details>
<summary>Solution</summary>

```bash
kubectl create configmap app-config --from-literal=APP_COLOR=blue
kubectl create secret generic app-secret --from-literal=PASSWORD=s3cr3t
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configured
spec:
  containers:
  - name: c
    image: busybox
    command: ["sh", "-c", "env; sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
    env:
    - name: PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: PASSWORD
```
</details>

---

## Lab 6 — RBAC: Role + RoleBinding

**Task:** In namespace `dev`, create a Role `pod-reader` that can `get`, `list`,
`watch` pods, and bind it to a user `jane`. Verify with `auth can-i`.

<details>
<summary>Solution</summary>

```bash
kubectl create namespace dev

kubectl create role pod-reader \
  --verb=get,list,watch --resource=pods -n dev

kubectl create rolebinding jane-pod-reader \
  --role=pod-reader --user=jane -n dev

# Verify
kubectl auth can-i list pods --as=jane -n dev     # yes
kubectl auth can-i delete pods --as=jane -n dev   # no
```

YAML equivalent:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jane-pod-reader
  namespace: dev
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
</details>

---

## Lab 7 — ServiceAccount + ClusterRoleBinding

**Task:** Create a ServiceAccount `builder` in `dev` and grant it the built-in
`view` ClusterRole cluster-wide.

<details>
<summary>Solution</summary>

```bash
kubectl create serviceaccount builder -n dev

kubectl create clusterrolebinding builder-view \
  --clusterrole=view \
  --serviceaccount=dev:builder

kubectl auth can-i list pods \
  --as=system:serviceaccount:dev:builder -A   # yes (read-only)
```
</details>

---

## Lab 8 — PersistentVolume and PersistentVolumeClaim

**Task:** Create a 1Gi hostPath PV `pv-data` (RWO), a matching PVC `pvc-data`,
and a Pod `pv-pod` that mounts the PVC at `/data`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
  - name: c
    image: nginx
    volumeMounts:
    - mountPath: /data
      name: vol
  volumes:
  - name: vol
    persistentVolumeClaim:
      claimName: pvc-data
```

```bash
kubectl apply -f pv.yaml
kubectl get pv,pvc            # PVC should show STATUS Bound
```
</details>

---

## Lab 9 — Node maintenance: drain and uncordon

**Task:** Safely take `node01` out of service for maintenance (evict pods,
ignore daemonsets, allow emptyDir), then return it to service.

<details>
<summary>Solution</summary>

```bash
kubectl drain node01 --ignore-daemonsets --delete-emptydir-data
# add --force if there are bare (unmanaged) pods

# ... perform maintenance ...

kubectl uncordon node01
kubectl get nodes           # node01 back to Ready + schedulable
```
</details>

---

## Lab 10 — etcd snapshot backup

**Task:** Take an etcd snapshot to `/opt/etcd-backup.db` and verify it.

<details>
<summary>Solution</summary>

```bash
# Read correct paths from /etc/kubernetes/manifests/etcd.yaml if unsure
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db

ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db --write-out=table
```
</details>

---

## Lab 11 — Static Pod

**Task:** Create a static Pod named `static-web` (`nginx`) on `node01` by placing
a manifest in the kubelet's static pod directory.

<details>
<summary>Solution</summary>

```bash
# The static pod dir is set by staticPodPath in the kubelet config
# (default /etc/kubernetes/manifests). Confirm:
grep staticPodPath /var/lib/kubelet/config.yaml

# On node01, write the manifest:
cat <<'EOF' | sudo tee /etc/kubernetes/manifests/static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
spec:
  containers:
  - name: web
    image: nginx
EOF
```

The kubelet auto-creates the pod; from the control plane it shows up as
`static-web-node01`. To delete a static pod, remove its manifest file (deleting
via `kubectl` won't work — the kubelet recreates it).

```bash
kubectl get pods -o wide | grep static-web
```
</details>

---

## Lab 12 — NetworkPolicy

**Task:** In namespace `prod`, deny all ingress to pods labeled `app=db` except
from pods labeled `app=api` on TCP 5432.

<details>
<summary>Solution</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-api
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
```

```bash
kubectl apply -f netpol.yaml
kubectl describe networkpolicy db-allow-api -n prod
```

Note: NetworkPolicies require a CNI that enforces them (Calico, Cilium, etc.).
</details>

---

## Lab 13 — Troubleshoot a broken Pod

**Task:** A Pod `broken` is stuck in `CrashLoopBackOff`. Diagnose and fix it. It
should run a web server. (Cause: the container command exits immediately.)

<details>
<summary>Solution</summary>

```bash
kubectl get pod broken
kubectl describe pod broken            # read Events + Last State + exit code
kubectl logs broken --previous         # why the last instance died

# Typical finds & fixes:
#  - wrong image tag        -> kubectl set image / edit
#  - command exits at once  -> fix command/args so process stays up
#  - missing configmap/secret -> create it
#  - crashing liveness probe -> fix probe path/port

kubectl edit pod broken                # or delete + reapply corrected YAML
```

Example fix — the pod ran `command: ["sh","-c","echo hi"]` (exits). Replace with
a long-running server:

```yaml
spec:
  containers:
  - name: web
    image: nginx:1.27         # long-running, no custom command needed
```
</details>

---

## Lab 14 — Taints, tolerations, and node scheduling

**Task:** Taint `node01` with `env=prod:NoSchedule`. Then schedule a Pod
`prod-pod` that tolerates it. Also pin the pod to `node01` with a nodeSelector.

<details>
<summary>Solution</summary>

```bash
kubectl taint nodes node01 env=prod:NoSchedule
kubectl label node node01 tier=frontend
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prod-pod
spec:
  nodeSelector:
    tier: frontend
  tolerations:
  - key: "env"
    operator: "Equal"
    value: "prod"
    effect: "NoSchedule"
  containers:
  - name: c
    image: nginx
```

```bash
kubectl apply -f prod-pod.yaml
kubectl get pod prod-pod -o wide       # should land on node01

# Remove the taint (note trailing minus):
kubectl taint nodes node01 env=prod:NoSchedule-
```
</details>

---

## Lab 15 — Cluster upgrade steps (control plane)

**Task:** Upgrade the control plane node to v1.31.1 with kubeadm.

<details>
<summary>Solution</summary>

```bash
# 1. Point apt repo at v1.31, then update
sudo apt-get update

# 2. Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.31.1-1.1
sudo apt-mark hold kubeadm

# 3. Plan + apply
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.31.1

# 4. Drain the control plane node
kubectl drain controlplane --ignore-daemonsets

# 5. Upgrade kubelet + kubectl and restart
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.31.1-1.1 kubectl=1.31.1-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 6. Uncordon
kubectl uncordon controlplane
kubectl get nodes
```
</details>

---

## Bonus Lab — Scale, autoscale, and label selectors

**Task:** Scale `web` to 5 replicas. Add an HPA (min 2, max 10, target CPU 70%).
List only pods labeled `app=web`.

<details>
<summary>Solution</summary>

```bash
kubectl scale deployment web --replicas=5

kubectl autoscale deployment web --min=2 --max=10 --cpu-percent=70

kubectl get pods -l app=web
kubectl get hpa
```
</details>

---

## Practice discipline for exam speed

- Alias `k=kubectl` and set up `--dry-run=client -o yaml` muscle memory.
- Use `kubectl explain <resource>.<field>` when you forget a field name.
- Always `kubectl config set-context --current --namespace=<ns>` when a task
  fixes a namespace, so you stop typing `-n`.
