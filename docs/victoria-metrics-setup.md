# VictoriaMetrics Cluster - Complete Setup Guide

This guide walks through deploying a production-ready VictoriaMetrics cluster on Amazon EKS for centralized metrics storage.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Infrastructure Preparation](#infrastructure-preparation)
3. [Installation Steps](#installation-steps)
4. [Post-Installation Verification](#post-installation-verification)
5. [Configuration Deep Dive](#configuration-deep-dive)
6. [Integration with Remote Clusters](#integration-with-remote-clusters)

---

## Prerequisites

### Required Tools
- `kubectl` (v1.24+)
- `helm` (v3.10+)
- `aws-cli` (v2.x) configured with appropriate credentials

### AWS Infrastructure
- EKS cluster running (1.28+)
- Node group with ARM64 instances (labeled: `nodepool=general-arm`, `architecture=arm64`)
- VPC with proper networking setup
- IAM roles configured for service accounts (IRSA)
- gp3 storage class available

### Access Requirements
- Cluster admin access
- Docker Hub account (for pulling VM images)
- AWS IAM permissions for creating load balancers

---

## Infrastructure Preparation

### 1. Create IAM Role for VictoriaMetrics

Create an IAM role for the service account to access AWS services (if needed):

```bash
# Set variables
CLUSTER_NAME="eks-aps1-infra-1"
REGION="ap-south-1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create IAM policy (optional - if using AWS services)
cat > vm-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::your-backup-bucket",
        "arn:aws:s3:::your-backup-bucket/*"
      ]
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name VictoriaMetricsStoragePolicy \
  --policy-document file://vm-policy.json

# Create IAM role with IRSA
eksctl create iamserviceaccount \
  --cluster=$CLUSTER_NAME \
  --region=$REGION \
  --namespace=monitoring \
  --name=victoria-metrics-storage \
  --attach-policy-arn=arn:aws:iam::${ACCOUNT_ID}:policy/VictoriaMetricsStoragePolicy \
  --approve \
  --override-existing-serviceaccounts
```

### 2. Verify Storage Class

```bash
kubectl get storageclass gp3

# If not exists, create it
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF
```

### 3. Create Namespace and Secrets

```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Create Docker Hub secret
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=YOUR_USERNAME \
  --docker-password=YOUR_PASSWORD \
  --docker-email=YOUR_EMAIL \
  -n monitoring

# Verify secret
kubectl get secret dockerhub-secret -n monitoring
```

---

## Installation Steps

### Step 1: Clone Repository

```bash
git clone https://github.com/YOUR_ORG/k8s-observability.git
cd k8s-observability/infra-cluster/helm/victoria-metrics/base
```

### Step 2: Review and Customize Values

#### Update passwords in `infra/values.yaml`:

```bash
# Edit the file
vim infra/values.yaml

# Update these sections with strong passwords:
vm:
  vmauth:
    config:
      users:
        - username: "dev-cluster"
          password: "CHANGE_THIS_PASSWORD"  # Use strong password
        - username: "stage-cluster"
          password: "CHANGE_THIS_PASSWORD"
        - username: "readonly"
          password: "CHANGE_THIS_PASSWORD"
        - username: "writeonly"
          password: "CHANGE_THIS_PASSWORD"
        - username: "admin"
          password: "CHANGE_THIS_PASSWORD"
```

#### Adjust resource limits if needed:

```yaml
# For smaller clusters, reduce resources:
vmstorage:
  resources:
    requests:
      cpu: "250m"      # Reduced from 500m
      memory: "512Mi"  # Reduced from 1Gi
```

#### Update ingress CIDR ranges:

```yaml
vmauth:
  ingress:
    annotations:
      alb.ingress.kubernetes.io/inbound-cidrs: "YOUR_VPC_CIDR,OTHER_ALLOWED_CIDRS"
```

### Step 3: Update Helm Dependencies

```bash
# Update dependencies to download the VM chart
helm dependency update

# Verify charts downloaded
ls charts/
# Should see: victoria-metrics-cluster-0.24.1.tgz
```

### Step 4: Dry Run (Optional but Recommended)

```bash
# Test the configuration
helm install victoria-metrics . \
  --namespace monitoring \
  --values values.yaml \
  --values infra/values.yaml \
  --dry-run --debug
```

### Step 5: Install VictoriaMetrics

```bash
# Install the chart
helm upgrade --install victoria-metrics . \
  --namespace monitoring \
  --values values.yaml \
  --values infra/values.yaml \
  --create-namespace \
  --timeout 10m

# Watch the deployment
kubectl get pods -n monitoring -w
```

### Step 6: Wait for All Pods to be Ready

```bash
# Check pod status
kubectl get pods -n monitoring -l app.kubernetes.io/part-of=victoria-metrics

# Expected output (all Running with 2/2 ready):
NAME                                    READY   STATUS    RESTARTS   AGE
victoria-metrics-vm-vmauth-xxx          2/2     Running   0          5m
victoria-metrics-vm-vmalert-xxx         2/2     Running   0          5m
victoria-metrics-vm-vminsert-xxx        2/2     Running   0          5m
victoria-metrics-vm-vmselect-xxx        2/2     Running   0          5m
victoria-metrics-vm-vmstorage-0         2/2     Running   0          5m
victoria-metrics-vm-vmstorage-1         2/2     Running   0          5m
```

---

## Post-Installation Verification

### 1. Verify Services

```bash
kubectl get svc -n monitoring

# Expected services:
# - vmauth (ClusterIP)
# - vminsert (ClusterIP)
# - vmselect (ClusterIP)
# - vmstorage (Headless)
# - vmalert (ClusterIP)
```

### 2. Verify PVCs

```bash
kubectl get pvc -n monitoring

# Should see:
# - vmstorage PVCs (2x 100Gi)
# - vmselect cache PVC (1x 50Gi)
```

### 3. Check Ingress

```bash
kubectl get ingress -n monitoring

# Note the ALB address
kubectl get ingress victoria-metrics-vm-vmauth -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### 4. Test Health Endpoints

```bash
# Port-forward vmauth
kubectl port-forward svc/victoria-metrics-vm-vmauth 8427:8427 -n monitoring

# Test health (in another terminal)
curl http://localhost:8427/health
# Should return: OK

# Test authentication
curl -u admin:admin-password http://localhost:8427/health
```

### 5. Test Write Path

```bash
# Port-forward vminsert
kubectl port-forward svc/victoria-metrics-vm-vminsert 8480:8480 -n monitoring

# Send test metric
curl -X POST http://localhost:8480/api/v1/import/prometheus \
  -d 'test_metric{label="value"} 123'

# Should return: {"status":"success"}
```

### 6. Test Query Path

```bash
# Port-forward vmselect
kubectl port-forward svc/victoria-metrics-vm-vmselect 8481:8481 -n monitoring

# Query the test metric
curl 'http://localhost:8481/select/0/prometheus/api/v1/query?query=test_metric'

# Should return JSON with your metric
```

---

## Configuration Deep Dive

### Component Roles

#### VMStorage
- **Purpose**: Long-term metrics storage
- **Replicas**: 2 (for HA)
- **Storage**: 100Gi gp3 per replica
- **Retention**: 15 days
- **Key Settings**:
  - Deduplication interval: 30s
  - Max concurrent inserts: 16

#### VMInsert
- **Purpose**: Write path / data ingestion
- **Replicas**: 2 (distributes load)
- **Connects to**: All VMStorage nodes
- **Key Settings**:
  - Max request size: 32MB
  - Max labels per time series: 50

#### VMSelect
- **Purpose**: Read path / query execution
- **Replicas**: 2 (query load balancing)
- **Cache**: 50Gi shared gp3 volume
- **Key Settings**:
  - Max query duration: 120s
  - Max concurrent requests: 32
  - Max points per time series: 1M

#### VMAuth
- **Purpose**: Authentication, authorization, load balancing
- **Replicas**: 2 (for HA)
- **Routes**:
  - Write requests → VMInsert
  - Read requests → VMSelect
- **Features**: Per-user rate limiting, IP discovery

#### VMAlert
- **Purpose**: Alerting engine (optional)
- **Replicas**: 2 (for HA)
- **Data Source**: VMSelect

### High Availability Features

1. **Component Redundancy**: All components run with 2+ replicas
2. **Pod Anti-Affinity**: Replicas scheduled on different nodes
3. **PodDisruptionBudget**: Ensures minimum 1 replica during updates
4. **Persistent Storage**: Data survives pod restarts
5. **Health Checks**: Automatic restart of unhealthy pods

### Security Configurations

1. **Pod Security Context**:
   - Run as non-root user (65534)
   - Read-only root filesystem
   - Drop all capabilities
   - Seccomp profile enabled

2. **Network Isolation**:
   - Internal ALB only
   - CIDR restrictions on ingress
   - Network policies (optional)

3. **Authentication**:
   - Basic auth via VMAuth
   - Per-environment credentials
   - Role-based access (read/write/admin)

---

## Integration with Remote Clusters

### Configure vmagent in Remote Clusters

Example vmagent configuration for Dev cluster:

```yaml
# In remote cluster (dev/staging/prod)
apiVersion: v1
kind: Secret
metadata:
  name: vm-remote-write-creds
  namespace: monitoring
type: Opaque
stringData:
  username: dev-cluster
  password: YOUR_DEV_CLUSTER_PASSWORD

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: vmagent-config
  namespace: monitoring
data:
  scrape.yml: |
    global:
      scrape_interval: 30s
      external_labels:
        cluster: dev
        environment: development
        region: ap-south-1
    
    scrape_configs:
      - job_name: 'kubernetes-nodes'
        kubernetes_sd_configs:
          - role: node
      
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod

---
# vmagent Deployment with remote write
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vmagent
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vmagent
  template:
    metadata:
      labels:
        app: vmagent
    spec:
      containers:
      - name: vmagent
        image: victoriametrics/vmagent:v1.116.0
        args:
          - -promscrape.config=/config/scrape.yml
          - -remoteWrite.url=http://victoria-metrics.infra.service/api/v1/write
          - -remoteWrite.basicAuth.username=$(VM_USER)
          - -remoteWrite.basicAuth.password=$(VM_PASS)
          - -remoteWrite.maxDiskUsagePerURL=1GB
        env:
        - name: VM_USER
          valueFrom:
            secretKeyRef:
              name: vm-remote-write-creds
              key: username
        - name: VM_PASS
          valueFrom:
            secretKeyRef:
              name: vm-remote-write-creds
              key: password
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: vmagent-config
```

### Verify Remote Write

From remote cluster:

```bash
# Check vmagent logs
kubectl logs -l app=vmagent -n monitoring

# Should see successful writes:
# "successfully sent request to http://victoria-metrics.infra.service/api/v1/write"
```

From infra cluster:

```bash
# Query metrics from dev cluster
kubectl port-forward svc/victoria-metrics-vm-vmselect 8481:8481 -n monitoring

curl 'http://localhost:8481/select/0/prometheus/api/v1/query?query=up{cluster="dev"}'
```

---

## Next Steps

1. **Set up Grafana** - Configure data source pointing to VMAuth
2. **Configure Alerting** - Set up alert rules in VMAlert
3. **Deploy Exporters** - Install node-exporter, kube-state-metrics
4. **Set up Dashboards** - Import VictoriaMetrics dashboards
5. **Configure Backup** - Set up backup strategy for VMStorage

---

## Troubleshooting

### Pods Not Starting

```bash
# Check events
kubectl describe pod <pod-name> -n monitoring

# Common issues:
# - Image pull errors: Check dockerhub-secret
# - PVC pending: Check storage class and PV availability
# - CrashLoopBackOff: Check logs for errors
```

### Storage Issues

```bash
# Check PVC status
kubectl get pvc -n monitoring

# If PVC is pending:
kubectl describe pvc <pvc-name> -n monitoring

# Check storage class
kubectl get sc gp3
```

### Cannot Write Metrics

```bash
# Test VMInsert directly
kubectl port-forward svc/victoria-metrics-vm-vminsert 8480:8480 -n monitoring

# Send test metric
curl -v -X POST http://localhost:8480/api/v1/import/prometheus \
  -d 'test{} 1'

# Check vminsert logs
kubectl logs -l app.kubernetes.io/name=vminsert -n monitoring
```

### High Memory Usage

- Check query patterns in VMSelect logs
- Adjust `search.maxSeries` and `search.maxPointsPerTimeseries`
- Increase cache size or resources
- Consider scaling horizontally

---

## Support and Resources

- **Official Docs**: https://docs.victoriametrics.com/
- **Helm Chart**: https://github.com/VictoriaMetrics/helm-charts
- **Community**: https://victoriametrics.com/community/
- **Slack**: victoriametrics.slack.com
