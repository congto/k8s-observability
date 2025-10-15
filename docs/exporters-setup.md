# Exporters - Complete Setup Guide

Step-by-step guide to deploy Node Exporter and Kube State Metrics for comprehensive Kubernetes monitoring.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Pre-Installation Checks](#pre-installation-checks)
3. [Node Exporter Installation](#node-exporter-installation)
4. [Kube State Metrics Installation](#kube-state-metrics-installation)
5. [Verification and Testing](#verification-and-testing)
6. [Integration with vmagent](#integration-with-vmagent)

---

## Prerequisites

### Required Tools
- `kubectl` (v1.24+)
- Cluster admin access

### Cluster Requirements
- Kubernetes 1.24 or higher
- Monitoring namespace
- Sufficient node resources

### Expected Resource Usage

**Per Node (Node Exporter)**:
- CPU: ~10-20m average
- Memory: ~50-100Mi

**Cluster-wide (Kube State Metrics)**:
- CPU: 100-200m (scales with cluster size)
- Memory: 190-512Mi (scales with object count)

---

## Pre-Installation Checks

### 1. Check Cluster Version

```bash
kubectl version --short

# Should be v1.24 or higher
```

### 2. Create Namespace

```bash
# Check if namespace exists
kubectl get namespace monitoring

# If not, create it
kubectl create namespace monitoring

# Verify
kubectl get namespace monitoring
```

### 3. Check Node Count

```bash
# Count nodes (Node Exporter will run on all)
kubectl get nodes --no-headers | wc -l

# Get node details
kubectl get nodes -o wide
```

### 4. Verify Available Resources

```bash
# Check node resources
kubectl describe nodes | grep -A 5 "Allocated resources"

# Ensure sufficient allocatable resources for exporters
```

---

## Node Exporter Installation

Node Exporter collects hardware and OS metrics from each node in your cluster.

### Step 1: Review Configuration

```bash
cd infra-cluster/exporters

# Review the manifest
cat node-exporter.yaml

# Key points to verify:
# - Image version: v1.9.1
# - Resource limits appropriate for your cluster
# - Tolerations allow running on all nodes
```

### Step 2: Deploy Node Exporter

```bash
# Apply the manifest
kubectl apply -f node-exporter.yaml

# Output should show:
# serviceaccount/node-exporter created
# daemonset.apps/node-exporter created
# service/node-exporter created
```

### Step 3: Verify Deployment

```bash
# Check DaemonSet status
kubectl get daemonset node-exporter -n monitoring

# Should show:
# NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
# node-exporter   4         4         4       4            4
# (Numbers should match your node count)

# Check pods
kubectl get pods -n monitoring -l app.kubernetes.io/name=node-exporter -o wide

# Should see one pod per node, all Running
```

### Step 4: Check Service

```bash
# Verify service
kubectl get svc node-exporter -n monitoring

# Should be ClusterIP: None (headless service)
# Port: 9100
```

### Step 5: Test Metrics Endpoint

```bash
# Get one pod name
POD_NAME=$(kubectl get pod -n monitoring -l app.kubernetes.io/name=node-exporter -o jsonpath='{.items[0].metadata.name}')

# Port-forward to test
kubectl port-forward -n monitoring $POD_NAME 9100:9100 &

# Test metrics (in another terminal or after backgrounding)
curl -s http://localhost:9100/metrics | head -20

# Should see Prometheus-formatted metrics starting with node_*
# Example output:
# node_cpu_seconds_total{cpu="0",mode="idle"} 12345.67
# node_memory_MemTotal_bytes 16777216000
# node_disk_read_bytes_total{device="sda"} 1234567890

# Stop port-forward
killall kubectl
```

### Step 6: Verify on All Nodes

```bash
# Check each node has a pod
kubectl get pods -n monitoring -l app.kubernetes.io/name=node-exporter -o wide

# Verify pod-to-node mapping
for node in $(kubectl get nodes -o name); do
  echo "Node: $node"
  kubectl get pods -n monitoring -l app.kubernetes.io/name=node-exporter \
    --field-selector spec.nodeName=${node#node/} -o wide
done
```

---

## Kube State Metrics Installation

Kube State Metrics exposes Kubernetes object state as Prometheus metrics.

### Step 1: Review Configuration

```bash
# Review the manifest
cat kube-state-metrics.yaml

# Key components:
# - ServiceAccount
# - ClusterRole (read-only access to K8s resources)
# - ClusterRoleBinding
# - Deployment (1 replica)
# - Service
```

### Step 2: Deploy Kube State Metrics

```bash
# Apply the manifest
kubectl apply -f kube-state-metrics.yaml

# Output should show:
# serviceaccount/kube-state-metrics created
# clusterrole.rbac.authorization.k8s.io/kube-state-metrics created
# clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics created
# deployment.apps/kube-state-metrics created
# service/kube-state-metrics created
```

### Step 3: Verify Deployment

```bash
# Check deployment status
kubectl get deployment kube-state-metrics -n monitoring

# Should show:
# NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
# kube-state-metrics    1/1     1            1           30s

# Check pod
kubectl get pods -n monitoring -l app.kubernetes.io/name=kube-state-metrics

# Should see one pod in Running state
```

### Step 4: Check RBAC

```bash
# Verify ClusterRole exists
kubectl get clusterrole kube-state-metrics

# Verify ClusterRoleBinding
kubectl get clusterrolebinding kube-state-metrics

# Test permissions (should return "yes" for all)
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:kube-state-metrics
kubectl auth can-i list nodes --as=system:serviceaccount:monitoring:kube-state-metrics
kubectl auth can-i list deployments --as=system:serviceaccount:monitoring:kube-state-metrics
```

### Step 5: Check Service

```bash
# Verify service
kubectl get svc kube-state-metrics -n monitoring

# Should show:
# - Type: ClusterIP
# - Ports: 8080 (metrics), 8081 (telemetry)
```

### Step 6: Test Metrics Endpoint

```bash
# Port-forward to test
kubectl port-forward -n monitoring deployment/kube-state-metrics 8080:8080 &

# Test metrics endpoint
curl -s http://localhost:8080/metrics | head -30

# Should see Kubernetes object metrics like:
# kube_pod_status_phase{namespace="default",pod="nginx",phase="Running"} 1
# kube_deployment_status_replicas{namespace="default",deployment="nginx"} 3
# kube_node_status_condition{node="node1",condition="Ready",status="true"} 1

# Test health endpoint
curl http://localhost:8080/healthz
# Should return: ok

# Stop port-forward
killall kubectl
```

### Step 7: Check Pod Logs

```bash
# View logs to confirm it's working
kubectl logs -n monitoring deployment/kube-state-metrics --tail=20

# Should see messages like:
# "Watching X resource"
# No errors about API connectivity
```

---

## Verification and Testing

### Overall Health Check

```bash
# Check all exporter pods
kubectl get pods -n monitoring -l 'app.kubernetes.io/name in (node-exporter,kube-state-metrics)'

# All should be Running
```

### Metrics Availability Check

```bash
# Create a test script
cat > test-exporters.sh <<'EOF'
#!/bin/bash

echo "Testing Node Exporter..."
POD=$(kubectl get pod -n monitoring -l app.kubernetes.io/name=node-exporter -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n monitoring $POD 9100:9100 &
PF_PID=$!
sleep 2
NODE_METRICS=$(curl -s http://localhost:9100/metrics | grep -c "node_")
kill $PF_PID
echo "Node Exporter metrics count: $NODE_METRICS"

echo -e "\nTesting Kube State Metrics..."
kubectl port-forward -n monitoring deployment/kube-state-metrics 8080:8080 &
PF_PID=$!
sleep 2
KSM_METRICS=$(curl -s http://localhost:8080/metrics | grep -c "kube_")
kill $PF_PID
echo "Kube State Metrics count: $KSM_METRICS"

if [ $NODE_METRICS -gt 100 ] && [ $KSM_METRICS -gt 100 ]; then
  echo -e "\n✅ Both exporters are working correctly!"
else
  echo -e "\n❌ Some exporters may have issues"
fi
EOF

chmod +x test-exporters.sh
./test-exporters.sh
```

### Resource Usage Check

```bash
# Check actual resource usage
kubectl top pods -n monitoring -l 'app.kubernetes.io/name in (node-exporter,kube-state-metrics)'

# Compare against limits
kubectl describe pods -n monitoring -l app.kubernetes.io/name=kube-state-metrics | grep -A 5 "Limits"
```

### Service Discovery Check

```bash
# Verify Prometheus annotations
kubectl get pods -n monitoring -l app.kubernetes.io/name=node-exporter -o jsonpath='{.items[0].metadata.annotations}' | jq

# Should show:
# {
#   "prometheus.io/scrape": "true",
#   "prometheus.io/port": "9100"
# }

kubectl get pods -n monitoring -l app.kubernetes.io/name=kube-state-metrics -o jsonpath='{.items[0].metadata.annotations}' | jq

# Should show:
# {
#   "prometheus.io/scrape": "true",
#   "prometheus.io/port": "8080"
# }
```

---

## Integration with vmagent

Now that exporters are running, configure vmagent to scrape them.

### vmagent Scrape Configuration

Add to your vmagent ConfigMap:

```yaml
scrape_configs:
  # Node Exporter - scrape from all nodes
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      # Replace kubelet port (10250) with node-exporter port (9100)
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
      
      # Add node labels as metric labels
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      
      # Add instance label
      - source_labels: [__meta_kubernetes_node_name]
        target_label: instance
      
      # Add cluster label
      - replacement: 'infra-cluster'
        target_label: cluster

  # Kube State Metrics - static config
  - job_name: 'kube-state-metrics'
    static_configs:
      - targets: ['kube-state-metrics.monitoring.svc.cluster.local:8080']
    relabel_configs:
      - replacement: 'infra-cluster'
        target_label: cluster
```

### Apply vmagent Configuration

```bash
# Update vmagent ConfigMap
kubectl edit configmap vmagent-config -n monitoring

# Add the scrape_configs above

# Reload vmagent (if it supports reload signal)
kubectl rollout restart deployment vmagent -n monitoring

# Or delete pod to pick up new config
kubectl delete pod -n monitoring -l app=vmagent
```

### Verify Scraping

```bash
# Check vmagent logs
kubectl logs -n monitoring -l app=vmagent --tail=100 | grep -E 'node-exporter|kube-state-metrics'

# Should see successful scrapes, e.g.:
# "Scrape succeeded" job="kubernetes-nodes"
# "Scrape succeeded" job="kube-state-metrics"
```

### Query Metrics in VictoriaMetrics

```bash
# Port-forward to VictoriaMetrics
kubectl port-forward -n monitoring svc/victoria-metrics-vm-vmselect 8481:8481 &

# Query node metrics
curl -s 'http://localhost:8481/select/0/prometheus/api/v1/query?query=node_cpu_seconds_total' | jq

# Query kube metrics
curl -s 'http://localhost:8481/select/0/prometheus/api/v1/query?query=kube_pod_status_phase' | jq

# Check target count
curl -s 'http://localhost:8481/select/0/prometheus/api/v1/targets' | jq '.data.activeTargets | length'
```

---

## Troubleshooting

### Node Exporter Not Starting

**Check DaemonSet events:**
```bash
kubectl describe daemonset node-exporter -n monitoring
```

**Common issues:**
1. **Insufficient node resources**: Check node allocatable resources
2. **Image pull errors**: Verify internet connectivity
3. **Host path permissions**: Check SELinux/AppArmor policies

**Check pod logs:**
```bash
kubectl logs -n monitoring POD_NAME
```

### Kube State Metrics CrashLoopBackOff

**Check logs:**
```bash
kubectl logs -n monitoring deployment/kube-state-metrics --tail=100
```

**Common issues:**
1. **RBAC permissions**: Verify ClusterRole and ClusterRoleBinding
   ```bash
   kubectl get clusterrole kube-state-metrics -o yaml
   kubectl get clusterrolebinding kube-state-metrics -o yaml
   ```

2. **API server connectivity**: Check service account token
   ```bash
   kubectl exec -n monitoring deployment/kube-state-metrics -- \
     cat /var/run/secrets/kubernetes.io/serviceaccount/token
   ```

3. **Resource limits**: Increase if OOMKilled
   ```bash
   kubectl describe pod -n monitoring -l app.kubernetes.io/name=kube-state-metrics | grep -A 5 "State"
   ```

### No Metrics Available

**Check if endpoints are accessible:**
```bash
# From within cluster
kubectl run curl-test --image=curlimages/curl -it --rm -- \
  curl http://node-exporter.monitoring.svc.cluster.local:9100/metrics

kubectl run curl-test --image=curlimages/curl -it --rm -- \
  curl http://kube-state-metrics.monitoring.svc.cluster.local:8080/metrics
```

### High Memory Usage (KSM)

For large clusters, Kube State Metrics may need more memory:

```bash
# Check current usage
kubectl top pod -n monitoring -l app.kubernetes.io/name=kube-state-metrics

# If consistently high, edit deployment
kubectl edit deployment kube-state-metrics -n monitoring

# Increase limits:
resources:
  requests:
    memory: 512Mi
  limits:
    memory: 1Gi
```

### Missing Node Exporter Pods

**If pod count doesn't match node count:**
```bash
# Check which nodes are missing pods
diff <(kubectl get nodes -o name | sort) \
     <(kubectl get pods -n monitoring -l app.kubernetes.io/name=node-exporter \
       -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort)

# Check DaemonSet status
kubectl get daemonset node-exporter -n monitoring -o yaml | grep -A 10 status
```

---

## Maintenance

### Updating Exporters

```bash
# Update image version in YAML
vim node-exporter.yaml
# Change: image: quay.io/prometheus/node-exporter:v1.9.1
# To:     image: quay.io/prometheus/node-exporter:v1.10.0

# Apply changes
kubectl apply -f node-exporter.yaml

# Watch rollout
kubectl rollout status daemonset/node-exporter -n monitoring

# Similarly for KSM
vim kube-state-metrics.yaml
kubectl apply -f kube-state-metrics.yaml
kubectl rollout status deployment/kube-state-metrics -n monitoring
```

### Restart Exporters

```bash
# Restart Node Exporter (rolling restart)
kubectl rollout restart daemonset/node-exporter -n monitoring

# Restart Kube State Metrics
kubectl rollout restart deployment/kube-state-metrics -n monitoring
```

---

## Performance Tuning

### For Large Clusters (100+ nodes)

**Kube State Metrics:**
```yaml
resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: 1
    memory: 2Gi
```

**Enable KSM sharding** (for 500+ nodes):
```yaml
# Create 2 replicas with sharding
replicas: 2
containers:
- name: kube-state-metrics
  args:
    - --shard=0
    - --total-shards=2
```

### Reduce Metric Cardinality

If metrics volume is too high, exclude certain collectors:

**Node Exporter:**
```yaml
args:
  # Add to exclude expensive collectors
  - --no-collector.arp
  - --no-collector.bcache
  - --no-collector.bonding
```

---

## Next Steps

1. **Configure Grafana Dashboards**
   - Import Node Exporter dashboard (ID: 1860)
   - Import Kubernetes dashboard (ID: 315)

2. **Set Up Alerts**
   - Node down alerts
   - High CPU/Memory alerts
   - Pod restart alerts

3. **Deploy Additional Exporters**
   - Application-specific exporters
   - Database exporters

4. **Implement Monitoring for Exporters**
   - Alert on exporter downtime
   - Monitor exporter resource usage

---

## Validation Checklist

- [ ] Node Exporter pods running on all nodes
- [ ] Kube State Metrics pod running
- [ ] Both services created and accessible
- [ ] Metrics endpoints returning data
- [ ] vmagent successfully scraping exporters
- [ ] Metrics visible in VictoriaMetrics/Grafana
- [ ] No errors in exporter logs
- [ ] Resource usage within limits

---

## Support Resources

- **Node Exporter**: https://github.com/prometheus/node_exporter
- **Kube State Metrics**: https://github.com/kubernetes/kube-state-metrics
- **Troubleshooting**: Check logs and GitHub issues
