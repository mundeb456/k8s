# Network Policies

A **NetworkPolicy** is a namespaced resource that controls the traffic allowed
to and from a set of Pods at the IP/port (L3/L4) level. It is the Kubernetes
equivalent of a firewall for Pods.

---

## Default behavior: allow all

**By default, all Pods are non-isolated** — every Pod can talk to every other
Pod, and all ingress and egress is allowed.

A Pod becomes **isolated** the moment *any* NetworkPolicy selects it:

- If a policy has `Ingress` in `policyTypes`, only the ingress rules in policies
  selecting that Pod are allowed; everything else is denied.
- If a policy has `Egress` in `policyTypes`, only the egress rules are allowed.
- Policies are **additive** — the union of all matching policies' rules applies.
  There is no "deny" rule and no ordering/priority.

Once a Pod is isolated for a direction, traffic in that direction is
**default-deny** unless explicitly permitted.

---

## CNI requirement

NetworkPolicy objects only take effect if the cluster's **CNI plugin enforces
them**. Plugins that enforce policies include **Calico**, Cilium, Weave Net, and
Antrea. The default **Flannel** CNI does **not** enforce NetworkPolicy.

You can create the objects on any cluster, but with a non-enforcing CNI they are
silently ignored.

```bash
# Common exam setup uses Calico
kubectl get pods -n kube-system | grep -i calico
```

---

## Anatomy of a NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example
  namespace: default
spec:
  podSelector:            # which pods this policy applies to
    matchLabels:
      role: db
  policyTypes:            # Ingress, Egress, or both
    - Ingress
    - Egress
  ingress:                # allowed incoming traffic
    - from:
        - podSelector:
            matchLabels:
              role: api
      ports:
        - protocol: TCP
          port: 5432
  egress:                 # allowed outgoing traffic
    - to:
        - podSelector:
            matchLabels:
              role: cache
      ports:
        - protocol: TCP
          port: 6379
```

- **`podSelector`** — selects the Pods (in the policy's namespace) the policy
  governs. An empty `podSelector: {}` selects **all Pods in the namespace**.
- **`policyTypes`** — which directions this policy affects. If omitted, it is
  inferred: `Ingress` if `ingress` is present, and `Egress` is added only if
  `egress` is present.

---

## from / to selectors

Both `ingress.from` and `egress.to` accept a list of peers. Each peer entry can
use `podSelector`, `namespaceSelector`, or `ipBlock`.

### podSelector

Selects Pods **within the same namespace** as the policy.

```yaml
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
```

### namespaceSelector

Selects entire namespaces (all Pods in matching namespaces).

```yaml
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: analytics
```

### Combining pod + namespace selectors (important!)

The distinction between one array element with two selectors vs two elements
changes the meaning (AND vs OR):

```yaml
  # AND: pods labeled app=web IN namespaces labeled team=analytics
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: analytics
          podSelector:
            matchLabels:
              app: web
```

```yaml
  # OR: (any pod in team=analytics namespaces) OR (app=web pods in this ns)
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: analytics
        - podSelector:
            matchLabels:
              app: web
```

The single `-` (one list item) with both keys = AND. Two `-` items = OR.

### ipBlock

Selects CIDR IP ranges (used for traffic to/from outside the cluster, or
specific node/external IPs). `except` carves out sub-ranges.

```yaml
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/16
            except:
              - 10.0.5.0/24
```

Note: `ipBlock` matches source/destination IPs. Pod IPs are ephemeral, so
`ipBlock` is normally used for external CIDRs, not for selecting Pods.

---

## ports

Restricts allowed ports/protocols for a rule. If `ports` is omitted, all ports
are allowed for that peer.

```yaml
      ports:
        - protocol: TCP
          port: 80
        - protocol: TCP
          port: 443
        - protocol: UDP
          port: 53
```

Port ranges are supported with `endPort`:

```yaml
      ports:
        - protocol: TCP
          port: 8000
          endPort: 8080
```

---

## Default-deny examples

### Deny all ingress in a namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: prod
spec:
  podSelector: {}          # all pods
  policyTypes:
    - Ingress
  # no ingress rules => nothing allowed in
```

### Deny all egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

### Deny all traffic (both directions)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### Allow DNS egress (common — you almost always need this)

When you apply default-deny egress, Pods lose DNS. Add this so they can resolve
names via CoreDNS in `kube-system`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

---

## Full example: 3-tier lockdown

Allow `frontend` → `api` on 8080, and `api` → `db` on 5432, deny everything
else. Policy attached to the `db` Pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: shop
spec:
  podSelector:
    matchLabels:
      tier: db
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: api
      ports:
        - protocol: TCP
          port: 5432
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

---

## Exam Tips

- Pods are **default-allow** until a policy selects them; selection flips that
  direction to **default-deny**.
- Policies are **additive** and have **no priority/order**. You cannot write an
  explicit "deny" rule — you deny by *not* allowing.
- `podSelector: {}` = all Pods in the namespace; empty `from`/`to` = all sources.
- `podSelector` inside `from`/`to` is scoped to the **policy's namespace**
  unless combined with a `namespaceSelector`.
- Watch the AND vs OR gotcha: two keys under one `-` = AND; separate `-`
  items = OR.
- NetworkPolicy is **namespaced** — check `-n <namespace>`.
- Requires an enforcing CNI (**Calico** on the exam); Flannel ignores policies.
- After default-deny egress, remember to **re-allow DNS (UDP/TCP 53 to
  kube-system)** or name resolution breaks.
- Use the `kubernetes.io/metadata.name` label (auto-added to every namespace) to
  target a namespace by name in a `namespaceSelector`.

---

## Key Commands

```bash
# List / inspect policies
kubectl get networkpolicy -n <ns>
kubectl get netpol -A
kubectl describe netpol <name> -n <ns>

# Create from file
kubectl apply -f policy.yaml

# See a pod's labels (needed to write selectors)
kubectl get pods -n <ns> --show-labels

# Label a namespace so a namespaceSelector can match it
kubectl label namespace <ns> team=analytics

# Explain the schema
kubectl explain networkpolicy.spec
kubectl explain networkpolicy.spec.ingress.from

# Test connectivity (deny = timeout/blocked)
kubectl exec -it <pod> -n <ns> -- wget -qO- --timeout=2 <target>:<port>
kubectl exec -it <pod> -n <ns> -- nc -zvw2 <target> <port>
```
