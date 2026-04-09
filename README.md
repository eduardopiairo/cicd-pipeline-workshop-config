# DevOps Porto Get-Together — CI/CD Workshop Kubernetes Config

This is the companion repository for the CI/CD Pipeline Workshop. It contains the Kubernetes manifests used by ArgoCD to deploy the **DevOps Porto Get-Together** application.

---

## How it fits in the workshop

```
cicd-pipeline-workshop-app          cicd-pipeline-workshop-config
──────────────────────────────      ──────────────────────────────
Application code                    Kubernetes manifests (this repo)
GitHub Actions CI pipeline    →     ArgoCD watches for changes
Builds & pushes Docker images →     Deploys to the cluster
      (Docker Hub)
```

When the CI pipeline pushes a new image to Docker Hub, update the image tag in the relevant manifest and push to this repository. ArgoCD detects the change and rolls out the new version automatically.

---

## Repository Structure

```
cicd-pipeline-workshop-config/
└── k8s/
    ├── backend/
    │   ├── deployment.yaml   # Backend pods (FastAPI, port 8000)
    │   ├── service.yaml      # ClusterIP service — internal access
    │   └── secret.yaml       # DATABASE_URL secret
    └── frontend/
        ├── deployment.yaml   # Frontend pods (nginx, port 80)
        └── service.yaml      # NodePort service — external access on :30000
```

---

## Prerequisites

- A running Kubernetes cluster (minikube or kind)
- `kubectl` configured to point at the cluster
- ArgoCD installed in the cluster

---

## Local Cluster Setup

Choose either **minikube** or **kind** to run Kubernetes locally.

### minikube

**Install**

```bash
# macOS
brew install minikube

# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

**Start / stop**

```bash
minikube start          # start with default driver (Docker recommended)
minikube status         # check cluster health
minikube stop           # stop the cluster
minikube delete         # destroy the cluster
```

**Useful extras**

```bash
minikube dashboard      # open the Kubernetes web UI
minikube addons enable ingress   # enable nginx ingress controller
```

---

### kind (Kubernetes IN Docker)

**Install**

```bash
# macOS
brew install kind

# Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x kind && sudo mv kind /usr/local/bin/kind
```

**Start / stop**

```bash
kind create cluster --name workshop   # create a cluster
kubectl cluster-info --context kind-workshop
kind delete cluster --name workshop   # destroy the cluster
```

> kind does not support NodePort access from the host directly. Use `kubectl port-forward` to reach services (see Access the application below).

---

## Quick Start

### 1. Apply the manifests manually (without ArgoCD)

```bash
kubectl apply -f k8s/backend/
kubectl apply -f k8s/frontend/
```

Verify everything is running:

```bash
kubectl get pods
kubectl get services
```

### 2. Access the application

**Frontend** (NodePort):
```bash
# minikube
minikube service frontend --url

# kind — use port-forward
kubectl port-forward service/frontend 3000:80
```

**Backend** (from inside the cluster only):
```bash
kubectl port-forward service/backend 8000:8000
curl http://localhost:8000/health
```

---

## Before Applying

Update the image placeholder in each deployment with your Docker Hub username:

```bash
# backend/deployment.yaml
image: <DOCKERHUB_USERNAME>/backend:latest

# frontend/deployment.yaml
image: <DOCKERHUB_USERNAME>/frontend:latest
```

---

## ArgoCD Setup

See [docs/ARGOCD.md](docs/ARGOCD.md) for full instructions on installing ArgoCD, accessing the UI, and registering this repository as an application.

---

## Updating the Image Tag

After a new CI pipeline run, update the image tag to deploy a new version:

```bash
# Example: update backend to a specific commit SHA
kubectl set image deployment/backend backend=<DOCKERHUB_USERNAME>/backend:<sha>
```

Or edit `k8s/backend/deployment.yaml` directly, commit, and push — ArgoCD will pick it up automatically.

---

## Related Repository

- **Application + CI pipeline**: [cicd-pipeline-workshop-app](https://github.com/eduardopiairo/cicd-pipeline-workshop-app)
