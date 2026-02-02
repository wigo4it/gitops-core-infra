# RabbitMQ Cluster Operator

## Overview

The RabbitMQ Cluster Kubernetes Operator automates provisioning, management, and operations of RabbitMQ clusters running on Kubernetes. It provides a declarative way to deploy and manage RabbitMQ clusters with custom resources.

**Official Documentation:** https://www.rabbitmq.com/kubernetes/operator/operator-overview.html  
**Helm Chart:** https://github.com/bitnami/charts/tree/main/bitnami/rabbitmq-cluster-operator  
**GitHub:** https://github.com/rabbitmq/cluster-operator

## What Does RabbitMQ Cluster Operator Do?

The operator manages the complete lifecycle of RabbitMQ clusters:

- **Automated Deployment**: Deploy RabbitMQ clusters with simple CRDs
- **Configuration Management**: Manage plugins, policies, and RabbitMQ configuration
- **Scaling**: Scale clusters up/down dynamically
- **Upgrades**: Safe, automated rolling upgrades
- **Monitoring**: Integration with Prometheus for metrics
- **High Availability**: Multi-node clusters with automatic failover
- **Storage Management**: Persistent volume management for message durability

### Common Use Cases

✅ **Message Queuing**: Asynchronous communication between microservices  
✅ **Event Streaming**: Publish-subscribe patterns for event-driven architectures  
✅ **Task Queues**: Distribute work across multiple workers  
✅ **Request/Reply**: RPC-style communication with correlation  
✅ **Topic Routing**: Complex message routing with topic exchanges  

## Architecture

```
┌────────────────────────────────────────────────────────┐
│            RabbitMQ Cluster Operator                   │
├────────────────────────────────────────────────────────┤
│                                                         │
│  ┌────────────────┐                                    │
│  │   Operator     │────▶ Watches RabbitmqCluster CRDs  │
│  │   Controller   │                                    │
│  └────────────────┘                                    │
│         │                                               │
│         ▼                                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │         Creates/Manages Resources:              │   │
│  │  • StatefulSet (RabbitMQ nodes)                 │   │
│  │  • Services (client, headless)                  │   │
│  │  • ConfigMaps (RabbitMQ config)                 │   │
│  │  • Secrets (default user credentials)           │   │
│  │  • PersistentVolumeClaims (storage)             │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└────────────────────────────────────────────────────────┘
                         │
                         ▼
          ┌─────────────────────────────────┐
          │      RabbitMQ Cluster           │
          │  ┌─────────┐  ┌─────────┐      │
          │  │  Node 1 │  │  Node 2 │      │
          │  │(Master) │──│(Mirror) │      │
          │  └─────────┘  └─────────┘      │
          │       │            │            │
          │       └────────────┘            │
          │    Clustered Messaging          │
          └─────────────────────────────────┘
```

## Deployment Configuration

### Version Management

Versions are managed per-cluster in [.argocd.yaml](.argocd.yaml):

```yaml
clusters:
  local-dev:
    chartVersion: "4.3.0"       # Latest for testing
  aks-ikv-nonprod-vnext:
    chartVersion: "4.3.0"       # Testing
  aks-ikv-nonprod:
    chartVersion: "4.3.0"       # Validated
  aks-ikv-prod:
    chartVersion: "4.2.5"       # Stable production
```

**Current Chart Version:** `4.3.0` (nonprod) / `4.2.5` (prod)  
**Chart Repository:** https://charts.bitnami.com/bitnami  
**Target Namespace:** `rabbitmq-system`

### Configuration Files

| File | Purpose | Applies To |
|------|---------|------------|
| [values.yaml](values.yaml) | Base configuration | All clusters |
| [values.local-dev.yaml](values.local-dev.yaml) | Local development overrides | Local testing |
| [values.aks-ikv-nonprod-vnext.yaml](values.aks-ikv-nonprod-vnext.yaml) | IKV testing environment | nonprod-vnext |
| [values.aks-ikv-nonprod.yaml](values.aks-ikv-nonprod.yaml) | IKV validation environment | nonprod |
| [values.aks-ikv-prod.yaml](values.aks-ikv-prod.yaml) | IKV production | prod |
| [values.aks-commonground-nonprod.yaml](values.aks-commonground-nonprod.yaml) | Commonground pre-prod | nonprod |
| [values.aks-commonground-prod.yaml](values.aks-commonground-prod.yaml) | Commonground production | prod |

## Key Configuration Options

### Resource Limits

**Default (base):**
```yaml
resources:
  limits: { cpu: 500m, memory: 512Mi }
  requests: { cpu: 100m, memory: 256Mi }
```

**Production:** Higher limits for operator stability  
**Local:** Reduced resources for development

### Operator Configuration

The operator itself runs as a single deployment and manages multiple RabbitMQ cluster instances.

## Creating RabbitMQ Clusters

After the operator is deployed, you can create RabbitMQ clusters using the `RabbitmqCluster` CRD:

### Basic Cluster Example

```yaml
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: my-rabbitmq
  namespace: default
spec:
  replicas: 3  # 3-node cluster
  
  rabbitmq:
    additionalPlugins:
      - rabbitmq_management
      - rabbitmq_prometheus
    
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1000m
      memory: 2Gi
  
  persistence:
    storageClassName: default
    storage: 10Gi
  
  service:
    type: ClusterIP
```

### Production Cluster Example

```yaml
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: production-rabbitmq
  namespace: messaging
spec:
  replicas: 5  # 5-node cluster for HA
  
  rabbitmq:
    additionalPlugins:
      - rabbitmq_management
      - rabbitmq_prometheus
      - rabbitmq_shovel
      - rabbitmq_federation
    
    additionalConfig: |
      cluster_partition_handling = pause_minority
      queue_master_locator = min-masters
      disk_free_limit.absolute = 2GB
  
  resources:
    requests:
      cpu: 1000m
      memory: 2Gi
    limits:
      cpu: 2000m
      memory: 4Gi
  
  persistence:
    storageClassName: premium-ssd
    storage: 50Gi
  
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app.kubernetes.io/name: production-rabbitmq
        topologyKey: kubernetes.io/hostname
  
  override:
    statefulSet:
      spec:
        template:
          spec:
            containers:
            - name: rabbitmq
              env:
              - name: RABBITMQ_ERLANG_COOKIE
                valueFrom:
                  secretKeyRef:
                    name: rabbitmq-erlang-cookie
                    key: cookie
```

## Upgrading RabbitMQ Cluster Operator

### Step-by-Step Process

**1. Test locally:**
```bash
vim .argocd.yaml
# Change: local-dev: chartVersion: "4.3.0" → "4.4.0"

git commit -am "test(rabbitmq): upgrade operator to 4.4.0 locally"
git push
```

**2. Validate existing clusters still work:**
```bash
kubectl get rabbitmqclusters --all-namespaces
kubectl describe rabbitmqcluster <name> -n <namespace>
```

**3. Promote through environments:** local-dev → nonprod-vnext → nonprod → prod

### Version Compatibility

Check compatibility matrix:  
https://www.rabbitmq.com/kubernetes/operator/operator-overview.html#compatibility

| Operator Version | RabbitMQ Version | Kubernetes Version |
|-----------------|------------------|--------------------|
| 4.3.x | 3.13.x | 1.26+ |
| 4.2.x | 3.12.x | 1.25+ |
| 4.1.x | 3.11.x | 1.24+ |

## Monitoring

### Health Checks

```bash
# Check operator status
kubectl get pods -n rabbitmq-system

# View operator logs
kubectl logs -n rabbitmq-system -l app.kubernetes.io/name=rabbitmq-cluster-operator --tail=100

# List RabbitMQ clusters
kubectl get rabbitmqclusters --all-namespaces

# Check specific cluster
kubectl describe rabbitmqcluster <name> -n <namespace>

# Check RabbitMQ pods
kubectl get pods -n <namespace> -l app.kubernetes.io/name=<cluster-name>
```

### RabbitMQ Cluster Status

```bash
# Get cluster status
kubectl get rabbitmqcluster <name> -n <namespace> -o yaml

# Check events
kubectl get events -n <namespace> --field-selector involvedObject.name=<cluster-name>

# Access RabbitMQ management UI
kubectl port-forward svc/<cluster-name> 15672:15672 -n <namespace>
# Open: http://localhost:15672
# Default credentials from secret: kubectl get secret <cluster-name>-default-user
```

### Prometheus Metrics

RabbitMQ exposes metrics at:
- `rabbitmq_queue_messages` - Message count per queue
- `rabbitmq_queue_messages_ready` - Messages ready for delivery
- `rabbitmq_queue_consumers` - Active consumers
- `rabbitmq_node_mem_used` - Memory usage
- `rabbitmq_node_disk_free` - Free disk space

## Common Operations

### Scale RabbitMQ Cluster

```bash
# Edit cluster spec
kubectl edit rabbitmqcluster <name> -n <namespace>
# Change: replicas: 3 → replicas: 5

# Or patch
kubectl patch rabbitmqcluster <name> -n <namespace> --type='json' \
  -p='[{"op": "replace", "path": "/spec/replicas", "value": 5}]'
```

### Update RabbitMQ Configuration

```bash
# Edit cluster to add/modify additionalConfig
kubectl edit rabbitmqcluster <name> -n <namespace>
```

### Access RabbitMQ Logs

```bash
# View logs from specific RabbitMQ node
kubectl logs <cluster-name>-server-0 -n <namespace>

# Follow logs
kubectl logs <cluster-name>-server-0 -n <namespace> -f

# All nodes
kubectl logs -l app.kubernetes.io/name=<cluster-name> -n <namespace> --all-containers
```

## Troubleshooting

### Operator Not Creating Clusters

**Symptoms:** RabbitmqCluster CRD created but no pods appear

**Check:**
```bash
# Verify CRD is installed
kubectl get crd rabbitmqclusters.rabbitmq.com

# Check operator logs
kubectl logs -n rabbitmq-system -l app.kubernetes.io/name=rabbitmq-cluster-operator -f

# Check cluster status
kubectl describe rabbitmqcluster <name> -n <namespace>
```

### Cluster Nodes Not Joining

**Symptoms:** RabbitMQ pods running but not forming cluster

**Check:**
```bash
# Check Erlang cookie secret
kubectl get secret <cluster-name>-erlang-cookie -n <namespace>

# View RabbitMQ node logs
kubectl logs <cluster-name>-server-0 -n <namespace>

# Check network connectivity between pods
kubectl exec <cluster-name>-server-0 -n <namespace> -- rabbitmq-diagnostics ping
```

### Storage Issues

**Symptoms:** Pods stuck in Pending, disk warnings

**Check:**
```bash
# Check PVC status
kubectl get pvc -n <namespace>

# View PVC details
kubectl describe pvc persistence-<cluster-name>-server-0 -n <namespace>

# Check storage class
kubectl get storageclass
```

## Best Practices

✅ **Production Clusters**: Use 3-5 replicas for HA  
✅ **Persistence**: Always enable for production workloads  
✅ **Resources**: Monitor and adjust based on message throughput  
✅ **Anti-Affinity**: Distribute nodes across availability zones  
✅ **Monitoring**: Enable Prometheus plugin and set up alerts  
✅ **Backups**: Implement backup strategy for critical queues  
✅ **Upgrades**: Test in non-production first, use rolling upgrades  

## Additional Resources

- **Operator Docs:** https://www.rabbitmq.com/kubernetes/operator/operator-overview.html
- **RabbitMQ Docs:** https://www.rabbitmq.com/documentation.html
- **GitHub:** https://github.com/rabbitmq/cluster-operator
- **Bitnami Chart:** https://github.com/bitnami/charts/tree/main/bitnami/rabbitmq-cluster-operator
- **Community:** https://groups.google.com/forum/#!forum/rabbitmq-users

## Support

For operator-specific issues:
1. Check operator logs in `rabbitmq-system` namespace
2. Review RabbitmqCluster resource status and events
3. Verify CRD installation
4. Check RBAC permissions

For RabbitMQ cluster issues:
- GitHub Issues: https://github.com/rabbitmq/cluster-operator/issues
- RabbitMQ Community: https://groups.google.com/forum/#!forum/rabbitmq-users
