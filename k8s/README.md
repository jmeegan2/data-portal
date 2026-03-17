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
  README.md
  config/
    dart-db-config.yaml    # ConfigMap — DB connection info (host, port, name, user)
    dart-db-secret.yaml    # Secret — DB password
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

### The Flow (RStudio example — VS Code is identical on port 8080)

```
You (localhost:8787)
  → kubectl port-forward (or Shift+F in k9s)
    → Service (rstudio:8787)
      → Pod (rstudio container:8787)
        → RStudio Server
          → reads DB_HOST, DB_PASSWORD, etc. from environment
            → connects to DART database
```

1. The **Deployment** creates and manages the pod running RStudio/VS Code
2. The **Service** gives that pod a stable network identity so you don't need to know the pod's name
3. Port-forwarding bridges your local machine to the Service
4. The pod gets its database connection info from the **ConfigMap** and **Secret** via environment variables
5. The **PVC** keeps user files alive across pod restarts

### Environment Variables: Docker Compose vs K8s

In Docker Compose, the `.env` file fed variables into containers via `env_file: .env`. Kubernetes doesn't have `env_file` — it uses ConfigMaps and Secrets instead.

| Docker Compose (.env)  | K8s Resource | File                          | Why                          |
|------------------------|--------------|-------------------------------|------------------------------|
| `DB_HOST`              | ConfigMap    | `config/dart-db-config.yaml`  | Not sensitive                |
| `DB_PORT`              | ConfigMap    | `config/dart-db-config.yaml`  | Not sensitive                |
| `DB_NAME`              | ConfigMap    | `config/dart-db-config.yaml`  | Not sensitive                |
| `DB_USER`              | ConfigMap    | `config/dart-db-config.yaml`  | Not sensitive                |
| `DB_PASSWORD`          | Secret       | `config/dart-db-secret.yaml`  | Credential — keep locked down|

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

## Applying Changes

ConfigMap and Secret must exist before the Deployments that reference them. Order matters:

```bash
# 1. Apply config (ConfigMap + Secret)
kubectl apply -f k8s/config/

# 2. Apply services (RStudio + VS Code)
kubectl apply -f k8s/rstudio/
kubectl apply -f k8s/vscode/
```

### Verify env vars are in the pods

```bash
kubectl exec deploy/rstudio -- env | grep DB_
kubectl exec deploy/vscode -- env | grep DB_
```

### Access the services

Using k9s: navigate to Services, select one, press `Shift+F` to port-forward.

Or from the command line:

```bash
kubectl port-forward svc/rstudio 8787:8787 &
kubectl port-forward svc/vscode 8080:8080
```

Then open `localhost:8787` (RStudio) or `localhost:8080` (VS Code) in your browser.
