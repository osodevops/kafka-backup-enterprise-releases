# Changelog

All notable changes to kafka-backup-enterprise are documented here.

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
