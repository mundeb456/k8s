# Kubernetes Affinity & Node Selector Cheat Sheet (CKA)

खाली CKA परीक्षेसाठी लागणारे **सर्व महत्त्वाचे Affinity आणि Node Selector syntax** एकाच ठिकाणी दिले आहेत.

---

# 1. nodeSelector

```yaml
spec:
  nodeSelector:
    color: blue
```

✅ **Meaning**

फक्त `color=blue` असलेल्या node वर Pod schedule होईल.

---

# 2. Node Affinity (Hard Rule)

**requiredDuringSchedulingIgnoredDuringExecution**

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

spec:
    affinity:
        nodeAffinity:
            requiredDuringSchedulingIgnoreDuringExecution:
                nodeSelectorTerms:
                    - matchExpressions:
                        - key: color
                          operator: In
                          values:
                          - red
```

✅ **Meaning**

जर `color=blue` node नसेल तर Pod **Pending** राहील.

---

# 3. Node Affinity (Soft Rule)

**preferredDuringSchedulingIgnoredDuringExecution**

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

spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoreDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: color
            operator: In
            values:
            - red
```

✅ **Meaning**

आधी blue node वर प्रयत्न करेल. नाही मिळाला तर दुसऱ्या node वर schedule होईल.

---

# 4. Operator: In

```yaml
matchExpressions:
- key: color
  operator: In
  values:
  - blue
```

### Meaning

```text
color=blue
```

---

# 5. Operator: NotIn

```yaml
matchExpressions:
- key: color
  operator: NotIn
  values:
  - blue
```

### Meaning

```text
color != blue
```

---

# 6. Operator: Exists

```yaml
matchExpressions:
- key: color
  operator: Exists
```

### Meaning

```text
color label असलाच पाहिजे.
```

---

# 7. Operator: DoesNotExist

```yaml
matchExpressions:
- key: color
  operator: DoesNotExist
```

### Meaning

```text
color label नसावा.
```

---

# 8. Operator: Gt (Greater Than)

```yaml
matchExpressions:
- key: cpu
  operator: Gt
  values:
  - "4"
```

### Matches

```text
cpu=8
cpu=16
```

---

# 9. Operator: Lt (Less Than)

```yaml
matchExpressions:
- key: cpu
  operator: Lt
  values:
  - "8"
```

### Matches

```text
cpu=2
cpu=4
```

---

# 10. Pod Affinity

एका Pod ला दुसऱ्या Pod च्या **जवळ (same topology)** schedule करायचे.

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

### Meaning

```text
frontend Pod ज्या node वर आहे,
हा Pod पण त्याच node वर जाईल.
```

---

# 11. Pod Anti-Affinity

एका node वर दोन सारखे Pods येऊ नयेत.

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

### Meaning

```text
nginx Pods वेगवेगळ्या nodes वर जातील.
```

---

# 12. Soft Pod Affinity

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

# 13. Soft Pod Anti-Affinity

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

# topologyKey म्हणजे काय?

हे Kubernetes ला सांगते **कशाच्या आधारावर Pods जवळ किंवा दूर ठेवायचे**.

### Node Level

```yaml
topologyKey: kubernetes.io/hostname
```

**Meaning**

> Node level वर scheduling करा.

### Availability Zone Level

```yaml
topologyKey: topology.kubernetes.io/zone
```

**Meaning**

> Availability Zone (AZ) level वर scheduling करा.

---

# CKA Cheat Sheet

| Requirement                     | Syntax                                            |
| ------------------------------- | ------------------------------------------------- |
| Specific node                   | `nodeSelector`                                    |
| Hard node rule                  | `requiredDuringSchedulingIgnoredDuringExecution`  |
| Soft node rule                  | `preferredDuringSchedulingIgnoredDuringExecution` |
| Same node as another Pod        | `podAffinity`                                     |
| Different node from another Pod | `podAntiAffinity`                                 |
| Node label exists               | `Exists`                                          |
| Node label doesn't exist        | `DoesNotExist`                                    |
| Specific values                 | `In`                                              |
| Exclude values                  | `NotIn`                                           |
| Greater than                    | `Gt`                                              |
| Less than                       | `Lt`                                              |

---

# CKA साठी लक्षात ठेवण्यासारखे

* **Node Affinity** मध्ये `nodeSelectorTerms` आणि `matchExpressions` वापरले जातात.
* **Pod Affinity / Pod Anti-Affinity** मध्ये `labelSelector`, `matchExpressions`, आणि `topologyKey` वापरले जातात.
* **Hard Rule** = `requiredDuringSchedulingIgnoredDuringExecution`
* **Soft Rule** = `preferredDuringSchedulingIgnoredDuringExecution`
* Soft Rule मध्ये `weight` आणि `preference` (Node Affinity) किंवा `podAffinityTerm` (Pod Affinity/Anti-Affinity) वापरले जाते.

---

# Quick Revision

| Feature                  | Use                                                |
| ------------------------ | -------------------------------------------------- |
| `nodeSelector`           | Simple label matching                              |
| `Node Affinity`          | Advanced node scheduling                           |
| `Pod Affinity`           | Pods together                                      |
| `Pod Anti-Affinity`      | Pods on different nodes                            |
| `required...`            | Mandatory                                          |
| `preferred...`           | Best effort                                        |
| `IgnoredDuringExecution` | Label changes after scheduling won't evict the Pod |
