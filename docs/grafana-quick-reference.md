# Grafana Quick Reference

Essential commands and operations for managing Grafana HA deployment.

## Quick Access

```bash
# Port-forward for local access
kubectl port-forward svc/grafana 3000:80 -n monitoring

# Get admin password
kubectl get secret grafana-admin-secret -n monitoring -o jsonpath='{.data.admin-password}' | base64 -d

# Get ALB URL
kubectl get ingress -n monitoring -o jsonpath='{.items[0].status.loadBalancer.ingress[0].hostname}'
```

## Status Checks

```bash
# Check pod status
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana

# Check logs
kubectl logs -l app.kubernetes.io/name=grafana -n monitoring --tail=50 -f

# Check External Secrets sync
kubectl get externalsecret -n monitoring
kubectl describe externalsecret grafana-db-secret -n monitoring

# Check ingress
kubectl get ingress -n monitoring
kubectl describe ingress -n monitoring
```

## Database Operations

```bash
# Test database connection from pod
kubectl exec -it POD_NAME -n monitoring -- \
  psql "postgresql://grafana:PASSWORD@HOST:5432/aurora_grafana?sslmode=require"

# Check database size
kubectl exec -it POD_NAME -n monitoring -- \
  psql "postgresql://grafana:PASSWORD@HOST:5432/aurora_grafana?sslmode=require" \
  -c "SELECT pg_size_pretty(pg_database_size('aurora_grafana'));"

# List tables
kubectl exec -it POD_NAME -n monitoring -- \
  psql "postgresql://grafana:PASSWORD@HOST:5432/aurora_grafana?sslmode=require" \
  -c "\dt"
```

## Configuration Updates

```bash
# Update Helm values
vim infra/values.yaml

# Apply changes
helm upgrade grafana . \
  --namespace monitoring \
  --values values.yaml \
  --values infra/values.yaml

# Rollback if needed
helm rollback grafana -n monitoring
```

## Scaling

```bash
# Scale to 3 replicas
kubectl scale deployment grafana --replicas=3 -n monitoring

# Or via Helm
helm upgrade grafana . \
  --set grafana.replicas=3 \
  --namespace monitoring \
  --reuse-values
```

## Restart Operations

```bash
# Restart all pods
kubectl rollout restart deployment grafana -n monitoring

# Delete specific pod (will auto-recreate)
kubectl delete pod POD_NAME -n monitoring

# Watch rollout status
kubectl rollout status deployment grafana -n monitoring
```

## Secret Management

```bash
# Update admin password in AWS Secrets Manager
aws secretsmanager update-secret \
  --secret-id infra/platform-tools/grafana/app-secrets \
  --secret-string '{"admin_password":"NewPassword123!"}'

# Force External Secret sync (restart ESO)
kubectl delete pod -n external-secrets-system -l app.kubernetes.io/name=external-secrets

# Or annotate ExternalSecret
kubectl annotate externalsecret grafana-admin-secret \
  force-sync=$(date +%s) -n monitoring --overwrite
```

## Backup & Export

```bash
# Export all dashboards
POD=$(kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n monitoring $POD -- \
  grafana-cli admin export-dashboards > dashboards-backup-$(date +%Y%m%d).json

# Backup Helm values
helm get values grafana -n monitoring > grafana-values-backup.yaml

# Backup secrets (encrypted)
kubectl get secret -n monitoring -o yaml > secrets-backup.yaml
```

## Troubleshooting Commands

```bash
# Check pod events
kubectl describe pod POD_NAME -n monitoring

# Check service endpoints
kubectl get endpoints grafana -n monitoring

# Check PVC status
kubectl get pvc grafana-efs-pvc -n monitoring
kubectl describe pvc grafana-efs-pvc -n monitoring

# Check EFS mount in pod
kubectl exec -it POD_NAME -n monitoring -- df -h | grep efs

# Test VictoriaMetrics connectivity
kubectl exec -it POD_NAME -n monitoring -- \
  curl -s http://victoria-metrics-vm-vmselect.monitoring.svc.cluster.local:8481/api/v1/query?query=up

# Check database connectivity
kubectl exec -it POD_NAME -n monitoring -- \
  nc -zv AURORA-HOST 5432
```

## Resource Usage

```bash
# Check resource usage
kubectl top pods -n monitoring -l app.kubernetes.io/name=grafana

# Check PVC usage
kubectl exec -it POD_NAME -n monitoring -- df -h /var/lib/grafana
```

## Grafana CLI Commands

```bash
# Execute Grafana CLI in pod
kubectl exec -it POD_NAME -n monitoring -- grafana-cli plugins list
kubectl exec -it POD_NAME -n monitoring -- grafana-cli plugins install PLUGIN_NAME

# Reset admin password (emergency)
kubectl exec -it POD_NAME -n monitoring -- \
  grafana-cli admin reset-admin-password NEW_PASSWORD
```

## Data Source Testing

```bash
# Port-forward and test via API
kubectl port-forward svc/grafana 3000:80 -n monitoring

# Test data source
curl -u admin:PASSWORD http://localhost:3000/api/datasources

# Test specific data source
curl -u admin:PASSWORD http://localhost:3000/api/datasources/1

# Test query
curl -u admin:PASSWORD \
  -H "Content-Type: application/json" \
  -d '{"queries":[{"refId":"A","datasourceId":1,"expr":"up"}]}' \
  http://localhost:3000/api/ds/query
```

## Performance Tuning

```bash
# Check database connections
kubectl exec -it POD_NAME -n monitoring -- \
  psql "postgresql://grafana:PASSWORD@HOST:5432/aurora_grafana?sslmode=require" \
  -c "SELECT count(*) FROM pg_stat_activity WHERE datname='aurora_grafana';"

# Check active sessions in Grafana
# UI: Server Admin → Stats

# Clear caching (if needed)
kubectl exec -it POD_NAME -n monitoring -- \
  rm -rf /var/lib/grafana/cache/*
```

## Monitoring Grafana

```bash
# Key metrics to watch
kubectl exec -it POD_NAME -n monitoring -- \
  curl -s http://localhost:3000/metrics | grep grafana_

# Important metrics:
# - grafana_api_response_status_total
# - grafana_database_connected
# - grafana_stat_totals_dashboard
# - grafana_stat_totals_user
```

## Upgrade Procedure

```bash
# 1. Backup current state
helm get values grafana -n monitoring > backup-values.yaml
kubectl exec -n monitoring POD_NAME -- \
  grafana-cli admin export-dashboards > backup-dashboards.json

# 2. Update Chart.yaml version
vim Chart.yaml

# 3. Update dependencies
helm dependency update

# 4. Test upgrade
helm upgrade grafana . \
  --namespace monitoring \
  --values values.yaml \
  --values infra/values.yaml \
  --dry-run --debug

# 5. Perform upgrade
helm upgrade grafana . \
  --namespace monitoring \
  --values values.yaml \
  --values infra/values.yaml

# 6. Watch rollout
kubectl rollout status deployment grafana -n monitoring

# 7. Verify
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana
kubectl logs -l app.kubernetes.io/name=grafana -n monitoring
```

## Emergency Procedures

### Grafana Not Responding

```bash
# 1. Check pod status
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana

# 2. Check logs
kubectl logs -l app.kubernetes.io/name=grafana -n monitoring --tail=100

# 3. Restart pods
kubectl rollout restart deployment grafana -n monitoring

# 4. If still failing, check database
kubectl exec -it POD_NAME -n monitoring -- \
  psql "postgresql://grafana:PASSWORD@HOST:5432/aurora_grafana?sslmode=require" -c "SELECT 1;"
```

### Database Connection Lost

```bash
# 1. Verify Aurora is up
aws rds describe-db-cluster-endpoints \
  --db-cluster-identifier YOUR-CLUSTER

# 2. Test connectivity from pod
kubectl exec -it POD_NAME -n monitoring -- nc -zv AURORA-HOST 5432

# 3. Check security groups
# Ensure Aurora SG allows EKS node SG on port 5432

# 4. Restart Grafana pods
kubectl rollout restart deployment grafana -n monitoring
```

### EFS Mount Failed

```bash
# 1. Check EFS CSI driver
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-efs-csi-driver

# 2. Check mount targets
aws efs describe-mount-targets --file-system-id fs-YOUR-EFS-ID

# 3. Check PVC
kubectl describe pvc grafana-efs-pvc -n monitoring

# 4. Delete and recreate PVC
kubectl delete -f grafana-efs-pvc.yaml
kubectl apply -f grafana-efs-pvc.yaml

# 5. Restart Grafana
kubectl rollout restart deployment grafana -n monitoring
```

## Useful URLs

- **Grafana UI**: http://grafana.infra.service
- **API Docs**: http://grafana.infra.service/docs/http_api/
- **Health Check**: http://grafana.infra.service/api/health
- **Metrics**: http://grafana.infra.service/metrics

## Default Credentials

- **Username**: admin
- **Password**: (stored in AWS Secrets Manager: `infra/platform-tools/grafana/app-secrets`)

## Important Files

- **Helm Values**: `infra-cluster/helm/grafana/base/infra/values.yaml`
- **EFS PVC**: `infra-cluster/helm/grafana/base/grafana-efs-pvc.yaml`
- **External Secrets**: `infra-cluster/helm/grafana/base/templates/external-secrets.yaml`

## Support

- Grafana Docs: https://grafana.com/docs/
- Helm Chart: https://github.com/grafana/helm-charts/tree/main/charts/grafana
- Community: https://community.grafana.com/
