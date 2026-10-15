# Changelog

All notable changes to kafka-backup-enterprise are documented here.

## [0.8.0] - 2026-10-15

Builds against kafka-backup-core **v0.25.2**, including its transport dependency
security updates.

- Rotate the commercial licence verification key away from the historical
  developer test signer. Commercial licence files issued for 0.7 and earlier
  require reissuance for 0.8. The built-in evaluation requires no signed file.
  The portal signer must switch with the public 0.8 release; older binaries
  cannot verify licences issued by the new signer.
- Persist evaluation state atomically with concurrent activation locking,
  strict integrity checks and exact fourteen-day expiry. Inspection commands
  do not start the evaluation. Retired public trial-key signatures are refused.
- Add versioned authorization capture, explicit-mapping plans, checksum-approved
  additive recovery, durable intent journals and read-only verification.
- Correct MDS direct-binding enumeration, principal/cluster filters and native
  cluster/resource classification; legacy snapshots remain inspection-only.
- Preserve complete Kafka ACL tuples and require complete protocol responses.
  ACL recovery uses a single-attempt SASL/PLAIN administration transport.
- Add Platform MDS and Cloud IAM adapters, structural Cloud CRNs, pagination
  guards, ownership checks and administrator opt-in.
- Add the `authorization backup|plan|apply|verify` commands and optional approved
  authorization recovery after registry, data and consumer-offset restoration.
- Requested unlicensed authorization/RBAC operations now refuse before transfer
  with exit 2. Existing `confluent_rbac` capture configuration remains accepted.
- Ship the matching 0.8.0 Helm chart with durable trial PVC retention/reuse.
- Native authorization qualification passed on CP 7.6.0 and 8.3.1 and for Cloud
  service accounts across two environments within OSO. The complete 81-case
  registry matrix passed against matching source hashes. Public-artifact
  verification and final credential cleanup are recorded separately.
- Require native cluster provenance even for ACL-only snapshots; reject
  provider translation, cross-organisation recovery and unqualified Cloud ACL
  principal formats.

## [0.7.0] - 2026-10-14

Builds against kafka-backup-core **v0.25.1** (previously v0.24.1) and
inherits its changes: read-committed backups, checkpoint durability,
idempotent and resumable restores, topic configs that restore into Kafka 4.x
and Confluent Cloud, and logs on stderr. The CLI is at parity with the
kafka-backup v0.25.0 CLI. New in Enterprise: **OAuth (SASL/OAUTHBEARER) for
Confluent Cloud** and other OIDC identity providers, for Kafka and Schema
Registry, and **Schema Registry restore with original IDs**.

### Upgrade notes
- **kafka-backup-core 0.25.0 upgrade notes apply** (see its changelog):
  - `backup.isolation_level` defaults to `read_committed`: aborted records
    and transaction markers are no longer archived or restored. Set
    `read_uncommitted` to keep the previous behaviour; alert on
    `kafka_backup_lso_lag` for hanging transactions.
  - One-time re-read after upgrading where an `offsets.db` checkpoint is
    ahead of the manifest; backups resume from the manifest without
    `offset_storage`.
  - Restores use an idempotent producer by default, which requires
    `restore.produce_acks: -1`; set `restore.idempotent_producer: false` to
    keep `0` or `1`. Restore rate limits are restore-wide.
  - Logs go to stderr (stdout carries only command output);
    `--log-format json` for structured logs.
  - Unknown keys under `source.security` / `target.security` are errors
    (`security.oauth` is rejected with a pointer to `enterprise.oauth`).
  - `restore.rewrite_schema_ids` fails on a truncated or unmapped schema ID
    frame (`schema_id_rewrite_strict`, default `true`).
- **MSK IAM migrations require the `oauth` licence feature** in addition to
  the migration feature (MSK IAM is SASL/OAUTHBEARER). **Licences issued for
  MSK IAM migrations must be reissued with `oauth` before upgrading**;
  without it `migrate msk-kraft` on an IAM cluster is refused (exit status 2).
- **Licence refusals exit with status 2** (OAuth without the `oauth` feature
  or after its grace period, erasure suppression or a Schema Registry
  restore without their features, `migrate msk-kraft` without its feature).
  Other errors still exit with 1.
- **Soft-deleted schema versions are backed up by default**
  (`enterprise.schema_registry.backup.include_soft_deleted: true`): records
  may still carry their IDs. Set `false` for the previous behaviour.
- **`enterprise.schema_registry.restore.strategy`** takes `preserve_ids`
  (default) or `into_existing`; `preserve` and `skip` are read as
  `into_existing`, `overwrite` is rejected and `force_ids` is removed (the
  section was parsed but unused before).
- **MSK ZooKeeper → KRaft migrations read the source `read_committed`**
  (SEED, TAIL and the cutover drain): transaction markers and aborted
  records are no longer copied to the target, where they used to arrive as
  ordinary records. A transaction still open on the source after the
  producer freeze now fails the cutover drain with an explicit error.

### Added
- **OAuth for Kafka (SASL/OAUTHBEARER) and Schema Registry** (licence feature
  `oauth`, included in the trial). `enterprise.oauth.providers.<name>` with
  `oidc_client_credentials` (client_secret_basic or _post, scope, audience,
  extra parameters, OIDC discovery; Entra ID, Okta, Keycloak),
  `azure_managed_identity` (IMDS), `azure_workload_identity` (AKS federated
  token exchange), `file` and `static` (tests). `enterprise.oauth.kafka.source`
  / `.target` bind a provider to a cluster with optional Confluent Cloud SASL
  extensions `logicalCluster` and `identityPoolId` (comma-separated lists
  allowed). Rust-native (`oauth2` crate over reqwest/rustls); no JVM,
  librdkafka or Azure SDK. See `docs/oauth-confluent-cloud.md`.
- One token cache per provider shared by every connection and by Schema
  Registry: single-flight refresh, refresh at 80 % of the lifetime or 5 min
  before expiry (`enterprise.oauth.cache`), stale-but-valid serving with
  backoff while the identity provider fails, handshake tokens with at least
  60 s left so KIP-368 re-authentication happens in place (with
  kafka-backup-core 0.25's SaslHandshake re-authentication), invalidation on
  `invalid_token` and on Schema Registry 401.
- Schema Registry / DEK Registry `auth: { type: oauth, provider,
  target_sr_cluster, identity_pool_id }`: bearer token plus the
  `target-sr-cluster` and (when set) `Confluent-Identity-Pool-Id` headers, on
  the Schema Registry backup and restore alike.
- Every command that connects to Kafka or Schema Registry applies OAuth:
  `backup`, `restore` (including `--schema-only`), `three-phase-restore`,
  `validate-restore`, `snapshot-groups`, `validation run`, and the offset
  commands via `--config <yaml> --cluster source|target`.
  `--sasl-mechanism OAUTHBEARER` parses and, without `--config`, says how to
  configure a provider (instead of the OSS "requires Kafka Backup
  Enterprise" error).
- **30-day OAuth grace period** after a commercial licence expires: a warning
  on every run, `license info` (text and JSON `oauth` object) shows the days
  left, and the metric `kafka_backup_enterprise_licence_grace_days_remaining`.
  After it, OAuth is refused; a continuous backup stops after its current
  cycle. Trials get no grace period.
- Metric `kafka_backup_enterprise_oauth_token_requests_total{provider,kind,outcome}`.
- **Schema Registry restore with original IDs** (licence feature
  `schema_registry`; `enterprise.schema_registry.restore`, the registry URL
  being the target). `restore` and `three-phase-restore` restore the registry
  first, then the records; `restore --schema-only` restores only the
  registry. Strategies:
  - `preserve_ids` (default): empty target; global IMPORT plus IMPORT on each
    context (global IMPORT does not reach contexts); every version registered
    with its original ID and version number; compatibility and modes
    restored; every version verified (ID, schema, references, soft-deleted
    state) along with subject compatibility/mode.
  - `into_existing`: populated target (for example a shared Confluent Cloud
    environment). Identical ID and schema is a skip; a free ID and version is
    imported under subject-level IMPORT; an ID or version held by a different
    schema is a conflict. Strict by default (fails before any change);
    `rewrite_ids` registers such schemas with new IDs and rewrites the IDs in
    the restored records' Confluent wire-format keys and values
    (kafka-backup-core `restore.rewrite_schema_ids` / `schema_id_mapping`),
    on the 1:1 and the repartitioning restore path alike.
  - The restore never overwrites an existing version (IMPORT mode silently
    does), orders registrations topologically on references at version level
    (ties by original ID), handles contexts (`:.ctx:subject`), restores
    soft-deleted versions with their original IDs then soft-deletes them again
    (`restore_soft_deleted`, default on), applies the backup's global settings
    by default only for `preserve_ids` (`restore_global_config`), and supports
    `subject_mapping`. The target's prior global, context and subject modes
    are put back on any failure.
  - Capacity: `max_schema_versions` fails the restore before any change when
    used (soft-deleted included) plus new versions would exceed it; a
    registry limit hit during the restore ("Schema Limit Exceeded") is
    reported as such.
  - Dry run (`dry_run`, or `restore.dry_run`) lists what would be registered,
    skipped, rewritten or in conflict. A JSON report goes to
    `{backup_id}/restore/schema-registry-report.json`
    (`.dry-run.json` for a dry run), on failure too; any failure exits
    non-zero and no record is restored.
  - A dropped or changed `metadata`/`ruleSet` on the target (a registry
    without the rule-set extension) is reported as a warning.
- The Schema Registry backup captures the versions that captured schemas
  reference outside the subject filter (`include_references`, which was not
  implemented) and context-level compatibility and mode.
- From the kafka-backup v0.25.0 CLI: `--log-format text|json` (global; also
  `KAFKA_BACKUP_LOG_FORMAT`) with every log line on stderr; `restore` and
  `three-phase-restore` record the restore metrics (`three-phase-restore`
  now serves `/metrics`, with the enterprise OAuth metrics when OAuth is in
  use); `restore` lists topic configuration skipped or adjusted on the
  target and per-topic schema ID rewrite counts; `describe` shows the
  isolation level, transaction markers and aborted records not archived, and
  snapshots that ended at the last stable offset; `offset-reset plan` marks
  interpolated offsets as approximate; `validate --deep` verifies SHA-256 and
  offset contiguity and `validate --verify-sha256` hashes without decoding
  (mismatches exit 1); `--sasl-mechanism` accepts Kafka's `SCRAM-SHA-256` /
  `SCRAM-SHA-512` spellings.

### Changed
- `LicenseValidator::verify` is split into `verify_signature` and
  `check_expiry`; `LicenseError::LicenseExpired` carries the verified payload,
  and `LicenseContext` gains `expired_license`, `expired_at`,
  `feature_status(_at)` and `feature_deadline(_at)`.
- `SchemaRegistryAuth::OAuth` is now a newtype over `SchemaRegistryOAuth`;
  the Schema Registry and DEK Registry clients' private token caches are
  replaced by the shared cache. The inline `token_url` form keeps working
  (client_secret_post, no `oauth` licence needed).
- `enterprise.oauth` rejects unknown keys.
- `validation run` expands `${ENV}` references in its configuration like
  every other command.
- With OAuth in use, `backup`, `restore` and `three-phase-restore` serve the
  enterprise metrics next to the OSS ones on the same endpoint; without
  OAuth the OSS metrics server runs unchanged.
- MSK IAM (the migrator) uses one token cache per region instead of a
  provider per connection.
- `rewrite_ids` hands the record restore the complete ID map (unchanged IDs
  map to themselves, listed as `ids_preserved` in the report), so
  kafka-backup-core's strict rewrite only fails on an ID the backup does not
  have, and scopes the rewrite to the restored topics
  (`restore.schema_id_rewrite_topics` = `target.topics.include` unless set).
- MSK ZooKeeper → KRaft migration, isolation chosen per read: source reads
  (SEED, TAIL, cutover drain, lag, seed size, validation) `read_committed`;
  target reads that check the migrator's own writes (log-start/LEO floor,
  sentinel) `read_uncommitted`; validation's view of the target
  `read_committed`. Consumer-group offsets on partitions whose replicated
  range has offsets without a record (markers, aborted records, compaction)
  are translated from the replicated records instead of linearly; validation
  reports a count drift equal to those offsets as a warning and spot-checks
  such partitions by position. `tail.progress.json` counts the markers and
  aborted records passed over.

### Fixed
- Registry reads retry when the server closes the connection before returning a
  response. Mutation requests remain one-shot. Core 0.25.1 also retries batched
  offset queries while a newly created target topic is waiting for its leader.
- Integrate the 0.6.1 registry safety and Apicurio restore work without losing
  OAuth or the context-aware Confluent restore path. Explicit common
  `enterprise.registry_restore` options retain the serializer-aware engine.
- Registry mutation requests are sent once after an uncertain response.
  Confluent restore checks sealed snapshot completeness and hashes before
  mutation; validation previews do not upload reports.
- An OAUTHBEARER broker rejection is no longer mistaken for success: the
  plugin answers the RFC 7628 error challenge with the `0x01` separator, so
  the broker returns a proper authentication error.
- Schema Registry backup: soft-deleted versions were never captured
  (`is_soft_deleted` was hard-coded to false and versions were fetched without
  `deleted=true`); they are now found by diffing the version lists with and
  without `deleted=true`.
- Schema Registry backup: crossing references between subjects (`a` v2 ->
  `b` v1, `b` v2 -> `a` v1) failed the whole backup with a reference cycle;
  ordering is now computed per version.
- Fixed while extracting the PR #15 restore: the target was left in IMPORT
  when the backup had no global mode, and was not reset on error (the prior
  global, context and subject modes are now restored on any failure); every
  subject got a forced READWRITE override (a subject without a mode in the
  source now has its override removed).
- MSK migration TAIL re-fetched the same offset forever when a fetch held
  only transaction markers or aborted records (its lag never reached 0
  while a log ended with a marker), and the cutover drain could spin until
  its deadline on an aborted transaction larger than its fetch size.

### Known limitations
- `rewrite_ids` is refused when a rewritten ID is also used in another
  context: the record rewrite is one ID map for every restored topic.
- `rewrite_ids` registrations go through the target subject's compatibility
  check; an incompatible target history fails the restore (modes restored).
- The Schema Registry API does not expose Confluent Cloud's schema limit, so
  it is checked only when `max_schema_versions` is set.
- MSK migration validation can only explain a count drift on a partition
  with transaction markers or aborted records from the migrator's own offset
  map, so such partitions validate as a warning, not a pass.
## [0.6.1] - 2026-10-13

Builds against kafka-backup-core **v0.24.1**. Schema Registry restoration now
runs before Kafka replay, with explicit serializer profiles and verified
destination identifiers.

### Added
- Confluent self-hosted and Apicurio v2/v3 restore planning, execution and
  verification; original-ID preservation, allocation and wire rewriting, and
  supported Avro binary, JSON Schema and Protobuf migrations.
- `restore --schema-only` and `validate-restore --schema-only`; registry plans
  and conflicts appear in validation output. Any dry-run setting makes the
  whole invocation read-only.
- Durable registry checkpoints bind mappings and settings recovery to the
  snapshot and target. Uncertain writes are reconciled on resume.
- A complete provider/version qualification matrix using independent pinned
  SDK producers and destination-only consumers, including repartitioning.

### Fixed
- Registry backup manifests record snapshot completeness, hashes, exact
  references, metadata, settings and lifecycle state. Exact dependency
  versions outside selection filters are captured recursively.
- Confluent schema-ID responses no longer require subject/version fields;
  deleted versions are fetched explicitly. Apicurio uses explicit v2/v3 API
  adapters and records structured and native archive scope independently.
- Missing identifiers, malformed configured frames and destination overflow
  block Kafka replay. Ordered duplicate/null headers and tombstones are
  preserved through all replay paths. Archived record bytes remain unchanged.
- Confluent IMPORT follows reference order even when a referenced later version
  must precede an earlier version of the same subject. Allocation rejects
  histories it cannot reproduce faithfully.
- Apicurio v3-to-v2 allocation verifies schema-derived display fields while
  retaining strict custom metadata and archive preservation checks.
- Registry HTTP clients explicitly use the Rust TLS backend for CA and mTLS
  identities; incomplete client-certificate/key pairs fail configuration.
- Operational logs use stderr so validation JSON/YAML reports remain
  machine-readable at the default log level, including retry warnings.

### Security
- Updated HTTP/2, QUIC, TLS and async/error dependencies to patched releases.
  AWS clients use the current HTTPS connector; the vulnerable legacy
  hyper 0.14 / rustls 0.21 fallback is no longer enabled. Vulnerability audit
  failures now block the required CI check.

### Release limits
- Confluent Cloud and MSK SEED/TAIL registry conversion are outside this
  release. Native Apicurio ID-preserving archives cannot cross API generations.
- CP 6.2 inherited-mode IMPORT, unsupported metadata/settings/states,
  custom serializers and ambiguous message/identifier mappings reject before
  mutation. Legacy snapshots cannot restore fidelity they did not capture.
- New Confluent rule-set registrations and rule-bearing Kafka replay are
  unsupported pending real rule-enabled registry/executor qualification;
  existing rule-bearing entries permit only registry-only verified exact reuse.
  Replay rejection includes encoding, domain/CSFLE ENCRYPT and migration rules.
- Native Apicurio nested Protobuf and unqualified native indexes-only/combined
  framing reject record preflight. Confluent and compatible framing support
  nested message indexes; native paths support multiple top-level messages.
- See `docs/schema-registry-restore.md` for configuration and
  `docs/schema-registry-compatibility.md` for executed evidence.

## [0.6.0] - 2026-10-13

Builds against kafka-backup-core **v0.23.8** (previously v0.22.0). The CLI is
at parity with the kafka-backup v0.23.8 CLI, and a test now fails when it is
not.

### Security
- **Release binaries embed the committed commercial licence key again** (first shipped as the 0.5.1 hotfix). The
  release workflow ran `scripts/generate-keypair.sh --test` before
  `cargo build --release`; the script overwrote `keys/license_public.key` with
  a random key, so the released binaries (not the Docker images) could not
  verify any commercial PEM licence. No workflow generates keys any more:
  - the core unit tests verify commercial signatures against a committed,
    test-only keypair (`keys/test_commercial_{private,public}.key`) that no
    non-test build embeds; `generate-keypair.sh --test` only regenerates that
    pair and never touches `license_public.key`;
  - `keys/license_public.key.sha256` pins the production key; a unit test and
    `scripts/verify-license-key.sh` fail when the key differs from it;
  - the release build checks the key before building and, after building,
    compares the binary's embedded key fingerprint (`license info --format
    json`) with the pin. The Docker jobs and the CI unit-test job run the same
    check.
- Commercial key rotation is tracked separately in issue #20 and is not part
  of this release.

### Added
- **CLI parity with kafka-backup v0.23.8**:
  - `validation run`, `validation evidence-list`, `validation evidence-get`,
    `validation evidence-verify` and `snapshot-groups`.
  - `--sasl-mechanism`, `--sasl-keytab`, `--sasl-krb5-config` and
    `--sasl-kerberos-service-name` (with their `KAFKA_*` environment variables)
    on `offset-reset execute`, `offset-reset-bulk` and `offset-rollback
    snapshot` / `rollback` / `verify`.
  - `sasl_mechanism: GSSAPI` in configuration files, behind the CLI crate's
    `gssapi` cargo feature (off by default, as in the OSS release builds).
  - The metrics server keep-alive (`metrics.keep_alive_seconds`) for `backup`
    and `restore`.
- **`--config <yaml> [--cluster source|target]`** (enterprise) on the offset
  commands that connect to a cluster: bootstrap servers and the full
  `security:` block (TLS files, SASL credentials, GSSAPI) come from a
  backup/restore configuration instead of flags and `KAFKA_USERNAME` /
  `KAFKA_PASSWORD`. `--cluster` defaults to `target`; an explicit
  `--bootstrap-servers` still wins.
- **`enterprise.snapshot_interval`** (humantime, default `15m`): how often a
  continuous backup re-snapshots the control plane.
- `license info --format json`; both formats print the SHA-256 of the licence
  verification keys embedded in the binary.
- Unknown configuration keys are warned about (`Ignoring unknown config key
  …`), in the OSS part of the file and inside `enterprise:`.

### Changed
- **Control-plane backups are independent.** Schema Registry, Confluent RBAC,
  encryption metadata and Apicurio Registry each run on their own `enabled`
  flag and licence feature and report their own outcome (completed / skipped /
  failed). Previously a disabled or unlicensed Schema Registry returned before
  RBAC, encryption and Apicurio, and the first failing component stopped the
  rest.
  - Encryption follows `encryption.enabled`, not `schema_registry.enabled`. It
    still needs the `schema_registry` section for the Schema Registry
    connection (the DEK Registry is a Schema Registry sub-API).
  - An unlicensed or disabled component is a logged skip; a failed component
    fails the run after the others have run.
- **A failed Kafka data backup no longer skips the control plane.** The
  control-plane snapshot still runs, then the command fails with the Kafka
  error (and any control-plane failures).
- **Continuous backups snapshot the control plane periodically**: once at
  start, every `enterprise.snapshot_interval` alongside the data backup, and
  once more at shutdown. One-shot backups take a single pass after the data
  backup, as before. Each pass replaces the previous snapshot.
- `restore` applies the consumer group offset reset (Phase 3) after the data
  restore when `reset_consumer_offsets` / `auto_consumer_groups` asks for it,
  prints a Phase 3 summary and fails if the reset fails (kafka-backup #148).
  Erasure suppression is still attached before the engine is built.
- `validate-restore` prints the full report including the Phase 1 header
  preflight and **exits 1 when the configuration is invalid** (it exited 0).
  The text output keeps the `Validation: PASSED/FAILED` line and the erasure
  block; JSON keeps `enterprise.erasure_suppression`. `three-phase-restore`
  prints the header preflight.
- Licence status lines that ran on every invocation ("license loaded",
  "auto-trial active", "trial expired", "Keygen.sh license verified") are
  logged at debug instead of info, so `--format json` output stays parseable
  at the default log level when a licence or trial is present.
- Builds against kafka-backup-core **v0.23.8**, which also brings: restore
  circuit-breaker settings and no more global restore stalls (kafka-backup
  #197); explicit consumer group offsets translated through `topic_mapping`
  (#214); OffsetFetch / OffsetCommit routed to the group coordinator (#224);
  storage errors no longer read as "missing" and S3 reporting `NotFound`
  (#218); `file://` URL decoding (#221).
- Helm chart `version` / `appVersion` are 0.6.0 (they were stuck at 0.3.2),
  and the default image tag is `v<appVersion>`, matching the tags the release
  pushes (the bare `0.3.2` tag never existed). The release refuses a tag that
  does not match the chart version.

### Fixed (ported from the kafka-backup CLI)
- `--path` on `offset-rollback`, `offset-reset`, `offset-reset-bulk`,
  `show-offset-mapping` and `status` honours storage URLs (`s3://`,
  `azure://`, `gcs://`, `file://`); a malformed URL such as `s3:/bucket` or
  `memory://` fails instead of writing to a relative local directory (#174).
- `file://` paths are percent-decoded and must be local (#221).
- `validation evidence-list` / `evidence-get` resolve `--path` like every other
  command (#219).
- `prune` and `status` report an unreadable `offsets.db` / manifest instead of
  treating it as absent (#218).
- `offset-rollback rollback --verify false` and `offset-reset plan --dry-run
  false` are accepted; both flags could not be turned off (#220).
- `offset-rollback show` no longer panics on long topic names (#223), and
  `three-phase-restore` truncates its report by character rather than byte.
- `show-offset-mapping` builds the mapping from the manifest, like the OSS
  command.

### Build and packaging
- `auto-tag.yml` is removed. It listened for a workflow named "CI" that does
  not exist (the workflow is "Tests"), so it never ran; enterprise releases
  are tagged by hand (`git tag vX.Y.Z && git push origin vX.Y.Z`).
- `docker-compose.yml` uses the Chainguard MinIO and MinIO client images,
  pinned by digest (`minio/minio` and `minio/mc` were removed from Docker Hub),
  as kafka-backup did in OSS #228.

### Tests
- The kafka-backup v0.23.8 CLI black-box tests (#100, #167, #174, #218, #219,
  #220, #221, #223, #224) run against the enterprise binary.
- `tests/oss_parity.rs` reads the OSS CLI command tree (subcommands and long
  flags) from the kafka-backup checkout and fails when the enterprise CLI lacks
  any of it.
- Control-plane orchestration tests (wiremock Schema Registry and MDS), CLI
  black-box tests for `backup` with an unreachable Kafka, and licence-key
  fingerprint tests.

## [0.5.1] - 2026-10-13

Hotfix for 0.5.0. Still builds against kafka-backup-core **v0.22.0**; no
functional changes beyond the licence key fix below.

### Security
- **The 0.5.0 binary archives and the Homebrew formula could not verify
  commercial licences.** The release workflow ran
  `scripts/generate-keypair.sh --test` before `cargo build --release`; the
  script overwrote `keys/license_public.key` with a random key, so the
  released binaries embedded a key no commercial PEM licence is signed with.
  The 0.5.0 container image (`osodevops/kafka-backup-enterprise:v0.5.0`) was
  built without that step and is unaffected.
  **Upgrade to 0.5.1 if you run a 0.5.0 binary archive or Homebrew install
  with a commercial licence.** Trial licences and OSS mode were not affected.
  No workflow generates keys any more:
  - the core unit tests verify commercial signatures against a committed,
    test-only keypair (`keys/test_commercial_{private,public}.key`) that no
    non-test build embeds; `generate-keypair.sh --test` only regenerates that
    pair and never touches `license_public.key`;
  - `keys/license_public.key.sha256` pins the production key; a unit test and
    `scripts/verify-license-key.sh` fail when the key differs from it;
  - the release build checks the key before building and, after building,
    compares the binary's embedded key fingerprint (`license info --format
    json`) with the pin. The Docker jobs and the CI unit-test job run the same
    check.
- Commercial key rotation is tracked separately in issue #20 and is not part
  of this release.

### Added
- `license info --format json`; both formats print the SHA-256 of the licence
  verification keys embedded in the binary, so an installed binary can be
  checked against `keys/license_public.key.sha256`.

### Changed
- Licence status lines that ran on every invocation ("license loaded",
  "auto-trial active", "trial expired", "Keygen.sh license verified") are
  logged at debug instead of info, so `--format json` output stays parseable
  at the default log level when a licence or trial is present.
- Helm chart `version` / `appVersion` are 0.5.1 (they were stuck at 0.3.2),
  and the default image tag is `v<appVersion>`, matching the tags the release
  pushes (the bare `0.3.2` tag never existed). The release refuses a tag that
  does not match the chart version, and the release notes' `docker pull` line
  uses the `v`-prefixed tag.

## [0.5.0] - 2026-09-07

### Added
- `kafka-backup prune`: delete aged or oversized segments from a backup set,
  safely. Plan-only by default, `--execute` to delete; the manifest is rewritten
  before any object is deleted, pruned ranges are recorded, and a run that looks
  live is refused unless `--force`. Same command and flags as kafka-backup 0.21.0
  (`--older-than`, `--before`, `--keep-segments`, `--max-total-bytes`).
- `describe`, `validate` and `prune` accept `--config <backup.yaml>` (storage,
  prefix and backup id come from the configuration, `enterprise:` section
  included) or `--path` together with `--backup-id`. `--path` on `describe`,
  `validate`, `prune`, `status` and `list` accepts any storage URL
  (`s3://bucket/prefix`, `azure://...`, `gcs://...`) as well as a local
  directory. Previously `describe`, `validate`, `status` and `list` only read a
  local filesystem path.
- `describe` and `validate` report pruned ranges, recorded data gaps and topics
  skipped under `backup.on_missing_topic: warn`.

### Changed
- Builds against kafka-backup-core **v0.22.0** (previously v0.21.0): brings
  `backup.on_missing_topic`, the `access_key_id`/`secret_access_key` aliases and
  `allow_http` inference from an `http://` endpoint, `path_style`,
  `offset_storage.sync_interval_secs`, the deprecation warnings for
  `offset_storage.s3_key`, `backup.checkpoint_interval_secs` and
  `offset_storage.backend: memory`, and Azure Workload Identity `client_id` /
  `tenant_id` from YAML.

### Tests
- CLI end-to-end tests for `prune` and for `describe` / `validate --config`,
  ported from kafka-backup issues #168 and #169.

## [0.4.0] - 2026-09-06

### Added
- **GDPR erasure — restore-time key suppression** (licence feature `erasure`).
  `enterprise.erasure.suppression` (`keys_file`, `topics`, `match: exact|prefix|regex`,
  `on_match: tombstone|drop`) re-applies an erasure list while restoring, on the
  kafka-backup v0.21.0 `RecordFilter` hook. `restore`, `three-phase-restore` and
  `validate-restore` print `Records suppressed: N (… sha256=…)` / an
  `Erasure Suppression (enterprise)` block; `validate-restore --format json|yaml`
  gains an `enterprise.erasure_suppression` object when configured.
- `features::ERASURE` and `features::ALL`.
- `CHANGELOG.md`.

### Changed
- The 14-day auto-trial now unlocks **every** enterprise feature (previously only
  `migrations:msk-kraft:plan`); `license info` prints the actual feature list.
- `validate-restore` now receives the licence context.

### Security
- Licences signed with the **trial** key are only accepted when `tier` is `trial`
  and the validity (issue → expiry) is at most 14 days. Previously a trial-key
  signature was treated as equivalent to the commercial key.

### Behaviour
- If erasure suppression is configured but the feature is not licensed, or the keys
  file cannot be read/parsed, the restore-shaped command is **refused** before any
  engine is constructed. A silently skipped erasure list is never an outcome.

## [0.3.3] - 2026-08-31
- Build against kafka-backup-core v0.21.0 (kafka-protocol 0.18, Option-valued record headers).
