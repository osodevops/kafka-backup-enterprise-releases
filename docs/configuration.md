# Configuration Reference

The enterprise binary uses the same YAML config as the OSS edition, with an additional `enterprise:` section for enterprise features. All OSS fields remain at the root level.

## Full Configuration

```yaml
# ═══════════════════════════════════════════════════════════
# OSS Configuration (same as kafka-backup)
# ═══════════════════════════════════════════════════════════

mode: backup                    # backup | restore
backup_id: "daily-backup-001"

source:
  bootstrap_servers:
    - kafka-1:9092
    - kafka-2:9092
  security:
    security_protocol: SASL_SSL
    sasl_mechanism: SCRAM-SHA-512
    sasl_username: ${KAFKA_USER}
    sasl_password: ${KAFKA_PASS}
  topics:
    include:
      - "*"
    exclude:
      - "__*"                   # Exclude internal topics

target:                         # For restore mode
  bootstrap_servers:
    - kafka-dr:9092

storage:
  backend: s3                   # s3 | azure | gcs | filesystem
  bucket: kafka-backups
  region: us-east-1
  prefix: production

backup:
  compression: zstd
  segment_max_bytes: 134217728  # 128MB

restore:
  time_window_start: 1743955200000
  time_window_end: 1744041600000

# ═══════════════════════════════════════════════════════════
# Enterprise Configuration
# ═══════════════════════════════════════════════════════════

enterprise:

  # ─── Confluent Schema Registry ──────────────────────────
  schema_registry:
    enabled: true               # Default: true when section present
    url: "https://schema-registry:8081"

    auth:
      type: basic               # none | basic | mtls | oauth
      username: ${SR_USER}
      password: ${SR_PASS}

    tls:
      ca_cert: /certs/ca.pem

    backup:
      subjects:                 # Glob patterns (default: ["*"])
        - "orders-*"
        - "payments-*"
      exclude:                  # Exclude patterns
        - "*-internal"
        - "*-test"
      include_soft_deleted: false
      include_versions: all     # all | latest
      include_references: true  # Auto-include referenced subjects

    connection:
      timeout_ms: 30000
      max_retries: 3
      retry_backoff_ms: 1000
      rate_limit_rps: 25

  # ─── Apicurio Registry v3 ──────────────────────────────
  apicurio_registry:
    enabled: true
    url: "https://apicurio:8080"

    storage_backend: sql        # sql | kafkasql | auto

    auth:
      type: oidc                # none | basic | mtls | oidc
      token_url: https://keycloak:8443/realms/registry/protocol/openid-connect/token
      client_id: ${APICURIO_CLIENT_ID}
      client_secret: ${APICURIO_CLIENT_SECRET}
      scope: registry-api

    tls:
      ca_cert: /certs/ca.pem

    backup:
      groups:                   # Glob patterns (default: ["*"])
        - "payments"
        - "orders"
      exclude_groups:
        - "test-*"
      artifacts:                # Glob patterns (default: ["*"])
        - "*"
      exclude_artifacts:
        - "*-internal"
      include_export: true      # Capture native export ZIP
      include_versions: all     # all | latest
      include_references: true

    connection:
      timeout_ms: 30000
      max_retries: 3
      retry_backoff_ms: 1000
      rate_limit_rps: 50

  # ─── Confluent RBAC (MDS) ──────────────────────────────
  confluent_rbac:
    enabled: true
    mds_url: "https://mds:8090"

    auth:
      username: ${MDS_USER}
      password: ${MDS_PASS}

    tls:
      ca_cert: /certs/ca.pem
      client_cert: /certs/client.crt
      client_key: /certs/client.key

    backup:
      principals:               # Glob patterns (default: ["*"])
        - "User:app-*"
        - "User:connect-*"
      exclude_principals:
        - "User:admin"
      cluster_filter: []        # Specific cluster IDs (default: all)
      include_sub_clusters: true

    connection:
      timeout_ms: 30000
      max_retries: 3
      retry_backoff_ms: 1000
      rate_limit_rps: 12

  # ─── Field-Level Encryption ─────────────────────────────
  encryption:
    enabled: true

    backup:
      kek_filter:               # KEK name patterns
        - "prod-*"
      exclude_keks:
        - "prod-test-*"
```

## Environment Variable Interpolation

All config values support `${ENV_VAR}` syntax:

```yaml
auth:
  username: ${SR_USER}
  password: ${SR_PASS}
```

## Feature Independence

Each enterprise feature is independently configurable. You can use any combination:

- Schema Registry only
- Apicurio Registry only
- Both registries + RBAC
- Everything

If a feature is configured but not licensed, it logs a warning and skips (no error).

## Auth Types

| Type | Config | Used By |
|------|--------|---------|
| `none` | No auth | Schema Registry, Apicurio |
| `basic` | `username` + `password` | Schema Registry, Apicurio, MDS |
| `mtls` | `ca_cert` + `client_cert` + `client_key` | Schema Registry, Apicurio |
| `oauth` | `token_url` + `client_id` + `client_secret` | Schema Registry |
| `oidc` | `token_url` + `client_id` + `client_secret` + `scope` | Apicurio |
