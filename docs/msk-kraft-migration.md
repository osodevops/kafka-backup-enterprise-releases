# AWS MSK ZooKeeper to KRaft Migration

Available from kafka-backup Enterprise `v0.3.2`.

This release adds the enterprise MSK migration workflow for moving Amazon MSK Provisioned clusters from ZooKeeper metadata mode to KRaft metadata mode. It is designed for planned production cutovers where data, topic configuration, ACL intent, and consumer group offsets all need to move together.

## What It Does

- Creates a migration plan and precheck report before making changes.
- Copies topic topology and compatible configuration to the target cluster.
- Seeds existing data through S3, then tails new source records until lag is inside the drain threshold.
- Coordinates a short producer freeze at cutover.
- Publishes sentinel records and drains them to the target.
- Translates source consumer group offsets to target offsets.
- Verifies target offset-floor safety before clients switch.
- Produces an Ed25519-signed JSON evidence bundle for audit.

## Release Evidence

The `v0.3.2` release was exercised end-to-end in AWS on May 7, 2026:

```text
migration_id=aws-e2e-r4-20260507T200820Z
source=kbe-msk-kraft-e2e-source-zk metadata=ZOOKEEPER kafka=3.7.x brokers=3
target=kbe-msk-kraft-e2e-target-kraft-r4 metadata=KRAFT kafka=3.9.x.kraft brokers=3
seed=120861 records / 123761664 bytes / 68 partitions
cutover=READY_FOR_CLIENT_SWITCH groups_translated=3 offsets_committed=37 warnings=0
counts_and_offsets=PASSED offset_floor_violations=0
spot_check_records=WARNING compared=122 matched=122 skipped=26 empty/zero-span partitions
sentinel_presence=PASSED sentinels=68
post_switch=e2e.heartbeat target_offset 201 -> 204; read_back=3/3
```

The validation warning above is not a data mismatch. It means some empty or zero-span partitions had no record available for spot comparison. Every comparable sample matched.

## Proven Auth Path

The full AWS end-to-end release rehearsal proved:

```text
SCRAM-SHA-512 source -> SCRAM-SHA-512 target
```

IAM and cross-auth paths are implemented and covered by automated tests, including endpoint selection, token handling, smoke tests, and IAM access-map generation. Rehearse IAM or cross-auth migrations in staging before using the result as production change evidence.

## Minimal Config

```yaml
enterprise:
  msk_kraft_migration:
    source:
      cluster_arn: arn:aws:kafka:eu-west-2:123456789012:cluster/prod-zk/abc-123
      auth:
        mode: scram-sha-512
        username: ${KAFKA_SOURCE_USER}
        password: ${KAFKA_SOURCE_PASSWORD}
    target:
      cluster_arn: arn:aws:kafka:eu-west-2:123456789012:cluster/prod-kraft/def-456
      auth:
        mode: scram-sha-512
        username: ${KAFKA_TARGET_USER}
        password: ${KAFKA_TARGET_PASSWORD}
    backup:
      s3_bucket: prod-msk-migration-segments
      s3_prefix: migrations/
    evidence:
      s3_bucket: prod-msk-migration-evidence
      s3_prefix: evidence/
      retention: 7y
    cutover:
      drain_timeout: 30m
      drain_max_partition_lag: 100
      drain_stable_window: 30s
      max_producer_freeze: 60s
      producer_freeze_webhook: https://internal.example.com/kafka/freeze
    validation:
      count_tolerance: 1
      spot_check_records_per_partition: 3
    acl:
      on_drift: merge
```

## Command Flow

`plan` and `precheck` are free and read-only:

```bash
kafka-backup-enterprise migrate msk-kraft plan \
  --config migration.yaml \
  --format all \
  --out-dir ./migration-plan

kafka-backup-enterprise migrate msk-kraft precheck \
  --config migration.yaml
```

Mutation commands require an enterprise license or active trial:

```bash
kafka-backup-enterprise migrate msk-kraft execute \
  --config migration.yaml \
  --journal-dir ./journal

kafka-backup-enterprise migrate msk-kraft cutover \
  --config migration.yaml \
  --migration-id <MIGRATION_ID> \
  --journal-dir ./journal

kafka-backup-enterprise migrate msk-kraft cutover-ack \
  --config migration.yaml \
  --migration-id <MIGRATION_ID> \
  --journal-dir ./journal

kafka-backup-enterprise migrate msk-kraft finalize \
  --config migration.yaml \
  --migration-id <MIGRATION_ID> \
  --journal-dir ./journal
```

## Validation Checks

Finalize runs five checks:

| Check | Pass criteria |
|-------|---------------|
| Topic parity | Topic partition counts match |
| Counts and offsets | Per-partition spans are within `count_tolerance`; `offset_floor_violations=0` |
| Spot-check records | Comparable sampled records match byte-for-byte |
| Sentinel presence | Cutover sentinels are present on target |
| Consumer group reconciliation | Target committed offsets match translated offsets |

The overall outcome can be `PASSED`, `WARNING`, or `FAILED`. Treat `WARNING` as acceptable only when the report explains a known condition, such as empty partitions with no spot-check sample, and all comparable records matched.

## Operational Notes

- Create the KRaft target cluster before running the migration.
- The source cluster is never modified.
- Rollback is available before cutover completes.
- Non-interactive cutover requires `cutover.producer_freeze_webhook`; the tool will not assume producers are frozen in a non-TTY shell.
- Reverse replication is intentionally not enabled in `v0.3.2`.
- Evidence is signed JSON. Keep the evidence bucket and consider S3 Object Lock for regulated environments.
