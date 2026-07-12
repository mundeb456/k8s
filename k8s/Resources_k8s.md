# Kubernetes Resource Requests and Limits

## What are Resource Requests and Limits?

In Kubernetes, **resource requests** and **resource limits** define how much CPU and memory a container needs.

- **Request** = Minimum resources guaranteed to the container.
- **Limit** = Maximum resources the container is allowed to use.

Think of it like booking a hotel room:

- **Request** → "I need at least this much space."
- **Limit** → "I cannot use more than this."

---

# Resource Requests

A **request** tells the Kubernetes scheduler how much CPU and memory to reserve for a container.

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
```

### Meaning

| Resource | Value |
|----------|-------|
| CPU | 500m = 0.5 CPU core |
| Memory | 256Mi = 256 MB |

---

## How Requests Work in the Background

Suppose a node has:

```
CPU: 2 Cores
Memory: 4Gi
```

A pod requests:

```
CPU: 500m
Memory: 256Mi
```

Scheduler checks:

```
Available CPU: 2 Cores
Available Memory: 4Gi

Need:
CPU: 0.5 Core
Memory: 256Mi

Result:
✓ Pod Scheduled
```

Now Kubernetes reserves:

```
Reserved CPU: 500m
Reserved Memory: 256Mi
```

Even if the application actually uses only:

```
CPU: 50m
Memory: 100Mi
```

The scheduler still considers:

```
Reserved CPU = 500m
Reserved Memory = 256Mi
```

This prevents overcommitting the node.

---

# Resource Limits

Limits define the maximum CPU and memory a container can consume.

Example:

```yaml
resources:
  limits:
    cpu: "1"
    memory: "512Mi"
```

Meaning:

| Resource | Maximum |
|----------|---------|
| CPU | 1 Core |
| Memory | 512Mi |

---

# Complete Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx

    resources:
      requests:
        cpu: "250m"
        memory: "128Mi"

      limits:
        cpu: "500m"
        memory: "256Mi"
```

This means:

### Guaranteed

```
CPU = 250m
Memory = 128Mi
```

### Maximum

```
CPU = 500m
Memory = 256Mi
```

---

# How Kubernetes Scheduler Works

Suppose Node1 has:

```
CPU = 4 Cores
Memory = 8Gi
```

## Pod A

```
Request

CPU = 1
Memory = 2Gi

Limit

CPU = 2
Memory = 4Gi
```

Scheduler checks only **requests**.

```
Available CPU = 4

Need = 1

✓ Schedule Pod A
```

Remaining resources:

```
CPU = 3
Memory = 6Gi
```

---

## Pod B

```
Request

CPU = 2
Memory = 3Gi
```

Scheduler checks:

```
Available CPU = 3
Available Memory = 6Gi

✓ Schedule Pod B
```

Remaining:

```
CPU = 1
Memory = 3Gi
```

---

## Pod C

```
Request

CPU = 2
```

Scheduler checks:

```
Need = 2

Available = 1

✗ Cannot Schedule
```

Result:

```
Pod Status: Pending
```

---

# Runtime Behavior

Suppose the pod has:

```
Request CPU = 250m

Limit CPU = 500m
```

## Case 1

Application uses:

```
300m CPU
```

Allowed because:

```
300m < 500m
```

---

## Case 2

Application suddenly needs:

```
700m CPU
```

Result:

```
CPU is throttled to 500m
```

The pod continues running.

---

# Memory Example

Configuration:

```
Request = 256Mi

Limit = 512Mi
```

Application uses:

```
400Mi
```

Allowed.

---

Application uses:

```
600Mi
```

Result:

```
Memory limit exceeded

Container terminated

Status: OOMKilled
```

Unlike CPU, memory cannot be throttled.

---

# Why Scheduler Uses Requests Instead of Limits

Node:

```
CPU = 2 Cores
```

Pods:

```
Pod A Request = 1 CPU

Pod B Request = 1 CPU
```

Actual usage:

```
Pod A = 100m

Pod B = 100m
```

Even though actual usage is small, Kubernetes considers:

```
Reserved CPU = 2 Cores
```

Another pod requesting:

```
500m CPU
```

Cannot be scheduled because all requested CPU is already reserved.

---

# CPU Example

```
Request = 500m

Limit = 1000m
```

Actual CPU usage:

| Usage | Result |
|--------|--------|
| 300m | Allowed |
| 700m | Allowed |
| 1000m | Allowed |
| 1200m | CPU Throttled |

---

# Memory Example

```
Request = 512Mi

Limit = 1Gi
```

Actual Memory:

| Usage | Result |
|--------|--------|
| 700Mi | Allowed |
| 900Mi | Allowed |
| 1Gi | Allowed |
| 1.2Gi | OOMKilled |

---

# Requests vs Limits

| Feature | Request | Limit |
|---------|----------|-------|
| Used by Scheduler | ✅ Yes | ❌ No |
| Guaranteed Resources | ✅ Yes | ❌ No |
| Maximum Usage | ❌ No | ✅ Yes |
| CPU Exceeded | N/A | Throttled |
| Memory Exceeded | N/A | OOMKilled |

---

# Real-World Example

Node Capacity:

```
CPU = 8 Cores

Memory = 16Gi
```

Pods:

| Pod | Request | Limit |
|------|----------|--------|
| Frontend | 500m / 512Mi | 1 CPU / 1Gi |
| Backend | 2 CPU / 4Gi | 4 CPU / 6Gi |
| Database | 4 CPU / 8Gi | 6 CPU / 10Gi |

Reserved by Scheduler:

```
CPU Reserved

0.5 + 2 + 4 = 6.5 CPUs

Remaining = 1.5 CPUs
```

Even if the frontend currently uses only 100m CPU, Kubernetes still reserves 500m because scheduling decisions are based on **requests**, not current usage.

---

# Key Points

- **Requests** determine where a pod can be scheduled.
- **Limits** control the maximum resources a container can consume.
- Scheduler considers **requests only**.
- CPU exceeding its limit is **throttled**.
- Memory exceeding its limit causes the container to be **OOMKilled**.
- Requests reserve resources to ensure predictable application performance.