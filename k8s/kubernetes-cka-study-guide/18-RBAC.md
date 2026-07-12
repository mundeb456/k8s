# RBAC (Role-Based Access Control)

**RBAC** regulates access to Kubernetes API resources based on the roles of individual users, groups, and ServiceAccounts. It is enabled by the `RBAC` authorization mode (`--authorization-mode=Node,RBAC` on the API server, the kubeadm default).

RBAC is built from four API objects, all in the API group **`rbac.authorization.k8s.io/v1`**:

- **Role** — a set of permissions **within a namespace**.
- **ClusterRole** — a set of permissions that are **cluster-wide** (or reusable across namespaces).
- **RoleBinding** — grants a Role (or ClusterRole) to subjects **within a namespace**.
- **ClusterRoleBinding** — grants a ClusterRole to subjects **cluster-wide**.

**Core rule:** A role defines *what* actions are allowed; a binding attaches those permissions to *who* (the subjects). Nothing is granted until a binding connects a role to a subject.

---

## Role vs ClusterRole

| | Role | ClusterRole |
|--|------|-------------|
| Scope | Single namespace | Cluster-wide |
| Grants access to | Namespaced resources in that namespace | Namespaced resources (in any/all ns), **cluster-scoped resources** (nodes, PVs, namespaces), and **non-resource URLs** (`/healthz`) |
| Requires `namespace` in metadata | Yes | No |

Use a **Role** when the permission is confined to one namespace. Use a **ClusterRole** for:

- Cluster-scoped resources (nodes, persistentvolumes, namespaces, storageclasses).
- Non-resource endpoints (`/healthz`, `/metrics`).
- Namespaced resources you want to grant **across all namespaces**.
- Reusable permission sets referenced by multiple RoleBindings in different namespaces.

---

## RoleBinding vs ClusterRoleBinding

- **RoleBinding** grants permissions **within its own namespace**. It can reference **either** a Role (same namespace) **or** a ClusterRole. When it references a ClusterRole, the permissions are scoped **only to the RoleBinding's namespace** (a common, powerful pattern — define once as a ClusterRole, grant per-namespace with RoleBindings).
- **ClusterRoleBinding** grants the permissions of a **ClusterRole across the entire cluster** (all namespaces + cluster-scoped resources).

| Binding | References | Effective scope |
|---------|-----------|-----------------|
| RoleBinding + Role | Role (same ns) | That namespace |
| RoleBinding + ClusterRole | ClusterRole | That namespace only |
| ClusterRoleBinding + ClusterRole | ClusterRole | Whole cluster |

> A RoleBinding **cannot** reference a Role in a different namespace. A ClusterRoleBinding **cannot** reference a Role (only a ClusterRole).

---

## Subjects (Who)

`subjects` in a binding can be:

- **User** — an authenticated user identity (from certs, OIDC, etc.). Kubernetes has no User objects; these are external identities. `kind: User`.
- **Group** — a set of users. `kind: Group`. Example: `system:authenticated`, or an OIDC/cert `O=` group.
- **ServiceAccount** — an in-cluster identity for Pods. `kind: ServiceAccount` with a `namespace`.

```yaml
subjects:
  - kind: User
    name: jane                    # certificate CN, exactly as authenticated
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: build-bot
    namespace: ci                 # ServiceAccounts have a namespace
    # NOTE: no apiGroup for ServiceAccount (core "" group)
```

---

## Rules: verbs, resources, apiGroups

Each `rule` in a Role/ClusterRole grants a combination of:

- **`apiGroups`** — the API group(s) of the resources. The **core group is `""`** (empty string): pods, services, configmaps, secrets, nodes, namespaces, persistentvolumeclaims. Other groups: `apps` (deployments, daemonsets, statefulsets, replicasets), `batch` (jobs, cronjobs), `rbac.authorization.k8s.io`, `networking.k8s.io`, `storage.k8s.io`.
- **`resources`** — the resource types (`pods`, `deployments`, `secrets`). Use subresources like `pods/log`, `pods/exec`, `deployments/scale`.
- **`verbs`** — the allowed actions: `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection`. Use `["*"]` for all.
- **`resourceNames`** (optional) — restrict the rule to specific named objects. Note: `list`, `watch`, `create`, and `deletecollection` cannot be restricted by `resourceNames`.

```yaml
rules:
  - apiGroups: [""]                    # core group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["db-credentials"]  # only this named secret
    verbs: ["get"]
```

---

## Aggregated ClusterRoles

A ClusterRole can **aggregate** rules from other ClusterRoles that carry matching labels. The controller manager automatically fills the aggregated role's `rules` by merging every ClusterRole whose labels match the `aggregationRule` selectors.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.example.com/aggregate-to-monitoring: "true"
rules: []   # left empty; controller fills it automatically
---
# Any ClusterRole with this label is merged into "monitoring"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
  - apiGroups: [""]
    resources: ["services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
```

The built-in `admin`, `edit`, and `view` ClusterRoles use aggregation, which is why CRDs can extend them.

---

## Full YAML Examples

### Role + RoleBinding (namespaced)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: development
subjects:
  - kind: User
    name: jane
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole + ClusterRoleBinding (cluster-wide)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes-global
subjects:
  - kind: Group
    name: cluster-ops
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole granted per-namespace via RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets-in-team-a
  namespace: team-a
subjects:
  - kind: ServiceAccount
    name: reader-sa
    namespace: team-a
roleRef:
  kind: ClusterRole          # reference a ClusterRole...
  name: secret-reader        # ...but scope it to team-a via this RoleBinding
  apiGroup: rbac.authorization.k8s.io
```

---

## kubectl auth can-i

Test whether an identity is allowed to perform an action:

```bash
# As yourself
kubectl auth can-i create deployments --namespace dev
kubectl auth can-i delete pods

# Impersonate another user/group/serviceaccount (needs impersonate perms)
kubectl auth can-i list secrets --as jane --namespace dev
kubectl auth can-i get pods --as system:serviceaccount:ci:build-bot -n ci
kubectl auth can-i '*' '*'                    # am I cluster-admin?

# List everything you can do in a namespace
kubectl auth can-i --list -n dev
```

---

## Imperative Creation

Very useful under exam time pressure:

```bash
# Create a Role
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods,pods/log \
  -n development

# Create a RoleBinding to a User
kubectl create rolebinding read-pods \
  --role=pod-reader \
  --user=jane \
  -n development

# RoleBinding to a ServiceAccount
kubectl create rolebinding sa-read-pods \
  --role=pod-reader \
  --serviceaccount=development:reader-sa \
  -n development

# Create a ClusterRole
kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

# ClusterRoleBinding
kubectl create clusterrolebinding read-nodes-global \
  --clusterrole=node-reader \
  --group=cluster-ops

# Bind a ClusterRole into ONE namespace via a RoleBinding
kubectl create rolebinding view-team-a \
  --clusterrole=view \
  --serviceaccount=team-a:reader-sa \
  -n team-a

# Scaffold YAML instead of applying
kubectl create role pod-reader --verb=get,list --resource=pods \
  --dry-run=client -o yaml
```

---

## Exam Tips

- **`roleRef` is immutable.** To change what a binding points to, delete and recreate it.
- **Core API group is `""`** (empty string), not `"core"`. Deployments are in `apps`, jobs/cronjobs in `batch`. Getting the apiGroup wrong is the #1 RBAC mistake.
- **RoleBinding + ClusterRole = permissions in that one namespace.** This is how the built-in `view`/`edit`/`admin` ClusterRoles get granted per-namespace.
- **ClusterRoleBinding = cluster-wide.** Do not use it when you only need one namespace.
- **ServiceAccount subject** uses `kind: ServiceAccount` with a `namespace` and **no `apiGroup`**. User/Group use `apiGroup: rbac.authorization.k8s.io`.
- **Verify with `kubectl auth can-i ... --as ...`** — the fastest way to confirm a binding works. `--as system:serviceaccount:<ns>:<name>` for SAs.
- Use **`kubectl create role/clusterrole/rolebinding/clusterrolebinding`** imperatively, then `--dry-run=client -o yaml` to tweak. Much faster than hand-writing.
- **`resourceNames`** cannot restrict `list`, `watch`, `create`, or `deletecollection`.
- Subresources are separate: to read logs you need `pods/log`; to exec you need `pods/exec`.
- RBAC is **additive/allow-only** — there are no deny rules. If any binding grants an action, it is allowed.

---

## Key Commands

```bash
kubectl create role NAME --verb=... --resource=... -n NS                 # role
kubectl create clusterrole NAME --verb=... --resource=...                # clusterrole
kubectl create rolebinding NAME --role=R --user=U -n NS                  # rolebinding (user)
kubectl create rolebinding NAME --role=R --serviceaccount=NS:SA -n NS    # rolebinding (SA)
kubectl create clusterrolebinding NAME --clusterrole=CR --group=G        # clusterrolebinding
kubectl auth can-i VERB RESOURCE -n NS                                   # test self
kubectl auth can-i VERB RESOURCE --as system:serviceaccount:NS:SA -n NS  # test SA
kubectl auth can-i --list -n NS                                          # list perms
kubectl get roles,rolebindings,clusterroles,clusterrolebindings -A       # inspect
```
