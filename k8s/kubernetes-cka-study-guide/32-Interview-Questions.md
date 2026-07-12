# 32 - Kubernetes / CKA Interview Questions

Forty-plus questions grouped by topic, each with a concise answer. Use these to
test recall and to prepare for job interviews after the CKA.

---

## Architecture & Cluster Components

**1. What are the main control plane components?**
kube-apiserver (front door / REST API), etcd (key-value store of all state),
kube-scheduler (assigns pods to nodes), kube-controller-manager (runs core
controllers), and cloud-controller-manager (cloud integration).

**2. What runs on every worker node?**
kubelet (agent that manages pods via the container runtime), kube-proxy (network
rules for Services), and a container runtime (containerd/CRI-O).

**3. What is the role of the kube-apiserver?**
It is the only component that reads/writes etcd. It validates and processes REST
requests, handles authentication/authorization/admission, and is the hub all
other components talk through.

**4. What does etcd store and why is it critical?**
The entire cluster state: objects, configs, secrets, node registrations. Losing
etcd without a backup means losing the cluster; hence snapshot backup/restore.

**5. How does the scheduler decide where to place a pod?**
Two phases: filtering (nodes that can't run the pod are removed — resources,
taints, affinity, nodeSelector) and scoring (remaining nodes ranked; highest
score wins).

**6. What is a static pod?**
A pod managed directly by the kubelet from a manifest file (default
`/etc/kubernetes/manifests`), not via the API server. Control plane components in
kubeadm clusters are static pods. Deleting the manifest removes the pod.

**7. What is the difference between a controller and an operator?**
A controller is a reconciliation loop driving current state toward desired state.
An operator is a controller plus a Custom Resource Definition (CRD) encoding
domain-specific operational knowledge.

**8. What is the CRI, CNI, CSI?**
Container Runtime Interface (runtime plugins like containerd), Container Network
Interface (network plugins like Calico), Container Storage Interface (storage
plugins). They standardize how Kubernetes talks to runtimes/networking/storage.

---

## Workloads & Scheduling

**9. Difference between a Deployment, ReplicaSet, and Pod?**
A Pod is the smallest deployable unit. A ReplicaSet keeps N pod replicas running.
A Deployment manages ReplicaSets and adds declarative rolling updates/rollbacks.

**10. When would you use a StatefulSet vs. a Deployment?**
StatefulSet for stateful apps needing stable network identities, ordered
deployment/scaling, and stable per-pod storage (databases). Deployment for
stateless apps.

**11. What is a DaemonSet?**
Ensures a copy of a pod runs on every (or selected) node — used for log
collectors, node monitoring, CNI agents.

**12. Job vs. CronJob?**
A Job runs a pod to completion (batch task). A CronJob schedules Jobs on a cron
schedule.

**13. What are requests and limits?**
Requests are the guaranteed resources used for scheduling. Limits are the maximum
a container may use; exceeding a memory limit gets the container OOM-killed, CPU
is throttled.

**14. Explain QoS classes.**
Guaranteed (requests == limits for all resources), Burstable (requests set but
not equal to limits), BestEffort (no requests/limits). Under pressure, BestEffort
pods are evicted first.

**15. What are taints and tolerations?**
Taints repel pods from a node (`key=value:effect`). Tolerations on a pod allow it
to schedule onto tainted nodes. Effects: NoSchedule, PreferNoSchedule, NoExecute.

**16. nodeSelector vs. node affinity vs. pod affinity?**
nodeSelector is a simple label match. Node affinity is richer (required/preferred,
operators). Pod affinity/anti-affinity schedules pods relative to other pods'
locations.

**17. How does a rolling update work and how do you roll back?**
The Deployment gradually replaces old ReplicaSet pods with new ones respecting
maxSurge/maxUnavailable. Roll back with `kubectl rollout undo deployment/<name>`.

**18. What is a PodDisruptionBudget?**
It limits how many pods of an app can be voluntarily disrupted (e.g. during a
drain) at once, protecting availability.

**19. What are init containers?**
Containers that run to completion sequentially before the main containers start —
used for setup/prerequisite checks.

**20. What is the difference between livenessProbe, readinessProbe, and startupProbe?**
Liveness restarts an unhealthy container. Readiness gates whether the pod
receives traffic (removed from endpoints when failing). Startup delays the other
probes until a slow-starting app is up.

---

## Networking

**21. What are the Kubernetes networking rules?**
Every pod gets its own IP; pods can communicate with all other pods without NAT;
agents on a node can reach all pods on that node.

**22. What are the Service types?**
ClusterIP (internal only, default), NodePort (exposes on each node's port),
LoadBalancer (cloud LB), ExternalName (CNAME alias). Headless (`clusterIP: None`)
returns pod IPs directly.

**23. How does a Service find its pods?**
Via a label selector. Matching, ready pods are recorded in the Service's
Endpoints / EndpointSlices; kube-proxy programs iptables/ipvs rules to load
balance to them.

**24. What is kube-proxy's job?**
It watches Services/Endpoints and programs the node's iptables or IPVS rules so
traffic to a Service ClusterIP is forwarded to a backend pod.

**25. What is an Ingress and an Ingress Controller?**
An Ingress is an API object defining HTTP(S) routing rules (host/path -> service).
An Ingress Controller (nginx, traefik) is the actual proxy that implements those
rules; the Ingress object alone does nothing without a controller.

**26. How does DNS work in the cluster?**
CoreDNS runs as a Deployment; the `kube-dns` Service is the cluster DNS IP that
pods use (in `/etc/resolv.conf`). Services resolve as
`<svc>.<ns>.svc.cluster.local`.

**27. What is a NetworkPolicy and what enforces it?**
A NetworkPolicy restricts pod ingress/egress by selectors. By default all traffic
is allowed; a policy selecting a pod switches it to default-deny for the specified
direction. Enforcement requires a compatible CNI (Calico/Cilium).

**28. What is a Gateway API and how does it relate to Ingress?**
Gateway API is the newer, more expressive successor to Ingress (Gateway,
HTTPRoute, etc.), separating infrastructure (Gateway) from routing (Routes). CKA
v1.31+ includes Gateway API awareness.

---

## Storage

**29. PV vs. PVC?**
A PersistentVolume is a cluster storage resource (provisioned by admin or
dynamically). A PersistentVolumeClaim is a user's request for storage that binds
to a matching PV.

**30. What are access modes?**
ReadWriteOnce (one node RW), ReadOnlyMany (many nodes RO), ReadWriteMany (many
nodes RW), ReadWriteOncePod (single pod RW).

**31. What is a StorageClass and dynamic provisioning?**
A StorageClass defines a provisioner and parameters. When a PVC references it, a
PV is created automatically (dynamic provisioning) instead of pre-creating PVs.

**32. What is a reclaim policy?**
Determines what happens to a PV when its PVC is deleted: Retain (keep data),
Delete (delete the volume), or Recycle (deprecated).

**33. emptyDir vs. hostPath vs. PVC?**
emptyDir lives for the pod's lifetime on the node. hostPath mounts a node
directory (fragile, node-bound). PVC-backed volumes provide real, portable
persistent storage.

---

## Security

**34. Explain the auth chain: authentication, authorization, admission.**
A request is first authenticated (who are you — certs, tokens, SA), then
authorized (may you do it — RBAC), then passed through admission controllers
(mutating/validating policies) before persisting.

**35. Role vs. ClusterRole (and their bindings)?**
Role/RoleBinding are namespaced. ClusterRole/ClusterRoleBinding are cluster-wide.
A ClusterRole can also be referenced by a namespaced RoleBinding to grant its
permissions within one namespace.

**36. What is a ServiceAccount?**
An identity for processes in pods to authenticate to the API server. Pods use the
`default` SA unless assigned one; tokens are mounted (projected) into the pod.

**37. How do you restrict what a pod can do at the OS level?**
securityContext (runAsNonRoot, runAsUser, readOnlyRootFilesystem, drop
capabilities) and Pod Security Admission (enforce/audit/warn at
privileged/baseline/restricted levels per namespace).

**38. How are Secrets stored and how do you secure them?**
Secrets are base64-encoded in etcd (not encrypted by default). Enable
encryption-at-rest (EncryptionConfiguration), restrict RBAC access, and consider
external secret stores.

**39. What is the principle you apply to RBAC?**
Least privilege — grant only the verbs/resources needed, prefer namespaced Roles
over ClusterRoles, and audit with `kubectl auth can-i`.

---

## Troubleshooting & Operations

**40. A pod is stuck in Pending. How do you diagnose?**
`kubectl describe pod` and read Events: insufficient resources, untolerated
taints, unschedulable due to affinity, or an unbound PVC.

**41. A pod is CrashLoopBackOff. What next?**
`kubectl logs <pod> --previous` to see why the last instance died; check the exit
code and lastState via describe; verify command, config/secrets, and probes.

**42. kubectl commands hang / connection refused. What's wrong?**
The API server is likely down. On the control plane node check the static pod
manifest, kubelet (`systemctl status/journalctl -u kubelet`), and use
`crictl ps -a` / `crictl logs` to inspect the apiserver container.

**43. A node shows NotReady. How do you investigate?**
`kubectl describe node` for conditions; on the node check the kubelet
(`systemctl status kubelet`, `journalctl -u kubelet`), the container runtime,
disk/memory pressure, certs, and the CNI.

**44. A Service works by pod IP but not by service name. Likely cause?**
DNS issue — check CoreDNS pods/logs and the `kube-dns` Service/endpoints; verify
the pod's `/etc/resolv.conf`.

**45. A Service has no endpoints. Why?**
The selector doesn't match any ready pod labels, or the pods aren't passing
readiness probes. Compare `kubectl describe svc` selector with pod labels.

**46. How do you back up and restore etcd?**
`etcdctl snapshot save` with the endpoint/CA/cert/key flags; restore with
`etcdctl snapshot restore --data-dir=<new dir>` and repoint the etcd static pod's
hostPath volume to that directory, then restart kubelet.

**47. Describe the kubeadm upgrade order.**
Control plane first (`kubeadm upgrade apply` on the primary, `kubeadm upgrade
node` on others), then workers one at a time — drain, upgrade kubeadm, `upgrade
node`, upgrade kubelet/kubectl, restart kubelet, uncordon.

**48. What is the version skew rule for kubelet?**
kubelet may be up to 3 minor versions older than the apiserver but never newer.
Upgrade one minor version at a time.

**49. What tool inspects containers when the API server is down?**
`crictl` — it talks to the CRI runtime directly (`crictl ps`, `crictl logs`,
`crictl pods`). Use it, not `docker`, on modern nodes.

**50. How do you safely take a node down for maintenance?**
`kubectl drain <node> --ignore-daemonsets [--delete-emptydir-data] [--force]` to
cordon and evict pods, do the work, then `kubectl uncordon <node>`.
