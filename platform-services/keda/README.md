# KEDA - Kubernetes Event Driven Autoscaler

## Overview

KEDA is a Kubernetes-based Event Driven Autoscaler. With KEDA, you can drive the scaling of any container in Kubernetes based on the number of events needing to be processed.

**Official Documentation:** https://keda.sh/  
**Helm Chart:** https://github.com/kedacore/charts

## What Does KEDA Do?

KEDA extends Kubernetes scaling capabilities beyond basic CPU/Memory metrics:

- **Event-Driven Scaling**: Scale workloads based on external event sources
- **Scalers**: 60+ built-in scalers (Azure Queue, RabbitMQ, Kafka, PostgreSQL, Prometheus, etc.)
- **Scale to Zero**: Reduce costs by scaling to zero replicas when idle
- **Custom Metrics**: Expose custom metrics to HPA (Horizontal Pod Autoscaler)
- **Serverless Framework**: Build serverless applications on Kubernetes

### Common Use Cases

✅ **Message Queue Processing**: Scale based on Azure Service Bus, RabbitMQ, SQS queue depth  
✅ **Database Operations**: Scale workers based on PostgreSQL, MySQL, MongoDB query results  
✅ **Event Streaming**: Scale consumers based on Kafka lag, Azure Event Hubs  
✅ **Cron Jobs**: Schedule-based scaling for periodic workloads  
✅ **Custom Metrics**: Scale based on Prometheus metrics, external APIs  

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    KEDA Components                       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────┐      ┌──────────────┐                │
│  │ KEDA Operator│─────▶│ ScaledObject │                │
│  └──────────────┘      └──────────────┘                │
│         │                      │                        │
│         │                      ▼                        │
│         │              ┌──────────────┐                │
│         └─────────────▶│     HPA      │                │
│                        └──────────────┘                │
│                               │                         │
│                               ▼                         │
│                        ┌──────────────┐                │
│                        │  Deployment  │                │
│                        │  StatefulSet │                │
│                        └──────────────┘                │
│                                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │         KEDA Metrics Server                     │   │
│  │  (Exposes custom metrics to Kubernetes HPA)     │   │
│  └─────────────────────────────────────────────────┘   │
│                                                          │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │  External Systems    │
              │  - Azure Queue       │
              │  - RabbitMQ          │
              │  - Kafka             │
              │  - PostgreSQL        │
              │  - Prometheus        │
              └──────────────────────┘
```

## Deployment Configuration

### Version Management

Versions are managed per-cluster in [.argocd.yaml](.argocd.yaml):

```yaml
clusters:
  local-dev:
    chartVersion: "2.14.0"      # Latest for testing
  aks-ikv-nonprod-vnext:
    chartVersion: "2.14.0"      # Testing
  aks-ikv-nonprod:
    chartVersion: "2.14.0"      # Validated
  aks-ikv-prod:
    chartVersion: "2.13.0"      # Stable production
```

**Current Chart Version:** `2.14.0` (nonprod) / `2.13.0` (prod)  
**Chart Repository:** https://kedacore.github.io/charts  
**Target Namespace:** `keda-system`

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
  operator:
    limits: { cpu: 500m, memory: 512Mi }
    requests: { cpu: 100m, memory: 128Mi }
  metricServer:
    limits: { cpu: 500m, memory: 512Mi }
    requests: { cpu: 100m, memory: 128Mi }
```

**Production overrides:** Higher limits, HA configuration  
**Local overrides:** Reduced resources for laptop development

### High Availability (Production)

Production environments use:
- Multiple replicas (2-3)
- Pod anti-affinity for node distribution
- Pod disruption budgets
- Increased resource limits

## Upgrading KEDA

### Step-by-Step Process

**1. Test locally first:**
```bash
# Edit .argocd.yaml
vim .argocd.yaml
# Change: local-dev: chartVersion: "2.14.0" → "2.15.0"

git commit -am "test(keda): upgrade to 2.15.0 locally"
git push
```

**2. Promote to nonprod-vnext:**
```bash
# After local validation
vim .argocd.yaml
# Change: aks-ikv-nonprod-vnext: chartVersion: "2.15.0"

git commit -am "feat(keda): upgrade to 2.15.0 in nonprod-vnext"
git push
```

**3. Soak period (1-2 weeks):**
- Monitor metrics
- Check scaler functionality
- Run integration tests

**4. Promote to nonprod, then production**

### Version Compatibility

Check compatibility between KEDA version and Kubernetes version:  
https://keda.sh/docs/latest/operate/cluster/

| KEDA Version | Kubernetes Version |
|--------------|-------------------|
| 2.14.x | 1.27+ |
| 2.13.x | 1.26+ |
| 2.12.x | 1.25+ |

## Monitoring

### Health Checks

```bash
# Check operator status
kubectl get pods -n keda-system

# View operator logs
kubectl logs -n keda-system -l app.kubernetes.io/name=keda-operator --tail=100

# Check metrics server
kubectl logs -n keda-system -l app.kubernetes.io/name=keda-metrics-apiserver --tail=100

# List ScaledObjects
kubectl get scaledobjects --all-namespaces

# Check specific ScaledObject
kubectl describe scaledobject <name> -n <namespace>
```

### Metrics

KEDA exposes Prometheus metrics:
- `keda_scaler_errors_total` - Scaler errors
- `keda_scaler_metrics_value` - Current metric values
- `keda_scaled_object_errors` - ScaledObject errors

## Common ScaledObject Examples

### Azure Service Bus Queue
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-servicebus-queue-scaledobject
spec:
  scaleTargetRef:
    name: azure-consumer-deployment
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: myqueue
      messageCount: "5"
```

### RabbitMQ Queue
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-scaledobject
spec:
  scaleTargetRef:
    name: rabbitmq-consumer
  triggers:
  - type: rabbitmq
    metadata:
      host: amqp://rabbitmq.default.svc.cluster.local:5672
      queueName: my-queue
      queueLength: "10"
```

### Cron Schedule
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: cron-scaledobject
spec:
  scaleTargetRef:
    name: scheduled-job
  minReplicaCount: 0
  maxReplicaCount: 10
  triggers:
  - type: cron
    metadata:
      timezone: Europe/Amsterdam
      start: 0 8 * * *
      end: 0 18 * * *
      desiredReplicas: "5"
```

## Troubleshooting

### Operator Not Scaling

**Symptoms:** ScaledObject exists but deployment not scaling

**Check:**
```bash
# View ScaledObject status
kubectl describe scaledobject <name> -n <namespace>

# Check HPA created by KEDA
kubectl get hpa -n <namespace>

# View operator logs
kubectl logs -n keda-system -l app.kubernetes.io/name=keda-operator -f
```

### Metrics Not Available

**Symptoms:** Custom metrics not exposed to HPA

**Check:**
```bash
# Check metrics server
kubectl get pods -n keda-system -l app.kubernetes.io/name=keda-metrics-apiserver

# Test metrics API
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1"

# View metrics server logs
kubectl logs -n keda-system -l app.kubernetes.io/name=keda-metrics-apiserver -f
```

### Authentication Issues

**Symptoms:** Scaler cannot authenticate to external system

**Solutions:**
- Check TriggerAuthentication/ClusterTriggerAuthentication resources
- Verify secrets exist and contain correct credentials
- Use Azure Workload Identity for Azure resources
- Check service account permissions

## Additional Resources

- **Official Docs:** https://keda.sh/docs/
- **Scalers Reference:** https://keda.sh/docs/scalers/
- **GitHub:** https://github.com/kedacore/keda
- **Slack Community:** https://kubernetes.slack.com (#keda)
- **Examples:** https://github.com/kedacore/samples

## Support

For issues specific to this deployment:
1. Check operator logs in `keda-system` namespace
2. Review ScaledObject status and events
3. Verify version compatibility with Kubernetes
4. Check resource constraints

For KEDA-specific issues:
- GitHub Issues: https://github.com/kedacore/keda/issues
- Community Support: Kubernetes Slack #keda channel
