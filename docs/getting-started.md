# Getting Started

## Prerequisites

- Apache Kafka cluster (any version 2.x+)
- Storage backend: S3, Azure Blob, GCS, or local filesystem
- (Optional) Confluent Schema Registry or Apicurio Registry

## Installation

### Homebrew (macOS / Linux)

```bash
brew install osodevops/tap/kafka-backup-enterprise
```

### Docker

```bash
docker pull osodevops/kafka-backup-enterprise:v0.3.1
```

### Binary Download

Download from [Releases](https://github.com/osodevops/kafka-backup-enterprise-releases/releases):

```bash
# Linux (x86_64)
curl -L https://github.com/osodevops/kafka-backup-enterprise-releases/releases/download/v0.3.1/kafka-backup-x86_64-linux.tar.gz | tar xz
sudo mv kafka-backup /usr/local/bin/kafka-backup-enterprise

# macOS (Apple Silicon)
curl -L https://github.com/osodevops/kafka-backup-enterprise-releases/releases/download/v0.3.1/kafka-backup-aarch64-macos.tar.gz | tar xz
sudo mv kafka-backup /usr/local/bin/kafka-backup-enterprise
```

## Verify Installation

```bash
kafka-backup-enterprise --version
# kafka-backup 0.2.1

kafka-backup-enterprise license info
# Status: Enterprise Auto-Trial
# Days left: 14
# Features: All enterprise features enabled
```

The 14-day auto-trial starts immediately — no signup, no license key needed.

## Your First Backup

### 1. Create a config file

```yaml
# backup.yaml
mode: backup
backup_id: "my-first-backup"

source:
  bootstrap_servers:
    - localhost:9092

storage:
  backend: s3
  bucket: my-kafka-backups
  region: us-east-1

enterprise:
  schema_registry:
    url: "http://localhost:8081"
```

### 2. Run the backup

```bash
kafka-backup-enterprise backup --config backup.yaml
```

This backs up:
- All Kafka topics and partitions
- All Schema Registry subjects, versions, and configs

### 3. Schema-only backup

To back up just the Schema Registry (no Kafka data):

```bash
kafka-backup-enterprise backup --config backup.yaml --schema-only
```

## With Docker

```bash
docker run --rm \
  -v ./backup.yaml:/config.yaml \
  osodevops/kafka-backup-enterprise:v0.3.1 \
  backup --config /config.yaml --schema-only
```

## Applying a License

After the 14-day trial, apply a license file:

```bash
kafka-backup-enterprise license apply --file license.lic
```

Verify it:

```bash
kafka-backup-enterprise license info
# License ID: abc-123
# Customer: Your Company
# Tier: Enterprise
# Features: encryption, schema_registry, rbac, audit, masking, wasm
# Expires: 2027-04-10 (365 days remaining)
# Status: Valid
```

In Docker, pass the license via environment variable:

```bash
docker run --rm \
  -e ENTERPRISE_LICENSE_KEY="$(base64 < license.lic)" \
  -v ./backup.yaml:/config.yaml \
  osodevops/kafka-backup-enterprise:v0.3.1 \
  backup --config /config.yaml
```

## Next Steps

- [Configuration Reference](configuration.md) — all enterprise config options
- [Schema Registry Backup](schema-registry.md) — Confluent SR details
- [Apicurio Registry Backup](apicurio-registry.md) — Apicurio v3 details
- [Kubernetes Deployment](kubernetes.md) — deploy to K8s
