# Changelog

## v0.3.2 (2026-05-08)

### Features
- **AWS MSK ZooKeeper to KRaft migration** — production-ready enterprise migration flow for moving MSK Provisioned clusters from ZooKeeper metadata mode to KRaft metadata mode.
- **Controlled cutover** — coordinated producer freeze, sentinel drain, translated consumer group offset commits, and explicit `READY_FOR_CLIENT_SWITCH` handoff.
- **Offset continuity** — source consumer group offsets are translated onto the target cluster so consumers resume from the same message position after the switch.
- **S3-backed seed and tail** — bulk seed through S3 followed by live tail replication until lag is within the configured drain threshold.
- **ACL and auth migration planning** — SCRAM-to-SCRAM E2E path, ACL drift policies, and IAM access-map output for cross-auth modernization paths.
- **Signed migration evidence** — Ed25519-signed JSON evidence bundle with the complete journal, topology plan, ACL plan, seed/tail stats, cutover report, offset translations, and validation results.

### Validation
- Full AWS end-to-end release rehearsal completed on May 7, 2026 with migration ID `aws-e2e-r4-20260507T200820Z`.
- Covered 11 topics, 68 partitions, compacted topics, tombstones, Kafka Connect internals, Kafka Streams topics, ACLs, consumer offsets, sentinels, and post-switch writes.
- Seeded 120,861 records / 123,761,664 bytes, translated 37 offsets across 3 consumer groups, observed 68 sentinels, and matched 122/122 comparable spot-check records.
- Final validation outcome was `WARNING` only because 26 empty or zero-span partitions had no record available for spot comparison; all comparable records matched and `offset_floor_violations=0`.

### Release Notes
- Enterprise release is built against OSS core `v0.15.4`.
- Helm chart and app version are `0.3.2`.
- The AWS-proven production path is SCRAM-SHA-512 source to SCRAM-SHA-512 target. IAM and cross-auth paths are implemented and covered by automated tests, but should be rehearsed in staging before being treated as equivalent production evidence.

## v0.3.1 (2026-04-10)

### CI/CD
- Fully automated release pipeline: tag → build → Docker Hub → public release sync → Homebrew formula update
- No manual steps required for any release

## v0.3.0 (2026-04-10)

### Features
- **OAuth/OIDC authentication** for Confluent Schema Registry and DEK Registry — supports Okta, Azure AD, Keycloak via standard client credentials flow with automatic token refresh
- **Tink AEAD decryption** — decrypt AES-256-GCM and AES-128-GCM encrypted field values using plaintext DEK bytes. Enables `Decrypt` and `ReEncrypt` restore modes.
- **Encryption restore** — restore backed-up KEKs and DEKs to a target DEK Registry with three conflict policies (skip, overwrite, error) and dry-run support

### Tests
- 6 new Tink crypto tests (encrypt/decrypt roundtrip, wrong key, tampered data, AES-128)
- 7 new encryption restore wiremock tests (full pipeline, conflict handling, dry-run, soft-deleted)

## v0.2.1 (2026-04-09)

### Features
- **Keygen.sh license file support** — Accept `.lic` files from Keygen.sh for automated license management
- Offline Ed25519 verification of Keygen.sh licenses (zero network calls)
- `license apply` and `license verify` commands support both `.lic` and `.key` formats
- License loader checks for `license.lic` (Keygen.sh) before `license.key` (custom PEM)

## v0.2.0 (2026-04-09)

### Features
- **Apicurio Registry v3 backup** — Full Core Registry API v3 client with Groups, Artifacts, Versions, Admin Export, and Rules at 3 scopes (global/group/artifact)
- **14-day auto-trial** — All enterprise features enabled on first run, no signup required
- **CI/CD pipeline** — GitHub Actions for testing, building (5 platforms), Docker Hub publishing
- **Docker image** — `osodevops/kafka-backup-enterprise` on Docker Hub (public)
- **Homebrew formula** — `brew install osodevops/tap/kafka-backup-enterprise`

### Security
- Trial state HMAC integrity check prevents tampering
- Fixed trial path (`/var/lib/kafka-backup/.trial`) prevents HOME override attacks
- Ed25519 license signature verification (offline, air-gapped)

## v0.1.0 (2026-04-04)

### Features
- **Confluent Schema Registry backup** — All subjects, versions, schemas, compatibility, modes, references
- **Confluent RBAC backup** — MDS role bindings, principals, cluster scopes
- **Field-level encryption backup** — CSFLE detection, DEK Registry, KEK/DEK inventory
- **Ed25519 license system** — Offline verification, PEM format, feature gating
- **License CLI** — `license apply`, `license info`, `license verify`
