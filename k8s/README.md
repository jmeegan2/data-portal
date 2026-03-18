# K8s Data Portal Infrastructure

## Concepts

### What is Kubernetes (K8s)?

Kubernetes is a container orchestration platform. It takes your Docker containers and manages running them — handling restarts, networking, scaling, and resource allocation. Instead of manually running `docker run`, you declare what you want in YAML files and Kubernetes makes it happen.

### What is k3d?

k3d is a tool that runs k3s (a lightweight Kubernetes distribution) inside Docker containers on your local machine. It gives you a real Kubernetes cluster without needing cloud infrastructure or heavy setup. Our cluster is named `data-portal`.

### What is a YAML Manifest?

A manifest is a YAML file that describes a Kubernetes resource — what kind it is, what it's named, and how it should be configured. You hand them to the cluster with `kubectl apply -f <file>.yaml` and Kubernetes creates or updates the resource to match what you declared. It's the K8s equivalent of `docker-compose.yml`, but each resource gets its own file.

### Key Resource Types

- **Pod** — the smallest unit in K8s. A wrapper around one or more containers. On its own it's ephemeral — if it dies, nobody brings it back.
- **Deployment** — manages pods. Keeps them running, restarts them if they crash, handles rolling updates when you change config or images.
- **Service** — gives a pod a stable network identity. Pods get new names every restart, but the Service always knows where the current pod is.
- **PersistentVolumeClaim (PVC)** — a request for disk storage that survives pod restarts. The K8s equivalent of a Docker volume.
- **ConfigMap** — key-value store for non-sensitive configuration (database host, port, etc.).
- **Secret** — like a ConfigMap but for sensitive data (passwords, credentials). Base64-encoded at rest, can be encrypted in production.

## Folder Structure

```
k8s/
  kustomization.yaml       # Kustomize entrypoint — lists all resources + configMapGenerators
  README.md
  config/
    dart-db-secret.yaml    # Secret — DB password
    test-scripts-configmap.yaml
  nginx/
    default.conf           # nginx server config — reverse proxy + basic auth
    deployment.yaml        # Deployment — runs the nginx pod
    service.yaml           # Service — ClusterIP on port 80 (ALB target)
    htpasswd-secret.yaml   # Secret — basic auth credentials
  rstudio/
    deployment.yaml        # Deployment — runs the RStudio pod
    service.yaml           # Service — stable network endpoint for RStudio
    pvc.yaml               # PVC — 5GB persistent storage at /home/rstudio
  vscode/
    deployment.yaml        # Deployment — runs the VS Code (code-server) pod
    service.yaml           # Service — stable network endpoint for VS Code
    pvc.yaml               # PVC — 5GB persistent storage at /home/coder
```

## How It All Fits Together

### The Flow

```
User (browser)
  → AWS ALB (ingress / TLS termination)
    → nginx Service (ClusterIP:80)
      → nginx Pod
        → basic auth check (.htpasswd)
        → /            → serves landing page (static/index.html)
        → /vscode/     → proxy_pass → vscode Service:8080 → VS Code pod
        → /rstudio/    → proxy_pass → rstudio Service:8787 → RStudio pod
                          → both read DB_HOST, DB_PASSWORD, etc. from environment
                            → connect to DART database
```

1. The **AWS ALB** handles ingress and routes traffic to the **nginx** Service
2. **nginx** enforces basic auth, serves the landing page at `/`, and reverse proxies `/vscode/` and `/rstudio/` to their respective Services
3. Each **Deployment** creates and manages the pod running its app (nginx, RStudio, VS Code)
4. Each **Service** gives a pod a stable network identity so nginx can proxy to them by name
5. The app pods get database connection info from the **ConfigMap** (via `configMapGenerator`) and **Secret** via environment variables
6. **PVCs** keep user files alive across pod restarts

### Environment Variables

Database config is managed through Kustomize's `configMapGenerator` (inline literals in `kustomization.yaml`) and a Secret:

| Variable       | K8s Resource       | Source                               | Why                          |
|----------------|--------------------|--------------------------------------|------------------------------|
| `DB_HOST`      | ConfigMap (generated) | `kustomization.yaml` → literals   | Not sensitive                |
| `DB_PORT`      | ConfigMap (generated) | `kustomization.yaml` → literals   | Not sensitive                |
| `DB_NAME`      | ConfigMap (generated) | `kustomization.yaml` → literals   | Not sensitive                |
| `DB_USER`      | ConfigMap (generated) | `kustomization.yaml` → literals   | Not sensitive                |
| `DB_PASSWORD`  | Secret             | `config/dart-db-secret.yaml`         | Credential — keep locked down|

The Deployments reference these via `valueFrom` in the `env:` block:

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: dart-db-config       # name of the ConfigMap resource
        key: DB_HOST               # key inside it
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: dart-db-secret       # name of the Secret resource
        key: DB_PASSWORD           # key inside it
```

Static values like `DISABLE_AUTH` and `ROOT` are hardcoded directly in the Deployment YAML since they're not sensitive and don't change.

### Persistent Storage (PVCs)

In Docker, you'd use a volume to keep data around when a container restarts. K8s splits this into two parts:

- **PersistentVolumeClaim (PVC)** — the pod says "I need 5GB of disk space"
- **PersistentVolume (PV)** — the cluster provisions actual storage to satisfy the claim (on k3s, local disk by default)

Each service has its own PVC:
- `rstudio/pvc.yaml` → 5GB at `/home/rstudio`
- `vscode/pvc.yaml` → 5GB at `/home/coder`

When a pod restarts, it reattaches to the same PVC — files, projects, and settings are still there.

### nginx — Reverse Proxy & Basic Auth

nginx sits between the ALB and the application pods. It handles three things:

1. **Basic auth** — all routes require a username/password from `.htpasswd` (stored in `nginx-htpasswd` Secret)
2. **Landing page** — serves `static/index.html` at `/` with links to each environment
3. **Reverse proxy** — forwards `/vscode/` and `/rstudio/` to the backend Services, with WebSocket support (`Upgrade` headers) and a 1-hour read timeout so idle sessions don't get dropped

The nginx config and landing page HTML are both managed as `configMapGenerator` entries in `kustomization.yaml` — no separate ConfigMap YAML files needed.

### Kustomize

All resources are declared in `kustomization.yaml` and applied with one command: `kubectl apply -k k8s/`

Kustomize handles:
- **`configMapGenerator`** — generates ConfigMaps from inline literals (DB config) or files (nginx config, portal HTML). Appends a hash suffix to names so config changes trigger pod restarts automatically.
- **`resources`** — references all Deployment, Service, PVC, and Secret manifests

### Flux

Flux is **GitOps for Kubernetes**. It watches the Git repo and automatically applies changes to the cluster when you push. The Git repo is the single source of truth — if someone manually changes something in the cluster, Flux reverts it to match Git.

**The workflow:** push code → Flux picks it up → cluster updates itself.

## Applying Changes

With Kustomize, one command applies everything in the right order:

```bash
kubectl apply -k k8s/
```

### Verify env vars are in the pods

```bash
kubectl exec deploy/rstudio -- env | grep DB_
kubectl exec deploy/vscode -- env | grep DB_
```

### Access the portal

In production, the AWS ALB routes traffic to the nginx Service automatically.

For local development, port-forward to nginx:

```bash
kubectl port-forward svc/nginx 8080:80
```

Then open `localhost:8080` in your browser. Log in with the basic auth credentials, and use the landing page to navigate to RStudio or VS Code.
