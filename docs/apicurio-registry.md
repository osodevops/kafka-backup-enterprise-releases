# Apicurio Registry v3 Backup

Backs up groups, artifacts, versions, content, references, and rules from Apicurio Registry via the Core Registry API v3. Captures both a native export ZIP and a structured artefact-by-artefact backup.

## What Gets Backed Up

| Item | Description |
|------|-------------|
| Export ZIP | Native `/admin/export` ZIP archive (primary, for disaster recovery) |
| Groups | All groups matching include/exclude patterns |
| Artifacts | All artifacts in matched groups (Avro, Protobuf, JSON Schema, OpenAPI, AsyncAPI, GraphQL, KCONNECT, WSDL, XSD) |
| Versions | Every version of every artifact (or latest only) |
| Content | Raw schema/definition bytes for each version |
| References | Cross-artifact dependencies (dependency ordering via topological sort) |
| Global rules | COMPATIBILITY, VALIDITY, INTEGRITY at global scope |
| Group rules | Rules at group scope |
| Artifact rules | Rules at artifact scope |

## Minimal Config

```yaml
enterprise:
  apicurio_registry:
    url: "http://apicurio:8080"
```

## With OIDC Authentication (Keycloak)

```yaml
enterprise:
  apicurio_registry:
    url: "https://apicurio:8080"
    auth:
      type: oidc
      token_url: https://keycloak:8443/realms/registry/protocol/openid-connect/token
      client_id: ${APICURIO_CLIENT_ID}
      client_secret: ${APICURIO_CLIENT_SECRET}
      scope: registry-api
```

## Group and Artifact Filtering

```yaml
enterprise:
  apicurio_registry:
    backup:
      groups:
        - "payments"
        - "orders"
      exclude_groups:
        - "test-*"
        - "dev-*"
      artifacts:
        - "*"
      exclude_artifacts:
        - "*-internal"
```

## Storage Layout

```
{backup_id}/apicurio-registry/
  _manifest.json           # Backup manifest with stats and dependency order
  _export.zip              # Native Apicurio export ZIP
  _global_rules.json       # Global rules
  groups/
    {groupId}/
      _metadata.json       # Group metadata + group rules
      artifacts/
        {artifactId}/
          _metadata.json   # Artifact metadata + artifact rules
          versions/
            {version}.json     # Version metadata (globalId, contentId, refs)
            {version}.content  # Raw content bytes
```

## Using Both Registries

You can back up Confluent SR and Apicurio in the same config:

```yaml
enterprise:
  schema_registry:
    url: "https://confluent-sr:8081"
  apicurio_registry:
    url: "https://apicurio:8080"
```

Both run sequentially — Confluent SR first, then Apicurio.

## KafkaSQL Backend

If your Apicurio instance uses the KafkaSQL storage backend, set:

```yaml
enterprise:
  apicurio_registry:
    storage_backend: kafkasql
```

This is informational for now — future versions will additionally back up the `kafkasql-journal` Kafka topic for point-in-time recovery.
