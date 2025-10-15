# Kubernetes Exporters

Production-ready Prometheus exporters for comprehensive Kubernetes cluster monitoring.

## Overview

This directory contains deployment manifests for essential Kubernetes exporters:
- **Node Exporter**: Hardware and OS metrics from cluster nodes
- **Kube State Metrics**: Kubernetes object state metrics

These exporters provide the foundation for monitoring cluster health, resource utilization, and workload status.

## Exporters

### Node Exporter
Collects hardware and operating system metrics from each node:
- CPU usage and load
- Memory utilization
- Disk I/O and space
- Network statistics
- Filesystem metrics

**Deployment Type**: DaemonSet (runs on every node)

### Kube State Metrics
Exposes Kubernetes object metrics:
- Pod status and lifecycle
- Deployment rollout status
- Node conditions
- Resource quotas
- PersistentVolume claims
- Service endpoints

**Deployment Type**: Deployment (single replica)

## Architecture

```
┌─────────────────────────────────────────────────────┐
│              Kubernetes Cluster                     │
│                                                     │
│  ┌────────────────────────────────────────────┐   │
│  │         Node Exporter DaemonSet            │   │
│  │    (One pod per node - hostNetwork)        │   │
│  │                                            │   │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  │   │
│  │  │ Node │  │ Node │  │ Node │  │ Node │  │   │
│  │  │  1   │  │  2   │  │  3   │  │  4   │  │   │
│  │  │:9100 │  │:9100 │  │:9100 │  │:9100 │  │   │
│  │  └──────┘  └──────┘  └──────┘  └──────┘  │   │
│  │                                            │   │
│  └────────────────────────────────────────────┘   │
│                                                     │
│  ┌────────────────────────────────────────────┐   │
│  │    Kube State Metrics Deployment           │   │
│  │    (Kubernetes API → Metrics)              │   │
│  │                                            │   │
│  │           ┌──────────────┐                 │   │
│  │           │     KSM      │                 │   │
│  │           │    :8080     │                 │   │
│  │           └──────────────┘                 │   │
│  │                                            │   │
│  └────────────────────────────────────────────┘   │
│                                                     │
│                       │                             │
│                       ▼                             │
│              ┌─────────────────┐                   │
│              │     vmagent     │                   │
│              │  (Scrapes and   │                   │
│              │   Remote Write) │                   │
│              └─────────────────┘                   │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
              ┌──────────────────┐
              │  VictoriaMetrics │
              │     Cluster      │
              └──────────────────┘
```

## Directory Structure

```
exporters/
├── README.md                    # This file
├── node-exporter.yaml          # Node Exporter DaemonSet
└── kube-state-metrics.yaml     # Kube State Metrics Deployment
```

## Prerequisites

- Kubernetes cluster (1.24+)
- Monitoring namespace created
- vmagent deployed (for metrics collection)

## Installation

### Quick Start

```bash
# Create monitoring namespace (if not exists)
kubectl create namespace monitoring

# Deploy both exporters
kubectl apply -f node-exporter.yaml
kubectl apply -f kube-state-metrics.yaml

# Verify deployment
kubectl get pods -n monitoring -l app.kubernetes.io/name=node-exporter
kubectl get pods -n monitoring -l app.kubernetes.io/name=kube-state-metrics
```

### Individual Deployment

```bash
# Deploy only Node Exporter
kubectl apply -f node-exporter.yaml

# Deploy only Kube State Metrics
kubectl apply -f kube-state-metrics.yaml
```

## Verification

### Check Pod Status

```bash
# Node Exporter (should be one pod per node)
kubectl get pods -n monitoring -l app.kubernetes.io/name=node-exporter -o wide

# Kube State Metrics
kubectl get pods -n monitoring -l app.kubernetes.io/name=kube-state-metrics
```

### Verify Services

```bash
kubectl get svc -n monitoring | grep -E 'node-exporter|kube-state-metrics'
```

### Test Metrics Endpoints

```bash
# Test Node Exporter
kubectl port-forward -n monitoring daemonset/node-exporter 9100:9100
curl http://localhost:9100/metrics

# Test Kube State Metrics
kubectl port-forward -n monitoring deployment/kube-state-metrics 8080:8080
curl http://localhost:8080/metrics
```

## Configuration Details

### Node Exporter

**Image**: `quay.io/prometheus/node-exporter:v1.9.1`

**Key Features**:
- Runs with `hostNetwork: true` for accurate network metrics
- Runs with `hostPID: true` for process metrics
- Mounts host `/proc`, `/sys`, and `/root` filesystems

**Resource Allocation**:
| Resource | Request | Limit |
|----------|---------|-------|
| CPU      | 50m     | 200m  |
| Memory   | 128Mi   | 256Mi |

**Collectors Configuration**:
- Excludes virtual filesystems (overlay, tmpfs, etc.)
- Excludes container network interfaces (veth, calico)
- Collects from host paths: `/host/proc`, `/host/sys`, `/host/root`

**Tolerations**:
- Runs on all nodes including control plane
- Tolerates all NoSchedule and NoExecute taints

**Security Context**:
- Runs as non-root (UID 65534)
- No privilege escalation
- Drops all capabilities
- Read-only root filesystem access

### Kube State Metrics

**Image**: `registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.16.0`

**Key Features**:
- Watches Kubernetes API for state changes
- Exposes metrics on port 8080
- Telemetry endpoint on port 8081

**Resource Allocation**:
| Resource | Request | Limit |
|----------|---------|-------|
| CPU      | 100m    | 200m  |
| Memory   | 190Mi   | 256Mi |

**RBAC Permissions**:
- ClusterRole with read-only access to:
  - Core resources (pods, services, nodes, etc.)
  - Apps resources (deployments, statefulsets, daemonsets)
  - Batch resources (jobs, cronjobs)
  - Autoscaling (HPA)
  - Storage (PV, PVC, StorageClass)
  - Networking (Ingress, NetworkPolicy)
  - RBAC (Roles, RoleBindings)

**Node Placement**:
- Scheduled on `general-arm` node pool
- ARM64 architecture

**Security Context**:
- Runs as non-root (UID 65534)
- No privilege escalation
- Drops all capabilities
- Read-only root filesystem

## Metrics Exposed

### Node Exporter Key Metrics

**CPU Metrics**:
```promql
node_cpu_seconds_total                # CPU time per mode
node_load1, node_load5, node_load15   # System load averages
```

**Memory Metrics**:
```promql
node_memory_MemTotal_bytes            # Total memory
node_memory_MemAvailable_bytes        # Available memory
node_memory_Buffers_bytes             # Buffer cache
node_memory_Cached_bytes              # Page cache
```

**Disk Metrics**:
```promql
node_disk_read_bytes_total            # Bytes read
node_disk_written_bytes_total         # Bytes written
node_disk_io_time_seconds_total       # I/O time
node_filesystem_avail_bytes           # Available space
node_filesystem_size_bytes            # Total size
```

**Network Metrics**:
```promql
node_network_receive_bytes_total      # Received bytes
node_network_transmit_bytes_total     # Transmitted bytes
node_network_receive_errs_total       # Receive errors
node_network_transmit_errs_total      # Transmit errors
```

### Kube State Metrics Key Metrics

**Pod Metrics**:
```promql
kube_pod_status_phase                 # Pod phase (Running, Pending, etc.)
kube_pod_container_status_restarts_total  # Container restart count
kube_pod_container_resource_requests  # Resource requests
kube_pod_container_resource_limits    # Resource limits
```

**Deployment Metrics**:
```promql
kube_deployment_status_replicas       # Desired replicas
kube_deployment_status_replicas_available  # Available replicas
kube_deployment_status_replicas_unavailable  # Unavailable replicas
```

**Node Metrics**:
```promql
kube_node_status_condition            # Node conditions (Ready, etc.)
kube_node_status_capacity             # Node capacity
kube_node_status_allocatable          # Allocatable resources
```

**Namespace Metrics**:
```promql
kube_namespace_status_phase           # Namespace phase
kube_resourcequota                    # Resource quota usage
```

## Usage with vmagent

These exporters are designed to be scraped by vmagent. Example vmagent scrape configuration:

```yaml
scrape_configs:
  # Node Exporter
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)

  # Kube State Metrics
  - job_name: 'kube-state-metrics'
    static_configs:
      - targets: ['kube-state-metrics.monitoring.svc.cluster.local:8080']
```

## Common Queries

### Node Exporter Queries

```promql
# CPU usage per node
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage per node
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))

# Disk usage per node
100 - ((node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100)

# Network receive rate
rate(node_network_receive_bytes_total[5m])

# Network transmit rate
rate(node_network_transmit_bytes_total[5m])
```

### Kube State Metrics Queries

```promql
# Pods not in Running state
count(kube_pod_status_phase{phase!="Running"}) by (namespace, phase)

# Container restart count (last 30m)
increase(kube_pod_container_status_restarts_total[30m]) > 0

# Deployments with unavailable replicas
kube_deployment_status_replicas_unavailable > 0

# Nodes not ready
kube_node_status_condition{condition="Ready",status="false"} == 1

# PVC pending
kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
```

## Maintenance

### Updating Versions

```bash
# Update image version in YAML files
vim node-exporter.yaml  # Update image tag
vim kube-state-metrics.yaml  # Update image tag

# Apply updates
kubectl apply -f node-exporter.yaml
kubectl apply -f kube-state-metrics.yaml

# Watch rollout
kubectl rollout status daemonset/node-exporter -n monitoring
kubectl rollout status deployment/kube-state-metrics -n monitoring
```

### Resource Tuning

For large clusters (100+ nodes), increase KSM resources:

```yaml
resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 1Gi
```

### Scaling Kube State Metrics

For very large clusters, consider sharding KSM:

```yaml
# Add sharding arguments
args:
  - --shard=0
  - --total-shards=2
```

## Troubleshooting

### Node Exporter Issues

**Pods not running on all nodes:**
```bash
# Check node count
kubectl get nodes --no-headers | wc -l

# Check node-exporter pod count
kubectl get pods -n monitoring -l app.kubernetes.io/name=node-exporter --no-headers | wc -l

# Should match. If not, check DaemonSet events:
kubectl describe daemonset node-exporter -n monitoring
```

**Cannot scrape metrics:**
```bash
# Check if pod is on correct node
kubectl get pods -n monitoring -l app.kubernetes.io/name=node-exporter -o wide

# Port-forward and test
kubectl port-forward -n monitoring POD_NAME 9100:9100
curl http://localhost:9100/metrics
```

**Host path mount issues:**
```bash
# Check pod logs
kubectl logs -n monitoring POD_NAME

# Check security context
kubectl get pod POD_NAME -n monitoring -o yaml | grep -A 10 securityContext
```

### Kube State Metrics Issues

**Pod CrashLoopBackOff:**
```bash
# Check logs
kubectl logs -n monitoring deployment/kube-state-metrics

# Common issues:
# - RBAC permissions: Check ClusterRole and ClusterRoleBinding
# - API server connectivity: Check service account token
```

**High memory usage:**
```bash
# Check resource usage
kubectl top pod -n monitoring -l app.kubernetes.io/name=kube-state-metrics

# If high, increase limits or enable sharding
```

**Missing metrics:**
```bash
# Check what resources KSM is watching
kubectl logs -n monitoring deployment/kube-state-metrics | grep "Watching"

# Verify RBAC permissions
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:kube-state-metrics
```

## Security Considerations

1. **RBAC**: KSM has read-only access to cluster resources
2. **Network Policies**: Consider restricting access to metrics ports
3. **Service Account**: Both exporters use dedicated service accounts
4. **Security Context**: Both run as non-root with minimal privileges
5. **Host Access**: Node Exporter requires host filesystem access (expected)

## Performance Impact

**Node Exporter**:
- Minimal CPU impact: ~10m average
- Memory footprint: ~50-100Mi per node
- Network: Minimal (only scraped periodically)

**Kube State Metrics**:
- CPU: Scales with cluster size (~100m for 100 nodes)
- Memory: Scales with object count (~200Mi for 10k objects)
- API Server load: Watches API events (efficient)

## Best Practices

1. **Monitoring**: Monitor the exporters themselves
2. **Alerts**: Set up alerts for exporter downtime
3. **Updates**: Keep exporters up-to-date for bug fixes
4. **Resources**: Adjust resources based on cluster size
5. **Retention**: Configure appropriate metric retention in VictoriaMetrics

## Version Information

- **Node Exporter**: v1.9.1
- **Kube State Metrics**: v2.16.0
- **Kubernetes**: 1.24+

## Additional Exporters

Consider adding these exporters for comprehensive monitoring:

- **cAdvisor**: Container metrics (included in kubelet)
- **Blackbox Exporter**: Endpoint probing
- **Elasticsearch Exporter**: If using Elasticsearch
- **Redis Exporter**: If using Redis
- **PostgreSQL Exporter**: If using PostgreSQL

## References

- [Node Exporter GitHub](https://github.com/prometheus/node_exporter)
- [Kube State Metrics GitHub](https://github.com/kubernetes/kube-state-metrics)
- [Node Exporter Metrics](https://github.com/prometheus/node_exporter#collectors)
- [KSM Metrics](https://github.com/kubernetes/kube-state-metrics/tree/main/docs)
