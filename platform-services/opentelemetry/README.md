# OpenTelemetry Operator

## Overview

The OpenTelemetry Operator is a Kubernetes operator for managing OpenTelemetry Collector and auto-instrumentation of workloads. It simplifies the deployment and configuration of OpenTelemetry components in Kubernetes.

**Official Documentation:** https://opentelemetry.io/docs/k8s-operator/  
**Helm Chart:** https://github.com/open-telemetry/opentelemetry-helm-charts  
**GitHub:** https://github.com/open-telemetry/opentelemetry-operator

## What Does OpenTelemetry Operator Do?

The operator manages OpenTelemetry infrastructure in Kubernetes:

- **Collector Management**: Deploy and manage OpenTelemetry Collectors as DaemonSets, Deployments, or StatefulSets
- **Auto-Instrumentation**: Automatically inject telemetry into applications without code changes
- **Configuration**: Manage collector pipelines and exporters through CRDs
- **Scaling**: Automatic scaling of collector instances based on load
- **Upgrades**: Manage collector version upgrades across the cluster

### Common Use Cases

✅ **Distributed Tracing**: Collect traces from microservices  
✅ **Metrics Collection**: Gather metrics from applications and infrastructure  
✅ **Log Aggregation**: Centralize logs with context  
✅ **Auto-Instrumentation**: Zero-code instrumentation for Java, Node.js, Python, .NET  
✅ **Multi-Backend**: Send telemetry to multiple backends (Prometheus, Jaeger, Grafana, Azure Monitor)  

## Architecture

```
┌────────────────────────────────────────────────────────┐
│         OpenTelemetry Operator                         │
├────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────────┐                                   │
│  │   Operator      │─────▶ Watches CRDs:               │
│  │   Controller    │       • OpenTelemetryCollector    │
│  └─────────────────┘       • Instrumentation           │
│         │                                               │
│         ▼                                               │
│  ┌──────────────────────────────────────────────────┐  │
│  │     Creates/Manages:                             │  │
│  │  • Collector Deployments/DaemonSets              │  │
│  │  • ConfigMaps (collector config)                 │  │
│  │  • Services (OTLP receivers)                     │  │
│  │  • Auto-instrumentation webhooks                 │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
└────────────────────────────────────────────────────────┘
                         │
                         ▼
          ┌─────────────────────────────────┐
          │   OpenTelemetry Collectors      │
          │  ┌──────────┐  ┌──────────┐    │
          │  │Gateway   │  │DaemonSet │    │
          │  │Collector │  │Collector │    │
          │  └──────────┘  └──────────┘    │
          │        │            │           │
          │        └────────────┘           │
          │     Receivers → Processors      │
          │              ↓                  │
          │         Exporters               │
          └─────────────────────────────────┘
                         │
                         ▼
          ┌─────────────────────────────────┐
          │   Observability Backends        │
          │  • Prometheus                   │
          │  • Jaeger                       │
          │  • Azure Monitor                │
          │  • Grafana                      │
          └─────────────────────────────────┘
```

## Deployment Configuration

### Version Management

Versions are managed per-cluster in [.argocd.yaml](.argocd.yaml):

```yaml
clusters:
  local-dev:
    enabled: true
    chartVersion: "0.40.0"      # Latest for testing
  # IKV Clusters - Disabled
  aks-ikv-*:
    enabled: false              # Not deployed on IKV
  # Commonground Clusters - Enabled
  aks-commonground-nonprod:
    enabled: true
    chartVersion: "0.40.0"
  aks-commonground-prod:
    enabled: true
    chartVersion: "0.39.0"
```

**Deployment Scope:** Commonground clusters + local development only  
**Current Chart Version:** `0.40.0` (nonprod) / `0.39.0` (prod)  
**Chart Repository:** https://open-telemetry.github.io/opentelemetry-helm-charts  
**Target Namespace:** `opentelemetry-system`

### Configuration Files

| File | Purpose | Applies To |
|------|---------|------------|
| [values.yaml](values.yaml) | Base configuration | All enabled clusters |
| [values.local-dev.yaml](values.local-dev.yaml) | Local development overrides | Local testing |
| [values.aks-commonground-nonprod.yaml](values.aks-commonground-nonprod.yaml) | Commonground pre-prod | nonprod |
| [values.aks-commonground-prod.yaml](values.aks-commonground-prod.yaml) | Commonground production | prod |

**Note:** No IKV values files - operator is disabled for IKV clusters

## Key Configuration Options

### Resource Limits

**Default (base):**
```yaml
manager:
  resources:
    limits: { cpu: 200m, memory: 256Mi }
    requests: { cpu: 50m, memory: 128Mi }
```

**Production:** Higher limits for operator stability  
**Local:** Minimal resources for development

### Operator Features

```yaml
# Enable admission webhooks for auto-instrumentation
admissionWebhooks:
  enabled: true
  autoGenerateCert: true

# Prometheus metrics
manager:
  prometheusRule:
    enabled: true
```

## Creating OpenTelemetry Collectors

After the operator is deployed, you can create collectors using the `OpenTelemetryCollector` CRD:

### Gateway Collector Example

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-gateway
  namespace: observability
spec:
  mode: deployment  # Gateway mode
  replicas: 3
  
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:
    
    processors:
      batch:
        timeout: 10s
        send_batch_size: 1024
      memory_limiter:
        limit_mib: 512
        spike_limit_mib: 128
    
    exporters:
      prometheus:
        endpoint: "0.0.0.0:8889"
      jaeger:
        endpoint: jaeger-collector:14250
        tls:
          insecure: true
      azuremonitor:
        connection_string: "${APPLICATIONINSIGHTS_CONNECTION_STRING}"
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [jaeger, azuremonitor]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [prometheus, azuremonitor]
  
  resources:
    limits:
      cpu: 1000m
      memory: 2Gi
    requests:
      cpu: 500m
      memory: 1Gi
```

### DaemonSet Collector Example

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-agent
  namespace: observability
spec:
  mode: daemonset  # Run on every node
  
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
      prometheus:
        config:
          scrape_configs:
          - job_name: 'kubernetes-pods'
            kubernetes_sd_configs:
            - role: pod
    
    processors:
      batch:
      resource:
        attributes:
        - key: cluster.name
          value: commonground-prod
          action: upsert
    
    exporters:
      otlp:
        endpoint: otel-gateway:4317
        tls:
          insecure: true
    
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch, resource]
          exporters: [otlp]
        metrics:
          receivers: [otlp, prometheus]
          processors: [batch, resource]
          exporters: [otlp]
  
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 200m
      memory: 256Mi
```

## Auto-Instrumentation

### Enable Auto-Instrumentation for Applications

Create an `Instrumentation` resource:

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-instrumentation
  namespace: default
spec:
  exporter:
    endpoint: http://otel-gateway:4318
  
  # Java auto-instrumentation
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
  
  # Node.js auto-instrumentation
  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:latest
  
  # Python auto-instrumentation
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:latest
  
  # .NET auto-instrumentation
  dotnet:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-dotnet:latest
```

Then annotate your pods:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-java: "true"
        # or inject-nodejs, inject-python, inject-dotnet
    spec:
      containers:
      - name: app
        image: my-app:latest
```

The operator will automatically inject telemetry libraries!

## Upgrading OpenTelemetry Operator

### Step-by-Step Process

**1. Test locally:**
```bash
vim .argocd.yaml
# Change: local-dev: chartVersion: "0.40.0" → "0.41.0"

git commit -am "test(otel): upgrade operator to 0.41.0 locally"
git push
```

**2. Validate existing collectors still work:**
```bash
kubectl get opentelemetrycollectors --all-namespaces
kubectl logs -n opentelemetry-system -l app.kubernetes.io/name=opentelemetry-operator
```

**3. Promote:** local-dev → nonprod → prod

### Version Compatibility

Check compatibility:  
https://github.com/open-telemetry/opentelemetry-operator#compatibility-matrix

| Operator Version | Collector Version | Kubernetes Version |
|-----------------|------------------|--------------------|
| 0.40.x+ | 0.91.0+ | 1.23+ |
| 0.39.x | 0.90.0+ | 1.23+ |
| 0.38.x | 0.89.0+ | 1.22+ |

## Monitoring

### Health Checks

```bash
# Check operator status
kubectl get pods -n opentelemetry-system

# View operator logs
kubectl logs -n opentelemetry-system -l app.kubernetes.io/name=opentelemetry-operator --tail=100

# List collectors
kubectl get opentelemetrycollectors --all-namespaces

# Check specific collector
kubectl describe opentelemetrycollector <name> -n <namespace>

# Check collector pods
kubectl get pods -n <namespace> -l app.kubernetes.io/instance=<collector-name>
```

### Collector Metrics

Collectors expose Prometheus metrics:
- `otelcol_receiver_accepted_spans` - Accepted spans
- `otelcol_processor_batch_batch_send_size` - Batch sizes
- `otelcol_exporter_sent_spans` - Exported spans
- `otelcol_exporter_send_failed_spans` - Failed exports

## Troubleshooting

### Operator Not Creating Collectors

**Symptoms:** OpenTelemetryCollector CRD created but no pods appear

**Check:**
```bash
# Verify CRD is installed
kubectl get crd opentelemetrycollectors.opentelemetry.io

# Check operator logs
kubectl logs -n opentelemetry-system -l app.kubernetes.io/name=opentelemetry-operator -f

# Check collector status
kubectl describe opentelemetrycollector <name> -n <namespace>
```

### Auto-Instrumentation Not Working

**Symptoms:** Pods not instrumented despite annotations

**Check:**
```bash
# Verify webhook is running
kubectl get mutatingwebhookconfiguration | grep opentelemetry

# Check webhook logs
kubectl logs -n opentelemetry-system -l app.kubernetes.io/component=webhook -f

# Verify Instrumentation resource exists
kubectl get instrumentation -n <namespace>

# Check pod events
kubectl describe pod <pod-name> -n <namespace>
```

### Telemetry Data Not Flowing

**Symptoms:** No traces/metrics reaching backends

**Check:**
```bash
# Check collector logs
kubectl logs -n <namespace> <collector-pod> -f

# Test OTLP endpoint
kubectl run test --rm -it --image=curlimages/curl -- \
  curl -v http://otel-gateway:4318/v1/traces

# Check exporter configuration
kubectl get opentelemetrycollector <name> -n <namespace> -o yaml
```

## Best Practices

✅ **Architecture**: Use DaemonSet collectors as agents, Deployment collectors as gateways  
✅ **Resource Limits**: Set appropriate limits based on traffic volume  
✅ **Batching**: Always enable batch processor to reduce overhead  
✅ **Memory Limiter**: Protect collectors from OOM with memory_limiter processor  
✅ **High Availability**: Run multiple gateway collector replicas  
✅ **Auto-Instrumentation**: Prefer auto-instrumentation over manual SDK integration  
✅ **Sampling**: Implement sampling for high-volume traces  

## Additional Resources

- **Operator Docs:** https://opentelemetry.io/docs/k8s-operator/
- **Collector Docs:** https://opentelemetry.io/docs/collector/
- **Specification:** https://opentelemetry.io/docs/specs/otel/
- **GitHub:** https://github.com/open-telemetry/opentelemetry-operator
- **Community:** https://cloud-native.slack.com (#otel-operator)

## Support

For operator-specific issues:
1. Check operator logs in `opentelemetry-system` namespace
2. Review OpenTelemetryCollector resource status and events
3. Verify CRD installation and webhook configuration
4. Check RBAC permissions

For OpenTelemetry questions:
- GitHub Issues: https://github.com/open-telemetry/opentelemetry-operator/issues
- CNCF Slack: #otel-operator channel
