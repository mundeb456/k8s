# ServiceAccounts

A **ServiceAccount (SA)** provides an identity for **processes running in Pods** to authenticate to the Kubernetes API server. Whereas Users are meant for humans (managed externally), ServiceAccounts are namespaced Kubernetes objects for **in-cluster workloads**.

API group: core (`v1`), kind `ServiceAccount`.

---

## The Default ServiceAccount

- Every namespace automatically gets a ServiceAccount named **`default`**.
- Any Pod that does not set `spec.serviceAccountName` is assigned the namespace's `default` SA.
- The `default` SA has **no RBAC permissions** beyond basic API discovery — it cannot read or modify cluster resources unless you bind a Role/ClusterRole to it.

```bash
kubectl get serviceaccount            # or: kubectl get sa
kubectl get sa default -o yaml
```

---

## Token Projection: Bound Tokens & the TokenRequest API

Modern Kubernetes (v1.22+, fully default v1.24+) uses **bound service account tokens** instead of long-lived Secret-based tokens.

- Tokens are issued by the **TokenRequest API** and **projected** into the Pod at `/var/run/secrets/kubernetes.io/serviceaccount/token` via a **projected volume** that the kubelet mounts automatically.
- These tokens are:
  - **Time-bound** — they expire (default ~1 hour) and are **auto-rotated** by the kubelet.
  - **Audience-bound** — scoped to specific audiences.
  - **Object-bound** — tied to the lifetime of the specific Pod; when the Pod is deleted, the token is invalid.
- Since **v1.24, creating a ServiceAccount no longer auto-generates a permanent Secret** with a token. To get a long-lived token you must explicitly create a Secret of type `kubernetes.io/service-account-token`, or request one on demand.

### Requesting a token imperatively

```bash
# Short-lived token (TokenRequest API), e.g. 1 hour
kubectl create token build-bot --namespace ci
kubectl create token build-bot --duration=3600s
```

### Creating a long-lived token Secret (when you truly need one)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: build-bot-token
  namespace: ci
  annotations:
    kubernetes.io/service-account.name: build-bot   # links Secret to the SA
type: kubernetes.io/service-account-token
```

The token controller populates the `token`, `ca.crt`, and `namespace` fields.

---

## Creating a ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
  namespace: ci
```

```bash
kubectl create serviceaccount build-bot -n ci
# scaffold YAML
kubectl create sa build-bot -n ci --dry-run=client -o yaml
```

---

## Mounting the SA Token in a Pod

Assign an SA to a Pod with `spec.serviceAccountName`. The kubelet then projects the token, CA cert, and namespace into the container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-client
  namespace: ci
spec:
  serviceAccountName: build-bot     # use this SA's identity
  containers:
    - name: app
      image: curlimages/curl:8.10.1
      command: ["sleep", "3600"]
```

Inside the container the projected files appear at:

```
/var/run/secrets/kubernetes.io/serviceaccount/token       # bearer token (auto-rotated)
/var/run/secrets/kubernetes.io/serviceaccount/ca.crt      # cluster CA
/var/run/secrets/kubernetes.io/serviceaccount/namespace   # pod namespace
```

### Explicit projected token volume (custom audience / expiry)

You can control audience and expiry explicitly with a `serviceAccountToken` projected volume:

```yaml
spec:
  serviceAccountName: build-bot
  containers:
    - name: app
      image: curlimages/curl:8.10.1
      volumeMounts:
        - name: api-token
          mountPath: /var/run/secrets/tokens
          readOnly: true
  volumes:
    - name: api-token
      projected:
        sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
              audience: api
```

---

## automountServiceAccountToken

By default the SA token is auto-mounted into every Pod. To prevent a Pod from receiving API credentials (good security practice for workloads that never call the API), disable it. It can be set on either the **ServiceAccount** or the **Pod** — the Pod-level setting wins.

```yaml
# On the ServiceAccount (applies to all pods using it)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-api-access
  namespace: ci
automountServiceAccountToken: false
---
# Or on the Pod (overrides the SA-level setting)
apiVersion: v1
kind: Pod
metadata:
  name: sealed
spec:
  serviceAccountName: build-bot
  automountServiceAccountToken: false     # this Pod gets no token
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "3600"]
```

---

## imagePullSecrets

A ServiceAccount can carry `imagePullSecrets`. Any Pod using that SA automatically gets those pull secrets, so you don't have to add them to every Pod spec — handy for pulling from private registries.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
  namespace: ci
imagePullSecrets:
  - name: my-registry-cred
```

Create the docker registry secret first:

```bash
kubectl create secret docker-registry my-registry-cred \
  --docker-server=registry.example.com \
  --docker-username=svc \
  --docker-password=s3cr3t \
  --docker-email=svc@example.com \
  -n ci
```

---

## Using a ServiceAccount with RBAC

An SA is just an identity — it has no permissions until you **bind a Role or ClusterRole** to it. Reference it as a subject:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: ci
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: build-bot-read-pods
  namespace: ci
subjects:
  - kind: ServiceAccount
    name: build-bot
    namespace: ci
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Or imperatively:

```bash
kubectl create rolebinding build-bot-read-pods \
  --role=pod-reader \
  --serviceaccount=ci:build-bot \
  -n ci
```

The SA's fully qualified RBAC name is:

```
system:serviceaccount:<namespace>:<name>      # e.g. system:serviceaccount:ci:build-bot
```

All ServiceAccounts are also members of the groups `system:serviceaccounts` and `system:serviceaccounts:<namespace>`.

Verify:

```bash
kubectl auth can-i list pods --as system:serviceaccount:ci:build-bot -n ci
```

---

## Full End-to-End YAML

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitor
  namespace: monitoring
automountServiceAccountToken: true
imagePullSecrets:
  - name: registry-cred
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metrics-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "nodes", "services"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitor-metrics-reader
subjects:
  - kind: ServiceAccount
    name: monitor
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: metrics-reader
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: monitor-agent
  namespace: monitoring
spec:
  serviceAccountName: monitor
  containers:
    - name: agent
      image: curlimages/curl:8.10.1
      command: ["sleep", "3600"]
```

---

## Exam Tips

- **`serviceAccountName` (not `serviceAccount`)** is the current Pod spec field. Every Pod without it uses the namespace `default` SA.
- **`default` SA has no permissions.** If a Pod gets 403 from the API, bind a Role/ClusterRole to its SA.
- **v1.24+ does not auto-create token Secrets.** Use `kubectl create token <sa>` for a short-lived token, or create a `kubernetes.io/service-account-token` Secret (with the `kubernetes.io/service-account.name` annotation) for a long-lived one.
- **Projected tokens live at** `/var/run/secrets/kubernetes.io/serviceaccount/token`, are time-bound, and auto-rotate. Know this path.
- **`automountServiceAccountToken: false`** stops the token from mounting — set it on the SA or the Pod (Pod wins). Common security-hardening question.
- **SA RBAC subject:** `kind: ServiceAccount`, with `namespace`, and **no `apiGroup`**. The identity string is `system:serviceaccount:<ns>:<name>`.
- **`imagePullSecrets` on an SA** auto-inject into every Pod using that SA — cleaner than per-Pod pull secrets.
- Test permissions with `kubectl auth can-i ... --as system:serviceaccount:<ns>:<name> -n <ns>`.
- `kubectl create token <sa> --duration=<d>` mints an on-demand bearer token via the TokenRequest API.

---

## Key Commands

```bash
kubectl create serviceaccount build-bot -n ci                              # create SA
kubectl get sa -n ci                                                       # list SAs
kubectl create token build-bot -n ci                                       # short-lived token
kubectl create token build-bot --duration=3600s                            # with expiry
kubectl create rolebinding rb --role=pod-reader \
  --serviceaccount=ci:build-bot -n ci                                      # bind SA to Role
kubectl create secret docker-registry cred --docker-server=... \
  --docker-username=... --docker-password=... -n ci                        # pull secret
kubectl auth can-i list pods --as system:serviceaccount:ci:build-bot -n ci # verify
kubectl set serviceaccount deployment/web build-bot                        # set SA on workload
```
