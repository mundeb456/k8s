# 30 - Helm

Helm is the package manager for Kubernetes. Since CKA v1.31 the exam includes
Helm-related tasks (installing/upgrading charts, managing repos, overriding
values). You do not need to author complex charts, but you must be fluent with
the CLI.

---

## Core concepts

| Term | Meaning |
|------|---------|
| **Chart** | A package of Kubernetes resource templates + metadata. The unit you install. |
| **Release** | A specific installed instance of a chart in a cluster (has a name and revision history). |
| **Repository** | An HTTP server hosting packaged charts (an index of `.tgz` files). |
| **Values** | Configuration inputs to a chart. Defaults live in `values.yaml`; you override them at install/upgrade time. |
| **Revision** | Each install/upgrade of a release increments a revision number; you can roll back to any revision. |

Helm renders the chart templates with the values to produce final Kubernetes
manifests, then applies them via the API server.

---

## Repositories

```bash
# Add a repo
helm repo add bitnami https://charts.bitnami.com/bitnami

# Refresh the local index for all repos (do this after add)
helm repo update

# List configured repos
helm repo list

# Search a repo for charts
helm search repo nginx
helm search repo bitnami/mysql --versions

# Remove a repo
helm repo remove bitnami
```

Search the public Artifact Hub (across many repos):

```bash
helm search hub wordpress
```

---

## Install

```bash
# helm install <release-name> <chart>
helm install my-nginx bitnami/nginx

# Install into a namespace (create it if missing)
helm install my-nginx bitnami/nginx -n web --create-namespace

# Pin a chart version
helm install my-nginx bitnami/nginx --version 15.4.4

# Preview rendered manifests WITHOUT installing
helm install my-nginx bitnami/nginx --dry-run --debug

# Install from a local chart directory or packaged .tgz
helm install my-app ./mychart
helm install my-app ./mychart-0.1.0.tgz
```

---

## Overriding values

Three ways, applied with increasing precedence (later overrides earlier):

```bash
# 1. Inline with --set (dotted paths; commas separate multiple)
helm install my-nginx bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=NodePort

# 2. A custom values file
helm install my-nginx bitnami/nginx -f my-values.yaml

# 3. Multiple files (last wins on conflicts) + --set on top
helm install my-nginx bitnami/nginx -f base.yaml -f prod.yaml --set image.tag=1.27
```

`--set` beats `-f`, and later `-f` files beat earlier ones. Inspect the merged
values a chart will use:

```bash
helm show values bitnami/nginx        # the chart's default values.yaml
helm get values my-nginx              # values used by an installed release
helm get values my-nginx --all        # include defaults
```

---

## List / status / inspect

```bash
helm list                    # releases in current namespace
helm list -A                 # all namespaces
helm list -n web

helm status my-nginx         # release status + notes
helm get manifest my-nginx   # the actual manifests Helm applied
helm history my-nginx        # revision history (for rollback)
```

---

## Upgrade

```bash
# Upgrade to a new chart version or new values
helm upgrade my-nginx bitnami/nginx --version 15.5.0
helm upgrade my-nginx bitnami/nginx --set replicaCount=5
helm upgrade my-nginx bitnami/nginx -f new-values.yaml

# Install if not present, otherwise upgrade (idempotent — very useful)
helm upgrade --install my-nginx bitnami/nginx -n web --create-namespace

# Preview an upgrade
helm upgrade my-nginx bitnami/nginx --dry-run --debug
```

---

## Rollback

Each upgrade creates a new revision. Roll back to any prior revision.

```bash
helm history my-nginx           # find the REVISION number
helm rollback my-nginx 1        # roll back to revision 1
helm rollback my-nginx          # roll back to the immediately previous revision
```

---

## Uninstall

```bash
helm uninstall my-nginx
helm uninstall my-nginx -n web

# Keep the release history (so you can roll back / audit)
helm uninstall my-nginx --keep-history
```

---

## Chart structure

A chart is a directory with this layout:

```
mychart/
├── Chart.yaml          # chart metadata
├── values.yaml         # default configuration values
├── charts/             # subcharts (dependencies)
├── templates/          # templated Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl    # reusable template snippets
│   └── NOTES.txt       # message shown after install
└── .helmignore
```

### Chart.yaml

```yaml
apiVersion: v2
name: mychart
description: A Helm chart for my application
type: application         # or "library"
version: 0.1.0            # chart version (SemVer)
appVersion: "1.27.0"      # version of the app being deployed
dependencies:
  - name: postgresql
    version: "15.x.x"
    repository: https://charts.bitnami.com/bitnami
```

### values.yaml

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.27"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 500m
    memory: 256Mi
```

---

## Templating (brief)

Templates in `templates/` use Go templating. Values are referenced via
`.Values`, release info via `.Release`, chart info via `.Chart`.

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-{{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

Render templates locally without touching the cluster:

```bash
helm template my-release ./mychart
helm template my-release ./mychart -f prod-values.yaml
helm lint ./mychart                 # validate chart correctness
```

---

## Managing chart dependencies

```bash
helm dependency update ./mychart     # pull subcharts listed in Chart.yaml into charts/
helm dependency list ./mychart
```

---

## Quick reference

```bash
helm repo add <name> <url>          # add a repo
helm repo update                    # refresh indexes
helm search repo <keyword>          # find charts
helm install <rel> <chart>          # install
helm install <rel> <chart> --set k=v -f vals.yaml   # with overrides
helm upgrade --install <rel> <chart>                # idempotent install/upgrade
helm list -A                        # all releases everywhere
helm history <rel>                  # revisions
helm rollback <rel> <rev>           # roll back
helm uninstall <rel>                # remove
helm show values <chart>            # default values
helm template <rel> <chart>         # render locally
```

---

## Common gotchas

- **Forgetting `helm repo update`** after `helm repo add` — searches/installs use
  a stale or empty index.
- **`--set` precedence** — inline `--set` overrides `-f` files; later `-f` files
  override earlier ones.
- **Namespace scope** — `helm list` only shows the current namespace unless you
  pass `-A` or `-n`.
- **`helm upgrade --install`** is the safe idempotent pattern for repeatable
  installs.
- **Chart version vs. appVersion** — `version` is the chart package version;
  `appVersion` is the packaged application's version. `--version` pins the chart.
