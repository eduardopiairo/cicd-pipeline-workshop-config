# ArgoCD

ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes. It monitors a Git repository and automatically syncs the cluster state to match the desired state defined in the repo.

## How it fits this project

This repository holds the Kubernetes manifests for the workshop application. ArgoCD watches this repo and applies any changes to the target cluster, replacing manual `kubectl apply` steps.

```
Git push → ArgoCD detects diff → syncs cluster
```

## Prerequisites

- A running Kubernetes cluster
- `kubectl` configured for the cluster
- ArgoCD installed in the cluster

## Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## Access the ArgoCD UI

```bash
# Port-forward the ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Retrieve the initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

Open `https://localhost:8080` and log in with username `admin` and the password above.


## Create an Application

### Option 1 — kubectl (YAML manifest)

Create one manifest per application and apply both:

**backend-app.yaml**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/eduardopiairo/cicd-pipeline-workshop-config
    targetRevision: main
    path: k8s/backend
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**frontend-app.yaml**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/eduardopiairo/cicd-pipeline-workshop-config
    targetRevision: main
    path: k8s/frontend
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
kubectl apply -f backend-app.yaml
kubectl apply -f frontend-app.yaml
```

### Option 2 — ArgoCD UI

Repeat the following steps twice — once for the backend, once for the frontend.

1. Open `https://localhost:8080` and log in.
2. Click **+ New App** in the top-left corner.
3. Fill in the **General** section:

   | Field | Backend | Frontend |
   |---|---|---|
   | **Application Name** | `backend` | `frontend` |
   | **Project** | `default` | `default` |
   | **Sync Policy** | `Automatic` | `Automatic` |

   Check **Prune Resources** and **Self Heal** for both.

4. Fill in the **Source** section:

   | Field | Backend | Frontend |
   |---|---|---|
   | **Repository URL** | `https://github.com/eduardopiairo/cicd-pipeline-workshop-config` | same |
   | **Revision** | `main` | `main` |
   | **Path** | `k8s/backend` | `k8s/frontend` |

5. Fill in the **Destination** section (same for both):
   - **Cluster URL**: `https://kubernetes.default.svc`
   - **Namespace**: `default`
6. Click **Create** at the top of the form.

Repeat for the second application. ArgoCD will sync each app independently from its respective path.

## Sync behavior

| Setting | Value | Effect |
|---|---|---|
| `automated` | enabled | ArgoCD syncs automatically on every Git change |
| `prune` | true | Resources removed from Git are deleted from the cluster |
| `selfHeal` | true | Manual cluster changes are reverted to match Git |

## Image updates in the CI/CD pipeline

The CI pipeline (e.g., GitHub Actions) should update the image tag in the deployment manifests after a successful build, then push the change to this repo. ArgoCD will detect the commit and roll out the new image automatically.

```
Build image → Push to registry → Update image tag in this repo → ArgoCD syncs
```
