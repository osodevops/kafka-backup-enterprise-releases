# Kafka Backup Enterprise

Back up Kafka records, recover schemas and restore explicitly mapped permissions onto existing identities. Run the CLI or container in your own infrastructure with your choice of backup storage.

[Documentation](https://kafkabackup.com/enterprise/) · [Downloads](https://github.com/osodevops/kafka-backup-enterprise-releases/releases) · [Pricing](https://enterprise.kafkabackup.com)

## Evaluate 0.8.0

The binary includes a fourteen-day evaluation of shipped Enterprise features. No signup or payment card is required, and there is no automatic charge. Inspection does not activate it; explicit activation or the first operational command in the Enterprise binary starts the clock.

```bash
brew install osodevops/tap/kafka-backup-enterprise
kafka-backup-enterprise license activate-trial --format json
kafka-backup-enterprise license info --format json
```

The Docker image targets Linux amd64. ARM hosts can use Docker amd64 emulation
or the native ARM binary. Persist trial state across restarts:

```bash
docker pull --platform linux/amd64 osodevops/kafka-backup-enterprise:v0.8.0
docker volume create kafka-backup-trial
docker run --rm \
  --mount type=volume,src=kafka-backup-trial,dst=/var/lib/kafka-backup \
  osodevops/kafka-backup-enterprise:v0.8.0 \
  license activate-trial --format json
```

Standalone downloads cover Linux x86_64/aarch64, macOS Intel/Apple Silicon and Windows x86_64. Linux binaries require glibc 2.39+; use Docker on older hosts. The unpacked executable is named `kafka-backup`; Homebrew installs `kafka-backup-enterprise`.

## Enterprise features

| Capability | Behavior |
| --- | --- |
| Schema recovery | Confluent Schema Registry and Apicurio v2/v3 backup and restore, dependency ordering and supported schema-ID rewriting |
| Authorization recovery | Versioned capture, explicit mappings, reviewed plans, additive apply/resume and assignment verification for MDS roles, Cloud IAM and Kafka ACLs |
| Enterprise authentication | OAuth/OIDC for Kafka and Schema Registry |
| Erasure suppression | Drop or tombstone listed keys during restore, with counts and checksum evidence |
| Encryption metadata | Back up CSFLE/DEK Registry metadata; KMS master keys remain external |
| Migration | Journalled migration, consumer-offset translation and cutover tooling; compare your provider's native options |
| Commercial licensing | Offline signed licences and commercial support |

Core Kafka backup/restore, storage backends, point-in-time recovery and consumer-offset management are also available in the [open-source edition](https://github.com/osodevops/kafka-backup).

Field-level record encryption, masking, general audit logging and WASM transformations remain planned. Metadata backup does not encrypt records or prove recovered applications can decrypt them.

## Rehearse permission recovery

Recover data, schemas and requested offsets before releasing application access. Use existing destination identities and externally provisioned bootstrap administration. Review every mapping and the plan checksum.

```bash
kafka-backup authorization backup --config authorization.yaml --out snapshot.json --format json
kafka-backup authorization plan --config authorization.yaml --snapshot snapshot.json --out plan.json --format json
kafka-backup authorization apply --config authorization.yaml --plan plan.json --approve REVIEWED_CHECKSUM --format json
kafka-backup authorization verify --config authorization.yaml --plan plan.json --format json
```

Recovery preserves target-only assignments and never automatically deletes or rolls back permissions. Administrator grants require explicit selection and opt-in. Verification establishes assignment presence; independently test allowed and denied application operations.

Platform enforcement is qualified on CP 7.6.0 and 8.3.1. Cloud qualification covers service-account principals across two environments within one organisation. Cross-organisation recovery, Cloud user/group/identity-pool principals, operator integration and complete cross-component recovery sets are excluded. Kafka ACL writes currently require SASL/PLAIN over SASL_SSL or SASL_PLAINTEXT. Legacy unversioned RBAC exports require fresh capture.

Follow the [recovery guide](https://kafkabackup.com/enterprise/authorization-recovery/), [evaluation guide](https://kafkabackup.com/enterprise/trial-evaluation/) and [registry recovery boundaries](https://kafkabackup.com/enterprise/schema-registry-restore/). Keep snapshots, plans, journals and licence files access controlled.

At expiry, requested Enterprise operations fail; OSS functionality remains available and existing permissions remain in place. Purchase a commercial licence through [enterprise.kafkabackup.com](https://enterprise.kafkabackup.com).
