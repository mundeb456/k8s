# Services

Pods are ephemeral and get new IPs when recreated. A **Service** provides a stable virtual IP (and DNS name) that load-balances traffic to a dynamic set of Pods selected by labels. Services are one of the most-tested topics on the CKA.

## How a Service Works

A Service selects Pods via a **label selector**. The endpoints controller watches for matching, ready Pods and populates an **EndpointSlice** (and legacy **Endpoints** object) with their IPs. `kube-proxy` on each node programs the data plane (iptables/IPVS/nftables rules) so that traffic to the Service's `ClusterIP` is DNAT'd to a backend Pod IP.

```
Client -> Service ClusterIP (stable) -> kube-proxy rules -> one of the backend Pod IPs
```

## Service Types

### ClusterIP (default)

Exposes the Service on an internal virtual IP, reachable **only inside the cluster**. Used for intra-cluster communication (e.g., frontend → backend).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80          # Service port (what clients hit)
    targetPort: 8080  # container port on the Pods
    protocol: TCP
```

### NodePort

Builds on ClusterIP and additionally opens a static port (`nodePort`, range **30000–32767**) on **every node**. External traffic to `<NodeIP>:<nodePort>` is routed to the Service. Good for exposing a service without a cloud LB.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-np
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080   # optional; auto-assigned if omitted
```

### LoadBalancer

Builds on NodePort and provisions an **external load balancer** via the cloud provider (or MetalLB on bare metal), assigning an external IP that forwards to the NodePort. Standard way to expose a service to the internet in the cloud.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

### ExternalName

No proxying and no selector. It maps the Service DNS name to an external DNS name via a **CNAME** record — useful to reference an external service (e.g. a managed database) by an in-cluster name.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.prod.example.com
```

## port vs targetPort vs nodePort

| Field | Where it lives | Meaning |
|-------|----------------|---------|
| `port` | on the Service | the port the Service listens on (clients connect here via ClusterIP/DNS). |
| `targetPort` | on the Service | the port on the **Pod/container** traffic is forwarded to. Can be a named port. |
| `nodePort` | NodePort/LoadBalancer | the static port opened on **every node** (30000–32767). |

`targetPort` can reference a container port **by name**:

```yaml
# Pod
ports:
- name: http
  containerPort: 8080
# Service
ports:
- port: 80
  targetPort: http     # resolves to 8080
```

## Endpoints and EndpointSlices

The set of backend IPs behind a Service:

```bash
kubectl get endpoints <svc>            # legacy Endpoints object
kubectl get endpointslices            # modern, scalable representation
kubectl describe svc <svc>            # shows Endpoints in the output
```

**If `Endpoints` is empty, the Service has no backends** — almost always a selector mismatch or no ready Pods. This is the #1 troubleshooting check.

## Headless Services

Set `clusterIP: None`. No virtual IP or proxying; instead DNS returns the **individual Pod IPs** (A/AAAA records). Used with StatefulSets so each Pod is addressable, and by clients that do their own load-balancing.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None
  selector:
    app: db
  ports:
  - port: 5432
    targetPort: 5432
```

With a StatefulSet, each Pod also gets a stable DNS name: `<pod>.<service>.<namespace>.svc.cluster.local`.

## kube-proxy Modes

`kube-proxy` implements Services on each node. Modes:

- **iptables** (default on most clusters) — installs iptables rules; picks a backend at random. Scales less well with thousands of Services.
- **IPVS** — uses the kernel IP Virtual Server; better performance and more LB algorithms (rr, lc, etc.) at scale. Requires the `ip_vs` kernel modules.
- **nftables** — newer backend, more efficient than iptables on large clusters.
- **userspace** — legacy, effectively unused.

```bash
kubectl -n kube-system get configmap kube-proxy -o yaml | grep mode
kubectl -n kube-system logs ds/kube-proxy | head
```

## DNS Names

CoreDNS gives every Service a DNS record:

```
<service>.<namespace>.svc.cluster.local
```

- Same namespace: just `<service>` works (e.g. `backend`).
- Cross-namespace: `<service>.<namespace>` (e.g. `backend.prod`).
- Fully qualified: `backend.prod.svc.cluster.local`.
- SRV records exist for named ports: `_http._tcp.<service>.<namespace>.svc.cluster.local`.

```bash
# Test DNS from a debug pod
kubectl run test --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup backend.default.svc.cluster.local
```

## Creating Services Imperatively

```bash
# Expose an existing deployment (creates a Service selecting its Pods)
kubectl expose deployment web --port=80 --target-port=8080
kubectl expose deployment web --type=NodePort --port=80

# Generate YAML to edit
kubectl expose deployment web --port=80 --target-port=8080 --dry-run=client -o yaml > svc.yaml

# Create directly
kubectl create service clusterip mysvc --tcp=80:8080
kubectl create service nodeport  mysvc --tcp=80:8080 --node-port=30080
```

`kubectl expose` reuses the target's labels as the selector automatically — the fastest way to make a correct Service.

## Troubleshooting Service Connectivity

Work outward from the Pods:

1. **Endpoints populated?** `kubectl get endpoints <svc>` — empty means selector mismatch or no ready Pods. Confirm `svc.spec.selector` matches the Pod labels and that Pods pass readiness.
2. **Pods actually ready?** `kubectl get pods -l <selector>` — a failing readiness probe removes a Pod from endpoints.
3. **Port mapping correct?** Verify `targetPort` equals the container's listening port.
4. **DNS resolving?** From a debug pod: `nslookup <svc>` / `wget -qO- <svc>:<port>`.
5. **kube-proxy healthy?** `kubectl -n kube-system get pods -l k8s-app=kube-proxy` and check its logs.
6. **NetworkPolicy blocking?** A default-deny policy can silently drop traffic even when the Service is correct.

```bash
kubectl describe svc <svc>
kubectl get endpoints <svc>
kubectl get pods -l <selector> -o wide
kubectl run curl --image=curlimages/curl --rm -it --restart=Never -- curl -s <svc>:<port>
```

## Exam Tips

- Empty `Endpoints`/`EndpointSlice` = selector mismatch or no ready Pods — check this first for any "service unreachable" task.
- Know the three port fields cold: `port` (Service), `targetPort` (Pod), `nodePort` (node, 30000–32767).
- `kubectl expose` is the fastest way to a correct Service because it copies the selector for you.
- ClusterIP = internal only; NodePort = every node's IP:port; LoadBalancer = external IP (superset of NodePort); ExternalName = CNAME, no selector/proxy.
- Headless = `clusterIP: None` → DNS returns Pod IPs (StatefulSets).
- DNS form is `<svc>.<ns>.svc.cluster.local`; same-namespace can use the short name.
- A NodePort Service also has a ClusterIP; a LoadBalancer also has a NodePort and ClusterIP.
- Use a throwaway `busybox`/`curl` Pod with `--rm -it --restart=Never` to test connectivity from inside the cluster.

## Key Commands

```bash
kubectl expose deployment <name> --port=<p> --target-port=<tp> [--type=NodePort]
kubectl expose deployment <name> --port=<p> --dry-run=client -o yaml
kubectl create service clusterip|nodeport <name> --tcp=<p>:<tp>
kubectl get svc [-o wide]
kubectl describe svc <name>
kubectl get endpoints <name>
kubectl get endpointslices
kubectl get pods -l <selector> -o wide
kubectl run test --image=busybox:1.36 --rm -it --restart=Never -- nslookup <svc>.<ns>
```
