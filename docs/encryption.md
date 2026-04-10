# Field-Level Encryption Backup

Backs up encryption metadata from Confluent's Client-Side Field-Level Encryption (CSFLE) system, including Key Encryption Keys (KEKs) and Data Encryption Keys (DEKs) from the DEK Registry.

## What Gets Backed Up

| Item | Description |
|------|-------------|
| KEKs | Key Encryption Key names and configurations |
| DEKs | Data Encryption Key registry entries (subject, version, algorithm) |
| Encrypted subjects | Which subjects have CSFLE-encrypted fields |

## Config

```yaml
enterprise:
  schema_registry:
    url: "https://schema-registry:8081"   # Required — DEK Registry is a sub-API of SR
  encryption:
    enabled: true
    backup:
      kek_filter:
        - "prod-*"
      exclude_keks:
        - "prod-test-*"
```

## Prerequisites

Encryption backup requires `schema_registry` to be configured because the DEK Registry is a sub-API of Confluent Schema Registry.
