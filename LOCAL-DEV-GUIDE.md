# Local Development Quick Start Guide

This guide helps you test the GitOps infrastructure locally before deploying to cloud clusters.

## Prerequisites

- Docker Desktop with Kubernetes enabled, OR
- kind, minikube, or k3d installed
- kubectl CLI
- ArgoCD CLI (optional but recommended)

## Step-by-Step Setup

### 1. Create Local Kubernetes Cluster

**Option A: Docker Desktop**
```bash
# Enable Kubernetes in Docker Desktop settings
# Cluster will be named: docker-desktop
kubectl config use-context docker-desktop
```

**Option B: kind (Kubernetes in Docker)**
```bash
kind create cluster --name local-dev
kubectl config use-context kind-local-dev
```

**Option C: minikube**
```bash
minikube start --profile local-dev
kubectl config use-context minikube
```

**Option D: k3d**
```bash
k3d cluster create local-dev
kubectl config use-context k3d-local-dev
```

### 2. Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Get admin password
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD admin password: $ARGOCD_PASSWORD"
```

### 3. Access ArgoCD UI

**Terminal 1: Port Forward**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

**Terminal 2: Login**
```bash
# Via CLI
argocd login localhost:8080 --username admin --password $ARGOCD_PASSWORD --insecure

# Via Browser
open https://localhost:8080
# Username: admin
# Password: (from step 2)
```

### 4. Register Local Cluster

```bash
# Add cluster to ArgoCD
argocd cluster add $(kubectl config current-context) --name local-dev

# Label the cluster
argocd cluster set local-dev --label cluster=local --label environment=dev

# Verify
argocd cluster list
```

### 5. Add Helm Repositories

```bash
argocd repo add https://kedacore.github.io/charts --type helm --name kedacore
argocd repo add https://charts.bitnami.com/bitnami --type helm --name bitnami
argocd repo add https://open-telemetry.github.io/opentelemetry-helm-charts --type helm --name open-telemetry
argocd repo add https://codecentric.github.io/helm-charts --type helm --name codecentric
```

### 6. Add Git Repository

```bash
# Add this repository (update URL first!)
argocd repo add https://github.com/wigo4it/gitops-core-infra.git
```

### 7. Bootstrap Local Cluster

```bash
# Apply the root application
kubectl apply -f clusters/local/dev/root.yaml

# Watch applications being created
argocd app list --watch
```

### 8. Verify Deployments

```bash
# Check all namespaces
kubectl get namespaces | grep -E "keda|rabbitmq|opentelemetry|keycloak"

# Check KEDA
kubectl get pods -n keda-system
kubectl logs -n keda-system -l app.kubernetes.io/name=keda-operator --tail=50

# Check RabbitMQ
kubectl get pods -n rabbitmq-system
kubectl logs -n rabbitmq-system -l app.kubernetes.io/name=rabbitmq-cluster-operator --tail=50

# Check OpenTelemetry
kubectl get pods -n opentelemetry-system

# Check Keycloak
kubectl get pods -n keycloak-system

# Check ArgoCD Applications
argocd app get keda-operator-local-dev
argocd app get rabbitmq-operator-local-dev
```

## Testing Operator Changes

### Test a Version Upgrade

```bash
# 1. Edit version in .argocd.yaml
vim operators/keda/.argocd.yaml
# Change local-dev: chartVersion: "2.14.0" → "2.15.0"

# 2. Commit and push
git add operators/keda/.argocd.yaml
git commit -m "test(keda): upgrade to 2.15.0 in local-dev"
git push

# 3. Refresh application in ArgoCD
argocd app get keda-operator-local-dev --refresh

# 4. Sync
argocd app sync keda-operator-local-dev

# 5. Verify
kubectl get pods -n keda-system -w
kubectl logs -n keda-system -l app.kubernetes.io/name=keda-operator --tail=100
```

### Test Configuration Changes

```bash
# 1. Edit values file
vim operators/keda/values.local-dev.yaml
# Modify resource limits, replicas, etc.

# 2. Commit and push
git add operators/keda/values.local-dev.yaml
git commit -m "test(keda): adjust local resource limits"
git push

# 3. Sync in ArgoCD
argocd app sync keda-operator-local-dev

# 4. Verify
kubectl get pods -n keda-system -o yaml | grep -A 10 resources:
```

## Troubleshooting

### ArgoCD Not Syncing

```bash
# Check ApplicationSet
kubectl get applicationset -n argocd
kubectl describe applicationset platform-operators -n argocd

# Check if Git repo is accessible
argocd repo list

# Force refresh
argocd app get keda-operator-local-dev --refresh --hard
```

### Pods Not Starting (Resource Constraints)

```bash
# Check node resources
kubectl top nodes

# Check pod resource requests
kubectl describe pod -n keda-system

# Reduce resources in values.local-dev.yaml if needed
```

### Port Conflicts

```bash
# If port 8080 is in use, use different port
kubectl port-forward svc/argocd-server -n argocd 9090:443

# Then access at: https://localhost:9090
```

## Cleanup

### Remove Operators

```bash
# Delete root application (cascades to all operators)
kubectl delete application local-dev-root -n argocd
```

### Remove ArgoCD

```bash
kubectl delete namespace argocd
```

### Delete Cluster

```bash
# kind
kind delete cluster --name local-dev

# minikube
minikube delete --profile local-dev

# k3d
k3d cluster delete local-dev

# Docker Desktop
# Disable Kubernetes in Docker Desktop settings
```

## Benefits of Local Testing

✅ **No Cloud Costs** - Test freely without AKS charges  
✅ **Fast Iteration** - Commit → Push → Sync in seconds  
✅ **Safe Experiments** - Break things without affecting cloud  
✅ **Full Feature Testing** - All 4 CNCF operators available  
✅ **Version Validation** - Test upgrades before cloud deployment  
✅ **Offline Development** - Work without cloud connectivity  

## Next Steps

Once validated locally:
1. Promote changes to `aks-ikv-nonprod-vnext`
2. Follow the promotion workflow in main README.md
3. Monitor and validate in cloud environments
4. Promote to production after soak period
