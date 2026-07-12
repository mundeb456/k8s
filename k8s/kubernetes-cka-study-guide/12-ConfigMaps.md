# ConfigMaps

A **ConfigMap** stores non-confidential configuration data as key-value pairs, decoupling configuration from container images. Pods consume ConfigMaps as **environment variables**, **command-line arguments**, or **files in a volume**. ConfigMaps are **namespaced** — a Pod can only reference a ConfigMap in its own namespace.

ConfigMaps are **not** for secrets. They are stored in plaintext in etcd and visible to anyone with read access. Use a Secret (see `13-Secrets.md`) for sensitive data.

Data limit: a single ConfigMap is capped at **1 MiB**.

## Creating ConfigMaps

### Imperatively — from literals

```bash
kubectl create configmap app-config \
  --from-literal=APP_COLOR=blue \
  --from-literal=APP_MODE=prod
```

### Imperatively — from a file

Each file becomes one key (the filename) whose value is the file's entire contents.

```bash
# Key = filename ("app.properties"), value = file contents
kubectl create configmap app-config --from-file=app.properties

# Override the key name
kubectl create configmap app-config --from-file=config=app.properties

# A whole directory: each file in it becomes a key
kubectl create configmap app-config --from-file=./config-dir/
```

### Imperatively — from an env file

An env file is `KEY=VALUE` lines; **each line becomes a separate key** (unlike `--from-file`, which makes the whole file one key).

```bash
# Given config.env containing:
#   APP_COLOR=blue
#   APP_MODE=prod
kubectl create configmap app-config --from-env-file=config.env
# Result: two keys, APP_COLOR and APP_MODE
```

Generate YAML without creating the object (great for the exam):

```bash
kubectl create configmap app-config \
  --from-literal=APP_COLOR=blue \
  --dry-run=client -o yaml > cm.yaml
```

### Declaratively (YAML)

`data` holds UTF-8 string values; `binaryData` holds base64-encoded binary values.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  APP_COLOR: blue
  APP_MODE: prod
  # A multi-line file-style value:
  app.properties: |
    color=blue
    mode=prod
    retries=3
```

## Consuming as Individual Environment Variables

Use `valueFrom.configMapKeyRef` to pull one key into one env var.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-env-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo $APP_COLOR $APP_MODE; sleep 3600"]
      env:
        - name: APP_COLOR
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_COLOR
        - name: APP_MODE
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_MODE
```

## Consuming with envFrom (all keys at once)

`envFrom` injects **every** key in the ConfigMap as an environment variable, using the keys as the variable names. An optional `prefix` prepends a string to each name.

```yaml
    spec:
      containers:
        - name: app
          image: busybox:1.36
          envFrom:
            - configMapRef:
                name: app-config
            # optional:
            # - configMapRef:
            #     name: app-config
            #   prefix: CFG_
```

Keys that are not valid environment variable names (e.g. containing `.`) are **skipped** and an event is logged.

## Consuming as a Volume Mount

Mounting a ConfigMap as a volume creates **one file per key**; the filename is the key and the contents are the value. This is how you inject config *files* (nginx.conf, application.yaml, etc.).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-vol-demo
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: config-vol
          mountPath: /etc/app-config      # each key becomes a file here
          readOnly: true
  volumes:
    - name: config-vol
      configMap:
        name: app-config
        # Optionally project only specific keys to specific paths:
        # items:
        #   - key: app.properties
        #     path: app.properties
```

Result: `/etc/app-config/APP_COLOR`, `/etc/app-config/APP_MODE`, `/etc/app-config/app.properties`.

ConfigMap volume updates propagate: when the ConfigMap changes, mounted files are eventually updated (typically within ~1 minute, unless the volume uses `subPath` — see below). Environment variables, by contrast, are **only** set at container start and never update live.

## subPath

`subPath` mounts a **single file** from the ConfigMap into a target path **without** replacing the entire directory. This is essential when you need to drop one config file into a directory that already contains other files (mounting the whole ConfigMap as a directory would hide them).

```yaml
    spec:
      containers:
        - name: app
          image: nginx:1.27
          volumeMounts:
            - name: nginx-conf
              mountPath: /etc/nginx/nginx.conf   # a single file, not a dir
              subPath: nginx.conf                # the key to mount
      volumes:
        - name: nginx-conf
          configMap:
            name: nginx-config
```

**Trade-off:** files mounted via `subPath` do **not** receive live updates when the ConfigMap changes — you must restart/roll the Pod to pick up new content.

## Immutable ConfigMaps

Setting `immutable: true` prevents any updates to the ConfigMap's data. Benefits: protects against accidental changes and improves cluster performance (the kubelet stops watching immutable ConfigMaps for changes). To change an immutable ConfigMap you must delete and recreate it (and roll the consuming Pods).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2
data:
  APP_COLOR: green
immutable: true
```

## Exam Tips

- `--from-file` makes the **whole file one key** (filename = key). `--from-env-file` makes **each line its own key**. This distinction is heavily tested.
- Env vars from a ConfigMap are captured **only at container start**; changing the ConfigMap does not update running env vars. Mounted volume files (without `subPath`) *do* update. `subPath` mounts do **not** update.
- Use `envFrom` + `configMapRef` to bring in all keys at once; use `valueFrom.configMapKeyRef` for a single key.
- Reference must be same-namespace; there is no cross-namespace ConfigMap reference.
- If a referenced ConfigMap or key is missing and not marked `optional: true`, the Pod fails to start. Add `optional: true` under the ref to tolerate absence.
- Fastest exam workflow: `kubectl create configmap ... --dry-run=client -o yaml` to generate the manifest, then edit.
- `kubectl describe configmap <name>` shows keys and values; `kubectl get cm <name> -o yaml` shows the full object.
- Remember the 1 MiB size limit and that ConfigMaps are plaintext — not for secrets.

## Key Commands

```bash
# Create imperatively
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=prod
kubectl create configmap app-config --from-file=app.properties
kubectl create configmap app-config --from-file=./config-dir/
kubectl create configmap app-config --from-env-file=config.env

# Generate YAML to edit (exam-friendly)
kubectl create configmap app-config --from-literal=APP_COLOR=blue --dry-run=client -o yaml > cm.yaml

# Inspect
kubectl get configmap
kubectl get cm app-config -o yaml
kubectl describe cm app-config

# Verify consumption
kubectl exec cm-env-demo -- env | grep APP_
kubectl exec cm-vol-demo -- ls /etc/app-config
kubectl exec cm-vol-demo -- cat /etc/app-config/APP_COLOR

# Edit / delete (recreate for immutable ones)
kubectl edit cm app-config
kubectl delete cm app-config

# Field reference
kubectl explain pod.spec.containers.envFrom
kubectl explain pod.spec.volumes.configMap
```
