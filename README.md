# ArgoCD Bootstrapping

This project implements the ArgoCD "App of Apps" pattern to deploy and manage multiple applications in a Kubernetes cluster using GitOps principles.

## Purpose

- **Centralized Application Management**: Define all applications in a single `values.yaml` file
- **Automated Deployments**: Applications are automatically synced and deployed when changes are pushed to the repository
- **Image Updates**: Integrated with ArgoCD Image Updater for automatic container image updates
- **Scalable GitOps**: Easy to add new applications by simply updating the configuration

## Prerequisites

Before using this repository, you need to install ArgoCD and ArgoCD Image Updater in your Kubernetes cluster.

### Install ArgoCD

```bash
# Add the ArgoCD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Create the argocd namespace
kubectl create namespace argocd

# Install ArgoCD
helm install argocd argo/argo-cd \
  --namespace argocd \
  --set server.extraArgs[0]="--insecure"
```

### Install ArgoCD Image Updater

```bash
# Install ArgoCD Image Updater
helm install argocd-image-updater argo/argocd-image-updater \
  --namespace argocd \
  --set config.argocd.serverAddress="http://argocd-server.argocd.svc.cluster.local"
```

### Get ArgoCD Admin Password

```bash
# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

## Usage

### Deploy the App of Apps

Create the parent ArgoCD application that will deploy all child applications:

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/learnwithtwoheads/argocd-bootstrapping.git
    targetRevision: main
    path: app-of-apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

### Add New Applications

To add a new application, simply edit `app-of-apps/values.yaml` and add it to the `applications` list:

```yaml
- name: my-new-app
  path: applications/my-new-app/overlays/prod
  namespace: my-namespace
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=ghcr.io/myorg/my-app
    argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/git-creds
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

The new application will be automatically deployed when the changes are synced.
