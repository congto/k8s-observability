# VictoriaMetrics Cluster Setup

High-availability VictoriaMetrics cluster deployment for centralized metrics storage across multiple environments.

## Overview

This setup deploys a production-ready VictoriaMetrics cluster with:
- **VMStorage**: 2 replicas for data persistence (15 days retention)
- **VMInsert**: 2 replicas for write path with load balancing
- **VMSelect**: 2 replicas for read path with caching
- **VMAuth**: 2 replicas for authentication and routing
- **VMAlert**: 2 replicas for alerting (optional)

## Architecture

```
┌─────────────────────────────────────────────────────┐
│              Remote Clusters (Dev/Stage/Prod)      │
│                     vmagent                         │
└──────────────────────┬──────────────────────────────┘
                       │ Remote Write
                       ▼
              ┌─────────────────┐
              │     VMAuth      │  ◄── Authentication & Routing
              │   (2 replicas)  │
              └────────┬────────┘
                       │
        ┌──────────────┴──────────────┐
        ▼                             ▼
┌──────────────┐              ┌──────────────┐
│   VMInsert   │              │   VMSelect   │
│ (2 replicas) │              │ (2 replicas) │
└──────┬───────┘              └──────┬───────┘
       │                             │
       ▼                             ▼
┌─────────────────────────────────────────┐
│            VMStorage                    │
│          (2 replicas)                   │
│    Persistent Storage: 100Gi gp3        │
└─────────────────────────────────────────┘
```

## Directory Structure

```
victoria-metrics/
└── base/
    ├── Chart.yaml              # Helm chart dependencies
    ├── values.yaml             # Base configuration
    ├── infra/
    │   └── values.yaml         # Infra-specific overrides
    ├── templates/
    │   ├── _helpers.tpl
    │   └── vmselect-pvc.yaml   # Shared cache PVC
    └── charts/
        └── victoria-metrics-cluster-0.24.1.tgz
```

## Prerequisites

- Kubernetes cluster (EKS recommended)
- Helm 3.x
- Storage class `gp3` configured
- Service account `victoria-metrics-storage` with appropriate IAM permissions
- Docker Hub credentials (for pulling images)

## Installation

### 1. Create Namespace
```bash
kubectl create namespace monitoring
```

### 2. Create Docker Hub Secret
```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<username> \
  --docker-password=<password> \
  -n monitoring
```

### 3. Deploy VictoriaMetrics
```bash
cd infra-cluster/helm/victoria-metrics/base

# Install with infra values
helm upgrade --install victoria-metrics . \
  --namespace monitoring \
  --values values.yaml \
  --values infra/values.yaml \
  --create-namespace
```

### 4. Verify Deployment
```bash
# Check all pods
kubectl get pods -n monitoring -l app.kubernetes.io/part-of=victoria-metrics

# Check services
kubectl get svc -n monitoring

# Check ingress
kubectl get ingress -n monitoring
```

## Configuration Details

### Storage Configuration
- **VMStorage**: 100Gi gp3 volumes per replica
- **VMSelect Cache**: 50Gi gp3 shared volume
- **Retention Period**: 15 days
- **Deduplication**: 30s interval

### Resource Allocation

| Component | Replicas | CPU Request | Memory Request | CPU Limit | Memory Limit |
|-----------|----------|-------------|----------------|-----------|--------------|
| VMStorage | 2 | 500m | 1Gi | 2 | 4Gi |
| VMInsert | 2 | 500m | 1Gi | 2 | 2Gi |
| VMSelect | 2 | 500m | 1Gi | 2 | 4Gi |
| VMAuth | 2 | 100m | 128Mi | 500m | 512Mi |
| VMAlert | 2 | 100m | 128Mi | 500m | 512Mi |

### Authentication (VMAuth)

Access credentials configured for different environments:

```yaml
users:
  - dev-cluster: Write-only access for Dev environment
  - stage-cluster: Write-only access for Stage environment
  - readonly: Read-only access (for Grafana)
  - writeonly: Write-only access
  - admin: Full read/write access
```

**Security Note**: Change default passwords in production!

### Node Placement
All components are scheduled on:
- Node pool: `general-arm`
- Architecture: `arm64`
- Anti-affinity rules ensure replicas run on different nodes

### Ingress Configuration
VMAuth is exposed via AWS ALB:
- **Host**: `victoria-metrics.infra.service`
- **Type**: Internal ALB
- **CIDR**: Restricted to VPC ranges
- **Health Check**: `/health` endpoint

## Usage

### Remote Write Configuration
From remote clusters (dev/stage/prod), configure vmagent to write to:

```yaml
remoteWrite:
  url: "http://victoria-metrics.infra.service/api/v1/write"
  basicAuth:
    username: "dev-cluster"  # or stage-cluster
    password: "<password>"
```

### Query Metrics (via VMSelect)
```bash
# Port-forward for local testing
kubectl port-forward svc/victoria-metrics-vm-vmselect 8481:8481 -n monitoring

# Query example
curl 'http://localhost:8481/select/0/prometheus/api/v1/query?query=up'
```

### Grafana Data Source
Configure Grafana to use VMAuth:
```yaml
url: http://victoria-metrics.infra.service
auth: Basic Auth
user: readonly
password: <readonly-password>
```

## Monitoring

### Health Checks
All components have liveness and readiness probes configured:
- Initial delay: 10-30s
- Period: 10-30s
- Timeout: 5s

### Service Monitors
Prometheus-style service monitors are enabled for scraping metrics from VM components themselves.

## Maintenance

### Scaling
```bash
# Scale VMInsert
helm upgrade victoria-metrics . \
  --set vm.vminsert.replicaCount=3 \
  --namespace monitoring

# Scale VMSelect
helm upgrade victoria-metrics . \
  --set vm.vmselect.replicaCount=3 \
  --namespace monitoring
```

### Backup & Recovery
VMStorage data is persisted on gp3 volumes with:
- Retention policy annotation: `helm.sh/resource-policy: keep`
- Volumes are retained even after helm uninstall

### Upgrading
```bash
# Update chart version in Chart.yaml
helm dependency update

# Upgrade
helm upgrade victoria-metrics . \
  --namespace monitoring \
  --values values.yaml \
  --values infra/values.yaml
```

## Troubleshooting

### Pod Not Starting
```bash
# Check pod events
kubectl describe pod <pod-name> -n monitoring

# Check logs
kubectl logs <pod-name> -n monitoring
```

### Storage Issues
```bash
# Check PVCs
kubectl get pvc -n monitoring

# Check PV status
kubectl get pv
```

### Authentication Issues
```bash
# Test VMAuth
curl -u dev-cluster:devcluster-123 \
  http://victoria-metrics.infra.service/health
```

### High Memory Usage
- Adjust query limits in VMSelect `extraArgs`
- Reduce retention period if needed
- Scale up resources or replicas

## Security Considerations

1. **Change default passwords** in `vmauth.config.users`
2. Use Kubernetes Secrets for credentials (not plain text)
3. Enable network policies in production
4. Restrict ingress CIDR ranges
5. Enable TLS/SSL for ingress
6. Regular security updates of VM images

## Performance Tuning

### For High Write Loads
- Increase `vminsert.replicaCount`
- Adjust `maxInsertRequestSize` and `maxConcurrentInserts`
- Scale VMStorage horizontally

### For High Query Loads
- Increase `vmselect.replicaCount`
- Increase cache size (`persistentVolume.size`)
- Adjust `search.maxConcurrentRequests`

## Version Information

- **Chart Version**: 0.24.1
- **App Version**: 1.116.0
- **Helm Chart**: victoria-metrics-cluster

## References

- [VictoriaMetrics Documentation](https://docs.victoriametrics.com/)
- [Helm Chart Repository](https://github.com/VictoriaMetrics/helm-charts)
- [Architecture Guide](https://docs.victoriametrics.com/Cluster-VictoriaMetrics.html)
