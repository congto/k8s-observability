# Grafana High Availability Setup

Production-ready Grafana deployment with PostgreSQL backend for centralized observability dashboards.

## Overview

This setup deploys Grafana in High Availability mode with:
- **PostgreSQL Backend**: Aurora PostgreSQL for session and dashboard storage
- **EFS Storage**: Shared persistent storage for plugins and configurations
- **External Secrets**: Managed credentials via AWS Secrets Manager
- **VictoriaMetrics Integration**: Pre-configured data sources
- **Pre-loaded Dashboards**: Kubernetes and VictoriaMetrics dashboards

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                  AWS ALB (Internal)                 │
│          grafana.infra.service                      │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  Grafana Pods   │
              │   (2 replicas)  │
              └────────┬────────┘
                       │
        ┏━━━━━━━━━━━━━┻━━━━━━━━━━━━━┓
        ▼                            ▼
┌──────────────────┐        ┌──────────────────┐
│  Aurora PostgreSQL│        │   EFS Storage    │
│  (Session + DB)  │        │ (Plugins + Data) │
│                  │        │                  │
│ - Dashboards     │        │ - Shared RWX     │
│ - Users          │        │ - 10Gi           │
│ - Settings       │        │                  │
└──────────────────┘        └──────────────────┘
                   
                   ▼
          ┌─────────────────────┐
          │  VictoriaMetrics    │
          │   (Data Source)     │
          │                     │
          │ - vmselect (read)   │
          │ - vminsert (write)  │
          └─────────────────────┘
```

## Directory Structure

```
grafana/
└── base/
    ├── Chart.yaml                  # Helm chart dependencies
    ├── values.yaml                 # Base configuration
    ├── grafana-efs-pvc.yaml       # EFS PVC/PV definition
    ├── infra/
    │   └── values.yaml            # Infra-specific overrides
    ├── templates/
    │   ├── _helpers.tpl
    │   └── external-secrets.yaml  # Secret management
    └── charts/
        └── grafana-8.5.2.tgz
```

## Key Features

### High Availability
- **2 Replicas**: Multiple Grafana instances for redundancy
- **Aurora PostgreSQL**: Centralized session and dashboard storage
- **EFS Storage**: Shared filesystem (ReadWriteMany) for all replicas
- **Pod Anti-Affinity**: Replicas spread across nodes
- **PodDisruptionBudget**: Ensures minimum availability during updates

### Security
- **External Secrets**: Credentials managed via AWS Secrets Manager
- **Database Encryption**: SSL/TLS enabled for PostgreSQL connections
- **Pod Security Context**: Run as non-root user (472)
- **Read-only Root Filesystem**: Enhanced security posture
- **Internal ALB**: Not exposed to public internet

### Data Sources
- **VictoriaMetrics (Read)**: For querying and visualization
- **VictoriaMetrics (Write)**: For recording rules (optional)
- **Basic Auth**: Configured with read-only/write-only users

### Pre-configured Dashboards
- **Kubernetes Cluster** (GrafanaLabs #315)
- **Node Exporter** (GrafanaLabs #1860)
- **VictoriaMetrics Cluster** (GrafanaLabs #11176)

## Prerequisites

- Kubernetes cluster (EKS)
- Aurora PostgreSQL database
- EFS filesystem provisioned
- EFS CSI driver installed
- External Secrets Operator configured
- AWS Secrets Manager with credentials
- VictoriaMetrics deployed
- Docker Hub credentials

## Installation

### 1. Prepare AWS Resources

#### Create Aurora PostgreSQL Database
```sql
-- Connect to Aurora cluster
CREATE DATABASE aurora_grafana;
CREATE USER grafana WITH PASSWORD 'your-secure-password';
GRANT ALL PRIVILEGES ON DATABASE aurora_grafana TO grafana;
```

#### Store Credentials in AWS Secrets Manager
```bash
# Database credentials
aws secretsmanager create-secret \
  --name infra/platform-tools/grafana/database-credentials \
  --secret-string '{
    "username":"grafana",
    "password":"your-secure-password",
    "host":"infra-aurora-common-pg.cluster-xxxxx.ap-south-1.rds.amazonaws.com",
    "database":"aurora_grafana"
  }'

# Admin credentials
aws secretsmanager create-secret \
  --name infra/platform-tools/grafana/app-secrets \
  --secret-string '{
    "admin_password":"your-admin-password"
  }'
```

### 2. Create EFS Resources

Update `grafana-efs-pvc.yaml` with your EFS filesystem ID:

```yaml
csi:
  driver: efs.csi.aws.com
  volumeHandle: fs-YOUR-EFS-ID  # Update this
```

Apply EFS resources:
```bash
kubectl apply -f grafana-efs-pvc.yaml
```

### 3. Create Namespace and Secrets

```bash
# Create namespace
kubectl create namespace monitoring

# Create Docker Hub secret
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=YOUR_USERNAME \
  --docker-password=YOUR_PASSWORD \
  -n monitoring
```

### 4. Update Configuration

Edit `infra/values.yaml`:

```yaml
# Update Aurora endpoint
database:
  host: "YOUR-AURORA-ENDPOINT.rds.amazonaws.com"

# Update VictoriaMetrics passwords
datasources:
  datasources.yaml:
    datasources:
      - name: VictoriaMetrics
        secureJsonData:
          basicAuthPassword: CHANGE_THIS  # Match VM password
```

### 5. Deploy Grafana

```bash
cd infra-cluster/helm/grafana/base

# Update dependencies
helm dependency update

# Install Grafana
helm upgrade --install grafana . \
  --namespace monitoring \
  --values values.yaml \
  --values infra/values.yaml \
  --create-namespace \
  --timeout 10m

# Watch deployment
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana -w
```

### 6. Verify Deployment

```bash
# Check pods
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana

# Check services
kubectl get svc -n monitoring -l app.kubernetes.io/name=grafana

# Check ingress
kubectl get ingress -n monitoring

# Check External Secrets
kubectl get externalsecret -n monitoring
kubectl get secret grafana-db-secret -n monitoring
kubectl get secret grafana-admin-secret -n monitoring
```

## Configuration Details

### Resource Allocation

| Component | Replicas | CPU Request | Memory Request | CPU Limit | Memory Limit |
|-----------|----------|-------------|----------------|-----------|--------------|
| Grafana   | 2        | 500m        | 1Gi            | 2         | 4Gi          |

### Storage Configuration
- **Type**: EFS (Elastic File System)
- **Access Mode**: ReadWriteMany
- **Size**: 10Gi
- **Storage Class**: efs-sc
- **Reclaim Policy**: Retain

### Database Configuration
- **Type**: PostgreSQL (Aurora)
- **SSL Mode**: Required
- **Session Storage**: PostgreSQL
- **Dashboard Storage**: PostgreSQL

### Node Placement
- **Node Pool**: `general-arm`
- **Architecture**: `arm64`
- **Workload Type**: `general`

### Health Checks

**Readiness Probe:**
- Path: `/api/health`
- Initial Delay: 10s
- Period: 5s
- Timeout: 2s
- Failure Threshold: 3

**Liveness Probe:**
- Path: `/api/health`
- Initial Delay: 30s
- Period: 10s
- Timeout: 5s
- Failure Threshold: 3

## Usage

### Access Grafana

#### Via Port Forward (Testing)
```bash
kubectl port-forward svc/grafana 3000:80 -n monitoring
```
Access at: http://localhost:3000

#### Via Internal ALB
Access at: http://grafana.infra.service

**Default Credentials:**
- Username: `admin`
- Password: Stored in AWS Secrets Manager

### Configure Data Sources

Data sources are automatically configured:

**VictoriaMetrics (Read):**
- URL: `http://victoria-metrics-vm-vmselect.monitoring.svc.cluster.local:8481/select/0/prometheus`
- Auth: Basic Auth (admin user)
- Used for: Dashboards and queries

**VictoriaMetrics (Write):**
- URL: `http://victoria-metrics-vm-vminsert.monitoring.svc.cluster.local:8480/insert/0/prometheus`
- Auth: Basic Auth (writeonly user)
- Used for: Recording rules

### Import Additional Dashboards

#### From Grafana.com
1. Navigate to **Dashboards** → **Import**
2. Enter Dashboard ID (e.g., 315 for Kubernetes)
3. Select **VictoriaMetrics** as data source
4. Click **Import**

#### From JSON
```bash
# Example: Import custom dashboard
kubectl create configmap my-dashboard \
  --from-file=dashboard.json \
  -n monitoring

# Add to values.yaml
dashboards:
  default:
    my-dashboard:
      json: |
        {{ .Files.Get "dashboard.json" | indent 8 }}
```

### Create Users and Teams

1. **Settings** → **Users** → **Invite**
2. Set appropriate roles:
   - **Admin**: Full access
   - **Editor**: Can edit dashboards
   - **Viewer**: Read-only access

### Configure Alerting

Grafana alerting is database-backed, so alerts persist across pod restarts.

```yaml
# Add alerting contact points in grafana.ini
alerting:
  enabled: true
  
# Or via UI:
# Alerting → Contact points → New contact point
```

## Maintenance

### Scaling

```bash
# Scale to 3 replicas
helm upgrade grafana . \
  --set grafana.replicas=3 \
  --namespace monitoring \
  --reuse-values
```

### Backup Strategy

**Database Backup:**
- Aurora automated backups (handled by AWS)
- Point-in-time recovery available

**Dashboard Backup:**
```bash
# Export all dashboards
kubectl exec -n monitoring grafana-xxx -- \
  grafana-cli admin export-dashboards > dashboards-backup.json
```

**Configuration Backup:**
```bash
# Backup helm values
helm get values grafana -n monitoring > grafana-values-backup.yaml
```

### Upgrading Grafana

```bash
# Update chart version in Chart.yaml
vim Chart.yaml
# Change: version: "8.5.2" to version: "8.6.0"

# Update dependencies
helm dependency update

# Upgrade
helm upgrade grafana . \
  --namespace monitoring \
  --values values.yaml \
  --values infra/values.yaml
```

### Database Migration

If migrating to a new Aurora cluster:

1. Dump old database:
```bash
pg_dump -h old-host -U grafana aurora_grafana > grafana-dump.sql
```

2. Restore to new database:
```bash
psql -h new-host -U grafana aurora_grafana < grafana-dump.sql
```

3. Update secrets in AWS Secrets Manager
4. Restart Grafana pods:
```bash
kubectl rollout restart deployment grafana -n monitoring
```

## Troubleshooting

### Pods Not Starting

```bash
# Check pod events
kubectl describe pod grafana-xxx -n monitoring

# Common issues:
# - External Secret not synced: Check ESO logs
# - EFS mount failed: Check EFS CSI driver
# - Database connection failed: Check Aurora connectivity
```

### External Secrets Not Syncing

```bash
# Check External Secret status
kubectl get externalsecret -n monitoring
kubectl describe externalsecret grafana-db-secret -n monitoring

# Check ClusterSecretStore
kubectl get clustersecretstore aws-clustersecretstore
kubectl describe clustersecretstore aws-clustersecretstore

# Verify secrets exist in AWS
aws secretsmanager get-secret-value \
  --secret-id infra/platform-tools/grafana/database-credentials
```

### Cannot Connect to Database

```bash
# Test from within pod
kubectl exec -it grafana-xxx -n monitoring -- /bin/sh

# Test PostgreSQL connection
psql "postgresql://grafana:PASSWORD@HOST:5432/aurora_grafana?sslmode=require"

# Check network connectivity
nc -zv YOUR-AURORA-HOST 5432
```

### Dashboard Not Loading

```bash
# Check Grafana logs
kubectl logs -l app.kubernetes.io/name=grafana -n monitoring

# Check data source connectivity
kubectl exec -it grafana-xxx -n monitoring -- \
  curl http://victoria-metrics-vm-vmselect.monitoring.svc.cluster.local:8481/api/v1/query?query=up
```

### EFS Mount Issues

```bash
# Check EFS CSI driver
kubectl get pods -n kube-system -l app=efs-csi-controller

# Check PVC status
kubectl get pvc grafana-efs-pvc -n monitoring

# Check mount in pod
kubectl exec -it grafana-xxx -n monitoring -- df -h
```

### High Memory Usage

- Check active sessions: **Server Admin** → **Stats**
- Reduce dashboard complexity
- Increase memory limits in `infra/values.yaml`
- Enable query result caching in data sources

## Security Best Practices

1. **Change Default Passwords**: Update admin password in AWS Secrets Manager
2. **Enable HTTPS**: Configure TLS termination at ALB
3. **Restrict Access**: Use VPC CIDR restrictions on ALB
4. **Regular Updates**: Keep Grafana version up-to-date
5. **Audit Logs**: Enable audit logging in `grafana.ini`
6. **Database Encryption**: Ensure Aurora encryption at rest
7. **Rotate Credentials**: Regularly rotate database passwords
8. **Network Policies**: Enable network policies in production

## Performance Tuning

### For Large Deployments

```yaml
# Increase resources
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "4"
    memory: "8Gi"

# Enable caching
grafana.ini:
  caching:
    enabled: true
    
# Optimize database queries
grafana.ini:
  database:
    max_open_conn: 50
    max_idle_conn: 10
```

### Dashboard Performance

- Use recording rules for complex queries
- Limit time ranges in dashboards
- Use query caching in data sources
- Avoid high-cardinality queries

## Monitoring Grafana

Self-monitoring is enabled via ServiceMonitor (optional):

```yaml
serviceMonitor:
  enabled: true
  labels:
    prometheus.io/operator: "true"
```

**Key Metrics to Watch:**
- `grafana_api_response_status_total`
- `grafana_database_connected`
- `grafana_stat_totals_dashboard`
- `grafana_alerting_active_alerts`

## Version Information

- **Chart Version**: 8.5.2
- **App Version**: 11.2.0
- **PostgreSQL**: Aurora (PostgreSQL 15+)
- **EFS CSI Driver**: Latest

## References

- [Grafana Documentation](https://grafana.com/docs/)
- [Grafana Helm Chart](https://github.com/grafana/helm-charts/tree/main/charts/grafana)
- [VictoriaMetrics Data Source](https://docs.victoriametrics.com/Single-server-VictoriaMetrics.html#grafana-setup)
- [External Secrets Operator](https://external-secrets.io/)
