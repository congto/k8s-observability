# Kubernetes Dashboards

Pre-configured Grafana dashboard JSON files for comprehensive Kubernetes monitoring.

## 📊 Available Dashboards

### 1. Kubernetes Cluster Overview
**File**: `k8s-cluster-overview.json`

Complete cluster-level monitoring dashboard showing:
- Total nodes, pods, and containers
- Cluster CPU and memory utilization
- Network bandwidth usage
- Storage capacity and usage
- Top pods by resource consumption
- Node status and conditions

**Best for**: Daily operations, overall cluster health

<img width="1910" height="937" alt="image" src="https://github.com/user-attachments/assets/4353b21b-3807-4487-ab0d-b4eeec867731" />

<img width="1914" height="949" alt="image" src="https://github.com/user-attachments/assets/c2e7da38-fe5b-44cd-82ec-06534fe1edd5" />

---

### 2. Kubernetes Namespace Overview
**File**: `k8s-namespace-overview.json`

Namespace-level resource monitoring:
- Resource quotas and limits
- Pod count per namespace
- CPU and memory usage by namespace
- Network traffic by namespace
- Storage usage per namespace
- Top namespaces by resource consumption

**Best for**: Multi-tenant clusters, namespace-level troubleshooting

<img width="1916" height="812" alt="image" src="https://github.com/user-attachments/assets/85f75b16-ed65-4d2b-9570-e2c3b583044c" />
<img width="1913" height="811" alt="image" src="https://github.com/user-attachments/assets/1b73e257-5058-438b-b419-db3c26a74459" />


---

### 3. Kubernetes Namespace Deep Dive
**File**: `k8s-namespace-deepdive.json`

Detailed namespace analysis:
- Per-pod resource utilization
- Container-level metrics
- Pod lifecycle events
- Resource requests vs actual usage
- Network policies and traffic
- Persistent volume claims

**Best for**: Debugging namespace issues, detailed investigation

<img width="1911" height="932" alt="image" src="https://github.com/user-attachments/assets/29b36dbb-cb91-4804-9639-6d5183523362" />
<img width="1911" height="762" alt="image" src="https://github.com/user-attachments/assets/01d29826-81ae-4d3d-8648-71c61b754dc1" />
<img width="1907" height="704" alt="image" src="https://github.com/user-attachments/assets/3fbc1ad8-7d65-4306-a279-6c3a174c00af" />
<img width="1907" height="432" alt="image" src="https://github.com/user-attachments/assets/8ea70b6c-ba66-4d93-baee-ee4ed4c57e74" />



---

### 4. Kubernetes Deployment & Service Overview
**File**: `k8s-deployment-service-overview.json`

Deployment and service monitoring:
- Deployment rollout status
- Replica count (desired vs actual)
- Service endpoints health
- Pod restarts and failures
- Deployment history
- Service latency and errors

**Best for**: Application deployment monitoring, service health

<img width="1904" height="782" alt="image" src="https://github.com/user-attachments/assets/ad8ff48d-dc09-4ae1-be57-868e0934a922" />
<img width="1912" height="876" alt="image" src="https://github.com/user-attachments/assets/7a65e603-0262-49e6-b0f4-3b5e1bf283ee" />


---

### 5. Kubernetes Nodes Overview
**File**: `k8s-nodes-overview.json`

Node-level metrics and health:
- CPU usage per node
- Memory usage per node
- Disk I/O per node
- Network traffic per node
- Node conditions (Ready, MemoryPressure, DiskPressure)
- Pods per node
- Node capacity and allocatable resources

**Best for**: Node troubleshooting, capacity planning

<img width="1913" height="862" alt="image" src="https://github.com/user-attachments/assets/7763dead-f63f-422d-b62b-51e82b98f2f2" />
<img width="1900" height="934" alt="image" src="https://github.com/user-attachments/assets/10efdbae-1d7a-4134-9db1-c58eaf124981" />


---

### 6. Kubernetes Namespace Pods Deep Dive
**File**: `k8s-namespace-pods-deepdive.json`

Pod-level detailed monitoring:
- Pod status (Running, Pending, Failed, Unknown)
- Container restarts by pod
- CPU and memory per container
- Network I/O per pod
- Filesystem usage per pod
- Pod events and logs
- Container state transitions

**Best for**: Pod-level debugging, container issues
<img width="1901" height="924" alt="image" src="https://github.com/user-attachments/assets/6f040d3d-cb4b-4401-a7f0-e5f743ec81f9" />
<img width="1915" height="848" alt="image" src="https://github.com/user-attachments/assets/8e92ed2a-e6f0-40f4-b39e-1e01e42977ed" />
<img width="1911" height="872" alt="image" src="https://github.com/user-attachments/assets/7e49da79-e0a1-43d9-85e6-eaf8bbb8a721" />

---

## 🚀 Quick Import

### Method 1: Via Grafana UI (Recommended)

```bash
# 1. Login to Grafana
http://grafana.infra.service

# 2. Navigate to Dashboards → Import

# 3. Upload JSON file or paste content

# 4. Configure:
#    - Name: Keep or customize
#    - Folder: Select destination
#    - Data Source: Select "VictoriaMetrics"

# 5. Click "Import"
```

### Method 2: Via Grafana API

```bash
# Set variables
export GRAFANA_URL="http://grafana.infra.service"
export GRAFANA_API_KEY="your-api-key-here"

# Import all dashboards
for dashboard in *.json; do
  echo "Importing $dashboard..."
  
  # Wrap JSON in required format
  jq -n --slurpfile dashboard "$dashboard" \
    '{dashboard: $dashboard[0], overwrite: true}' | \
  curl -X POST \
    -H "Authorization: Bearer $GRAFANA_API_KEY" \
    -H "Content-Type: application/json" \
    -d @- \
    "$GRAFANA_URL/api/dashboards/db"
  
  echo ""
done
```

### Method 3: Automated Import Script

```bash
#!/bin/bash
# import-dashboards.sh

GRAFANA_URL="http://grafana.infra.service"
GRAFANA_USER="admin"
GRAFANA_PASS="your-password"

for dashboard in dashboards/*.json; do
  echo "Importing $(basename $dashboard)..."
  
  curl -X POST \
    -u "$GRAFANA_USER:$GRAFANA_PASS" \
    -H "Content-Type: application/json" \
    -d @"$dashboard" \
    "$GRAFANA_URL/api/dashboards/db"
done
```

---

## 📋 Dashboard Variables

All dashboards include these variables for filtering:

| Variable | Type | Description | Example Values |
|----------|------|-------------|----------------|
| `$datasource` | Datasource | VictoriaMetrics datasource | VictoriaMetrics |
| `$cluster` | Query | Select cluster | eks-aps1-dev-1, eks-aps1-prod-1 |
| `$namespace` | Query | Filter by namespace | default, kube-system, monitoring |
| `$node` | Query | Select specific node | ip-10-0-1-100.ec2.internal |
| `$deployment` | Query | Filter by deployment | nginx, api-service |
| `$pod` | Query | Select specific pod | nginx-abc123, api-xyz789 |

### Variable Queries Used

```promql
# Cluster selection
label_values(up, cluster)

# Namespace selection  
label_values(kube_namespace_status_phase{cluster="$cluster"}, namespace)

# Node selection
label_values(kube_node_info{cluster="$cluster"}, node)

# Deployment selection
label_values(kube_deployment_labels{cluster="$cluster", namespace="$namespace"}, deployment)

# Pod selection
label_values(kube_pod_info{cluster="$cluster", namespace="$namespace"}, pod)
```

---

## 🎯 Use Cases

### Daily Operations
- **Start with**: Cluster Overview
- **Check**: Node Overview for capacity
- **Monitor**: Deployment & Service Overview for app health

### Troubleshooting
- **Application Issues**: Deployment → Namespace Pods Deep Dive
- **Node Problems**: Nodes Overview → Check specific node metrics
- **Namespace Investigation**: Namespace Overview → Namespace Deep Dive

### Capacity Planning
- **Cluster Level**: Cluster Overview (trends over 7-30 days)
- **Node Level**: Nodes Overview (utilization patterns)
- **Namespace Level**: Namespace Overview (quota usage)

### Multi-Cluster Monitoring
All dashboards support the `$cluster` variable:
- Switch between dev/staging/prod with dropdown
- Compare metrics across environments
- Track deployment differences

---

## 🔧 Customization

### Adding Custom Panels

1. **Open dashboard in Grafana**
2. **Click "Add panel"** (top right)
3. **Add new panel**
4. **Enter query**:
   ```promql
   # Example: Custom metric
   your_custom_metric{cluster="$cluster", namespace="$namespace"}
   ```
5. **Configure visualization**
6. **Save dashboard**

### Modifying Queries

1. **Edit panel** (click panel title → Edit)
2. **Update query** in Query tab
3. **Test** with "Run queries"
4. **Apply** changes
5. **Save dashboard**

### Changing Time Ranges

Default time ranges:
- **Overview dashboards**: Last 1 hour
- **Deep dive dashboards**: Last 15 minutes
- **Trend analysis**: Last 24 hours

Change via dashboard settings or time picker (top right).

### Adjusting Refresh Rate

1. **Click refresh icon** (top right)
2. **Select interval**: 5s, 10s, 30s, 1m, 5m
3. **Or set auto-refresh** in dashboard settings

---

## 📈 Key Metrics Explained

### CPU Metrics
```promql
# Node CPU usage
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Pod CPU usage
sum(rate(container_cpu_usage_seconds_total{pod="$pod"}[5m])) by (container)
```

### Memory Metrics
```promql
# Node memory usage
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Pod memory usage
sum(container_memory_working_set_bytes{pod="$pod"}) by (container)
```

### Network Metrics
```promql
# Network receive rate
rate(node_network_receive_bytes_total[5m])

# Network transmit rate
rate(node_network_transmit_bytes_total[5m])
```

### Pod Metrics
```promql
# Pods by phase
count(kube_pod_status_phase{phase="Running"}) by (namespace)

# Pod restarts
increase(kube_pod_container_status_restarts_total[1h]) > 0
```

---

## 🔍 Troubleshooting

### Dashboard shows "No data"

**Check:**
1. ✅ VictoriaMetrics datasource is selected
2. ✅ Time range includes recent data
3. ✅ Variables are populated (cluster, namespace)
4. ✅ Metrics exist in VictoriaMetrics

**Test query in Explore:**
```promql
up{cluster="eks-aps1-dev-1"}
```

### Variables are empty

**Fix:**
1. Check datasource is configured
2. Verify variable query syntax
3. Ensure metrics have required labels
4. Check regex filters in variable settings

### Dashboard loads slowly

**Optimize:**
1. Reduce time range (24h → 1h)
2. Limit max data points (Settings → Query options)
3. Use recording rules for complex queries
4. Disable auto-refresh during investigation

---

## 📤 Exporting Dashboards

### Export for Backup

```bash
# From Grafana UI:
# 1. Open dashboard
# 2. Click ⚙️ (Settings)
# 3. JSON Model
# 4. Copy to clipboard or save to file

# From API:
DASHBOARD_UID="your-dashboard-uid"
curl -H "Authorization: Bearer $GRAFANA_API_KEY" \
  "$GRAFANA_URL/api/dashboards/uid/$DASHBOARD_UID" | \
  jq '.dashboard' > dashboard-backup.json
```

### Share Dashboard

1. Click **"Share"** (top right)
2. Select **"Export"** tab
3. Toggle **"Export for sharing externally"**
4. Click **"Save to file"**
5. Commit to git repository

---

## 🛡️ Best Practices

### Dashboard Organization

✅ **Create Folders**:
- `Kubernetes` - All K8s dashboards
- `Infrastructure` - VictoriaMetrics, node metrics
- `Applications` - App-specific dashboards

✅ **Use Tags**:
- `kubernetes`, `monitoring`, `infrastructure`
- Makes searching easier

✅ **Version Control**:
- Commit dashboard JSON to git
- Track changes over time
- Easy rollback if needed

### Query Optimization

✅ **Use Variables**: 
- Filter by cluster/namespace to reduce data
- Improves performance

✅ **Appropriate Time Ranges**:
- Real-time monitoring: 5m-1h
- Troubleshooting: 15m-4h
- Trend analysis: 24h-7d

✅ **Limit Visualizations**:
- Keep dashboards focused (8-12 panels max)
- Create multiple dashboards instead of one huge one

---

## 📚 Additional Resources

### Documentation
- [Grafana Dashboards](https://grafana.com/docs/grafana/latest/dashboards/)
- [PromQL Guide](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [VictoriaMetrics MetricsQL](https://docs.victoriametrics.com/MetricsQL.html)

### Community Dashboards
- [Grafana Dashboard Gallery](https://grafana.com/grafana/dashboards/)
- Search tags: `kubernetes`, `prometheus`, `victoriametrics`

### Metrics Documentation
- [Kube State Metrics](https://github.com/kubernetes/kube-state-metrics/tree/main/docs)
- [Node Exporter](https://github.com/prometheus/node_exporter#collectors)
- [cAdvisor Metrics](https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md)

---

## 🤝 Contributing

### Adding New Dashboards

1. **Create dashboard in Grafana**
2. **Test with real data**
3. **Export JSON**:
   - Settings → JSON Model → Copy
4. **Save to file**:
   - Naming: `k8s-{purpose}-{detail}.json`
   - Example: `k8s-ingress-monitoring.json`
5. **Update this README**
6. **Submit PR**

### Improving Existing Dashboards

1. **Make changes in Grafana**
2. **Test thoroughly**
3. **Export updated JSON**
4. **Replace file in repo**
5. **Document changes in commit message**
6. **Submit PR**

---

## ℹ️ Dashboard Details

| Dashboard | Panels | Variables | Best For |
|-----------|--------|-----------|----------|
| Cluster Overview | 12 | cluster | Daily monitoring |
| Namespace Overview | 10 | cluster, namespace | Multi-tenant |
| Namespace Deep Dive | 15 | cluster, namespace | Investigation |
| Deployment & Service | 14 | cluster, namespace, deployment | App monitoring |
| Nodes Overview | 16 | cluster, node | Node troubleshooting |
| Namespace Pods Deep Dive | 18 | cluster, namespace, pod | Pod debugging |

---

**Need help?** Check the [main documentation](../docs/) or open an issue on [GitHub](https://github.com/jasaulakh1988/k8s-observability/issues).
