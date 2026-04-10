# Kubernetes Deployment

## Docker Image

```bash
docker pull osodevops/kafka-backup-enterprise:v0.2.1
```

Note: The image is built for `linux/amd64`. For ARM clusters, download the `aarch64-linux` binary from [Releases](https://github.com/osodevops/kafka-backup-enterprise-releases/releases) and build a custom image.

## Deployment

### 1. Create ConfigMap with backup config

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-backup-config
  namespace: kafka-backup
data:
  backup.yaml: |
    mode: backup
    backup_id: "production-backup"
    source:
      bootstrap_servers:
        - kafka-bootstrap:9092
    storage:
      backend: s3
      bucket: kafka-backups
      region: us-east-1
    enterprise:
      schema_registry:
        url: "http://schema-registry:8081"
```

### 2. Create license Secret

```bash
kubectl create namespace kafka-backup

kubectl create secret generic kafka-backup-license \
  --namespace kafka-backup \
  --from-literal=license-b64="$(base64 < license.lic)"
```

### 3. Deploy

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: kafka-backup
  namespace: kafka-backup
spec:
  schedule: "0 */6 * * *"   # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: kafka-backup
              image: osodevops/kafka-backup-enterprise:v0.2.1
              args: ["backup", "--config", "/config/backup.yaml"]
              env:
                - name: ENTERPRISE_LICENSE_KEY
                  valueFrom:
                    secretKeyRef:
                      name: kafka-backup-license
                      key: license-b64
                - name: AWS_ACCESS_KEY_ID
                  valueFrom:
                    secretKeyRef:
                      name: s3-credentials
                      key: access-key
                - name: AWS_SECRET_ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: s3-credentials
                      key: secret-key
              volumeMounts:
                - name: config
                  mountPath: /config
                  readOnly: true
              resources:
                requests:
                  memory: "256Mi"
                  cpu: "100m"
                limits:
                  memory: "1Gi"
                  cpu: "500m"
          volumes:
            - name: config
              configMap:
                name: kafka-backup-config
          restartPolicy: OnFailure
```

## Without a License (auto-trial)

If you skip the license Secret, the 14-day auto-trial activates automatically. The trial state is stored inside the container at `/var/lib/kafka-backup/.trial`.

Note: If the Pod is recreated, the trial restarts (new 14 days). For persistent trial tracking, mount a PersistentVolumeClaim at `/var/lib/kafka-backup/`.

## Schema-Only Backup

For a dedicated schema backup job:

```yaml
args: ["backup", "--config", "/config/backup.yaml", "--schema-only"]
```
