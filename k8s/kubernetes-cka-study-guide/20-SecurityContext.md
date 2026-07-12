# Security Contexts

A **SecurityContext** defines privilege and access control settings for a Pod
or Container. It controls things like the user/group a process runs as, Linux
capabilities, filesystem permissions, and kernel security modules (seccomp,
AppArmor, SELinux).

Security settings can be applied at two levels:

- **Pod level** (`spec.securityContext`) — applies to all containers in the Pod
  and, for some fields, to mounted volumes.
- **Container level** (`spec.containers[].securityContext`) — applies only to
  that container.

When a setting exists at both levels, the **container-level value overrides the
Pod-level value**.

---

## Pod-level vs Container-level

| Field | Pod level | Container level |
|-------|-----------|-----------------|
| `runAsUser` | Yes | Yes (overrides) |
| `runAsGroup` | Yes | Yes (overrides) |
| `runAsNonRoot` | Yes | Yes (overrides) |
| `fsGroup` | Yes (Pod only) | No |
| `fsGroupChangePolicy` | Yes (Pod only) | No |
| `supplementalGroups` | Yes (Pod only) | No |
| `seLinuxOptions` | Yes | Yes |
| `seccompProfile` | Yes | Yes |
| `privileged` | No | Yes (container only) |
| `allowPrivilegeEscalation` | No | Yes (container only) |
| `capabilities` | No | Yes (container only) |
| `readOnlyRootFilesystem` | No | Yes (container only) |

Key point for the exam: **`privileged`, `capabilities`,
`allowPrivilegeEscalation`, and `readOnlyRootFilesystem` are container-only**.
`fsGroup` and `supplementalGroups` are Pod-only.

---

## Core identity fields

### runAsUser / runAsGroup

- `runAsUser` — the UID all processes in the container run as.
- `runAsGroup` — the primary GID for all processes. If unset, the primary GID
  is 0 (root group).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: id-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

Verify inside the container:

```bash
kubectl exec -it id-demo -- id
# uid=1000 gid=3000
```

### runAsNonRoot

Boolean gate that forces the kubelet to **reject the container at startup if it
would run as UID 0**. It does not pick a UID for you — it only validates.

```yaml
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
```

If the image tries to run as root and `runAsNonRoot: true` is set, the Pod fails
with `CreateContainerConfigError`.

---

## fsGroup

`fsGroup` is a Pod-level field. Kubernetes changes the group ownership of mounted
volumes to this GID and adds it as a supplemental group to every container
process. This lets non-root containers write to a shared volume.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fsgroup-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      emptyDir: {}
```

```bash
kubectl exec -it fsgroup-demo -- sh -c 'touch /data/file && ls -l /data'
# file is group-owned by GID 2000
```

`fsGroupChangePolicy` controls when the recursive `chown` happens:

- `Always` (default) — always change ownership/permissions on mount.
- `OnRootMismatch` — only change if the top-level dir's ownership differs
  (faster for large volumes).

---

## privileged

A **privileged** container has essentially all capabilities of the host and can
access host devices. This effectively disables container isolation.

```yaml
      securityContext:
        privileged: true
```

Only use for system-level workloads (e.g. CNI/CSI agents). It is
container-level only.

---

## allowPrivilegeEscalation

Controls whether a process can gain **more** privileges than its parent
(the `no_new_privs` flag / setuid binaries). Should almost always be `false`
for hardened workloads.

```yaml
      securityContext:
        allowPrivilegeEscalation: false
```

Note: it is implicitly `true` when `privileged: true` or when the container adds
`CAP_SYS_ADMIN`.

---

## Linux capabilities (add / drop)

Instead of full root, grant only the specific capabilities a process needs.
Capabilities are container-level. Names are given **without the `CAP_` prefix**.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: caps-demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        capabilities:
          drop:
            - ALL
          add:
            - NET_ADMIN
            - SYS_TIME
```

Best practice: `drop: ["ALL"]` then `add` only what you need.

```bash
kubectl exec -it caps-demo -- sh -c 'grep Cap /proc/1/status'
```

---

## readOnlyRootFilesystem

Mounts the container's root filesystem read-only. Any writable paths must be
provided via volumes (e.g. `emptyDir`). This blocks tampering with binaries and
config at runtime.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readonly-demo
spec:
  containers:
    - name: app
      image: nginx:1.27
      securityContext:
        readOnlyRootFilesystem: true
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
  volumes:
    - name: tmp
      emptyDir: {}
    - name: cache
      emptyDir: {}
    - name: run
      emptyDir: {}
```

---

## seccompProfile

Seccomp filters the syscalls a container may make. Since Kubernetes 1.19+ the
recommended default is `RuntimeDefault` (the container runtime's profile).

```yaml
      securityContext:
        seccompProfile:
          type: RuntimeDefault
```

`type` values:

- `RuntimeDefault` — use the container runtime's default profile.
- `Unconfined` — no filtering (least secure).
- `Localhost` — use a custom profile file; requires
  `localhostProfile: profiles/my-profile.json` (relative to the kubelet's
  seccomp root, usually `/var/lib/kubelet/seccomp`).

```yaml
      securityContext:
        seccompProfile:
          type: Localhost
          localhostProfile: profiles/audit.json
```

---

## Full example (Pod + container combined)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    runAsNonRoot: true
    fsGroup: 2000
    fsGroupChangePolicy: OnRootMismatch
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginx:1.27
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        privileged: false
        capabilities:
          drop:
            - ALL
          add:
            - NET_BIND_SERVICE
        seccompProfile:
          type: RuntimeDefault
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
      ports:
        - containerPort: 80
  volumes:
    - name: tmp
      emptyDir: {}
    - name: cache
      emptyDir: {}
    - name: run
      emptyDir: {}
```

---

## Exam Tips

- Remember which fields are **Pod-only** (`fsGroup`, `supplementalGroups`,
  `fsGroupChangePolicy`) and which are **container-only** (`privileged`,
  `capabilities`, `allowPrivilegeEscalation`, `readOnlyRootFilesystem`).
- Container securityContext **overrides** Pod securityContext for overlapping
  fields.
- Capability names drop the `CAP_` prefix in YAML (`NET_ADMIN`, not
  `CAP_NET_ADMIN`).
- `runAsNonRoot: true` only **validates** — it does not set a UID. Pair it with
  `runAsUser`.
- If the task says "must not run as root" or "least privilege," think
  `runAsNonRoot`, `drop: [ALL]`, `allowPrivilegeEscalation: false`,
  `readOnlyRootFilesystem: true`.
- Use `kubectl exec <pod> -- id` and `-- whoami` to verify identity quickly.
- `kubectl explain pod.spec.securityContext` and
  `kubectl explain pod.spec.containers.securityContext` are your friends when
  you forget a field.

---

## Key Commands

```bash
# See fields available at each level
kubectl explain pod.spec.securityContext
kubectl explain pod.spec.containers.securityContext

# Verify the effective user/group inside a container
kubectl exec -it <pod> -- id
kubectl exec -it <pod> -- whoami

# Check granted capabilities
kubectl exec -it <pod> -- sh -c 'grep Cap /proc/1/status'

# Inspect why a hardened pod failed to start
kubectl describe pod <pod>
kubectl get pod <pod> -o yaml | grep -A20 securityContext
```
