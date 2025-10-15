# vmagent - Local Metrics Collection (Infra Cluster)

vmagent deployment for collecting metrics from the infrastructure cluster and writing to local VictoriaMetrics.

## Overview

This vmagent instance is deployed in the infra cluster to:
- Scrape metrics from local exporters (Node Exporter, Kube State Metrics)
- Collect Kubernetes component metrics (API server, kubelet, cAdvisor)
- Monitor VictoriaMetrics components
- Scrape pods with Prometheus annotations
- Write all metrics to local VictoriaMetrics cluster

## Architecture

```
┌────────────────────────────────────────────────────────┐
│          Infrastructure Cluster (k8s-infra-1)     │
│                                                         │
│  ┌─────────────────────────────────────────────────┐  │
│  │              Scrape Targets                     │  │
│  │                                                 │  │
│  │  • Kubernetes API Server                       │  │
│  │  • Kubelet (nodes)                             │  │
│  │  • cAdvisor (container metrics)                │  │
│  │  • Node Exporter (host metrics)                │  │
│  │  • Kube State Metrics (K8s objects)            │  │
│  │  • VictoriaMetrics components                  │  │
│  │  • Karpenter (node provisioner)                │  │
│  │  • Annotated pods (prometheus.io/scrape)       │  │
│  │                                                 │  │
│  └────────────────┬────────────────────────────────┘  │
│                   │                                     │
│                   ▼                                     │
│         ┌──────────────────┐                           │
│         │     vmagent      │                           │
│         │   StatefulSet    │                           │
│         │  (1 replica)     │                           │
│         │                  │                           │
│         │ • Scrape every   │                           │
│         │   30 seconds     │                           │
│         │ • Queue metrics  │                           │
│         │ • Add labels     │                           │
│         │                  │                           │
│         └────────┬─────────┘                           │
│                  │                                      │
│                  │ Remote Write                         │
│                  ▼                                      │
│       ┌────────────────────┐                           │
│       │  VictoriaMetrics   │                           │
│       │     vminsert       │                           │
│       │   (Local Write)    │                           │
│       └────────────────────┘                           │
│                  │                                      │
│                  ▼                                      │
│       ┌────────────────────┐                           │
│       │  VictoriaMetrics   │                           │
│       │     vmstorage      │                           │
│       │  (Persistence)     │                           │
│       └────────────────────┘                           │
└────────────────────────────────────────────────────────┘
```

## Key Features

### Local Collection
- **No Remote Write Authentication**: Direct write to local VMInsert
- **Low Latency**: Metrics stay within the same cluster
- **High Throughput**: Optimized for local collection

### Comprehensive Scraping
- **System Metrics**: API server, kubelet, cAdvisor
- **Exporters**: Node Exporter, Kube State Metrics
- **Platform**: VictoriaMetrics, Karpenter
- **Applications**: Auto-discovery via annotations

### Reliability
- **Persistent Queue**: 10Gi PVC for temporary metric storage
- **Batch Processing**: Efficient remote write with configurable queues
- **Health Checks**: Automatic restart on failure

### Labels
All metrics are enriched with:
- `cluster: k8s-infra-1`
- `environment: infra`
- `region: ap-south-1`
- `account: infra-myop`

## Configuration

### Global Settings

```yaml
global:
  scrape_interval: 30s        # Default scrape frequency
  scrape_timeout: 10s         # Scrape timeout
  external_labels:            # Added to all metrics
    cluster: 'k8s-infra-1'
    environment: 'infra'
    region: 'ap-south-1'
    account: 'infra-myop'
```

### Scrape Targets

| Job Name | Target | Port | Purpose |
|----------|--------|------|---------|
| vmagent | localhost | 8429 | Self-monitoring |
| kubernetes-apiservers | Kubernetes API | 443 | API server metrics |
| kubernetes-nodes | Kubelet | 10250 | Node metrics |
| kubernetes-nodes-cadvisor | cAdvisor | 10250 | Container metrics |
| node-exporter | Node Exporter | 9100 | Host hardware metrics |
| kube-state-metrics | KSM | 8080 | K8s object metrics |
| karpenter | Karpenter | 8080 | Node provisioning metrics |
| victoria-metrics-components | VM Components | Various | VM cluster metrics |
| kubernetes-pods | Annotated Pods | Various | App metrics |

### Remote Write Configuration

```yaml
remoteWrite:
  url: http://victoria-metrics-vm-vminsert.monitoring.svc.cluster.local:8480/insert/0/prometheus/api/v1/write
  queues: 4                          # Parallel write queues
  maxDiskUsagePerURL: 1GB           # Max persistent queue size
  flushInterval: 5s                 # Write interval
  maxBlockSize: 8MB                 # Max batch size
```

### Resource Allocation

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | 250m | 1 core |
| Memory | 512Mi | 1Gi |
| Storage | 10Gi | - |

### Storage
- **Type**: StatefulSet with PVC
- **Storage Class**: gp3
- **Size**: 10Gi
- **Purpose**: Temporary metric buffering during network issues

## Prerequisites

- Monitoring namespace created
- VictoriaMetrics cluster deployed
- Exporters deployed (Node Exporter, Kube State Metrics)
- Docker Hub credentials (for image pull)
- gp3 storage class available

## Installation

### Step 1: Verify Prerequisites

```bash
# Check VictoriaMetrics is running
kubectl get pods -n monitoring | grep victoria-metrics

# Check exporters are running
kubectl get pods -n monitoring | grep -E 'node-exporter|kube-state-metrics'

# Check storage class
kubectl get sc gp3
```

### Step 2: Deploy vmagent

```bash
cd infra-cluster/vmagent

# Review configuration
cat vmagent.yaml

# Apply manifest
kubectl apply -f vmagent.yaml

# Watch deployment
kubectl get pods -n monitoring -l app.kubernetes.io/name=vmagent -w
```

### Step 3: Verify Deployment

```bash
# Check StatefulSet
kubectl get statefulset vmagent -n monitoring

# Check pod
kubectl get pods -n monitoring -l app.kubernetes.io/name=vmagent

# Check service
kubectl get svc vmagent -n monitoring

# Check PVC
kubectl get pvc -n monitoring -l app.kubernetes.io/name=vmagent
```

## Verification

### Check vmagent Health

```bash
# Port-forward
kubectl port-forward -n monitoring statefulset/vmagent 8429:8429

# Check health endpoint
curl http://localhost:8429/health
# Should return: OK

# Check metrics
curl http://localhost:8429/metrics | head -20
```

### Verify Scraping

```bash
# Check active targets
curl -s http://localhost:8429/targets | jq '.data.activeTargets | length'

# List all jobs
curl -s http://localhost:8429/targets | jq '.data.activeTargets | group_by(.labels.job) | map({job: .[0].labels.job, count: length})'

# Check for scrape errors
curl -s http://localhost:8429/targets | jq '.data.activeTargets[] | select(.health != "up")'
```

### Check Logs

```bash
# View logs
kubectl logs -n monitoring statefulset/vmagent --tail=50 -f

# Should see successful scrapes:
# "successfully scraped target" job="node-exporter"
# "successfully scraped target" job="kube-state-metrics"
```

### Verify Metrics in VictoriaMetrics

```bash
# Port-forward VMSelect
kubectl port-forward -n monitoring svc/victoria-metrics-vm-vmselect 8481:8481

# Query metrics from infra cluster
curl -s 'http://localhost:8481/select/0/prometheus/api/v1/query?query=up{cluster="k8s-infra-1"}' | jq

# Check metric labels
curl -s 'http://localhost:8481/select/0/prometheus/api/v1/query?query=node_cpu_seconds_total' | jq '.data.result[0].metric'
# Should include: cluster, environment, region, account labels
```

## Usage

### Querying Metrics

```bash
# Port-forward vmagent
kubectl port-forward -n monitoring statefulset/vmagent 8429:8429
```

**Available Endpoints:**
- `/health` - Health check
- `/metrics` - vmagent internal metrics
- `/targets` - Active scrape targets
- `/api/v1/targets` - Prometheus-compatible targets API

### Example Queries

**Check target health:**
```bash
curl http://localhost:8429/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health, lastScrape: .lastScrape}'
```

**View scrape statistics:**
```bash
curl http://localhost:8429/metrics | grep vmagent_remotewrite
```

**Check queue size:**
```bash
curl http://localhost:8429/metrics | grep vmagent_remotewrite_pending_data_bytes
```

## Monitoring vmagent

### Key Metrics to Monitor

```promql
# Scrape success rate
rate(vmagent_promscrape_scrapes_total{status="success"}[5m]) 
/ 
rate(vmagent_promscrape_scrapes_total[5m])

# Remote write success rate  
rate(vmagent_remotewrite_requests_total{status_code="2xx"}[5m])
/ 
rate(vmagent_remotewrite_requests_total[5m])

# Queue size (should be near 0)
vmagent_remotewrite_pending_data_bytes

# Scrape duration
vmagent_promscrape_scrape_duration_seconds

# Memory usage
process_resident_memory_bytes{job="vmagent"}
```

### Alerts

**vmagent Down:**
```promql
up{job="vmagent"} == 0
```

**High Scrape Failure Rate:**
```promql
rate(vmagent_promscrape_scrapes_failed_total[5m]) > 0.1
```

**Remote Write Errors:**
```promql
rate(vmagent_remotewrite_requests_total{status_code!~"2.."}[5m]) > 0
```

**Queue Growing:**
```promql
vmagent_remotewrite_pending_data_bytes > 500000000  # 500MB
```

## Configuration Updates

### Update Scrape Configuration

```bash
# Edit ConfigMap
kubectl edit configmap vmagent-config -n monitoring

# Restart vmagent to reload config
kubectl rollout restart statefulset vmagent -n monitoring

# Watch restart
kubectl rollout status statefulset vmagent -n monitoring
```

### Add New Scrape Target

Example: Add a new job for custom exporter

```yaml
# Edit ConfigMap
kubectl edit configmap vmagent-config -n monitoring

# Add under scrape_configs:
- job_name: 'my-custom-exporter'
  static_configs:
    - targets: ['my-exporter.monitoring.svc.cluster.local:9999']
  scrape_interval: 30s

# Restart vmagent
kubectl rollout restart statefulset vmagent -n monitoring
```

### Adjust Resources

```bash
# Edit StatefulSet
kubectl edit statefulset vmagent -n monitoring

# Update resources:
resources:
  requests:
    cpu: 500m      # Increase CPU
    memory: 1Gi    # Increase memory
  limits:
    cpu: 2
    memory: 2Gi
```

## Troubleshooting

### Pod Not Starting

```bash
# Check events
kubectl describe statefulset vmagent -n monitoring

# Check pod events
kubectl describe pod vmagent-0 -n monitoring

# Common issues:
# - PVC not bound: Check storage class
# - ImagePullBackOff: Check dockerhub-secret
# - CrashLoopBackOff: Check logs
```

### Cannot Scrape Targets

```bash
# Check vmagent logs
kubectl logs -n monitoring vmagent-0 --tail=100 | grep -i error

# Test target connectivity from vmagent pod
kubectl exec -it vmagent-0 -n monitoring -- /bin/sh
wget -O- http://node-exporter.monitoring.svc.cluster.local:9100/metrics

# Check RBAC permissions
kubectl auth can-i get nodes --as=system:serviceaccount:monitoring:vmagent
```

### High Memory Usage

```bash
# Check current usage
kubectl top pod vmagent-0 -n monitoring

# Check metrics for specific jobs
kubectl exec -it vmagent-0 -n monitoring -- wget -O- http://localhost:8429/metrics | grep scrape_samples

# Solutions:
# 1. Increase memory limits
# 2. Reduce scrape frequency
# 3. Filter out high-cardinality metrics
```

### Remote Write Errors

```bash
# Check logs for write errors
kubectl logs -n monitoring vmagent-0 | grep remotewrite

# Check VMInsert availability
kubectl get pods -n monitoring | grep vminsert

# Test connectivity
kubectl exec -it vmagent-0 -n monitoring -- \
  wget -O- http://victoria-metrics-vm-vminsert.monitoring.svc.cluster.local:8480/health
```

### Persistent Queue Growing

```bash
# Check queue size
kubectl exec -it vmagent-0 -n monitoring -- du -sh /var/lib/vmagent-remotewrite

# Check PVC usage
kubectl exec -it vmagent-0 -n monitoring -- df -h /var/lib/vmagent-remotewrite

# If queue is full:
# 1. Fix remote write endpoint
# 2. Increase PVC size (requires StatefulSet recreation)
# 3. Temporarily increase flush interval
```

## Maintenance

### Restart vmagent

```bash
# Rolling restart
kubectl rollout restart statefulset vmagent -n monitoring

# Or delete pod (will auto-recreate)
kubectl delete pod vmagent-0 -n monitoring
```

### Update vmagent Version

```bash
# Edit StatefulSet
kubectl edit statefulset vmagent -n monitoring

# Update image version:
image: victoriametrics/vmagent:v1.117.0

# Apply changes (will trigger rolling update)
kubectl rollout status statefulset vmagent -n monitoring
```

### Expand Storage

```bash
# Edit PVC (only if storage class supports expansion)
kubectl edit pvc data-vmagent-0 -n monitoring

# Update storage size:
spec:
  resources:
    requests:
      storage: 20Gi

# Restart pod to apply
kubectl delete pod vmagent-0 -n monitoring
```

### Backup Configuration

```bash
# Export ConfigMap
kubectl get configmap vmagent-config -n monitoring -o yaml > vmagent-config-backup.yaml

# Export StatefulSet
kubectl get statefulset vmagent -n monitoring -o yaml > vmagent-statefulset-backup.yaml
```

## Performance Tuning

### For High-Volume Metrics

```yaml
# Increase write queues
args:
  - -remoteWrite.queues=8              # More parallel queues
  - -remoteWrite.maxBlockSize=16MB     # Larger batches
  - -remoteWrite.flushInterval=10s     # Less frequent writes

# Increase resources
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 2
    memory: 4Gi
```

### Reduce Memory Usage

```yaml
# Enable streaming parse
args:
  - -promscrape.streamParse=true       # Already enabled
  - -memory.allowedPercent=60          # Reduce from 80
  - -remoteWrite.maxBlockSize=4MB      # Smaller batches
```

### Optimize Scraping

```yaml
# Adjust scrape intervals per job
scrape_configs:
  - job_name: 'high-frequency'
    scrape_interval: 15s               # More frequent
    
  - job_name: 'low-frequency'
    scrape_interval: 60s               # Less frequent
```

## Security

- **RBAC**: Limited to read-only access for service discovery
- **Network**: Internal service, not exposed externally
- **Service Account**: Dedicated SA with minimal permissions
- **Security Context**: Runs as non-root (UID 65534)
- **No Privilege Escalation**: Capabilities dropped

## Version Information

- **Image**: victoriametrics/vmagent:v1.116.0
- **Kubernetes**: 1.24+
- **Storage**: gp3

## References

- [vmagent Documentation](https://docs.victoriametrics.com/vmagent.html)
- [Configuration Examples](https://github.com/VictoriaMetrics/VictoriaMetrics/tree/master/deployment/docker)
- [Scrape Config Reference](https://docs.victoriametrics.com/#how-to-scrape-prometheus-exporters-such-as-node-exporter)
