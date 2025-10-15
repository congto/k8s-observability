# Dev Cluster - Quick Setup Guide

Step-by-step guide to deploy monitoring stack in the development Kubernetes cluster.

## Overview

Deploy lightweight monitoring to dev cluster that sends all metrics to central VictoriaMetrics in infra cluster.

**What Gets Deployed:**
- Node Exporter (DaemonSet) - host metrics
- Kube State Metrics (Deployment) - K8s object metrics  
- vmagent (StatefulSet) - scraping and remote write

**What Doesn't Get Deployed:**
- VictoriaMetrics (only in infra cluster)
- Grafana (only in infra cluster)

## Prerequisites

### 1. Verify Cluster Access

```bash
# Switch to dev cluster
kubectl config use-context k8s-dev-1

# Verify connection
kubectl cluster-info
kubectl get nodes
```

### 2. Verify Network Connectivity

```bash
# Test connectivity to central VictoriaMetrics
# From any pod in dev cluster or from your machine if you have network access:

curl -v http://victoria-metrics.infra.service/health

# Or test with authentication
curl -u dev-cluster:devcluster-123 http://victoria-metrics.infra.service/health

# Should return: OK
```

### 3. Check Storage Class

```bash
# Verify gp3-expandable storage class exists
kubectl get storageclass gp3-expandable

# If not, create it or adjust vmagent.yaml to use existing storage class
```

### 4. Verify Node Pool

```bash
# Check if dev-tools-graviton node pool exists
kubectl get nodes -l nodepool=dev-tools-graviton

# If using different node pool, update nodeSelector in manifests
```

## Installation Steps

### Step 1: Create Namespace

```bash
kubectl create namespace monitoring-dev

# Verify
kubectl get namespace monitoring-dev
```

### Step 2: Deploy Node Exporter

```bash
cd remote-clusters/dev/exporters

# Review configuration
cat node-exporter.yaml

# Deploy
kubectl apply -f node-exporter.yaml

# Verify deployment
kubectl get daemonset node-exporter -n monitoring-dev
kubectl get pods -n monitoring-dev -l app.kubernetes.io/name=node-exporter

# Should see one pod per node, all Running
```

### Step 3: Deploy Kube State Metrics

```bash
# Deploy
kubectl apply -f kube-state-metrics.yaml

# Verify deployment
kubectl get deployment kube-state-metrics -n monitoring-dev
kubectl get pods -n monitoring-dev -l app.kubernetes.io/name=kube-state-metrics

# Should see 1 pod Running
```

### Step 4: Verify Exporters

```bash
# Check all exporter pods
kubectl get pods -n monitoring-dev

# Test Node Exporter metrics
POD=$(kubectl get pod -n monitoring-dev -l app.kubernetes.io/name=node-exporter -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n monitoring-dev $POD 9100:9100 &
curl -s http://localhost:9100/metrics | head -20
killall kubectl

# Test Kube State Metrics
kubectl port-forward -n monitoring-dev deployment/kube-state-metrics 8080:8080 &
curl -s http://localhost:8080/metrics | head -20
killall kubectl
```

### Step 5: Deploy vmagent

```bash
cd ../vmagent

# IMPORTANT: Review and update credentials if needed
vim vmagent.yaml

# Look for this line and verify credentials:
# -remoteWrite.url=http://dev-cluster:devcluster-123@victoria-metrics.infra.service/insert/0/prometheus/api/v1/write

# Deploy
kubectl apply -f vmagent.yaml

# Watch deployment
kubectl get statefulset vmagent -n monitoring-dev -w
# Press Ctrl+C when Ready is 1/1
```

### Step 6: Verify vmagent

```bash
# Check StatefulSet
kubectl get statefulset vmagent -n monitoring-dev

# Check pod
kubectl get pods -n monitoring-dev -l app=vmagent

# Check logs (should show successful scrapes)
kubectl logs -n monitoring-dev vmagent-0 --tail=50

# Look for:
# ✅ "successfully scraped target" job="node-exporter"
# ✅ "successfully scraped target" job="kube-state-metrics"
# ✅ "successfully pushed metrics" (or similar remote write success)
```

### Step 7: Test vmagent Metrics

```bash
# Port-forward vmagent
kubectl port-forward -n monitoring-dev vmagent-0 8429:8429 &

# Check health
curl http://localhost:8429/health
# Should return: OK

# Check vmagent metrics
curl -s http://localhost:8429/metrics | grep vmagent_remotewrite

# Key metrics to check:
# vmagent_remotewrite_requests_total{status_code="2xx"} - Successful writes
# vmagent_remotewrite_pending_data_bytes - Queue size (should be low)

# Stop port-forward
killall kubectl
```

## Post-Installation Verification

### Check All Pods Running

```bash
kubectl get pods -n monitoring-dev

# Expected output (adjust for your node count):
# NAME                                  READY   STATUS    RESTARTS   AGE
# kube-state-metrics-xxxxxxxxxx-xxxxx   1/1     Running   0          5m
# node-exporter-xxxxx                   1/1     Running   0          5m
# node-exporter-yyyyy                   1/1     Running   0          5m
# node-exporter-zzzzz                   1/1     Running   0          5m
# vmagent-0                             1/1     Running   0          3m
```

### Verify Node Exporter Coverage

```bash
# Count should match
echo "Total nodes: $(kubectl get nodes --no-headers | wc -l)"
echo "Node Exporter pods: $(kubectl get pods -n monitoring-dev -l app.kubernetes.io/name=node-exporter --no-headers | wc -l)"
```

### Check Services

```bash
kubectl get svc -n monitoring-dev

# Expected:
# NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
# kube-state-metrics   ClusterIP   10.x.x.x     <none>        8080/TCP,8081/TCP
# node-exporter        ClusterIP   None         <none>        9100/TCP
# vmagent              ClusterIP   None         <none>        8429/TCP
```

### Verify Remote Write Success

```bash
# Check vmagent logs for successful writes
kubectl logs -n monitoring-dev vmagent-0 | grep -i "write\|push\|remote" | tail -20

# Check metrics
kubectl port-forward -n monitoring-dev vmagent-0 8429:8429 &
curl -s http://localhost:8429/metrics | grep vmagent_remotewrite_requests_total
killall kubectl

# vmagent_remotewrite_requests_total{status_code="2xx"} should be increasing
```

### Query from Grafana (Final Verification)

```bash
# Access Grafana at: http://grafana.infra.service
# Or port-forward from infra cluster

# Run these queries to verify dev cluster metrics are arriving:

# 1. Check cluster is reporting
up{cluster="k8s-dev-1"}

# 2. Check node metrics
node_cpu_seconds_total{cluster="k8s-dev-1"}

# 3. Check K8s metrics
kube_pod_status_phase{cluster="k8s-dev-1"}

# 4. Count time series from dev cluster
count({cluster="k8s-dev-1"})
```

## Troubleshooting

### Issue: vmagent Cannot Connect to VictoriaMetrics

**Error in logs:**
```
dial tcp: lookup victoria-metrics.infra.service: no such host
```

**Solution 1 - Check DNS:**
```bash
kubectl exec -it vmagent-0 -n monitoring-dev -- nslookup victoria-metrics.infra.service
```

**Solution 2 - Use External URL:**
If DNS doesn't work, use the ALB hostname:
```bash
# Get ALB hostname from infra cluster
kubectl get ingress -n monitoring victoria-metrics-vm-vmauth -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# Update vmagent args in StatefulSet
kubectl edit statefulset vmagent -n monitoring-dev

# Change:
# -remoteWrite.url=http://dev-cluster:devcluster-123@victoria-metrics.infra.service/...
# To:
# -remoteWrite.url=http://dev-cluster:devcluster-123@<ALB-HOSTNAME>/...
```

### Issue: Authentication Failed (401/403)

**Error in logs:**
```
remote write: unexpected status code 401
```

**Solution:**
```bash
# Verify credentials in vmagent args
kubectl get statefulset vmagent -n monitoring-dev -o yaml | grep remoteWrite.url

# Test credentials manually
curl -v -u dev-cluster:devcluster-123 http://victoria-metrics.infra.service/health

# If fails, get correct credentials from infra team
```

### Issue: Node Exporter Not on All Nodes

**Check:**
```bash
kubectl describe daemonset node-exporter -n monitoring-dev

# Look for:
# - Pods status
# - Events showing failures
```

**Common fixes:**
- Add tolerations for node taints
- Check node selector matches your nodes
- Verify image can be pulled

### Issue: High vmagent Memory/CPU

**Check queue size:**
```bash
kubectl exec -it vmagent-0 -n monitoring-dev -- df -h /var/lib/vmagent-remotewrite
```

**If queue is growing:**
1. Verify remote write is working
2. Increase resources in StatefulSet
3. Add more write queues: `-remoteWrite.queues=8`

### Issue: Metrics Not Appearing in Grafana

**Debug steps:**
```bash
# 1. Check vmagent is scraping
kubectl logs vmagent-0 -n monitoring-dev | grep "successfully scraped"

# 2. Check vmagent is writing
kubectl logs vmagent-0 -n monitoring-dev | grep "remotewrite"

# 3. Check labels
kubectl logs vmagent-0 -n monitoring-dev | grep "external_labels"

# 4. Query with correct cluster label
# In Grafana: up{cluster="k8s-dev-1"}

# 5. Check time range - metrics may be recent only
```

## Configuration Changes

### Update Scrape Interval

```bash
kubectl edit configmap vmagent-config -n monitoring-dev

# Change:
global:
  scrape_interval: 30s  # Change to 60s for less frequent scraping

# Restart vmagent
kubectl rollout restart statefulset vmagent -n monitoring-dev
```

### Add Custom Scrape Target

```bash
kubectl edit configmap vmagent-config -n monitoring-dev

# Add new scrape_config:
scrape_configs:
  # ... existing configs ...
  
  - job_name: 'my-app'
    static_configs:
      - targets: ['my-app-service.default.svc.cluster.local:8080']

# Restart vmagent
kubectl rollout restart statefulset vmagent -n monitoring-dev
```

### Change Remote Write URL

```bash
kubectl edit statefulset vmagent -n monitoring-dev

# Update args section:
args:
  - "-remoteWrite.url=http://NEW-URL/..."

# Save and exit (pod will restart automatically)
```

## Cleanup (If Needed)

```bash
# Delete all monitoring components
kubectl delete namespace monitoring-dev

# This removes:
# - vmagent (and its PVC)
# - Node Exporter
# - Kube State Metrics
# - All associated resources
```

## Next Steps

1. **Monitor vmagent Health**
   - Set up alerts for vmagent down
   - Monitor remote write queue size

2. **Create Dashboards in Grafana**
   - Import Kubernetes cluster dashboard
   - Filter by `cluster="k8s-dev-1"`

3. **Set Up Alerts**
   - Node down alerts
   - Pod crash alerts
   - High resource usage

4. **Deploy to Other Environments**
   - Follow same process for staging
   - Follow same process for production
   - Update labels and credentials accordingly

## Quick Reference

```bash
# View all monitoring pods
kubectl get pods -n monitoring-dev

# Check vmagent logs
kubectl logs -f vmagent-0 -n monitoring-dev

# Port-forward vmagent UI
kubectl port-forward -n monitoring-dev vmagent-0 8429:8429

# Restart vmagent
kubectl rollout restart statefulset vmagent -n monitoring-dev

# Check remote write stats
kubectl exec -it vmagent-0 -n monitoring-dev -- wget -O- http://localhost:8429/metrics | grep remotewrite

# Delete and redeploy vmagent
kubectl delete -f vmagent/vmagent.yaml
kubectl apply -f vmagent/vmagent.yaml
```

## Support

If you encounter issues:
1. Check logs: `kubectl logs -n monitoring-dev <pod-name>`
2. Check events: `kubectl describe pod -n monitoring-dev <pod-name>`
3. Verify connectivity to infra cluster
4. Contact infra team for VictoriaMetrics credentials

---

**Installation Time**: ~10 minutes  
**Components**: 3 (Node Exporter, KSM, vmagent)  
**Storage Required**: 10Gi (vmagent buffer)  
**Network**: ~10KB/s average remote write traffic
