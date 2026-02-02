# 🎨 Visual Architecture Diagrams

## 1. Overall System Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                           Developer Workflow                            │
│                                                                          │
│  1. Edit helm-values/overlays/ikv/prod/keda-operator.yaml              │
│  2. git commit -m "Increase KEDA replicas to 3"                        │
│  3. git push origin main                                                │
└────────────────────────────────┬───────────────────────────────────────┘
                                  │
                                  ▼
┌────────────────────────────────────────────────────────────────────────┐
│                          Git Repository                                 │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ helm-values/          argocd/              clusters/             │  │
│  │  ├── base/            ├── applicationset-  ├── ikv/             │  │
│  │  │   ├── keda.yaml    │   ikv.yaml         │   └── prod.yaml   │  │
│  │  │   └── ...          └── applicationset-  └── commonground/    │  │
│  │  └── overlays/            commonground.yaml    └── prod.yaml   │  │
│  │      └── ikv/prod/                                              │  │
│  │          └── keda.yaml                                          │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────┬───────────────────────────────────────┘
                                  │
                                  ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    ArgoCD (Running in Hub Cluster)                      │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ ApplicationSet Controller                                        │  │
│  │  - Watches: argocd/applicationset-*.yaml                        │  │
│  │  - Generates: Individual Application resources per operator     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Application Controller                                           │  │
│  │  - Fetches: Git repo (values) + Helm repo (charts)             │  │
│  │  - Renders: helm template with merged values                    │  │
│  │  - Syncs: To target clusters                                    │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────┬───────────────────┬───────────────────┬──────────────────┘
             │                   │                   │
    ┌────────▼────────┐ ┌───────▼────────┐ ┌───────▼────────┐
    │  Helm Repos     │ │  Git Repo      │ │ Target Clusters│
    │  - kedacore     │ │  (values)      │ │ - ikv-prod     │
    │  - bitnami      │ │                │ │ - cg-prod      │
    │  - open-telem.  │ │                │ │ - etc.         │
    └─────────────────┘ └────────────────┘ └────────────────┘
                                                    │
                                                    ▼
                                  ┌─────────────────────────────┐
                                  │   Deployed Operators        │
                                  │   ├── keda-operator-system  │
                                  │   ├── rabbitmq-op-system    │
                                  │   └── opentelemetry-system  │
                                  └─────────────────────────────┘
```

## 2. Values Inheritance Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    Chart Default Values                          │
│  (Built into Helm chart, e.g., kedacore/keda)                  │
│                                                                  │
│  replicaCount: 1                                                │
│  image:                                                         │
│    repository: ghcr.io/kedacore/keda                          │
│    tag: "2.14.0"                                               │
│  resources:                                                     │
│    requests:                                                    │
│      cpu: 100m                                                 │
│      memory: 100Mi                                             │
└──────────────────────────────┬──────────────────────────────────┘
                               │ Overridden by
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│           Base Values (All Environments)                         │
│  File: helm-values/base/keda-operator.yaml                      │
│                                                                  │
│  resources:                                                     │
│    limits:                                                      │
│      cpu: 500m          # Add limits                           │
│      memory: 512Mi                                             │
│    requests:                                                    │
│      cpu: 100m          # Inherited from chart                 │
│      memory: 128Mi      # Override chart default               │
│  podSecurityContext:                                           │
│    runAsNonRoot: true   # Add security                         │
└──────────────────────────────┬──────────────────────────────────┘
                               │ Overridden by
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│         Environment Overlay (Production Specific)                │
│  File: helm-values/overlays/ikv/prod/keda-operator.yaml        │
│                                                                  │
│  replicaCount: 2        # Override for HA                      │
│  resources:                                                     │
│    limits:                                                      │
│      cpu: 1000m         # Higher for prod                      │
│      memory: 1Gi        # Higher for prod                      │
│  affinity:                                                      │
│    podAntiAffinity:     # Add HA policy                        │
│      ...                                                        │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Final Merged Values                           │
│  (What gets deployed to ikv-prod cluster)                       │
│                                                                  │
│  replicaCount: 2                    # From overlay              │
│  image:                                                         │
│    repository: ghcr.io/kedacore/keda  # From chart default    │
│    tag: "2.14.0"                    # From chart default       │
│  resources:                                                     │
│    limits:                                                      │
│      cpu: 1000m                     # From overlay             │
│      memory: 1Gi                    # From overlay             │
│    requests:                                                    │
│      cpu: 100m                      # From base                │
│      memory: 128Mi                  # From base                │
│  podSecurityContext:                                           │
│    runAsNonRoot: true               # From base                │
│  affinity:                                                      │
│    podAntiAffinity: ...             # From overlay             │
└─────────────────────────────────────────────────────────────────┘
```

## 3. Multi-Cluster Deployment Flow

```
┌────────────────────────────────────────────────────────────────┐
│            ApplicationSet: ikv-operators                        │
│                                                                  │
│  Generates 9 Applications (3 clusters × 3 operators):          │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Matrix Generator                                         │  │
│  │  Clusters:                    Operators:                 │  │
│  │  - ikv-nonprod-vnext    ×    - keda-operator            │  │
│  │  - ikv-nonprod                - rabbitmq-operator        │  │
│  │  - ikv-prod                   - redir-operator           │  │
│  │                                                           │  │
│  │  = 9 Application resources generated                     │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         │                        │                        │
         ▼                        ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Application:    │    │ Application:    │    │ Application:    │
│ keda-operator-  │    │ rabbitmq-op-    │    │ redir-operator- │
│ ikv-nonprod-    │    │ ikv-nonprod-    │    │ ikv-nonprod-    │
│ vnext           │    │ vnext           │    │ vnext           │
│                 │    │                 │    │                 │
│ Sources:        │    │ Sources:        │    │ Sources:        │
│ 1. Git (values) │    │ 1. Git (values) │    │ 1. Git (values) │
│ 2. Helm chart   │    │ 2. Helm chart   │    │ 2. Git (raw)    │
│                 │    │                 │    │                 │
│ Destination:    │    │ Destination:    │    │ Destination:    │
│ ikv-nonprod-    │    │ ikv-nonprod-    │    │ ikv-nonprod-    │
│ vnext cluster   │    │ vnext cluster   │    │ vnext cluster   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                        │                        │
         └────────────────────────┼────────────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  ikv-nonprod-vnext       │
                    │  AKS Cluster             │
                    │                          │
                    │  Namespaces:             │
                    │  ├── keda-system         │
                    │  ├── rabbitmq-op-system  │
                    │  └── redir-op-system     │
                    └──────────────────────────┘
```

## 4. Upgrade Flow

```
Developer Action:
┌────────────────────────────────────────────────────────────┐
│ 1. Edit: argocd/applicationset-ikv.yaml                    │
│    - name: keda-operator                                   │
│      version: "2.15.0"  ← Change from 2.14.0              │
│                                                            │
│ 2. git commit -m "Upgrade KEDA to 2.15.0"                 │
│ 3. git push origin main                                    │
└──────────────────────────────┬─────────────────────────────┘
                               │
                               ▼
ArgoCD Detects Change:
┌────────────────────────────────────────────────────────────┐
│ ApplicationSet Controller:                                  │
│ - Detects ApplicationSet changed                           │
│ - Regenerates 9 Application resources                      │
│ - Updates each Application with new chart version          │
└──────────────────────────────┬─────────────────────────────┘
                               │
                               ▼
ArgoCD Syncs (Sequential by Environment):
┌────────────────────────────────────────────────────────────┐
│                                                             │
│  Step 1: nonprod-vnext (First Environment)                 │
│  ┌────────────────────────────────────────────────────┐   │
│  │ - Fetch kedacore/keda:2.15.0                       │   │
│  │ - Merge with values files                          │   │
│  │ - Generate manifests                               │   │
│  │ - Apply to ikv-nonprod-vnext cluster               │   │
│  │ - Health check: ✅ Healthy                         │   │
│  └────────────────────────────────────────────────────┘   │
│           │ Success ✅                                      │
│           ▼                                                 │
│  Step 2: nonprod (Second Environment)                      │
│  ┌────────────────────────────────────────────────────┐   │
│  │ - Same process for ikv-nonprod                     │   │
│  │ - Health check: ✅ Healthy                         │   │
│  └────────────────────────────────────────────────────┘   │
│           │ Success ✅                                      │
│           ▼                                                 │
│  Step 3: prod (Final Environment)                          │
│  ┌────────────────────────────────────────────────────┐   │
│  │ - Same process for ikv-prod                        │   │
│  │ - Health check: ✅ Healthy                         │   │
│  └────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
All Clusters Upgraded ✅
┌────────────────────────────────────────────────────────────┐
│ Result:                                                     │
│ - All IKV clusters now running KEDA 2.15.0                │
│ - All Commonground clusters now running KEDA 2.15.0       │
│ - Total time: ~5-10 minutes                                │
│ - Zero downtime (rolling updates)                          │
└────────────────────────────────────────────────────────────┘
```

## 5. Comparison: Before vs After

### Before (Kustomize-based)

```
Update KEDA across all clusters:
1. Edit 5 deployment.yaml files manually
2. Update image tags in each file
3. Edit 5 kustomization.yaml files
4. Commit 10+ files
5. ArgoCD syncs raw manifests
6. Manual CRD management
7. Hope nothing breaks
```

### After (Helm-based)

```
Update KEDA across all clusters:
1. Edit 1 line in ApplicationSet (version: "2.15.0")
2. Commit 1 file
3. ArgoCD pulls official chart
4. Helm handles CRDs automatically
5. Rolling update across all clusters
6. Done! ✅
```

## 6. Progressive Delivery Visual

```
                    Commit Change to Git
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │   Stage 1: nonprod-vnext (Testing)    │
        │   - Lower resources                    │
        │   - Debug logging                      │
        │   - 1 replica                          │
        │   - Canary for issues                  │
        └──────────────┬────────────────────────┘
                       │ ✅ Validated
                       ▼
        ┌───────────────────────────────────────┐
        │   Stage 2: nonprod (Pre-prod)         │
        │   - Moderate resources                 │
        │   - Info logging                       │
        │   - 1 replica                          │
        │   - Final validation                   │
        └──────────────┬────────────────────────┘
                       │ ✅ Validated
                       ▼
        ┌───────────────────────────────────────┐
        │   Stage 3: prod (Production)          │
        │   - High resources                     │
        │   - Info logging                       │
        │   - 2+ replicas (HA)                   │
        │   - Anti-affinity rules                │
        │   - Production-grade SLAs              │
        └───────────────────────────────────────┘
                       │
                       ▼
                   Deployed ✅
```

---

These diagrams show the complete flow from developer commit to production deployment using the Helm-based GitOps architecture.
