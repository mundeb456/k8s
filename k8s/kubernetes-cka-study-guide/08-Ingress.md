# Ingress

An **Ingress** manages external HTTP/HTTPS access to Services inside the cluster, providing host- and path-based routing, TLS termination, and a single entry point — instead of one LoadBalancer/NodePort per Service. An Ingress is only a set of routing rules; it does nothing until an **Ingress controller** is running to implement it.

## Ingress vs. Ingress Controller

- **Ingress (object)** — the API resource that declares routing rules (which host/path goes to which Service).
- **Ingress controller** — the actual reverse proxy (e.g. **ingress-nginx**, Traefik, HAProxy, cloud controllers) that watches Ingress objects and configures itself accordingly. Without a controller, an Ingress object has no effect.

The controller itself is typically exposed via a `LoadBalancer` or `NodePort` Service — that is the true external entry point.

```bash
kubectl get pods -n ingress-nginx            # is the controller running?
kubectl get svc  -n ingress-nginx            # how is it exposed?
kubectl get ingressclass                     # available classes
```

## IngressClass

`IngressClass` identifies which controller should serve an Ingress. Reference it via `spec.ingressClassName`. One class can be marked default with the annotation `ingressclass.kubernetes.io/is-default-class: "true"`, used when an Ingress omits `ingressClassName`.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
```

## Host and Path Rules

An Ingress routes on the request **Host** header and URL **path**. Each rule maps a host to one or more paths, and each path points to a backend Service + port.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: shop.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 8080
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin
            port:
              number: 80
```

Note the `apiVersion: networking.k8s.io/v1` and the nested `backend.service.name` / `backend.service.port.number` structure (this changed from older beta versions — a common exam gotcha).

## pathType

Every path must set `pathType`:

- **Prefix** — matches based on URL path prefix split by `/`. `/foo` matches `/foo` and `/foo/bar`, but **not** `/foobar`.
- **Exact** — matches the exact path only (case-sensitive). `/foo` matches only `/foo`.
- **ImplementationSpecific** — matching is up to the controller.

## TLS Termination

Terminate HTTPS at the Ingress by referencing a TLS **Secret** (type `kubernetes.io/tls`) containing the cert and key. The `hosts` in the `tls` block must match the rule hosts.

Create the Secret:

```bash
kubectl create secret tls shop-tls \
  --cert=tls.crt --key=tls.key
```

Reference it:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - shop.example.com
    secretName: shop-tls
  rules:
  - host: shop.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

The referenced Secret must live in the **same namespace** as the Ingress.

## Default Backend

Traffic that matches no rule is sent to the **default backend**. With ingress-nginx this is a controller-level setting, but an Ingress can also declare a `spec.defaultBackend`:

```yaml
spec:
  defaultBackend:
    service:
      name: default-http-backend
      port:
        number: 80
```

If no default backend exists and nothing matches, the controller returns 404.

## Rewrite Annotations (ingress-nginx)

Annotations tune controller behavior. A very common one is URL rewriting, so `/api/users` at the edge becomes `/users` at the backend.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 8080
```

Other useful ingress-nginx annotations:

```yaml
nginx.ingress.kubernetes.io/ssl-redirect: "true"        # force HTTPS
nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"    # talk HTTPS to backend
nginx.ingress.kubernetes.io/rewrite-target: /
```

## Creating Ingress Imperatively

```bash
kubectl create ingress app \
  --rule="shop.example.com/=frontend:80" \
  --rule="shop.example.com/api=api:8080" \
  --class=nginx \
  --dry-run=client -o yaml > ingress.yaml

# With TLS
kubectl create ingress secure \
  --rule="shop.example.com/*=frontend:80,tls=shop-tls" \
  --class=nginx
```

The `--rule` format is `HOST/PATH=SERVICE:PORT[,tls=SECRET]`.

## Inspecting and Troubleshooting

```bash
kubectl get ingress
kubectl get ingress -o wide          # shows hosts, address, ports
kubectl describe ingress <name>      # rules, backends, events, TLS
```

Common problems:

- **No `ADDRESS` on the Ingress**: no controller is running or the class doesn't match any controller. Check `kubectl get pods -n ingress-nginx` and `ingressClassName`.
- **404 from the controller**: host/path didn't match, or the backend Service has no endpoints. Verify `kubectl get endpoints <svc>`.
- **`pathType` missing**: required in `networking.k8s.io/v1`; omission causes rejection.
- **TLS not working**: Secret must be type `kubernetes.io/tls`, in the same namespace, and its `hosts` must match the rule host.
- **Wrong apiVersion / backend structure**: must be `networking.k8s.io/v1` with `backend.service.name` and `backend.service.port.number`.

## Exam Tips

- An Ingress does nothing without a running controller — always confirm the controller and its IngressClass exist first.
- Use `apiVersion: networking.k8s.io/v1`, and remember the `backend.service.name`/`backend.service.port.number` nesting — old examples use a flatter, now-invalid layout.
- `pathType` is mandatory. Know `Prefix` (path segment prefix) vs `Exact`.
- TLS Secret must be type `kubernetes.io/tls`, same namespace, hosts matching the rules — create it with `kubectl create secret tls`.
- Generate manifests with `kubectl create ingress ... --rule=HOST/PATH=SVC:PORT --class=nginx $do` to save time.
- The Ingress backend points to a **Service** (not Pods); if it 404s, check the Service's endpoints.
- Rewrite behavior lives in controller-specific **annotations**, not the spec.

## Key Commands

```bash
kubectl get ingressclass
kubectl create ingress <name> --rule="HOST/PATH=SVC:PORT" --class=nginx --dry-run=client -o yaml
kubectl create secret tls <name> --cert=tls.crt --key=tls.key
kubectl get ingress [-o wide]
kubectl describe ingress <name>
kubectl get pods -n ingress-nginx
kubectl get svc  -n ingress-nginx
kubectl get endpoints <backend-svc>
kubectl explain ingress.spec.rules.http.paths
```
