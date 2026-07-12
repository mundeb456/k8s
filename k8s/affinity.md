
# Kubernetes Affinity & Node Affinity Cheat Sheet (CKA)


## 1. nodeSelector

Use `nodeSelector` when you want to schedule a Pod on a node with an exact label match.

### Example

```yaml
spec:
  nodeSelector:
    color: blue
```

**Result:** The Pod will run only on nodes having the label:

```text
color=blue
```

---

# 2. Node Affinity

Node Affinity is the advanced version of `nodeSelector`.

It supports:

* In
* NotIn
* Exists
* DoesNotExist
* Gt
* Lt

There are two types of Node Affinity:

* `requiredDuringSchedulingIgnoredDuringExecution` (Hard Rule)
* `preferredDuringSchedulingIgnoredDuringExecution` (Soft Rule)

---

## 2.1 Required Node Affinity (Hard Rule)

The Pod **must** satisfy the rule before scheduling.

If no matching node exists, the Pod remains **Pending**.

### Syntax

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: color
            operator: In
            values:
            - blue
```

---

## 2.2 Preferred Node Affinity (Soft Rule)

The scheduler tries to place the Pod on a matching node.

If no matching node is available, it schedules the Pod elsewhere.

### Syntax

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: color
            operator: In
            values:
            - blue
```

---

# 3. Meaning of IgnoredDuringExecution

Suppose the Pod is scheduled on:

```text
node01
color=blue
```

Later someone removes the label:

```bash
kubectl label node node01 color-
```

The Pod **continues running**.

Reason:

`IgnoredDuringExecution` means Kubernetes checks the rule **only during scheduling**, not after the Pod is running.

---

# 4. Operators

## In

```yaml
matchExpressions:
- key: color
  operator: In
  values:
  - blue
```

Matches:

```text
color=blue
```

---

## NotIn

```yaml
matchExpressions:
- key: color
  operator: NotIn
  values:
  - blue
```

Matches every node except:

```text
color=blue
```

---

## Exists

```yaml
matchExpressions:
- key: color
  operator: Exists
```

The label only needs to exist.

Examples:

```text
color=blue
color=red
```

Both match.

---

## DoesNotExist

```yaml
matchExpressions:
- key: color
  operator: DoesNotExist
```

The label **must not** exist.

---

## Gt (Greater Than)

```yaml
matchExpressions:
- key: cpu
  operator: Gt
  values:
  - "4"
```

Matches:

```text
cpu=8
cpu=16
```

---

## Lt (Less Than)

```yaml
matchExpressions:
- key: cpu
  operator: Lt
  values:
  - "8"
```

Matches:

```text
cpu=2
cpu=4
```

---

# 5. Pod Affinity

Used when you want Pods to run **close to other Pods**.

Example:

* Backend Pod should run on the same node as Frontend Pod.

### Syntax

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - frontend
        topologyKey: kubernetes.io/hostname
```

---

# 6. Pod Anti-Affinity

Used when similar Pods should **not** run on the same node.

Example:

Three replicas should be distributed across different nodes.

### Syntax

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - nginx
        topologyKey: kubernetes.io/hostname
```

---

# 7. Preferred Pod Affinity

```yaml
spec:
  affinity:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - frontend
          topologyKey: kubernetes.io/hostname
```

---

# 8. Preferred Pod Anti-Affinity

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - nginx
          topologyKey: kubernetes.io/hostname
```

---

# 9. topologyKey

`topologyKey` defines the level at which affinity rules apply.

### Same Node

```yaml
topologyKey: kubernetes.io/hostname
```

### Same Availability Zone

```yaml
topologyKey: topology.kubernetes.io/zone
```

---

# 10. nodeSelector vs Node Affinity

| Feature            | nodeSelector | Node Affinity |
| ------------------ | ------------ | ------------- |
| Easy to use        | ✅            | ❌             |
| Exact match        | ✅            | ✅             |
| Supports operators | ❌            | ✅             |
| Hard scheduling    | ✅            | ✅             |
| Soft scheduling    | ❌            | ✅             |

---

# 11. Pod Affinity vs Pod Anti-Affinity

| Feature           | Purpose                      |
| ----------------- | ---------------------------- |
| Pod Affinity      | Schedule Pods together       |
| Pod Anti-Affinity | Keep Pods on different nodes |

---

# 12. CKA Cheat Sheet

| Requirement                | Use                                               |
| -------------------------- | ------------------------------------------------- |
| Run Pod on a specific node | `nodeSelector`                                    |
| Mandatory node selection   | `requiredDuringSchedulingIgnoredDuringExecution`  |
| Preferred node selection   | `preferredDuringSchedulingIgnoredDuringExecution` |
| Run Pods together          | `podAffinity`                                     |
| Separate Pods              | `podAntiAffinity`                                 |
| Label must exist           | `Exists`                                          |
| Label must not exist       | `DoesNotExist`                                    |
| Match values               | `In`                                              |
| Exclude values             | `NotIn`                                           |
| Greater than               | `Gt`                                              |
| Less than                  | `Lt`                                              |

---

# 13. Quick Memory Tips

### Node Affinity

> **Which node should my Pod run on?**

### Pod Affinity

> **Which Pod should my Pod run with?**

### Pod Anti-Affinity

> **Which Pod should my Pod stay away from?**

---

# 14. Exam Tips (CKA)

* Use `nodeSelector` for simple label matching.
* Use **Node Affinity** for advanced scheduling rules.
* Remember:

  * `required` = Hard Rule
  * `preferred` = Soft Rule
* `IgnoredDuringExecution` means label changes after scheduling do **not** evict the Pod.
* For Pod Affinity/Anti-Affinity, always remember:

  * `labelSelector`
  * `topologyKey`
