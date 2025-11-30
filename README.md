# mysql

A simple, standalone MySQL Helm chart with optional cluster support

## Features

- ✅ **Official MySQL Images**: Uses `docker.io/mysql:8.4.2`
- ✅ **Standalone Mode**: Simple single-instance deployment
- ✅ **Persistent Storage**: StatefulSet with PVC support (default 20Gi)
- ✅ **Configurable**: Customizable MySQL configuration
- ✅ **Secure**: Kubernetes secrets for passwords
- ✅ **Production Ready**: Resource limits, health checks, security contexts
- ✅ **Database Initialization**: Automated database and user creation (optional)
- ✅ **Automated Backups**: Scheduled backups to S3-compatible storage (optional)
- ✅ **Restore Operations**: One-time restore from S3-compatible storage (optional)
- ✅ **Global Labels**: Common labels support for all resources

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- PV provisioner support (if persistence is enabled)

## Quick Start

### Basic Installation

```bash
helm install my-mysql ./
```

### With Custom Values

```bash
helm install my-mysql ./ \
  --set auth.rootPassword=mypassword \
  --set auth.database=mydb \
  --set primary.persistence.size=50Gi
```

### Using a Values File

```bash
helm install my-mysql ./ -f my-values.yaml
```

## Examples

See the `examples/` directory for complete example configurations:

- **Basic**: `simple-values.yaml`, `opik-values.yaml`
- **Persistence**: `existing-pvc-values.yaml`
- **Initialization**: `init-job-mixed-values.yaml`, `init-job-secretref-values.yaml`
- **Backups**: `backup-aws-s3-values.yaml`, `backup-aws-iam-role-values.yaml`, `backup-minio-values.yaml`
- **Restore**: `restore-aws-s3-values.yaml`, `restore-aws-iam-role-values.yaml`, `restore-minio-values.yaml`
- **Complete Setup**: `complete-setup-values.yaml`
- **Labels**: `common-labels-values.yaml`

## Advanced Features

### Database Initialization

Automatically create multiple databases and users on installation:

```yaml
initJob:
  enabled: true
  databases:
    - name: opik
      username: opik
      password: opik_password  # or use passwordSecretRef
    - name: comet
      username: comet
      passwordSecretRef:
        secretName: comet-db-secret
        secretKey: password
```

### Automated Backups

Schedule regular backups to S3-compatible storage (AWS S3, MinIO, etc.):

```yaml
backup:
  enabled: true
  schedule: "0 2 * * *"  # Daily at 2 AM
  storage:
    bucket: "my-backups"
    region: "us-east-1"
    prefix: "mysql-backups"
    endpoint: ""  # Empty for AWS S3, or "http://minio:9000" for MinIO
    existingSecret: "aws-s3-credentials"  # Optional - use IAM role if empty
  retention: 7
  databases: []  # Empty = all databases
```

### Restore from Backup

One-time restore operation:

```yaml
restore:
  enabled: true
  backupFile: "mysql_backup_20250130_020000.sql.gz"
  storage:
    bucket: "my-backups"
    region: "us-east-1"
    prefix: "mysql-backups"
    existingSecret: "aws-s3-credentials"
```

## Connecting to MySQL

### From Within the Cluster

```bash
mysql -h <release-name>-mysql -u root -p
```

### Port Forward (for local development)

```bash
kubectl port-forward svc/<release-name>-mysql 3306:3306
mysql -h 127.0.0.1 -P 3306 -u root -p
```

## Upgrading

```bash
helm upgrade my-mysql ./ -f values.yaml
```

## Uninstalling

```bash
helm uninstall my-mysql
```

**Note**: This removes all Kubernetes components but **does not delete PVCs** by default. To delete PVCs:

```bash
kubectl delete pvc -l app.kubernetes.io/name=mysql
```

## Troubleshooting

### Check Pod Status

```bash
kubectl get pods -l app.kubernetes.io/name=mysql
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Check PVC

```bash
kubectl get pvc
kubectl describe pvc data-<release-name>-mysql-0
```

### Connect to MySQL Pod

```bash
kubectl exec -it <pod-name> -- /bin/bash
```

## Architecture

```
┌─────────────────────────────────────┐
│         Service (ClusterIP)         │
│      <release-name>-mysql:3306      │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│         StatefulSet (Primary)        │
│      <release-name>-mysql-0         │
│  - MySQL 8.4.2 (Official Image)     │
│  - Port 3306                         │
│  - Health Checks                     │
│  - Resource Limits                   │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│      PersistentVolumeClaim          │
│   data-<release-name>-mysql-0       │
│  - /var/lib/mysql                    │
│  - Size: 20Gi (default)              │
│  - Storage Class: configurable       │
└─────────────────────────────────────┘
```

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| architecture.mode | string | `"standalone"` |  |
| architecture.replication.replicas | int | `2` |  |
| auth.database | string | `"my_database"` |  |
| auth.existingSecret | string | `""` |  |
| auth.password | string | `""` |  |
| auth.rootPassword | string | `"changeme"` |  |
| auth.username | string | `""` |  |
| backup.databases | list | `[]` |  |
| backup.enabled | bool | `false` |  |
| backup.nodeSelector | object | `{}` |  |
| backup.resources.limits.cpu | string | `"500m"` |  |
| backup.resources.limits.memory | string | `"512Mi"` |  |
| backup.resources.requests.cpu | string | `"250m"` |  |
| backup.resources.requests.memory | string | `"256Mi"` |  |
| backup.retention | int | `7` |  |
| backup.schedule | string | `"0 2 * * *"` |  |
| backup.serviceAccountName | string | `""` |  |
| backup.storage.bucket | string | `""` |  |
| backup.storage.endpoint | string | `""` |  |
| backup.storage.existingSecret | string | `""` |  |
| backup.storage.prefix | string | `"mysql-backups"` |  |
| backup.storage.region | string | `"us-east-1"` |  |
| backup.tolerations | list | `[]` |  |
| containerSecurityContext.enabled | bool | `true` |  |
| containerSecurityContext.runAsNonRoot | bool | `true` |  |
| containerSecurityContext.runAsUser | int | `999` |  |
| fullnameOverride | string | `""` |  |
| global.commonLabels | object | `{}` |  |
| global.imageRegistry | string | `""` |  |
| global.storageClass | string | `""` |  |
| image.pullPolicy | string | `"IfNotPresent"` |  |
| image.pullSecrets | list | `[]` |  |
| image.registry | string | `"docker.io"` |  |
| image.repository | string | `"mysql"` |  |
| image.tag | string | `"8.4.2"` |  |
| initJob.annotations."helm.sh/hook" | string | `"post-install,post-upgrade"` |  |
| initJob.annotations."helm.sh/hook-delete-policy" | string | `"before-hook-creation,hook-succeeded"` |  |
| initJob.annotations."helm.sh/hook-weight" | string | `"5"` |  |
| initJob.databases | list | `[]` |  |
| initJob.enabled | bool | `false` |  |
| initJob.resources.limits.cpu | string | `"500m"` |  |
| initJob.resources.limits.memory | string | `"256Mi"` |  |
| initJob.resources.requests.cpu | string | `"100m"` |  |
| initJob.resources.requests.memory | string | `"128Mi"` |  |
| nameOverride | string | `""` |  |
| podSecurityContext.enabled | bool | `true` |  |
| podSecurityContext.fsGroup | int | `999` |  |
| primary.affinity | object | `{}` |  |
| primary.configuration | string | `"[mysqld]\ndefault-authentication-plugin=mysql_native_password\nskip-name-resolve\nexplicit_defaults_for_timestamp\nport=3306\ndatadir=/var/lib/mysql\nmax_allowed_packet=16M\nbind-address=0.0.0.0\ncharacter-set-server=utf8mb4\ncollation-server=utf8mb4_unicode_ci\nslow_query_log=0\nlong_query_time=10.0"` |  |
| primary.existingConfigmap | string | `""` |  |
| primary.nodeSelector | object | `{}` |  |
| primary.persistence.accessModes[0] | string | `"ReadWriteOnce"` |  |
| primary.persistence.annotations | object | `{}` |  |
| primary.persistence.enabled | bool | `true` |  |
| primary.persistence.existingClaim | string | `""` |  |
| primary.persistence.selector | object | `{}` |  |
| primary.persistence.size | string | `"20Gi"` |  |
| primary.persistence.storageClass | string | `""` |  |
| primary.podAnnotations | object | `{}` |  |
| primary.podLabels | object | `{}` |  |
| primary.resources.limits.cpu | string | `"2000m"` |  |
| primary.resources.limits.memory | string | `"2Gi"` |  |
| primary.resources.requests.cpu | string | `"500m"` |  |
| primary.resources.requests.memory | string | `"512Mi"` |  |
| primary.tolerations | list | `[]` |  |
| restore.backupFile | string | `""` |  |
| restore.enabled | bool | `false` |  |
| restore.nodeSelector | object | `{}` |  |
| restore.resources.limits.cpu | string | `"1000m"` |  |
| restore.resources.limits.memory | string | `"1Gi"` |  |
| restore.resources.requests.cpu | string | `"500m"` |  |
| restore.resources.requests.memory | string | `"512Mi"` |  |
| restore.serviceAccountName | string | `""` |  |
| restore.storage.bucket | string | `""` |  |
| restore.storage.endpoint | string | `""` |  |
| restore.storage.existingSecret | string | `""` |  |
| restore.storage.prefix | string | `"mysql-backups"` |  |
| restore.storage.region | string | `"us-east-1"` |  |
| restore.tolerations | list | `[]` |  |
| service.annotations | object | `{}` |  |
| service.clusterIP | string | `""` |  |
| service.loadBalancerIP | string | `""` |  |
| service.loadBalancerSourceRanges | list | `[]` |  |
| service.nodePort | string | `""` |  |
| service.port | int | `3306` |  |
| service.type | string | `"ClusterIP"` |  |
| serviceAccount.annotations | object | `{}` |  |
| serviceAccount.create | bool | `true` |  |
| serviceAccount.name | string | `""` |  |

## License

This chart is provided as-is for Comet ML internal use.

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.14.2](https://github.com/norwoodj/helm-docs/releases/v1.14.2)

