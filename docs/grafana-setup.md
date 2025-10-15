# Grafana HA - Complete Setup Guide

Step-by-step guide to deploy Grafana with PostgreSQL backend and EFS storage on Amazon EKS.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [AWS Infrastructure Setup](#aws-infrastructure-setup)
3. [Kubernetes Prerequisites](#kubernetes-prerequisites)
4. [Installation Steps](#installation-steps)
5. [Post-Installation Configuration](#post-installation-configuration)
6. [Integration with VictoriaMetrics](#integration-with-victoriametrics)

---

## Prerequisites

### Required Tools
- `kubectl` (v1.24+)
- `helm` (v3.10+)
- `aws-cli` (v2.x)
- `psql` (PostgreSQL client)

### AWS Resources Needed
- Aurora PostgreSQL cluster (v15+)
- EFS filesystem
- EFS CSI driver installed in cluster
- External Secrets Operator configured
- VPC with proper networking

### Access Requirements
- AWS IAM permissions for Secrets Manager
- Aurora database admin access
- EKS cluster admin access

---

## AWS Infrastructure Setup

### 1. Create Aurora PostgreSQL Database

#### Create Database and User

```bash
# Connect to Aurora cluster
psql -h YOUR-AURORA-ENDPOINT.rds.amazonaws.com -U postgres

# Create database and user
CREATE DATABASE aurora_grafana;
CREATE USER grafana WITH PASSWORD 'YourSecurePassword123!';

# Grant privileges
GRANT ALL PRIVILEGES ON DATABASE aurora_grafana TO grafana;

# Connect to the database
\c aurora_grafana

# Grant schema permissions
GRANT ALL ON SCHEMA public TO grafana;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO grafana;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO grafana;

# Set default privileges for future objects
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO grafana;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO grafana;

\q
```

#### Verify Connection

```bash
# Test connection from your machine
psql "postgresql://grafana:YourSecurePassword123!@YOUR-AURORA-ENDPOINT:5432/aurora_grafana?sslmode=require"

# Should connect successfully
\l  # List databases
\q
```

### 2. Create EFS Filesystem

#### Option A: Using AWS Console
1. Navigate to EFS console
2. Click **Create file system**
3. Select your VPC
4. Create mount targets in all AZ subnets
5. Note the filesystem ID (e.g., `fs-0e23e50a6d2608c33`)

#### Option B: Using AWS CLI

```bash
# Create EFS filesystem
aws efs create-file-system \
  --region ap-south-1 \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --tags Key=Name,Value=grafana-efs Key=Environment,Value=infra

# Get filesystem ID
EFS_ID=$(aws efs describe-file-systems \
  --region ap-south-1 \
  --query "FileSystems[?Name=='grafana-efs'].FileSystemId" \
  --output text)

echo "EFS Filesystem ID: $EFS_ID"

# Create mount targets (replace with your subnet IDs)
for SUBNET in subnet-xxx subnet-yyy subnet-zzz; do
  aws efs create-mount-target \
    --file-system-id $EFS_ID \
    --subnet-id $SUBNET \
    --security-groups sg-your-efs-sg
done
```

#### Create Security Group for EFS

```bash
# Create security group
EFS_SG=$(aws ec2 create-security-group \
  --group-name grafana-efs-sg \
  --description "Security group for Grafana EFS" \
  --vpc-id vpc-YOUR-VPC-ID \
  --query 'GroupId' \
  --output text)

# Allow NFS from EKS node security group
aws ec2 authorize-security-group-ingress \
  --group-id $EFS_SG \
  --protocol tcp \
  --port 2049 \
  --source-group eks-node-sg-id
```

### 3. Store Credentials in AWS Secrets Manager

#### Database Credentials

```bash
aws secretsmanager create-secret \
  --name infra/platform-tools/grafana/database-credentials \
  --description "Grafana Aurora database credentials" \
  --secret-string '{
    "username": "grafana",
    "password": "YourSecurePassword123!",
    "host": "YOUR-AURORA-ENDPOINT.rds.amazonaws.com",
    "database": "aurora_grafana"
  }' \
  --region ap-south-1

# Verify
aws secretsmanager get-secret-value \
  --secret-id infra/platform-tools/grafana/database-credentials \
  --region ap-south-1
```

#### Admin Credentials

```bash
# Generate strong password
ADMIN_PASSWORD=$(openssl rand -base64 32)
echo "Admin Password: $ADMIN_PASSWORD"  # Save this!

aws secretsmanager create-secret \
  --name infra/platform-tools/grafana/app-secrets \
  --description "Grafana admin credentials" \
  --secret-string "{
    \"admin_password\": \"$ADMIN_PASSWORD\"
  }" \
  --region ap-south-1
```

---

## Kubernetes Prerequisites

### 1. Install EFS CSI Driver

```bash
# Add EFS CSI driver repo
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update

# Install driver
helm upgrade --install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --set image.repository=602401xxxxxxx.dkr.ecr.ap-south-1.amazonaws.com/eks/aws-efs-csi-driver \
  --set controller.serviceAccount.create=true \
  --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::ACCOUNT_ID:role/EKS-EFS-CSI-DriverRole"

# Verify
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-efs-csi-driver
```

### 2. Create EFS Storage Class

```bash
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-YOUR-EFS-ID  # Replace with your EFS ID
  directoryPerms: "700"
  gidRangeStart: "1000"
  gidRangeEnd: "2000"
  basePath: "/grafana"
EOF

# Verify
kubectl get sc efs-sc
```

### 3. Install External Secrets Operator (if not installed)

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets-system \
  --create-namespace \
  --set installCRDs=true

# Verify
kubectl get pods -n external-secrets-system
```

### 4. Create ClusterSecretStore

```bash
# Create IAM role for External Secrets (if not exists)
# This role should have permissions to read from Secrets Manager

cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-clustersecretstore
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets-system
EOF

# Verify
kubectl get clustersecretstore
```

### 5. Create Namespace and Docker Secret

```bash
# Create namespace
kubectl create namespace monitoring

# Create Docker Hub secret
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=YOUR_DOCKERHUB_USERNAME \
  --docker-password=YOUR_DOCKERHUB_PASSWORD \
  --docker-email=YOUR_EMAIL \
  -n monitoring

# Verify
kubectl get secret dockerhub-secret -n monitoring
```

---

## Installation Steps

### Step 1: Clone Repository

```bash
git clone https://github.com/YOUR_ORG/k8s-observability.git
cd k8s-observability/infra-cluster/helm/grafana/base
```

### Step 2: Update EFS Configuration

Edit `grafana-efs-pvc.yaml`:

```bash
vim grafana-efs-pvc.yaml

# Update the volumeHandle with your EFS ID
csi:
  driver: efs.csi.aws.com
  volumeHandle: fs-YOUR-EFS-ID  # Replace this
```

Apply EFS resources:

```bash
kubectl apply -f grafana-efs-pvc.yaml

# Verify PV and PVC
kubectl get pv grafana-efs-pv
kubectl get pvc grafana-efs-pvc -n monitoring
```

### Step 3: Update Configuration Values

Edit `infra/values.yaml`:

```bash
vim infra/values.yaml
```

**Update these sections:**

1. **Aurora Endpoint:**
```yaml
database:
  host: "YOUR-AURORA-ENDPOINT.rds.amazonaws.com"
```

2. **Grafana Domain:**
```yaml
grafana.ini:
  server:
    domain: grafana.infra.service  # Or your domain
    root_url: "http://grafana.infra.service/"
```

3. **VictoriaMetrics Passwords:**
```yaml
datasources:
  datasources.yaml:
    datasources:
      - name: VictoriaMetrics
        secureJsonData:
          basicAuthPassword: YOUR_VM_ADMIN_PASSWORD  # From VM setup
      - name: VictoriaMetrics-Write
        secureJsonData:
          basicAuthPassword: YOUR_VM_WRITEONLY_PASSWORD
```

4. **Session Provider:**
```yaml
grafana.ini:
  session:
    provider_config: "user=grafana dbname=aurora_grafana sslmode=require host=YOUR-AURORA-ENDPOINT port=5432"
```

### Step 4: Update Helm Dependencies

```bash
# Download Grafana chart
helm dependency update

# Verify
ls charts/
# Should see: grafana-8.5.2.tgz
```

### Step 5: Dry Run (Recommended)

```bash
helm install grafana . \
  --namespace monitoring \
  --values values.yaml \
  --values infra/values.yaml \
  --dry-run --debug > grafana-dry-run.yaml

# Review the output
less grafana-dry-run.yaml
```

### Step 6: Install Grafana

```bash
helm upgrade --install grafana . \
  --namespace monitoring \
  --values values.yaml \
  --values infra/values.yaml \
  --create-namespace \
  --timeout 10m

# Watch deployment
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana -w
```

### Step 7: Wait for Pods to be Ready

```bash
# Check pod status
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana

# Expected output (may take 2-3 minutes):
NAME                       READY   STATUS    RESTARTS   AGE
grafana-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
grafana-xxxxxxxxxx-yyyyy   1/1     Running   0          2m
```

---

## Post-Installation Configuration

### 1. Verify External Secrets

```bash
# Check External Secrets
kubectl get externalsecret -n monitoring

# Should show both secrets with status: SecretSynced
# - grafana-db-secret
# - grafana-admin-secret

# Verify secrets created
kubectl get secret grafana-db-secret -n monitoring -o yaml
kubectl get secret grafana-admin-secret -n monitoring -o yaml
```

### 2. Verify Services

```bash
kubectl get svc -n monitoring -l app.kubernetes.io/name=grafana

# Should see ClusterIP service on port 80
```

### 3. Check Ingress

```bash
kubectl get ingress -n monitoring

# Get ALB hostname
ALB_HOST=$(kubectl get ingress -n monitoring \
  -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}')

echo "Grafana ALB: http://$ALB_HOST"
```

### 4. Verify Database Connection

```bash
# Check Grafana logs
kubectl logs -l app.kubernetes.io/name=grafana -n monitoring | grep -i database

# Should see: "Database connected" or similar
# Should NOT see connection errors
```

### 5. Verify EFS Mount

```bash
# Check EFS mount in pod
kubectl exec -it $(kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana -o jsonpath='{.items[0].metadata.name}') -n monitoring -- df -h | grep efs

# Should see EFS mount at /var/lib/grafana
```

---

## Integration with VictoriaMetrics

### 1. Test Data Source Connection

```bash
# Port-forward Grafana
kubectl port-forward svc/grafana 3000:80 -n monitoring

# In browser, navigate to:
# http://localhost:3000

# Login with:
# Username: admin
# Password: (from AWS Secrets Manager)

# Navigate to: Configuration → Data Sources
# Click on "VictoriaMetrics"
# Click "Save & Test"
# Should see: "Data source is working"
```

### 2. Verify Pre-loaded Dashboards

```bash
# Navigate to: Dashboards → Browse
# You should see:
# - Kubernetes Cluster (from GrafanaLabs)
# - Node Exporter (from GrafanaLabs)
# - VictoriaMetrics Cluster (from GrafanaLabs)
```

### 3. Test Query

Navigate to **Explore** and run:

```promql
up{job="kubernetes-nodes"}
```

Should return metrics from your cluster.

---

## Accessing Grafana

### Method 1: Port Forward (Development)

```bash
kubectl port-forward svc/grafana 3000:80 -n monitoring

# Access at: http://localhost:3000
```

### Method 2: Internal ALB (Production)

```bash
# Get ALB hostname
kubectl get ingress -n monitoring

# Access at: http://grafana.infra.service
# (Ensure DNS is configured or add to /etc/hosts)
```

### Method 3: kubectl proxy

```bash
kubectl proxy

# Access at:
# http://localhost:8001/api/v1/namespaces/monitoring/services/grafana:80/proxy/
```

---

## Initial Configuration

### 1. Change Admin Password (Recommended)

```bash
# Get current password from secret
ADMIN_PASS=$(kubectl get secret grafana-admin-secret -n monitoring -o jsonpath='{.data.admin-password}' | base64 -d)

echo "Current admin password: $ADMIN_PASS"

# To change: Update in AWS Secrets Manager and restart pods
aws secretsmanager update-secret \
  --secret-id infra/platform-tools/grafana/app-secrets \
  --secret-string '{"admin_password":"NewSecurePassword123!"}'

# Restart Grafana
kubectl rollout restart deployment grafana -n monitoring
```

### 2. Create Additional Users

UI: **Configuration → Users → Invite**

Or via API:
```bash
curl -X POST http://grafana.infra.service/api/admin/users \
  -H "Content-Type: application/json" \
  -u admin:password \
  -d '{
    "name":"Jasvinder",
    "email":"jasaulakh1988@gmail.com",
    "login":"jasaulakh",
    "password":"password123",
    "OrgId": 1
  }'
```

### 3. Configure Teams

1. Navigate to **Configuration → Teams**
2. Click **New Team**
3. Add team members
4. Assign dashboard permissions

### 4. Set Up Alert Notifications

1. Navigate to **Alerting → Contact points**
2. Click **New contact point**
3. Choose notification type (Slack, Email, etc.)
4. Configure and test

---

## Troubleshooting

### Pods in CrashLoopBackOff

```bash
# Check logs
kubectl logs -l app.kubernetes.io/name=grafana -n monitoring --tail=100

# Common issues:
# 1. Database connection failed
kubectl exec -it POD_NAME -n monitoring -- \
  psql "postgresql://grafana:PASS@HOST:5432/aurora_grafana?sslmode=require"

# 2. EFS mount failed
kubectl describe pod POD_NAME -n monitoring | grep -A 10 Events

# 3. External Secret not synced
kubectl describe externalsecret grafana-db-secret -n monitoring
```

### Cannot Access via ALB

```bash
# Check ALB creation
kubectl describe ingress -n monitoring

# Check ALB in AWS console
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[?contains(LoadBalancerName, `grafana`)].LoadBalancerArn'

# Check target health
aws elbv2 describe-target-health \
  --target-group-arn TARGET_GROUP_ARN
```

### Database Connection Issues

```bash
# Test from pod
kubectl exec -it POD_NAME -n monitoring -- /bin/sh

# Inside pod
nc -zv YOUR-AURORA-ENDPOINT 5432
psql "postgresql://grafana:PASS@HOST:5432/aurora_grafana?sslmode=require"

# Check security groups
# Ensure Aurora SG allows traffic from EKS node SG on port 5432
```

### EFS Mount Failures

```bash
# Check EFS CSI driver
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-efs-csi-driver

# Check mount targets
aws efs describe-mount-targets --file-system-id fs-YOUR-EFS-ID

# Check security group
# Ensure EFS SG allows NFS (2049) from EKS node SG
```

---

## Next Steps

1. **Import Additional Dashboards** - Browse grafana.com/dashboards
2. **Set Up Alerting** - Configure alert rules and notifications
3. **Create Custom Dashboards** - Build dashboards for your workloads
4. **Configure SSO** - Set up OAuth/SAML for authentication
5. **Set Up Backup** - Regular exports of dashboards and config

---

## Maintenance Tasks

### Regular Tasks
- **Weekly**: Review dashboard performance
- **Monthly**: Update Grafana version, rotate passwords
- **Quarterly**: Audit user access, review alert rules

### Backup Dashboards

```bash
# Export all dashboards
kubectl exec -n monitoring POD_NAME -- \
  grafana-cli admin export-dashboards > dashboards-$(date +%Y%m%d).json
```

### Monitor Database Size

```bash
# Connect to Aurora
psql -h AURORA-ENDPOINT -U grafana aurora_grafana

# Check database size
SELECT pg_size_pretty(pg_database_size('aurora_grafana'));
```

---

## Support Resources

- **Grafana Docs**: https://grafana.com/docs/
- **Helm Chart**: https://github.com/grafana/helm-charts
- **Community Forums**: https://community.grafana.com/
- **Slack**: grafana.slack.com
