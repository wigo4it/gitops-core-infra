# GitOps Infrastructure - Complete Guide

**Multi-Cluster Kubernetes Operator Management with ArgoCD**

---

## 📖 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Quick Start](#quick-start)
- [Repository Structure](#repository-structure)
- [Version Management](#version-management)
- [Operations Guide](#operations-guide)
- [Cluster Reference](#cluster-reference)
- [Migration from Legacy](#migration-from-legacy)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

---

## Overview

This repository implements **best-practice GitOps** for managing Kubernetes operators across multiple AKS clusters using ArgoCD with Helm charts.

### Key Features

✅ **Discovery-Based Deployment** - Operators automatically discovered from Git  
✅ **Centralized Version Control** - Single `.argocd.yaml` per operator  
✅ **Progressive Delivery** - nonprod-vnext → nonprod → prod workflow  
✅ **App-of-Apps Pattern** - Simple cluster bootstrapping  
✅ **Git-Tracked Versions** - Complete audit trail  
✅ **Modular Structure** - One directory per operator  

### Cluster Groups

**Local Development** (4 operators)
- `local-dev` - Local testing (kind/minikube/k3d/Docker Desktop)
- Operators: KEDA, RabbitMQ, OpenTelemetry, Keycloak
- Purpose: Test operator changes locally before pushing to cloud

**IKV Clusters** (2 operators)
- `aks-ikv-nonprod-vnext` - Testing environment
- `aks-ikv-nonprod` - Validation environment
- `aks-ikv-prod` - Production environment
- Operators: KEDA, RabbitMQ

**Commonground Clusters** (4 operators)
- `aks-commonground-nonprod` - Pre-production (⚠️ TODO: Update actual name)
- `aks-commonground-prod` - Production (⚠️ TODO: Update actual name)
- Operators: KEDA, RabbitMQ, OpenTelemetry, Keycloak

---

## Architecture

### Design Principles

**1. Discovery-Based ApplicationSet**
```yaml
# Automatically discovers operators via .argocd.yaml files
generators:
  - git:
      files:
        - path: "operators/**/.argocd.yaml"
```

Benefits: Add new operator = create directory, no ApplicationSet changes needed

**2. Centralized Version Management**
```yaml
# operators/keda/.argocd.yaml
clusters:
  aks-ikv-nonprod-vnext:
    chartVersion: "2.14.0"  # Latest for testing
  aks-ikv-nonprod:
    chartVersion: "2.14.0"  # Validated
  aks-ikv-prod:
    chartVersion: "2.13.0"  # Stable for production
```

Benefits: One file to update, clear promotion path, Git tracks changes

**3. Values Inheritance**
```
Chart Defaults
    ↓
operators/{name}/values.yaml (base)
    ↓
operators/{name}/values.{cluster}.yaml (environment-specific)
    ↓
Final Deployment
```

**4. Official Helm Charts**

| Operator | Chart | Repository |
|----------|-------|------------|
| KEDA | `kedacore/keda` | https://kedacore.github.io/charts |
| RabbitMQ | `bitnami/rabbitmq-cluster-operator` | https://charts.bitnami.com/bitnami |
| OpenTelemetry | `open-telemetry/opentelemetry-operator` | https://open-telemetry.github.io/opentelemetry-helm-charts |
| Keycloak | `codecentric/keycloakx` | https://codecentric.github.io/helm-charts |

---

## Quick Start

### Prerequisites

- ArgoCD installed (hub cluster or local)
- kubectl configured with cluster access
- Git repository access
- Helm 3.x (included in ArgoCD)
- **For local testing**: kind, minikube, k3d, or Docker Desktop with Kubernetes enabled

### Local Development Setup

**Perfect for testing operator changes before deploying to cloud clusters!**

**1. Create local Kubernetes cluster**

```bash
# Using kind (recommended)
kind create cluster --name local-dev

# Or using minikube
minikube start --profile local-dev

# Or using k3d
k3d cluster create local-dev

# Or use Docker Desktop Kubernetes (enable in settings)
```

**2. Install ArgoCD**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward to access UI (in a separate terminal)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Login via CLI
argocd login localhost:8080
```

**3. Register local cluster**

```bash
# Register the cluster
kubectl config use-context kind-local-dev  # or docker-desktop, or minikube
argocd cluster add kind-local-dev --name local-dev

# Label the cluster
argocd cluster set local-dev --label cluster=local --label environment=dev
```

**4. Add Helm repositories**

```bash
argocd repo add https://kedacore.github.io/charts --type helm --name kedacore
argocd repo add https://charts.bitnami.com/bitnami --type helm --name bitnami
argocd repo add https://open-telemetry.github.io/opentelemetry-helm-charts --type helm --name open-telemetry
argocd repo add https://codecentric.github.io/helm-charts --type helm --name codecentric
```

**5. Bootstrap local cluster**

```bash
# Update Git repo URL first
vim clusters/local/dev/root.yaml  # Update repoURL

# Apply root application
kubectl apply -f clusters/local/dev/root.yaml

# Watch deployments
argocd app list --watch
```

**6. Verify operators**

```bash
kubectl get pods -n keda-system
kubectl get pods -n rabbitmq-system
kubectl get pods -n opentelemetry-system
kubectl get pods -n keycloak-system
```

### Cloud Cluster Setup

### 1. Update Git Repository URLs

Repository URL: `https://github.com/wigo4it/gitops-core-infra.git`

Update in the following locations if you fork this repository:
- `argocd/applicationset-operators.yaml` (2 occurrences)
- `clusters/ikv/*/root.yaml` (3 files)
- `clusters/commonground/*/root.yaml` (2 files)

```bash
# Quick update
find argocd/ clusters/ -type f -name "*.yaml" -exec sed -i 's|https://github.com/your-org/ikvcluster-core-infra2.git|YOUR_REPO_URL|g' {} \;
```

### 2. Register Clusters in ArgoCD

```bash
# Get AKS credentials
az aks get-credentials --resource-group <rg> --name aks-ikv-nonprod-vnext
az aks get-credentials --resource-group <rg> --name aks-ikv-nonprod
az aks get-credentials --resource-group <rg> --name aks-ikv-prod

# Register with ArgoCD
argocd cluster add aks-ikv-nonprod-vnext --name aks-ikv-nonprod-vnext
argocd cluster add aks-ikv-nonprod --name aks-ikv-nonprod
argocd cluster add aks-ikv-prod --name aks-ikv-prod

# Label clusters
argocd cluster set aks-ikv-nonprod-vnext --label cluster=ikv --label environment=nonprod-vnext
argocd cluster set aks-ikv-nonprod --label cluster=ikv --label environment=nonprod
argocd cluster set aks-ikv-prod --label cluster=ikv --label environment=prod
```

### 3. Add Helm Repositories to ArgoCD

```bash
argocd repo add https://kedacore.github.io/charts --type helm --name kedacore
argocd repo add https://charts.bitnami.com/bitnami --type helm --name bitnami
argocd repo add https://open-telemetry.github.io/opentelemetry-helm-charts --type helm --name open-telemetry
argocd repo add https://codecentric.github.io/helm-charts --type helm --name codecentric
```

### 4. Bootstrap a Cluster

```bash
# Apply root application for a cluster
kubectl apply -f clusters/ikv/nonprod-vnext/root.yaml

# The ApplicationSet will automatically discover and deploy all enabled operators
```

### 5. Monitor Deployment

```bash
# Watch applications
argocd app list

# Check specific operator
argocd app get keda-operator-aks-ikv-nonprod-vnext

# Monitor pods
kubectl get pods -n keda-system -w
```

---

## Repository Structure

```
.
├── operators/                      # One directory per operator
│   ├── keda/
│   │   ├── .argocd.yaml           # Version metadata (per cluster)
│   │   ├── values.yaml            # Base Helm values
│   │   ├── values.aks-ikv-nonprod-vnext.yaml
│   │   ├── values.aks-ikv-nonprod.yaml
│   │   ├── values.aks-ikv-prod.yaml
│   │   ├── values.aks-commonground-nonprod.yaml
│   │   └── values.aks-commonground-prod.yaml
│   ├── rabbitmq/                  # Same structure
│   ├── opentelemetry/             # Commonground only
│   └── keycloak/                  # Commonground only
│
├── argocd/
│   └── applicationset-operators.yaml  # Discovery-based ApplicationSet
│
├── clusters/
│   ├── ikv/
│   │   ├── nonprod-vnext/
│   │   │   └── root.yaml          # App-of-Apps bootstrap
│   │   ├── nonprod/
│   │   │   └── root.yaml
│   │   └── prod/
│   │       └── root.yaml
│   └── commonground/
│       ├── nonprod/
│       │   └── root.yaml
│       └── prod/
│           └── root.yaml
│
└── .archive/
    └── legacy/                     # Archived old structure
```

### Key Files Explained

**`.argocd.yaml` - Version Control Metadata**

Each operator has this file defining versions per cluster:

```yaml
# operators/keda/.argocd.yaml
chartName: keda
chartRepoURL: https://kedacore.github.io/charts
targetNamespace: keda-system

clusters:
  aks-ikv-nonprod-vnext:
    enabled: true
    chartVersion: "2.14.0"
  aks-ikv-nonprod:
    enabled: true
    chartVersion: "2.14.0"
  aks-ikv-prod:
    enabled: true
    chartVersion: "2.13.0"
```

**`root.yaml` - App-of-Apps Bootstrap**

Each cluster has a root application:

```yaml
# clusters/ikv/prod/root.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ikv-prod-root
spec:
  source:
    repoURL: https://github.com/your-org/ikvcluster-core-infra2.git
    path: argocd
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
```

To bootstrap: `kubectl apply -f clusters/ikv/prod/root.yaml`

---

## Version Management

### Version Promotion Workflow

```
┌──────────────────┐
│ local-dev        │  Local testing
│ chartVersion:    │  ↓ Test changes locally
│ "2.15.0"         │  ↓ Verify functionality
└──────────────────┘  ↓ Fast iteration
         ↓
┌──────────────────┐
│ nonprod-vnext    │  Cloud testing
│ chartVersion:    │  ↓ 1-2 weeks validation
│ "2.15.0"         │  ↓ Run smoke tests
└──────────────────┘  ↓ Monitor metrics
         ↓
┌──────────────────┐
│ nonprod          │  Promote after validation
│ chartVersion:    │  ↓ 1-2 weeks soak period
│ "2.15.0"         │  ↓ Load testing
└──────────────────┘  ↓ Monitor stability
         ↓
┌──────────────────┐
│ prod             │  Promote to production
│ chartVersion:    │  ✓ Proven stable
│ "2.15.0"         │
└──────────────────┘
```

### Upgrading an Operator

**Example: Upgrade KEDA to v2.15.0**

**Step 1: Test in nonprod-vnext**

```bash
git checkout -b upgrade-keda-2.15.0

# Edit operators/keda/.argocd.yaml
vim operators/keda/.argocd.yaml
# Change: aks-ikv-nonprod-vnext: chartVersion: "2.14.0" → "2.15.0"

git add operators/keda/.argocd.yaml
git commit -m "feat(keda): upgrade to 2.15.0 in nonprod-vnext"
git push origin upgrade-keda-2.15.0
# Create PR, review, merge
```

**Step 2: Monitor & Validate (1-2 weeks)**

```bash
# Check KEDA deployment
kubectl get pods -n keda-system --context aks-ikv-nonprod-vnext

# Monitor logs
kubectl logs -n keda-system deployment/keda-operator -f

# Run tests
# Verify scalers work correctly
```

**Step 3: Promote to nonprod**

```bash
git checkout main && git pull
git checkout -b promote-keda-2.15.0-nonprod

# Edit operators/keda/.argocd.yaml
# Change: aks-ikv-nonprod: chartVersion: "2.14.0" → "2.15.0"

git add operators/keda/.argocd.yaml
git commit -m "feat(keda): promote 2.15.0 to nonprod"
git push origin promote-keda-2.15.0-nonprod
# Create PR, merge
```

**Step 4: Soak Period (1-2 weeks)**

Monitor nonprod performance, run load tests

**Step 5: Promote to Production**

```bash
git checkout main && git pull
git checkout -b promote-keda-2.15.0-prod

# Edit operators/keda/.argocd.yaml
# Change: aks-ikv-prod: chartVersion: "2.14.0" → "2.15.0"

git add operators/keda/.argocd.yaml
git commit -m "feat(keda): promote 2.15.0 to production"
git push origin promote-keda-2.15.0-prod
# Create PR with thorough review
# Schedule maintenance window
# Merge during business hours
```

### Viewing Current Versions

```bash
# View all versions for an operator
yq '.clusters' operators/keda/.argocd.yaml

# Compare versions across all operators
for op in operators/*/; do
  echo "=== $(basename $op) ==="
  yq '.clusters | to_entries | .[] | .key + ": " + .value.chartVersion' $op/.argocd.yaml
done
```

### Rollback Procedures

**Quick Rollback via ArgoCD**

```bash
# Via CLI
argocd app rollback keda-operator-aks-ikv-prod <revision-id>

# Or via UI: Application → History → Select previous sync → Rollback
```

**Git Rollback**

```bash
# Revert the version change commit
git revert <commit-sha>
git push origin main

# ArgoCD will sync the reverted version
```

### Version Pinning Best Practices

```yaml
# ✅ Good - Exact version
chartVersion: "2.14.0"

# ❌ Bad - Version ranges or latest
chartVersion: "^2.14.0"
chartVersion: "2.x"
chartVersion: "latest"
```

---

## Operations Guide

### Adding a New Operator

**1. Create operator directory**

```bash
mkdir -p operators/my-operator
```

**2. Create `.argocd.yaml`**

```yaml
# operators/my-operator/.argocd.yaml
chartName: my-operator
chartRepoURL: https://charts.example.com/
targetNamespace: my-operator-system

clusters:
  aks-ikv-nonprod-vnext:
    enabled: true
    chartVersion: "1.0.0"
  aks-ikv-nonprod:
    enabled: true
    chartVersion: "1.0.0"
  aks-ikv-prod:
    enabled: true
    chartVersion: "1.0.0"
  aks-commonground-nonprod:
    enabled: false  # Not needed
  aks-commonground-prod:
    enabled: false  # Not needed
```

**3. Create base values**

```yaml
# operators/my-operator/values.yaml
replicaCount: 1

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

**4. Create environment-specific values**

```yaml
# operators/my-operator/values.aks-ikv-prod.yaml
replicaCount: 2  # HA for production

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

# Anti-affinity for HA
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: my-operator
          topologyKey: kubernetes.io/hostname
```

**5. Commit and push**

```bash
git add operators/my-operator/
git commit -m "feat(my-operator): add new operator"
git push origin main
```

**The operator will be automatically discovered and deployed!**

### Customizing Operator Configuration

**Modify base values (affects all environments):**

```bash
vim operators/keda/values.yaml
# Make changes
git commit -am "config(keda): update base configuration"
git push
```

**Modify environment-specific values:**

```bash
vim operators/keda/values.aks-ikv-prod.yaml
# Make production-specific changes
git commit -am "config(keda): increase prod resources"
git push
```

ArgoCD auto-syncs changes within 3 minutes.

### Disabling an Operator

**Disable for specific cluster:**

```yaml
# operators/keda/.argocd.yaml
clusters:
  aks-ikv-nonprod-vnext:
    enabled: false  # ← Disable
```

The ApplicationSet will skip disabled operators automatically.

### Monitoring Deployments

```bash
# Watch all applications
argocd app list --watch

# Check specific operator status
argocd app get keda-operator-aks-ikv-prod

# View sync history
argocd app history keda-operator-aks-ikv-prod

# Check pod status
kubectl get pods -n keda-system --context aks-ikv-prod -w

# View logs
kubectl logs -n keda-system deployment/keda-operator -f --context aks-ikv-prod

# Check resource usage
kubectl top pods -n keda-system --context aks-ikv-prod
```

---

## Cluster Reference

### IKV Clusters (Production Ready)

| Environment | ArgoCD Name | AKS Resource | Purpose |
|-------------|-------------|--------------|---------|
| nonprod-vnext | `aks-ikv-nonprod-vnext` | aks-ikv-nonprod-vnext | Testing ground, latest versions |
| nonprod | `aks-ikv-nonprod` | aks-ikv-nonprod | Validation, load testing |
| prod | `aks-ikv-prod` | aks-ikv-prod | Production, stable versions, HA |

**Operators Deployed:** KEDA, RabbitMQ, Redir (3 total)

### Commonground Clusters (TODO: Update Names)

| Environment | ArgoCD Name | AKS Resource | Status |
|-------------|-------------|--------------|--------|
| nonprod | `aks-commonground-nonprod` | TBD | ⚠️ Placeholder |
| prod | `aks-commonground-prod` | TBD | ⚠️ Placeholder |

**Operators Deployed:** KEDA, RabbitMQ, Redir, OpenTelemetry, Keycloak (5 total)

**When actual cluster names are available, update:**
- `operators/*/.argocd.yaml` (5 files)
- `argocd/applicationset-operators.yaml`

### Local Development Cluster

| Environment | ArgoCD Name | Infrastructure | Purpose |
|-------------|-------------|----------------|---------|
| dev | `local-dev` | kind/minikube/k3d/Docker Desktop | Local testing before cloud deployment |

**Operators Deployed:** KEDA, RabbitMQ, OpenTelemetry, Keycloak (4 total)

**Benefits:**
- ✅ Test operator changes locally without cloud costs
- ✅ Fast iteration cycle for development
- ✅ Safe experimentation environment
- ✅ All operators enabled for full testing
- ✅ Minimal resource allocation for laptops
- ✅ Perfect for testing version upgrades

### Application Naming Convention

Applications follow pattern: `<operator-name>-<cluster-name>`

**Examples:**
- `keda-operator-aks-ikv-nonprod-vnext`
- `rabbitmq-operator-aks-ikv-prod`
- `opentelemetry-operator-aks-commonground-prod`

---

## Migration from Legacy

If you have the old structure with hardcoded ApplicationSets, follow these steps:

### Step 1: Verify New Structure

```bash
# Check operators directory exists
ls -la operators/*/

# Check ApplicationSet exists
ls -la argocd/applicationset-operators.yaml

# Validate .argocd.yaml files
for f in operators/**/.argocd.yaml; do
  yq eval '.' $f > /dev/null && echo "✓ $f" || echo "✗ $f"
done
```

### Step 2: Remove Old ApplicationSets

**⚠️ Do this during a maintenance window**

```bash
# Delete old ApplicationSets (does NOT delete running operators)
kubectl delete applicationset ikv-operators -n argocd
kubectl delete applicationset commonground-operators -n argocd

# Verify Applications are still running
kubectl get applications -n argocd | grep operator
```

### Step 3: Deploy New ApplicationSet

```bash
# Apply discovery-based ApplicationSet
kubectl apply -f argocd/applicationset-operators.yaml

# Verify creation
kubectl get applicationset -n argocd

# Watch applications sync
argocd app list --watch
```

### Step 4: Archive Legacy Files

```bash
# Move to archive after successful migration
mkdir -p .archive/legacy
mv argocd/applicationset-ikv.yaml .archive/legacy/ 2>/dev/null
mv argocd/applicationset-commonground.yaml .archive/legacy/ 2>/dev/null
mv helm-values/ .archive/legacy/ 2>/dev/null

git add .archive/
git commit -m "chore: archive legacy configuration files"
```

---

## Troubleshooting

### Issue: Applications Not Created

**Symptoms:** No applications appear after deploying ApplicationSet

**Solutions:**

```bash
# 1. Check ApplicationSet exists
kubectl get applicationset -n argocd

# 2. Check ApplicationSet status
kubectl describe applicationset platform-operators -n argocd

# 3. Check ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-applicationset-controller

# 4. Verify Git repository is accessible
argocd repo list

# 5. Validate .argocd.yaml files
for f in operators/**/.argocd.yaml; do
  echo "Validating $f"
  yq eval '.' $f > /dev/null || echo "ERROR in $f"
done
```

### Issue: Wrong Version Deployed

**Symptoms:** Deployed version doesn't match `.argocd.yaml`

**Solutions:**

```bash
# 1. Check .argocd.yaml syntax
yq '.clusters.aks-ikv-nonprod-vnext' operators/keda/.argocd.yaml

# 2. Force refresh
argocd app get keda-operator-aks-ikv-nonprod-vnext --refresh

# 3. Check Application manifest
argocd app manifests keda-operator-aks-ikv-nonprod-vnext | grep targetRevision

# 4. Hard refresh (recreate from ApplicationSet)
kubectl delete application keda-operator-aks-ikv-nonprod-vnext -n argocd
# ApplicationSet recreates it within 3 minutes
```

### Issue: Values Not Applied

**Symptoms:** Environment-specific values not taking effect

**Solutions:**

```bash
# 1. Check file naming matches cluster name
ls operators/keda/values.aks-ikv-*

# Expected:
# values.aks-ikv-nonprod-vnext.yaml
# values.aks-ikv-nonprod.yaml
# values.aks-ikv-prod.yaml

# 2. Verify values file syntax
yq eval '.' operators/keda/values.aks-ikv-prod.yaml

# 3. Check Application sources
argocd app get keda-operator-aks-ikv-prod -o yaml | grep -A 20 sources:

# 4. View effective values
helm get values keda -n keda-system --kube-context aks-ikv-prod
```

### Issue: Sync Failures

**Symptoms:** Application shows "OutOfSync" or "Failed"

**Solutions:**

```bash
# 1. Check sync status
argocd app get <app-name>

# 2. View detailed error
argocd app get <app-name> -o yaml | grep -A 20 conditions:

# 3. Check pod events
kubectl describe pod -n <namespace> <pod-name>

# 4. Force sync
argocd app sync <app-name> --force

# 5. Check CRD installation
kubectl get crd | grep <operator>
```

### Common Error Messages

**"git repository not accessible"**
- Solution: Check ArgoCD has Git credentials: `argocd repo list`

**"helm chart not found"**
- Solution: Verify Helm repo added: `argocd repo list | grep helm`

**"cluster not found"**
- Solution: Check cluster registered: `argocd cluster list`

**".argocd.yaml not found"**
- Solution: Verify file exists with dot prefix: `ls operators/*/.argocd.yaml`

---

## Best Practices

### Git Commit Messages

Follow conventional commits:

```bash
# Features/Upgrades
feat(keda): upgrade to 2.15.0 in nonprod-vnext
feat(keda): promote 2.15.0 to nonprod
feat(keda): promote 2.15.0 to production

# Fixes
fix(rabbitmq): rollback to 4.3.0 due to memory leak

# Configuration
config(opentelemetry): adjust resource limits in prod

# New operators
feat(my-operator): add new operator
```

### Version Validation Checklist

Before promoting to production:

- [ ] **Test Coverage**: Run integration tests in nonprod-vnext
- [ ] **Metric Baseline**: Compare metrics before/after upgrade
- [ ] **Log Review**: Check for errors/warnings in logs
- [ ] **Resource Usage**: Monitor CPU/memory consumption
- [ ] **Soak Period**: Minimum 1 week in nonprod-vnext
- [ ] **Load Testing**: Validate under production-like load
- [ ] **Documentation**: Update changelog
- [ ] **Team Review**: Get approval from team members
- [ ] **Rollback Plan**: Document rollback procedure
- [ ] **Communication**: Notify stakeholders

### Change Windows

| Environment | Change Window | Purpose |
|-------------|---------------|---------|
| local-dev | Anytime (24/7) | Local development and testing |
| nonprod-vnext | Anytime (24/7) | Cloud testing environment |
| nonprod | Business hours preferred | Validation environment |
| prod | Scheduled maintenance windows only | Production systems |

### Monitoring After Upgrades

```bash
# Application health
argocd app get <operator>-<cluster>

# Pod status
kubectl get pods -n <namespace> -w

# Logs
kubectl logs -n <namespace> deployment/<operator> --tail=100 -f

# Metrics (if Prometheus available)
# - CPU/Memory usage
# - Error rates
# - Request latency
# - Custom operator metrics
```

### Security Best Practices

- Always pin exact versions (never use `latest`)
- Review Helm chart changes before upgrading
- Use Pull Requests for all version changes
- Require approvals for production changes
- Monitor Git audit trail
- Use Azure Workload Identity for pod authentication
- Keep secrets in Azure Key Vault
- Enable ArgoCD RBAC

### Scalability Guidelines

**When adding many operators:**
- Keep one `.argocd.yaml` per operator
- Use consistent naming conventions
- Document operator purpose in README
- Group related operators in subdirectories if needed

**When managing many clusters:**
- Use cluster labels for filtering
- Consider separate ApplicationSets per cluster group
- Document cluster purposes clearly
- Maintain cluster inventory

---

## Summary

This GitOps infrastructure follows **industry best practices** and provides:

✅ **Easy Version Management** - Single file updates  
✅ **Safe Deployments** - Clear promotion path  
✅ **Full Visibility** - Git-tracked changes  
✅ **Auto-Discovery** - Add operators by creating directories  
✅ **Proven Patterns** - Based on Commonground Haven+ reference  

**Key Files to Remember:**
- `operators/{name}/.argocd.yaml` - Version control
- `operators/{name}/values.yaml` - Base configuration
- `operators/{name}/values.{cluster}.yaml` - Environment overrides
- `clusters/{group}/{env}/root.yaml` - Bootstrap clusters
- `argocd/applicationset-operators.yaml` - Discovery engine

**Need Help?**
- Check troubleshooting section above
- Review `.argocd.yaml` files for examples
- Check ArgoCD UI for sync status
- Review Git commits for version history

---

**Last Updated:** February 2026  
**Architecture Version:** 2.0 (Discovery-Based)
