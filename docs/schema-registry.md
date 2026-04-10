# Confluent Schema Registry Backup

Backs up all subjects, schema versions, compatibility configs, modes, and schema references from Confluent Schema Registry.

## What Gets Backed Up

| Item | Description |
|------|-------------|
| Subjects | All subject names matching include/exclude patterns |
| Versions | Every schema version (or latest only) |
| Schemas | Full schema content (Avro, Protobuf, JSON Schema) |
| Compatibility | Global and per-subject compatibility levels |
| Mode | Global and per-subject modes (READWRITE, READONLY) |
| References | Cross-subject schema references (dependency ordering) |

## Minimal Config

```yaml
enterprise:
  schema_registry:
    url: "http://schema-registry:8081"
```

## With Authentication

```yaml
enterprise:
  schema_registry:
    url: "https://schema-registry:8081"
    auth:
      type: basic
      username: ${SR_USER}
      password: ${SR_PASS}
    tls:
      ca_cert: /certs/ca.pem
```

## Subject Filtering

```yaml
enterprise:
  schema_registry:
    backup:
      subjects:
        - "orders-*"
        - "payments-*"
      exclude:
        - "*-test"
        - "*-internal"
```

## Version Selection

```yaml
enterprise:
  schema_registry:
    backup:
      include_versions: all      # all | latest
      include_soft_deleted: false
```

## Schema References

When schemas reference other schemas (e.g., Protobuf imports), the backup engine:

1. Discovers all references across subjects
2. Builds a dependency graph (DAG)
3. Performs topological sort for correct restore order
4. Detects circular references (error)

Enable with `include_references: true` (default).

## Storage Layout

```
{backup_id}/schema-registry/
  _manifest.json          # Backup manifest with stats
  _global_config.json     # Global compatibility + mode
  subjects/
    {subject}/
      _metadata.json      # Subject metadata + per-subject config
      v1.json             # Schema version 1
      v2.json             # Schema version 2
```

## Running

```bash
# Full backup (Kafka data + Schema Registry)
kafka-backup-enterprise backup --config backup.yaml

# Schema Registry only (no Kafka data)
kafka-backup-enterprise backup --config backup.yaml --schema-only
```
