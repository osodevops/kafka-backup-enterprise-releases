# kafka-backup-enterprise

Enterprise edition of [kafka-backup](https://github.com/osodevops/kafka-backup) — backup and restore for Apache Kafka with Schema Registry, RBAC, and encryption support.

**14-day free trial on first run — no signup required.**

## Install

**Homebrew (macOS / Linux):**
```bash
brew install osodevops/tap/kafka-backup-enterprise
```

**Docker:**
```bash
docker pull osodevops/kafka-backup-enterprise:v0.2.1
```

**Binary download:** See [Releases](https://github.com/osodevops/kafka-backup-enterprise-releases/releases) for Linux (x86_64, aarch64), macOS (Intel, Apple Silicon), and Windows.

## Quick Start

```bash
# Check license status (auto-trial activates on first run)
kafka-backup-enterprise license info

# Backup Kafka + Schema Registry
kafka-backup-enterprise backup --config backup.yaml

# Schema-only backup
kafka-backup-enterprise backup --config backup.yaml --schema-only
```

Minimal config:

```yaml
mode: backup
backup_id: "my-backup"
source:
  bootstrap_servers: ["kafka:9092"]
storage:
  backend: s3
  bucket: my-kafka-backups
enterprise:
  schema_registry:
    url: "https://schema-registry:8081"
```

## Enterprise Features

| Feature | Description |
|---------|-------------|
| **Confluent Schema Registry backup** | All subjects, versions, schemas, compatibility configs, modes |
| **Apicurio Registry v3 backup** | Groups, artifacts, versions, content, rules, export ZIP |
| **Confluent RBAC backup** | MDS role bindings, principals, cluster scopes |
| **Field-level encryption backup** | CSFLE/DEK Registry metadata, KEK/DEK inventory |
| **Both registries simultaneously** | Configure Confluent SR and Apicurio in the same config |

All features work with the OSS backup engine — topics, partitions, offsets, consumer groups, three-phase restore, and everything else in [kafka-backup](https://github.com/osodevops/kafka-backup).

## Documentation

| Guide | Description |
|-------|-------------|
| [Getting Started](docs/getting-started.md) | Install, first backup, apply a license |
| [Configuration Reference](docs/configuration.md) | Full YAML config for all enterprise features |
| [Schema Registry Backup](docs/schema-registry.md) | Confluent SR backup and restore |
| [Apicurio Registry Backup](docs/apicurio-registry.md) | Apicurio v3 backup and restore |
| [RBAC Backup](docs/rbac.md) | Confluent MDS role binding backup |
| [Encryption Backup](docs/encryption.md) | Field-level encryption metadata backup |
| [Licensing](docs/licensing.md) | How the license system works, trial, purchasing |
| [Kubernetes Deployment](docs/kubernetes.md) | Deploy to K8s with license secret |

## Examples

| Example | Description |
|---------|-------------|
| [basic-backup.yaml](examples/basic-backup.yaml) | Minimal backup config |
| [confluent-sr.yaml](examples/confluent-sr.yaml) | Confluent Schema Registry with auth |
| [apicurio.yaml](examples/apicurio.yaml) | Apicurio Registry with OIDC |
| [full-enterprise.yaml](examples/full-enterprise.yaml) | All enterprise features |
| [kubernetes/](examples/kubernetes/) | K8s deployment manifests |

## Licensing

The binary includes a **14-day free trial** — all enterprise features are enabled automatically on first run, no signup needed.

After the trial:
- Get a 30-day extended trial at [kafkabackup.com/enterprise](https://kafkabackup.com/enterprise)
- Purchase an annual license for production use

Without a license, the binary continues to work as the full OSS edition.

```bash
# Apply a license file
kafka-backup-enterprise license apply --file license.lic

# Check license status
kafka-backup-enterprise license info
```

## Links

- [kafkabackup.com/enterprise](https://kafkabackup.com/enterprise) — Pricing and trials
- [kafka-backup OSS](https://github.com/osodevops/kafka-backup) — Open-source core (MIT)
- [Docker Hub](https://hub.docker.com/r/osodevops/kafka-backup-enterprise) — Docker images
- [Changelog](CHANGELOG.md)
