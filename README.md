MySQL Helm Chart

![Version: 0.1.29](https://img.shields.io/badge/Version-0.1.29-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 8.4.2](https://img.shields.io/badge/AppVersion-8.4.2-informational?style=flat-square)

A simple, standalone MySQL Helm chart

## Features

- **Official MySQL Images**: Uses `docker.io/mysql:8.4.2`
- **Standalone Mode**: Simple single-instance deployment
- **Persistent Storage**: StatefulSet with PVC support (default 20Gi)
- **Configurable**: Customizable MySQL configuration
- **Secure**: Kubernetes secrets for passwords
- **Production Ready**: Resource limits, health checks, security contexts
- **Database Initialization**: Automated database and user creation (optional)
- **Automated Backups**: Scheduled backups to S3-compatible storage (optional)
- **Restore Operations**: One-time restore from S3-compatible storage (optional)
- **Global Labels**: Common labels support for all resources

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

- **Basic**: `simple-values.yaml`
- **Persistence**: `existing-pvc-values.yaml`
- **Configuration**: `extra-args-values.yaml` (command-line arguments)
- **Initialization**: `initdb-scripts-values.yaml` (SQL scripts), `init-job-mixed-values.yaml`, `init-job-secretref-values.yaml`
- **Availability**: `pdb-values.yaml` (Pod Disruption Budget)
- **Security**: `network-policy-values.yaml` (Network Policy), `tls-values.yaml` (TLS/SSL)
- **Backups**: `backup-aws-s3-values.yaml`, `backup-aws-iam-role-values.yaml`, `backup-minio-values.yaml`
- **Restore**: `restore-aws-s3-values.yaml`, `restore-aws-iam-role-values.yaml`, `restore-minio-values.yaml`
- **Complete Setup**: `complete-setup-values.yaml`
- **Labels**: `common-labels-values.yaml`

## Advanced Features

### Database Initialization

The chart supports two methods for database initialization:

#### Method 1: Init Scripts (initdbScripts)

SQL scripts that run automatically on first initialization (Bitnami-style):

```yaml
initdbScripts:
  createdb.sql: |-
    CREATE DATABASE IF NOT EXISTS myapp
      DEFAULT CHARACTER SET utf8
      DEFAULT COLLATE utf8_general_ci;
    CREATE USER IF NOT EXISTS 'myapp'@'%' IDENTIFIED BY 'myapp_password';
    GRANT ALL ON `myapp`.* TO 'myapp'@'%';
    FLUSH PRIVILEGES;
```

**Characteristics:**
- Runs automatically on first initialization only
- Scripts execute in alphabetical order
- Mounted to `/docker-entrypoint-initdb.d` (MySQL standard)
- Only runs when data directory is empty

#### Method 2: Init Job (Helm Hook)

Flexible init job that runs after MySQL is ready:

```yaml
initJob:
  enabled: true
  databases:
    - name: myapp
      username: myapp
      password: myapp_password  # or use passwordSecretRef
    - name: production
      username: produser
      passwordSecretRef:
        secretName: prod-db-secret
        secretKey: password
```

**Characteristics:**
- Runs as Helm post-install/post-upgrade hook
- Supports both plain text passwords and secret references
- Runs every time (can be idempotent with IF NOT EXISTS)
- More flexible for dynamic configurations

**You can use both methods together** - initdbScripts run first, then initJob runs after.

```yaml
initJob:
  enabled: true
  databases:
    - name: myapp
      username: myapp
      password: myapp_password  # or use passwordSecretRef
    - name: production
      username: produser
      passwordSecretRef:
        secretName: prod-db-secret
        secretKey: password
```

### MySQL Command-Line Arguments

You can pass additional command-line arguments directly to the `mysqld` process:

```yaml
primary:
  extraArgs:
    - "--max-connections=500"
    - "--max-allowed-packet=64M"
    - "--log-bin-trust-function-creators=1"
    - "--thread-stack=256K"
```

**Use cases:**
- Override configuration values at runtime
- Set values that can't be configured in `my.cnf`
- Apply temporary settings without changing the config file
- Test different MySQL settings

**Note:** Command-line arguments take precedence over `my.cnf` configuration. See `examples/extra-args-values.yaml` for a complete example.

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

### Pod Disruption Budget

Protect MySQL from voluntary disruptions (node drains, pod evictions):

```yaml
podDisruptionBudget:
  enabled: true
  minAvailable: 1  # Keep at least 1 pod available (for single replica)
  # OR
  # maxUnavailable: 0  # Prevent any disruption
```

### Network Policy

Restrict network traffic to/from MySQL pods:

```yaml
networkPolicy:
  enabled: true
  allowExternal: true  # Allow all pods in namespace
  # OR restrict to specific namespaces/pods:
  # allowExternal: false
  # allowedNamespaces:
  #   - matchLabels:
  #       name: production
  # allowedPods:
  #   - matchLabels:
  #       app: myapp
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

## Migrate from Bitnami MySQL

When migrating from Bitnami MySQL chart to this custom MySQL chart:

1. **Set dataDir to Bitnami's path** in your values:

```yaml
primary:
  dataDir: /bitnami/mysql/data
```

2. **Delete the MySQL StatefulSet before upgrading** (PVCs are preserved):

```bash
kubectl delete statefulset <release-name>-mysql --cascade=orphan
```

This allows Helm to recreate the StatefulSet with the new chart while reusing the existing PVC and data.

**Note**: The volume will be mounted at `/bitnami/mysql` (base folder), and MySQL will use `/bitnami/mysql/data` as its datadir, matching Bitnami's structure.

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
| containerSecurityContext.runAsNonRoot | bool | `false` |  |
| containerSecurityContext.runAsUser | string | `""` |  |
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
| initdbScripts | object | `{}` |  |
| nameOverride | string | `""` |  |
| networkPolicy.allowExternal | bool | `false` |  |
| networkPolicy.allowedNamespaces | list | `[]` |  |
| networkPolicy.allowedPods | list | `[]` |  |
| networkPolicy.enabled | bool | `false` |  |
| networkPolicy.extraEgress | list | `[]` |  |
| networkPolicy.extraIngress | list | `[]` |  |
| podDisruptionBudget.enabled | bool | `false` |  |
| podDisruptionBudget.maxUnavailable | int | `1` |  |
| podDisruptionBudget.minAvailable | string | `""` |  |
| podSecurityContext.enabled | bool | `true` |  |
| podSecurityContext.fsGroup | int | `999` |  |
| primary.affinity | object | `{}` |  |
| primary.configuration | string | `"[mysqld]\nauthentication_policy='* ,,'\nskip-name-resolve\nexplicit_defaults_for_timestamp\nport=3306\ndatadir=/var/lib/mysql\nsocket=/var/run/mysqld/mysqld.sock\npid-file=/var/run/mysqld/mysqld.pid\nmax_allowed_packet=16M\nbind-address=0.0.0.0\ncharacter-set-server=utf8mb4\ncollation-server=utf8mb4_unicode_ci\nslow_query_log=0\nlong_query_time=10.0"` |  |
| primary.dataDir | string | `"/var/lib/mysql"` |  |
| primary.existingConfigmap | string | `""` |  |
| primary.extraArgs | list | `[]` |  |
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
| tls.certCAFilename | string | `"ca.crt"` |  |
| tls.certFilename | string | `"tls.crt"` |  |
| tls.certKeyFilename | string | `"tls.key"` |  |
| tls.enabled | bool | `false` |  |
| tls.existingSecret | string | `""` |  |
| tls.requireSecureTransport | bool | `false` |  |

## License

This chart is provided as-is.

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.14.2](https://github.com/norwoodj/helm-docs/releases/v1.14.2)

