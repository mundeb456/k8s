# Secrets

A **Secret** stores small amounts of sensitive data — passwords, tokens, keys, TLS certificates — so it does not have to live in a Pod spec or a container image. Like ConfigMaps, Secrets are **namespaced** and consumed as environment variables or volume-mounted files, but they receive additional handling: they are only distributed to nodes that run a Pod requiring them, and (when configured) can be encrypted at rest in etcd.

Size limit: **1 MiB** per Secret.

> Important: by default a Secret is **not encrypted**, only **base64-encoded**. Anyone who can `kubectl get secret -o yaml` and run `base64 -d` can read it. Base64 is encoding, not encryption. Real protection comes from RBAC + etcd encryption at rest.

## Secret Types

| Type | Purpose |
|------|---------|
| `Opaque` | Arbitrary user-defined key/value data (the default). |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials for pulling private images (`imagePullSecrets`). |
| `kubernetes.io/tls` | A TLS certificate + private key; keys must be `tls.crt` and `tls.key`. |
| `kubernetes.io/service-account-token` | A ServiceAccount token (largely superseded by short-lived projected tokens in modern clusters, but still creatable). |
| `kubernetes.io/basic-auth` | Basic auth credentials (`username`, `password`). |
| `kubernetes.io/ssh-auth` | SSH private key (`ssh-privatekey`). |

## base64 Encoding

In a Secret's `data` field, all values must be **base64-encoded**. The `stringData` field lets you provide plaintext that Kubernetes encodes for you (see below).

```bash
# Encode
echo -n 'S3cr3t!' | base64            # -n avoids a trailing newline -> UzNjcjN0IQ==

# Decode (to verify)
echo 'UzNjcjN0IQ==' | base64 -d
```

Always use `echo -n` — a trailing newline becomes part of the secret value and causes hard-to-debug auth failures.

## Creating Secrets Imperatively

`kubectl create secret` encodes values for you, so you pass **plaintext**.

```bash
# Generic (Opaque) from literals
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASS='S3cr3t!'

# Generic from files (key = filename, value = contents)
kubectl create secret generic tls-files \
  --from-file=ssh-privatekey=./id_rsa

# TLS secret
kubectl create secret tls my-tls \
  --cert=tls.crt --key=tls.key

# Docker registry secret (for private image pulls)
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=deploy \
  --docker-password='p@ss' \
  --docker-email=ops@example.com
```

Generate YAML without creating (exam-friendly):

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_PASS='S3cr3t!' \
  --dry-run=client -o yaml > secret.yaml
```

## Creating Secrets Declaratively

### Using `data` (values must be base64-encoded)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_USER: YWRtaW4=          # base64 of "admin"
  DB_PASS: UzNjcjN0IQ==      # base64 of "S3cr3t!"
```

### Using `stringData` (plaintext, Kubernetes encodes it)

`stringData` is write-only convenience: you write plaintext, and it is merged into `data` (encoded) when stored. If a key appears in both `stringData` and `data`, `stringData` wins. When you read the object back, values appear under `data` as base64.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_USER: admin
  DB_PASS: S3cr3t!
```

### A TLS Secret declaratively

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-tls
type: kubernetes.io/tls
data:
  tls.crt: <base64 of the certificate>
  tls.key: <base64 of the private key>
```

## Mounting as Environment Variables

Single key with `secretKeyRef`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo user=$DB_USER; sleep 3600"]
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_USER
```

All keys at once with `envFrom` + `secretRef`:

```yaml
    spec:
      containers:
        - name: app
          image: busybox:1.36
          envFrom:
            - secretRef:
                name: db-secret
```

Values are automatically decoded before being placed into the environment — the container sees plaintext.

## Mounting as a Volume

Each key becomes a file (filename = key, contents = decoded value). Files are backed by tmpfs (RAM), so they are never written to node disk.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-vol-demo
spec:
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: secret-vol
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-vol
      secret:
        secretName: db-secret
        defaultMode: 0400          # restrictive file permissions
        # items:                   # optionally project selected keys
        #   - key: DB_PASS
        #     path: db_password
```

Result: `/etc/secrets/DB_USER` and `/etc/secrets/DB_PASS`, each containing the decoded plaintext. Like ConfigMaps, volume-mounted Secrets update when the Secret changes (unless mounted via `subPath`); env-var Secrets do not update live.

## Using a Docker Registry Secret

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  containers:
    - name: app
      image: registry.example.com/team/app:1.0
  imagePullSecrets:
    - name: regcred
```

## Security Notes — Encryption at Rest (brief)

By default Secrets are stored **base64-encoded, not encrypted**, in etcd. Two hardening measures matter for CKA:

1. **RBAC** — restrict who can `get`/`list`/`watch` Secrets. Broad read access effectively means broad secret access.
2. **Encryption at rest** — configure the API server with an `EncryptionConfiguration` so Secrets are encrypted in etcd. This is enabled via the `--encryption-provider-config` flag on `kube-apiserver`.

```yaml
# /etc/kubernetes/enc/enc.yaml  (referenced by kube-apiserver --encryption-provider-config)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:                       # a provider that actually encrypts
          keys:
            - name: key1
              secret: <base64-encoded 32-byte key>
      - identity: {}                  # fallback: no encryption (must not be first if you want encryption)
```

After enabling, re-encrypt existing Secrets so they are stored with the new provider:

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

Additional good practices: mark Secrets `immutable: true` where possible, prefer volume mounts over env vars (env vars can leak via `/proc`, logs, and crash dumps), and use `defaultMode: 0400`.

## Exam Tips

- `data` values must be base64-encoded; `stringData` takes plaintext and encodes it for you. Reading the object back always shows `data` (base64). Know both.
- `kubectl create secret` takes **plaintext** and encodes automatically — do not pre-encode when using the imperative command.
- Always use `echo -n` when hand-encoding, or a trailing newline poisons the value.
- Base64 is **not** encryption. If asked "are Secrets encrypted by default?" the answer is **no** — they are only encoded; encryption at rest requires an `EncryptionConfiguration`.
- TLS secrets require exactly the keys `tls.crt` and `tls.key`; `type: kubernetes.io/tls`.
- Private image pulls need a `kubernetes.io/dockerconfigjson` secret referenced via `imagePullSecrets`.
- `envFrom` + `secretRef` injects all keys as env vars; `valueFrom.secretKeyRef` injects one. Same pattern as ConfigMaps.
- Volume-mounted secrets update on change (not via `subPath`); env-var secrets are set at start only.
- Fast exam path: `kubectl create secret generic ... --dry-run=client -o yaml`.

## Key Commands

```bash
# Create imperatively (values are plaintext; kubectl encodes)
kubectl create secret generic db-secret --from-literal=DB_USER=admin --from-literal=DB_PASS='S3cr3t!'
kubectl create secret tls my-tls --cert=tls.crt --key=tls.key
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com --docker-username=deploy \
  --docker-password='p@ss' --docker-email=ops@example.com

# Generate YAML to edit
kubectl create secret generic db-secret --from-literal=DB_PASS='S3cr3t!' --dry-run=client -o yaml > secret.yaml

# base64 helpers
echo -n 'S3cr3t!' | base64
echo 'UzNjcjN0IQ==' | base64 -d

# Inspect (values are base64 in output)
kubectl get secret
kubectl get secret db-secret -o yaml
kubectl describe secret db-secret          # shows keys and sizes, not values

# Decode a specific key
kubectl get secret db-secret -o jsonpath='{.data.DB_PASS}' | base64 -d

# Verify consumption
kubectl exec secret-env-demo -- env | grep DB_
kubectl exec secret-vol-demo -- cat /etc/secrets/DB_PASS

# Field reference
kubectl explain pod.spec.containers.env.valueFrom.secretKeyRef
kubectl explain pod.spec.volumes.secret
```
