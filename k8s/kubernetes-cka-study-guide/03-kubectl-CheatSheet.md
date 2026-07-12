# kubectl Cheat Sheet

`kubectl` is the primary command-line tool for interacting with the Kubernetes API server. On the CKA exam, fluency with `kubectl` is the difference between finishing and running out of time. This guide covers the commands, flags, and speed tricks you must know cold.

## Basics: How kubectl Works

`kubectl` reads a **kubeconfig** file (default `~/.kube/config`, overridable with `$KUBECONFIG` or `--kubeconfig`) to know which cluster to talk to, which user credentials to present, and which namespace to default to. Every command builds an HTTP request to the API server.

```bash
kubectl version                      # client + server version
kubectl cluster-info                 # control plane endpoints
kubectl api-resources                # all resource types + short names + apiVersion
kubectl api-versions                 # all API groups/versions enabled
kubectl get --raw /healthz           # hit an API endpoint directly
```

`kubectl api-resources` is invaluable — it tells you the shortname (`po`, `deploy`, `svc`), whether a resource is namespaced, and the `KIND`/`APIVERSION` you need in YAML.

## Contexts and kubeconfig

A **context** bundles a cluster + user + namespace. The exam environment may require you to switch clusters between tasks, so know these by heart.

```bash
kubectl config view                              # show merged kubeconfig
kubectl config view --minify                     # show only the current context
kubectl config get-contexts                      # list all contexts (* marks current)
kubectl config current-context                   # print current context name
kubectl config use-context <context-name>        # SWITCH context (do this first each task!)
kubectl config set-context --current --namespace=<ns>   # set default namespace for context
kubectl config rename-context <old> <new>
```

```bash
# Point kubectl at a specific kubeconfig for one command
kubectl --kubeconfig /root/cluster1.conf get nodes

# Persist across the shell session
export KUBECONFIG=/root/cluster1.conf
```

The kubeconfig has three top-level lists: `clusters`, `users`, and `contexts`. A context references one cluster and one user by name:

```yaml
apiVersion: v1
kind: Config
current-context: dev-context
clusters:
- name: dev-cluster
  cluster:
    server: https://10.0.0.1:6443
    certificate-authority: /etc/kubernetes/pki/ca.crt
users:
- name: dev-user
  user:
    client-certificate: /etc/kubernetes/pki/dev.crt
    client-key: /etc/kubernetes/pki/dev.key
contexts:
- name: dev-context
  context:
    cluster: dev-cluster
    user: dev-user
    namespace: development
```

## Imperative vs. Declarative

- **Imperative**: You tell Kubernetes exactly what to do, step by step (`kubectl create`, `kubectl run`, `kubectl expose`, `kubectl scale`, `kubectl set image`). Fast for the exam.
- **Declarative**: You describe the desired end state in a file and let Kubernetes reconcile (`kubectl apply -f`). Idempotent and repeatable.

Best exam strategy: use imperative commands with `--dry-run=client -o yaml` to **generate** a manifest, then edit and `apply` it. This is faster than writing YAML from scratch and less error-prone than pure imperative.

## Core Verbs

```bash
kubectl get pods                          # list pods in current namespace
kubectl get pods -A                       # all namespaces (alias for --all-namespaces)
kubectl get pods -n kube-system           # a specific namespace
kubectl get pod,svc,deploy                # multiple resource types at once
kubectl get pods -w                       # watch for changes
kubectl get pods --show-labels            # display labels column
kubectl get events --sort-by=.metadata.creationTimestamp   # recent events, ordered

kubectl describe pod <name>               # full detail incl. events (great for debugging)
kubectl create -f pod.yaml                # create; fails if it already exists
kubectl apply -f pod.yaml                 # create or update declaratively
kubectl replace -f pod.yaml               # replace an existing object entirely
kubectl replace --force -f pod.yaml       # delete + recreate (for immutable fields)
kubectl edit deploy <name>                # open live object in $EDITOR, apply on save
kubectl delete pod <name>                 # delete a resource
kubectl delete -f pod.yaml                # delete everything defined in a file
kubectl delete pod <name> --grace-period=0 --force   # force-delete a stuck pod
```

`create` vs `apply`: `create` is imperative and errors on an existing object; `apply` is declarative, tracks the last-applied config in an annotation, and can be run repeatedly.

## Output Formats

```bash
kubectl get pods -o wide                  # extra columns (node, IP)
kubectl get pod <name> -o yaml            # full YAML (includes status, defaults)
kubectl get pod <name> -o json            # full JSON
kubectl get pods -o name                  # just resource/name

# JSONPath — extract specific fields
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
kubectl get pod <name> -o jsonpath='{.spec.containers[0].image}'

# Custom columns
kubectl get pods -o custom-columns='NAME:.metadata.name,IMAGE:.spec.containers[0].image'
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase

# Sort
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pv --sort-by=.spec.capacity.storage
```

## Generating YAML with --dry-run

`--dry-run=client` builds the object locally without sending it to the server. Combine with `-o yaml` to print a manifest you can save and edit.

```bash
# Pod
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Deployment
kubectl create deployment web --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml

# Service (ClusterIP) exposing a deployment
kubectl expose deployment web --port=80 --target-port=8080 --dry-run=client -o yaml > svc.yaml

# Service (NodePort) via create
kubectl create service nodeport web --tcp=80:8080 --dry-run=client -o yaml

# ConfigMap / Secret
kubectl create configmap app-cfg --from-literal=KEY=value --dry-run=client -o yaml
kubectl create secret generic app-sec --from-literal=PASS=s3cr3t --dry-run=client -o yaml

# Job / CronJob
kubectl create job pi --image=perl --dry-run=client -o yaml -- perl -Mbignum=bpi -wle 'print bpi(200)'
kubectl create cronjob report --image=busybox --schedule="*/5 * * * *" --dry-run=client -o yaml -- date
```

`--dry-run=server` sends the object to the API server for validation (admission checks, defaulting) without persisting — useful to catch schema errors.

## kubectl explain

Discover fields and their meaning without leaving the terminal — a lifesaver when you forget YAML structure.

```bash
kubectl explain pod                       # top-level fields
kubectl explain pod.spec                   # drill down
kubectl explain pod.spec.containers        # container fields
kubectl explain pod.spec.containers.livenessProbe
kubectl explain deployment.spec.strategy --recursive    # entire subtree
```

## Labels and Selectors

```bash
kubectl get pods -l app=web                       # equality selector
kubectl get pods -l 'env in (prod,staging)'       # set-based selector
kubectl get pods -l app=web,tier=frontend         # AND of two labels
kubectl get pods -l '!canary'                      # pods without the 'canary' label

kubectl label pod <name> env=prod                  # add a label
kubectl label pod <name> env=dev --overwrite       # change a label
kubectl label pod <name> env-                      # remove the 'env' label

kubectl annotate pod <name> description="my pod"   # add an annotation
```

## Logs, Exec, Cp, Port-Forward

```bash
kubectl logs <pod>                        # container logs
kubectl logs <pod> -c <container>         # multi-container pod, pick one
kubectl logs <pod> -f                     # follow (stream)
kubectl logs <pod> --previous             # logs from previous (crashed) container
kubectl logs -l app=web --tail=50         # logs from all pods matching a label

kubectl exec <pod> -- ls /app             # run a command
kubectl exec -it <pod> -- /bin/sh         # interactive shell
kubectl exec -it <pod> -c <container> -- bash

kubectl cp <pod>:/etc/config.txt ./config.txt      # copy out of a pod
kubectl cp ./local.txt <pod>:/tmp/local.txt        # copy into a pod

kubectl port-forward pod/<pod> 8080:80             # local 8080 -> pod 80
kubectl port-forward svc/<svc> 8080:80             # forward to a service
```

## kubectl top (Metrics)

Requires the **metrics-server** add-on to be installed.

```bash
kubectl top nodes                         # CPU/memory per node
kubectl top pods                          # per pod in current namespace
kubectl top pods -A --sort-by=memory      # highest memory across cluster
kubectl top pod <name> --containers       # per-container breakdown
```

## Scaling, Rollout, Autoscale

```bash
kubectl scale deployment web --replicas=5
kubectl scale --replicas=3 -f deploy.yaml
kubectl autoscale deployment web --min=2 --max=10 --cpu-percent=80

kubectl rollout status deployment web
kubectl rollout history deployment web
kubectl rollout undo deployment web                # roll back one revision
kubectl rollout undo deployment web --to-revision=2
kubectl rollout restart deployment web             # restart all pods (new rollout)
kubectl rollout pause deployment web
kubectl rollout resume deployment web

kubectl set image deployment web nginx=nginx:1.27
```

## Exam Speed Tricks (Set These Up First)

At the start of the exam, configure aliases and a shell function. This alone can save several minutes.

```bash
alias k=kubectl
export do='--dry-run=client -o yaml'      # "dry output"
export now='--force --grace-period=0'      # fast delete

# Enable bash completion + make it work with the alias
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
```

Usage after setup:

```bash
k run nginx --image=nginx $do > pod.yaml     # expands the $do dry-run flags
k delete pod nginx $now                       # instant delete
```

Add these to `~/.bashrc` so they persist in new terminals (the exam allows this).

### Vim Setup for YAML

YAML is whitespace-sensitive; set up `~/.vimrc` so indentation and tabs behave. Do this before writing any manifests.

```vim
set expandtab       " tabs become spaces
set tabstop=2       " a tab is 2 spaces wide
set shiftwidth=2    " autoindent uses 2 spaces
set number          " show line numbers
```

Handy vim commands during the exam: `:set paste` before pasting to avoid cascading indentation, and `Esc` then `:x` to save+quit.

## Namespaces Quick Reference

```bash
kubectl create namespace dev
kubectl get ns
kubectl config set-context --current --namespace=dev   # stop typing -n dev
kubectl delete namespace dev                            # deletes everything in it
```

## Exam Tips

- **First action of every task: `kubectl config use-context <ctx>`** if the question names a cluster/context. Solving a task in the wrong cluster earns zero points.
- Never hand-write YAML you can generate. `k run`/`k create ... $do > file.yaml` then edit.
- Use `kubectl explain <resource>.<field> --recursive` when you forget nesting; it is faster than the docs.
- `kubectl describe` and `kubectl get events --sort-by=.metadata.creationTimestamp` are your first debugging moves — check the events at the bottom.
- Set `export do='--dry-run=client -o yaml'` and `alias k=kubectl` immediately; enable completion for the alias.
- For a stuck/terminating pod, use `--force --grace-period=0`.
- Prefer `kubectl edit` for a quick single-field change; use `kubectl replace --force -f` when changing immutable fields.
- You are allowed to browse kubernetes.io docs during the exam — bookmark the kubectl cheat sheet and the concepts pages, but relying on `explain` is usually faster.
- Set your namespace on the context so you stop appending `-n`.

## Key Commands

```bash
kubectl config get-contexts
kubectl config use-context <ctx>
kubectl config set-context --current --namespace=<ns>
kubectl get <res> -o wide|yaml|json
kubectl get <res> -o jsonpath='{...}'
kubectl get <res> -o custom-columns='NAME:.metadata.name'
kubectl run <name> --image=<img> --dry-run=client -o yaml
kubectl create deployment <name> --image=<img> --dry-run=client -o yaml
kubectl explain <res>.<field> --recursive
kubectl describe <res> <name>
kubectl logs <pod> [-c <ctr>] [-f] [--previous]
kubectl exec -it <pod> -- sh
kubectl cp <pod>:/path ./path
kubectl port-forward svc/<svc> 8080:80
kubectl top nodes|pods
kubectl scale deployment <name> --replicas=<n>
kubectl rollout status|history|undo|restart deployment <name>
kubectl set image deployment <name> <ctr>=<img>
alias k=kubectl; export do='--dry-run=client -o yaml'
```
