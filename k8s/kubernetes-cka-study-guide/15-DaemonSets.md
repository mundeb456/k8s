# DaemonSets

A **DaemonSet** ensures that **all (or some) nodes run a copy of a Pod**. As nodes are added to the cluster, Pods are automatically added to them. As nodes are removed, those Pods are garbage collected. Deleting a DaemonSet cleans up the Pods it created.

DaemonSets are ideal for **node-level agents** — one Pod per node — rather than a fixed replica count.

---

## Purpose and Behavior

Unlike a Deployment (which schedules N replicas anywhere) or a StatefulSet (ordered, stateful), a DaemonSet's replica count is **implicitly the number of matching nodes**. The DaemonSet controller creates exactly **one Pod per eligible node**.

Key properties:

- **One Pod per node** (by default, on every eligible node).
- **Automatic scheduling on new nodes** — when a node joins, its Pod appears with no manual action.
- **Automatic cleanup** — when a node is removed, its Pod is removed.
- DaemonSet Pods are scheduled by the **default scheduler** (using node affinity injected by the controller), and they can also run on nodes that a normal workload could not, via tolerations.

---

## Common Use Cases

- **Log collection:** Fluentd, Fluent Bit, Filebeat, Vector — tail logs from every node.
- **Monitoring / metrics:** Prometheus `node-exporter`, Datadog agent, cAdvisor.
- **Networking (CNI):** Calico, Cilium, Flannel, Weave — network plugin pods run on every node.
- **kube-proxy:** The core cluster component that programs iptables/IPVS rules is itself run as a DaemonSet in kubeadm clusters.
- **Storage daemons:** Ceph, GlusterFS node agents.
- **Security agents:** Falco, node compliance scanners.

---

## Targeting Specific Nodes

By default a DaemonSet runs on **every** eligible node. To restrict it to a subset, use `nodeSelector`, node affinity, and/or tolerations in the Pod template.

### nodeSelector

Runs the Pod only on nodes with a matching label.

```yaml
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd
        node-role: storage
```

Label a node so it matches:

```bash
kubectl label node worker-1 disktype=ssd
```

### Node Affinity

More expressive than `nodeSelector`. Use `requiredDuringSchedulingIgnoredDuringExecution` to hard-restrict placement.

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/os
                    operator: In
                    values:
                      - linux
```

### Tolerations

DaemonSets frequently need to run on **tainted** nodes (e.g., control-plane nodes, or nodes with special taints). Tolerations let the Pod tolerate those taints. The DaemonSet controller automatically adds several tolerations (e.g., `node.kubernetes.io/not-ready`, `node.kubernetes.io/unreachable`, `node.kubernetes.io/disk-pressure`) so agents keep running even when a node is unhealthy.

To also run on control-plane nodes:

```yaml
spec:
  template:
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
```

---

## Update Strategy

`spec.updateStrategy.type`:

### RollingUpdate (default)

After a template change, old Pods are killed and new Pods created in a controlled fashion.

- `maxUnavailable` — how many Pods can be unavailable during the update (number or percent, default `1`).
- `maxSurge` — (v1.22+) how many extra Pods may be created above the desired count (number or percent, default `0`). Cannot be used together with `maxUnavailable` set to 0 improperly; typically use one or the other.

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      # maxSurge: 0
```

### OnDelete

New Pods are only created when you **manually delete** the old DaemonSet Pods. Gives full control over rollout timing.

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

---

## Full YAML Example

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-logging
  namespace: kube-system
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      # run on control-plane nodes too
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: fluentd
          image: quay.io/fluentd_elasticsearch/fluentd:v4.7.0
          resources:
            requests:
              cpu: 100m
              memory: 200Mi
            limits:
              memory: 400Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
```

---

## Common Operations

```bash
# Create
kubectl apply -f daemonset.yaml

# List DaemonSets (note DESIRED/CURRENT/READY = node count)
kubectl get daemonset            # or: kubectl get ds
kubectl get ds -n kube-system

# See which node each Pod landed on
kubectl get pods -l app=fluentd -o wide

# Roll out an update and watch
kubectl rollout status ds/fluentd-logging -n kube-system

# Roll back
kubectl rollout undo ds/fluentd-logging -n kube-system

# Restart (recreate all Pods)
kubectl rollout restart ds/fluentd-logging -n kube-system
```

---

## Exam Tips

- **No `replicas` field.** The count equals the number of eligible nodes. If you see a `replicas` field it is not a DaemonSet.
- **`kubectl get ds`** columns: `DESIRED CURRENT READY UP-TO-DATE AVAILABLE`. DESIRED = number of nodes that should run the Pod. If DESIRED does not equal your node count, check `nodeSelector`/affinity/taints.
- **Tolerations for control-plane:** to run an agent on control-plane nodes, tolerate `node-role.kubernetes.io/control-plane:NoSchedule`. This is a very common exam scenario (e.g., "deploy monitoring on all nodes including the master").
- **kube-proxy and CNI plugins are DaemonSets** — good to recognize when inspecting `kube-system`.
- The DaemonSet controller auto-adds tolerations for node conditions (not-ready, unreachable, memory/disk/PID pressure) so agents survive node problems.
- **`nodeSelector` requires the node to be labeled.** If a DaemonSet shows DESIRED=0, likely no node matches the selector — label a node with `kubectl label node <node> key=value`.
- Default update strategy is **RollingUpdate** with `maxUnavailable: 1`. Use **OnDelete** for manual control.
- Use `kubectl rollout restart ds/<name>` to force-recreate all Pods (e.g., after a ConfigMap change).

---

## Key Commands

```bash
kubectl apply -f ds.yaml                              # create/update
kubectl get ds -A                                     # list all daemonsets
kubectl describe ds fluentd-logging -n kube-system    # detail + events
kubectl get pods -l app=fluentd -o wide               # pod-to-node mapping
kubectl rollout status ds/<name>                      # update progress
kubectl rollout undo ds/<name>                        # roll back
kubectl rollout restart ds/<name>                     # recreate all pods
kubectl label node <node> disktype=ssd                # label node for selector
```
