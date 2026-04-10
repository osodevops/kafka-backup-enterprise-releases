# Confluent RBAC Backup

Backs up all role bindings from Confluent Metadata Service (MDS), preserving the complete RBAC configuration for disaster recovery.

## What Gets Backed Up

| Item | Description |
|------|-------------|
| Role bindings | All principal-to-role assignments across clusters |
| Principals | Users and service accounts with their roles |
| Cluster scopes | Kafka, Schema Registry, Connect, ksqlDB cluster bindings |
| Resource patterns | Topic, group, transactional ID, and subject-level bindings |

## Config

```yaml
enterprise:
  confluent_rbac:
    mds_url: "https://mds:8090"
    auth:
      username: ${MDS_USER}
      password: ${MDS_PASS}
    backup:
      principals:
        - "User:app-*"
        - "User:connect-*"
      exclude_principals:
        - "User:admin"
```

## Authentication

MDS uses bearer token auth: the client exchanges Basic credentials for a JWT token, which is auto-refreshed on 401.

## Principal Filtering

```yaml
backup:
  principals:
    - "User:*"             # All users
  exclude_principals:
    - "User:backup-svc"   # Exclude the backup service account
  cluster_filter: []       # Specific cluster IDs (default: all)
  include_sub_clusters: true
```
