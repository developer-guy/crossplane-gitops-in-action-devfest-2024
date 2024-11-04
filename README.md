# Crossplane & GitOps in Action

## Prerequisites
* kubectl
* kind
* argocd
* kustomize
* up

## Setup

### Kubernetes Cluster (Control Plane Cluster)

Create a local Kubernetes cluster using [kind](https://kind.sigs.k8s.io/) to run Crosplane and manage our infrastructure on GCP.

```bash
kind create cluster --wait 5m
```

### ArgoCD

You need to apply the kustomization under `gitops/argocd` directory:

```bash
kubectl apply -k ./gitops/argocd

# Check the argocd namespace
kubectl get pods -n argocd

# Wait until argo is ready
kubectl wait pod --all --for=condition=Ready --namespace=argocd --timeout=600s
```

To access argocd, you can use the CLI tool or the Web UI:

```bash
# First, port forward to the argocd server
kubectl port-forward svc/argocd-server 8080:443 -n argocd

# Get the initial admin password
ARGO_PASS=$(kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d)

# Connect using CLI
# If you want to connect to the UI, you can use https://localhost:8080
argocd login localhost:8080 --username admin --password ${ARGO_PASS} --insecure
```

### Argo Applications

We are ready to use GitOps. Let's start with Crossplane deploymenet:

First, craete the app of the apps:

```bash
cat <<EOF > ./gitops/crossplane/crossplane.yaml
##########################################################
## The ArgoCD App of Apps for all Crossplane components ##
##########################################################
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane-bootstrap
  namespace: argocd
  labels:
    crossplane.jonashackt.io: bootstrap
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  source:
    repoURL: https://github.com/koksay/crossplane-gitops-in-action-devfest-2024.git
    targetRevision: HEAD
    path: gitops/crossplane/bootstrap
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true    
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 1
      backoff:
        duration: 5s 
        factor: 2 
        maxDuration: 1m
EOF
```

Then, create the Crossplane installation:

```bash
cat <<EOF > ./gitops/crossplane/bootstrap/crossplane-install.yaml
# The ArgoCD Application for crossplane core components themselves
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: crossplane
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  project: default
  source:
    chart: crossplane
    repoURL: https://charts.crossplane.io/stable
    targetRevision: 1.17.2 # this was the latest version of the Helm Chart at the we were working on this.
    helm:
      releaseName: crossplane
  destination:
    server: https://kubernetes.default.svc
    namespace: crossplane-system
  syncPolicy:
    automated:
      prune: true    
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 1
      backoff:
        duration: 5s 
        factor: 2 
        maxDuration: 1m
EOF
```

We need to push our changes to our repo. Then, when we create the `crossplane-bootstrap` app, ArgoCD will take care of deploying Crossplane Helm Chart for us.

```bash
# Push changes
git add .
git commit -am "Deploy crossplane"
git push

# Create the crossplane-bootstrap app
kubectl apply -f ./gitops/crossplane/crossplane.yaml

# You can manually sync argocd, and check the app status

```

## Manage GCP