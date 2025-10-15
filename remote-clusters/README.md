# Remote Clusters - Monitoring Setup

Deployment configurations for metrics collection in remote Kubernetes clusters (dev, staging, production) with remote write to central VictoriaMetrics.

## Overview

Each remote cluster deploys a lightweight monitoring stack that:
- **Collects local metrics** using exporters (Node Exporter, Kube State Metrics)
- **Scrapes metrics** using vmagent
- **Adds cluster labels** for multi-tenant isolation
- **Writes remotely** to central VictoriaMetrics in infra cluster
- **Buffers metrics** during network issues with persistent storage

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│           Remote Cluster (Dev/Staging/Prod)             │
│                                                          │
│  ┌────────────────────────────────────────────────┐    │
│  │              Metrics Sources                   │    │
│  │                                                │    │
│  │  • Kubernetes API Server                      │    │
│  │  • Kubelet (nodes)                            │    │
│  │  • cAdvisor (container metrics)               │    │
│  │  • Node Exporter (host metrics)               │    │
│  │  • Kube State Metrics (K8s objects)           │    │
│  │  • Annotated pods (app metrics)               │    │
│  │                                                │    │
│  └────────────────┬───────────────────────────────┘    │
│                   │ Scrape (30s interval)               │
│                   ▼                                     │
│         ┌──────────────────┐                           │
│         │     vmagent      │                           │
│         │   StatefulSet    │                           │
│         │                  │                           │
│         │ • Add labels:    │                           │
│         │   - cluster      │                           │
│         │   - environment  │                           │
│         │   - region       │                           │
│         │ • Buffer (10Gi)  │                           │
│         │ • Compress       │                           │
│         │                  │                           │
│         └────────┬─────────┘                           │
│                  │                                      │
│                  │ Remote Write                         │
│                  │ (Authenticated)                      │
└──────────────────┼──────────────────────────────────────┘
                   │
                   │ HTTP POST with Basic Auth
                   │ (dev-cluster:devcluster-123)
                   │
                   ▼
┌──────────────────────────────────────────────────────────┐
│          Infrastructure Cluster (Infra Account)          │
│                                                          │
│         ┌──────────────────────────┐                    │
│         │        VMAuth            │                    │
│         │  (Authentication)        │                    │
│         └────────┬─────────────────┘                    │
│                  │                                       │
│                  ▼                                       │
│         ┌──────────────────────────┐                    │
│         │  VictoriaMetrics Cluster │                    │
│         │    (Central Storage)     │                    │
│         └──────────────────────────┘                    │
└──────────────────────────────────────────────────────────┘
```

## Directory Structure

```
remote-clusters/
├── dev/
│   ├── exporters/
│   │   ├── node-exporter.yaml          # Host metrics exporter
│   │   └── kube-state-metrics.yaml     # K8s object metrics
│   ├── vmagent/
│   │   └── vmagent.yaml                # Metrics collection & remote write
│   └── README.md                       # Dev cluster guide
├── staging/
│   └── ... (similar structure)
└── prod/
    └── ... (similar structure)
```

## Key Differences from Infra Cluster

| Aspect | Infra Cluster | Remote Cluster |
|--------|---------------|----------------|
| **VictoriaMetrics** | Full cluster deployed | Not deployed |
| **Grafana** | HA deployment | Not deployed |
| **vmagent** | Writes locally | Writes remotely with auth |
| **Namespace** | `monitoring` | `monitoring-dev/staging/prod` |
| **Labels** | `cluster: infra` | `cluster: dev/staging/prod` |
| **Storage** | VictoriaMetrics storage | Only vmagent buffer (10Gi) |

## Components

### 1. Node Exporter (DaemonSet)

**Purpose**: Collects hardware and OS metrics from each node

**Key Features**:
- Runs on every node (including control plane)
- Collects CPU, memory, disk, network metrics
- Minimal resource footprint (50m CPU, 128Mi memory)
- Tolerates all taints for complete coverage

**Deployment**: `exporters/node-exporter.yaml`

### 2. Kube State Metrics (Deployment)

**Purpose**: Exposes Kubernetes object state as metrics

**Key Features**:
- Single replica per cluster
- Monitors pods, deployments, nodes, services, etc.
- Read-only RBAC permissions
- Cluster-wide visibility

**Deployment**: `exporters/kube-state-metrics.yaml`

### 3. vmagent (StatefulSet)

**Purpose**: Scrapes metrics and writes to central VictoriaMetrics

**Key Features**:
- **Service Discovery**: Automatically finds scrape targets
- **Label Enrichment**: Adds cluster/environment/region labels
- **Persistent Queue**: 10Gi buffer for network resilience
- **Authenticated Write**: Basic auth to central VictoriaMetrics
- **Compression**: Reduces network bandwidth
- **Parallel Queues**: 4 concurrent write streams

**Deployment**: `vmagent/vmagent.yaml`

## Configuration

### Cluster Labels (Automatic)

Each remote cluster's metrics are automatically labeled:

**Dev Cluster:**
```yaml
external_labels:
  cluster: "k8s-dev-1"
  environment: "dev"
  region: "ap-south-1"
  account: "dev-myop"
```

**Staging Cluster:**
```yaml
external_labels:
  cluster: "k8s-stage-1"
  environment: "staging"
  region: "ap-south-1"
  account: "staging-myop"
```

**Production Cluster:**
```yaml
external_labels:
  cluster: "k8s-prod-1"
  environment: "production"
  region: "ap-south-1"
  account: "prod-myop"
```

### Remote Write Configuration

vmagent writes to central VictoriaMetrics using:

```yaml
remoteWrite:
  url: http://dev-cluster:devcluster-123@victoria-metrics.infra.service/insert/0/prometheus/api/v1/write
  
  # Credentials embedded in URL (Basic Auth)
  # Format: http://username:password@host/path
  
  queues: 4                          # Parallel write streams
  maxDiskUsagePerURL: 1GB           # Buffer limit
  flushInterval: 5s                 # Write frequency
  maxBlockSize: 8MB                 # Batch size
```

**Important**: Each environment uses different credentials:
- Dev: `dev-cluster:devcluster-123`
- Staging: `stage-cluster:stagecluster-123`
- Production: `prod-cluster:prodcluster-123`

### Scrape Targets

vmagent automatically discovers and scrapes:

| Job Name | Target | Purpose |
|----------|--------|---------|
| `vmagent` | Self | vmagent health monitoring |
| `kubernetes-apiservers` | API Server | Control plane metrics |
| `kubernetes-nodes` | Kubelet | Node metrics |
| `kubernetes-nodes-cadvisor` | cAdvisor | Container metrics |
| `kubelet` | Kubelet | Kubelet-specific metrics |
| `kube-state-metrics` | KSM Service | K8s object metrics |
| `node-exporter` | Node Exporter Pods | Host hardware metrics |
| `kubernetes-pods` | Annotated Pods | Application metrics |

## Prerequisites

Before deploying to a remote cluster:

1. **Access to Remote Cluster**
   ```bash
   kubectl config use-context dev-cluster
   kubectl cluster-info
   ```

2. **Network Connectivity**
   - Remote cluster must reach `victoria-metrics.infra.service`
   - DNS resolution configured (or use IP/external hostname)
   - Firewall rules allow HTTPS traffic

3. **VictoriaMetrics Credentials**
   - Obtain credentials from infra team
   - Test connectivity: `curl -u dev-cluster:devcluster-123 http://victoria-metrics.infra.service/health`

4. **Storage Class Available**
   - vmagent needs `gp3-expandable` storage class
   - Verify: `kubectl get sc gp3-expandable`

5. **Node Pool Requirements**
   - Dev cluster uses `dev-tools-graviton` node pool
   - Adjust `nodeSelector` and `tolerations` for your environment

## Installation

### Dev Cluster Deployment

```bash
# 1. Switch to dev cluster context
kubectl config use-context k8s-dev-1

# 2. Create namespace
kubectl create namespace monitoring-dev

# 3. Deploy exporters
cd remote-clusters/dev/exporters
kubectl apply -f node-exporter.yaml
kubectl apply -f kube-state-metrics.yaml

# 4. Verify exporters are running
kubectl get pods -n monitoring-dev

# 5. Deploy vmagent
cd ../vmagent
kubectl apply -f vmagent.yaml

# 6. Verify vmagent deployment
kubectl get statefulset vmagent -n monitoring-dev
kubectl get pods -n monitoring-dev -l app=vmagent
```

### Staging/Production Deployment

```bash
# Similar steps, adjust namespace and context:
# - Namespace: monitoring-staging or monitoring-prod
# - Context: k8s-stage-1 or k8s-prod-1
# - Update labels in manifests before applying
```

## Verification

### 1. Check Pod Status

```bash
# All pods should be Running
kubectl get pods -n monitoring-dev

# Expected output:
# NAME                                  READY   STATUS    RESTARTS   AGE
# node-exporter-xxxxx                   1/1     Running   0          5m
# node-exporter-yyyyy                   1/1     Running   0          5m
# kube-state-metrics-xxxxxxxxxx-zzzzz   1/1     Running   0          5m
# vmagent-0                             1/1     Running   0          5m
```

### 2. Verify Node Exporter Coverage

```bash
# Node exporter should run on ALL nodes
NODE_COUNT=$(kubectl get nodes --no-headers | wc -l)
NE_COUNT=$(kubectl get pods -n monitoring-dev -l app.kubernetes.io/name=node-exporter --no-headers | wc -l)

echo "Nodes: $NODE_COUNT"
echo "Node Exporter pods: $NE_COUNT"
# Should match!
```

### 3. Check vmagent Logs

```bash
# View vmagent logs
kubectl logs -n monitoring-dev vmagent-0 --tail=50

# Should see successful scrapes:
# "successfully scraped target" job="node-exporter"
# "successfully scraped target" job="kube-state-metrics"
# "successfully pushed metrics" remote_storage="http://victoria-metrics.infra.service"
```

### 4. Test Metrics Endpoints

```bash
# Port-forward and test
kubectl port-forward -n monitoring-dev pod/node-exporter-xxxxx 9100:9100 &
curl http://localhost:9100/metrics | head -20

kubectl port-forward -n monitoring-dev deployment/kube-state-metrics 8080:8080 &
curl http://localhost:8080/metrics | head -20

kubectl port-forward -n monitoring-dev vmagent-0 8429:8429 &
curl http://localhost:8429/metrics | grep vmagent_
```

### 5. Verify Remote Write

```bash
# Check vmagent metrics for successful writes
kubectl port-forward -n monitoring-dev vmagent-0 8429:8429

# Check remote write stats
curl -s http://localhost:8429/metrics | grep vmagent_remotewrite

# Key metrics:
# vmagent_remotewrite_requests_total - Total write requests
# vmagent_remotewrite_requests_total{status_code="2xx"} - Successful writes
# vmagent_remotewrite_pending_data_bytes - Buffered data (should be low)
```

### 6. Query from Central Grafana

```bash
# In Grafana (victoria-metrics.infra.service), run queries:

# Check if dev cluster metrics are present
up{cluster="k8s-dev-1"}

# Check node metrics from dev
node_cpu_seconds_total{cluster="k8s-dev-1"}

# Check pod metrics from dev
kube_pod_status_phase{cluster="k8s-dev-1"}
```

## Troubleshooting

### vmagent Cannot Write to VictoriaMetrics

**Symptoms:**
- Logs show connection errors
- `vmagent_remotewrite_requests_total{status_code!="2xx"}` increasing

**Solutions:**

1. **Check connectivity:**
   ```bash
   kubectl exec -it vmagent-0 -n monitoring-dev -- /bin/sh
   wget -O- http://victoria-metrics.infra.service/health
   ```

2. **Verify credentials:**
   ```bash
   # Test with curl
   kubectl exec -it vmagent-0 -n monitoring-dev -- /bin/sh
   wget -O- --user=dev-cluster --password=devcluster-123 \
     http://victoria-metrics.infra.service/insert/0/prometheus/api/v1/write
   ```

3. **Check DNS resolution:**
   ```bash
   kubectl exec -it vmagent-0 -n monitoring-dev -- nslookup victoria-metrics.infra.service
   ```

4. **Check VMAuth logs in infra cluster:**
   ```bash
   kubectl logs -n monitoring -l app.kubernetes.io/name=vmauth | grep dev-cluster
   ```

### Node Exporter Not Running on All Nodes

**Check DaemonSet:**
```bash
kubectl describe daemonset node-exporter -n monitoring-dev

# Look for:
# - SchedulingDisabled
# - ImagePullBackOff
# - Insufficient resources
```

**Fix tolerations if needed:**
```yaml
tolerations:
  - operator: Exists  # Run on all nodes regardless of taints
```

### Kube State Metrics CrashLoopBackOff

**Check RBAC:**
```bash
kubectl auth can-i list pods --as=system:serviceaccount:monitoring-dev:kube-state-metrics

# Should return "yes"
```

**Check logs:**
```bash
kubectl logs -n monitoring-dev deployment/kube-state-metrics
```

### High vmagent Memory Usage

**Check queue size:**
```bash
kubectl exec -it vmagent-0 -n monitoring-dev -- df -h /var/lib/vmagent-remotewrite
```

**If queue is growing:**
1. Verify remote write is working
2. Increase flush frequency: `-remoteWrite.flushInterval=2s`
3. Add more queues: `-remoteWrite.queues=8`
4. Increase resources

### Metrics Not Appearing in Grafana

**Verify in order:**

1. **Exporter is running:**
   ```bash
   kubectl get pods -n monitoring-dev
   ```

2. **vmagent is scraping:**
   ```bash
   kubectl logs vmagent-0 -n monitoring-dev | grep "successfully scraped"
   ```

3. **vmagent is writing:**
   ```bash
   kubectl logs vmagent-0 -n monitoring-dev | grep "remotewrite"
   ```

4. **Check labels are correct:**
   ```bash
   # In Grafana, query with labels
   up{cluster="k8s-dev-1"}
   ```

5. **Check time range in Grafana** - metrics may be recent only

## Maintenance

### Update vmagent Version

```bash
# Edit StatefulSet
kubectl edit statefulset vmagent -n monitoring-dev

# Update image version:
# image: victoriametrics/vmagent:v1.107.0
# to: victoriametrics/vmagent:v1.116.0

# Restart
kubectl rollout restart statefulset vmagent -n monitoring-dev
```

### Update Exporter Versions

```bash
# Edit manifest files
vim exporters/node-exporter.yaml  # Update image tag
vim exporters/kube-state-metrics.yaml  # Update image tag

# Apply changes
kubectl apply -f exporters/
```

### Expand vmagent Storage

```bash
# Edit PVC (requires storage class with allowVolumeExpansion: true)
kubectl edit pvc data-vmagent-0 -n monitoring-dev

# Update size:
spec:
  resources:
    requests:
      storage: 20Gi  # Increased from 10Gi

# Delete pod to trigger resize
kubectl delete pod vmagent-0 -n monitoring-dev
```

### Change Remote Write Credentials

```bash
# Update ConfigMap or vmagent args
kubectl edit statefulset vmagent -n monitoring-dev

# Update remoteWrite.url with new credentials
# Restart vmagent
kubectl rollout restart statefulset vmagent -n monitoring-dev
```

## Resource Usage (Expected)

For a typical dev cluster with 3-5 nodes:

| Component | Replicas | CPU | Memory | Storage | Network |
|-----------|----------|-----|--------|---------|---------|
| Node Exporter | 3-5 (per node) | 50m each | 128Mi each | - | ~1KB/s each |
| Kube State Metrics | 1 | 100m | 190Mi | - | ~2KB/s |
| vmagent | 1 | 250m | 512Mi | 10Gi | ~5KB/s |
| **Total** | **5-7 pods** | **~400m** | **~1Gi** | **10Gi** | **~10KB/s** |

## Security Considerations

### Network Security
- vmagent writes over HTTP with basic auth (consider HTTPS for production)
- Credentials embedded in URL (use Kubernetes secrets for better security)
- Only vmagent needs external network access

### RBAC
- Node Exporter: Minimal permissions (just service account)
- Kube State Metrics: Cluster-wide read-only access
- vmagent: Read-only access to K8s API for service discovery

### Pod Security
- All containers run as non-root (UID 65534)
- Capabilities dropped
- Read-only root filesystem (where possible)

### Recommendations for Production
1. **Use Kubernetes Secrets** for VictoriaMetrics credentials
2. **Enable TLS** for remote write
3. **Network Policies** to restrict pod communication
4. **Resource Quotas** on monitoring namespace
5. **Pod Security Policies** or Pod Security Standards

## Next Steps

1. **Configure Grafana Dashboards**
   - Import Kubernetes cluster dashboard
   - Filter by `cluster="k8s-dev-1"`

2. **Set Up Alerts**
   - Node down alerts
   - High resource usage alerts
   - Pod crash alerts

3. **Deploy to Other Environments**
   - Repeat for staging cluster
   - Repeat for production cluster
   - Adjust labels and credentials accordingly

4. **Monitor the Monitoring**
   - Create alerts for vmagent health
   - Monitor remote write queue size
   - Track scrape success rate

## FAQ

**Q: Do I need VictoriaMetrics in remote clusters?**
A: No, remote clusters only need vmagent and exporters. VictoriaMetrics runs only in the infra cluster.

**Q: Can I have multiple vmagents?**
A: Yes, but typically one per cluster is sufficient. Use multiple only for very large clusters.

**Q: What if the infra cluster is down?**
A: vmagent will buffer metrics in its persistent queue (10Gi) and retry when connectivity is restored.

**Q: How much network bandwidth does remote write use?**
A: Typically 5-20KB/s per cluster, depending on metric cardinality. vmagent uses compression.

**Q: Can I add custom scrape targets?**
A: Yes, edit the vmagent ConfigMap and add new `scrape_configs` entries. Restart vmagent after changes.

**Q: How do I access vmagent UI?**
A: Port-forward: `kubectl port-forward vmagent-0 8429:8429 -n monitoring-dev`, then open http://localhost:8429

## References

- [vmagent Documentation](https://docs.victoriametrics.com/vmagent.html)
- [Node Exporter Collectors](https://github.com/prometheus/node_exporter#collectors)
- [Kube State Metrics](https://github.com/kubernetes/kube-state-metrics)
- [Prometheus Remote Write](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write)
