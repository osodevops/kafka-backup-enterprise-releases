# Schema Registry restore

Enterprise restores registries before Kafka replay. Registry conflicts, missing
objects or dependencies, unknown record identifiers, verification failures and
settings recovery failures block replay. `restore --schema-only` performs the
registry phase without requiring a Kafka destination. `validate-restore
--schema-only --format json` inspects its plan without writes.
Operational logs use stderr; JSON/YAML command reports use stdout.

Confluent connections use `enterprise.schema_registry`. Apicurio connections
use `enterprise.apicurio_registry` and an explicit `api_version: v2` or `v3`.
Those connections identify the destination and supply its authentication/TLS
settings. The `schema_registry` entitlement is required whenever registry
restoration is configured. Confluent Cloud and MSK SEED/TAIL are outside this
release's registry restore scope.

## Configuration

ID preservation is the default. Legacy `rewrite_ids: true` selects allocation;
combining it with `force_ids: true` is rejected. Destructive `overwrite` is
rejected. Existing entries are reused only after checking their schema envelope
and references. A dry-run flag in any registry section makes the entire
invocation dry-run.

A Confluent-to-Apicurio example:

```yaml
enterprise:
  apicurio_registry:
    url: https://destination-registry.example
    api_version: v3
  registry_restore:
    source: confluent
    destination: apicurio
    id_policy: rewrite
    restore_global_settings: false
    coordinates:
      - source_group: ""
        source_name: orders-value
        target_group: orders
        target_name: orders-value
    serializers:
      orders:
        key:
          source: { type: raw }
          target: { type: raw }
        value:
          source: { type: confluent, namespace: "" }
          target:
            type: apicurio
            namespace: orders
            id_kind: global
            width: 4
```

Profiles use exact source topic names. Keys and values are independent. Every
selected replay topic needs a profile, including explicitly raw topics.
Confluent namespaces identify contexts; Apicurio namespaces identify groups.
Cross-provider restores require mappings for every selected artifact/subject.
References use the actual destination versions returned by registration.

Available framing profiles:

| `type` | Identifier | Framing |
| --- | --- | --- |
| `raw` | None | Unchanged bytes |
| `confluent` | Numeric, context scoped | Magic byte 0 and signed 32-bit ID |
| `confluent_guid_header` | GUID, context scoped | Named header containing version byte 1, 16 GUID bytes and Protobuf message indexes when applicable; requires CP 8.1+ |
| `apicurio` | `global` or `content`, group scoped | Magic byte 0 and signed 4/8-byte ID; `header` selects an 8-byte native ID header |
| `apicurio_compatible` | Explicit native `global`/`content` ID | Confluent-compatible framing using Apicurio identifiers |

Apicurio v2 and v3 SDK defaults differ. Select the actual header/prefix mode, ID
kind and width explicitly. No serializer is inferred from a magic byte. A
4-byte destination must fit a positive signed 32-bit integer.

For Protobuf, Confluent and compatible profiles require `indexes`. Qualified
native Apicurio profiles use `type_reference`, or v2 `bare` with its explicit
native type header. Native indexes-only and combined index/type-reference
profiles reject record preflight in this release because SDK consumption has
not been qualified. Frame conversion requires a
fully qualified `message`, such as `acme.Event`. Imports are compiled at exact
referenced versions. Source and destination message index paths are derived
separately from descriptors. Ambiguous identities fail validation.

Native Apicurio Protobuf serializers cannot faithfully resolve nested messages
in the tested SDKs. Those records reject preflight before registry/Kafka writes.
Nested messages use Confluent or Confluent-compatible framing, or registry-only
restoration. Multiple top-level messages are qualified with an explicit identity.

Confluent Protobuf GUID framing carries message indexes in the GUID header;
its record payload is bare Protobuf. Apicurio v2 native Protobuf headers carry
the message type separately and also use a bare payload. Configure that side
with `bare` and its explicit type header, for example:

```yaml
value:
  source:
    type: apicurio
    namespace: default
    id_kind: global
    width: 8
    header: apicurio.registry.globalId
  target: { type: confluent, namespace: "" }
  protobuf_source: bare
  protobuf_source_type_header: apicurio.protobuf.type
  protobuf_target: indexes
  message: acme.Event
```

Use `protobuf_target_type_header` when writing the corresponding destination
format. Header names must match the configured SDK serializer.

## Preservation and recovery

Confluent preservation uses subject-level IMPORT with explicit original IDs and
versions. A conflicting global ID/version, populated subject that needs an
unsupported IMPORT transition, or soft-deleted history that could be purged is
rejected. Targets without mode-override deletion (including CP 6.2.0) reject
IMPORT into subjects whose mode is inherited, because exact cleanup is
unavailable. Explicit IMPORT versions follow reference order, including valid
histories where version 2 must precede version 1. Allocation retains required
history edges and rejects histories it cannot reproduce faithfully. Global
forced IMPORT is never used. Temporary subject modes are
captured durably and restored, including after failure/cancellation. Unexpected
concurrent mode/configuration changes are not overwritten.

Apicurio preservation requires a complete unfiltered native archive, an empty
target, matching API generations and matching archive coordinates, IDs,
contents, references and states. A whole-registry ZIP is never used for a
filtered restore. An uncertain archive import is reconciled in full and never
blindly repeated. Original global/content IDs and resulting versions are
verified through the destination API.

Global destination settings are preserved by default. Source global settings
require `restore_global_settings: true` and an empty destination. Source global
IMPORT cannot be restored through a forced transition. Existing conflicting
subject/group/artifact policy is rejected before registration.

Registry progress lives beside `restore.checkpoint_state`, with separate files
per source/destination provider. Without that option, progress lives under
`.kafka-backup/registry-restore/`. Journals bind the operation to the snapshot,
configuration and target identity, record uncertain registrations and capture
settings recovery data. Reports include created/reused entries, complete ID and
version mappings, verification, cleanup and failures. Keep the journal when
resuming an interrupted operation.

Record preflight first validates source identifiers and configured frames.
After registry allocation and verification, it scans again with actual
destination mappings before any Kafka replay. Destination integer overflow,
unavailable identifiers and malformed configured frames block replay and are
saved in the structured failure report. A resumed operation reconciles registry
writes and uses the same transform fingerprint; completed Kafka checkpoints
prevent duplicate replay.

## Snapshot fidelity and limits

New manifests contain a version 2 `snapshot` integrity index. Required object
failures leave `_status.json` explicitly incomplete. Confluent snapshots retain
schema envelopes, GUIDs when supplied, metadata/rules, normalization and explicit
configuration overrides, exact reference versions and deleted versions.
Apicurio snapshots retain original global/content IDs, state, metadata, rules,
content SHA-256 values and independent structured/archive scope. Dependencies
outside filters are included recursively unless disabled; unresolved dependencies
are recorded and block restoration. Ordering operates on versions and retains
same-subject history edges.

Legacy snapshots remain inspectable. Restore requests are rejected when they
lack the metadata/settings/state information needed to make a faithful restore;
missing information is never synthesized. Cross-provider conversion supports
Avro binary, JSON Schema and Protobuf when their semantics and framing match.
Avro top-level `bytes` length encoding is translated. Unsupported custom
serializers, Avro JSON encoding, rules/states without a faithful target
representation and nonredundant metadata without a supported mapping fail
preflight. Registry-only OpenAPI/GraphQL snapshots use native registry APIs and
archives rather than Kafka serializer conversion.

New Confluent registrations containing `ruleSet` are unsupported in this
release and fail before target mutation. Existing entries may be reused only
after exact envelope verification. Server version and existing target rule
policies are insufficient qualification: the public images tested here omitted
submitted rules. Full rule-preservation compatibility remains unverified. Kafka
replay of rule-bearing identifiers rejects until a verified rule-executor/key
integration is available. This covers encoding, domain (including CSFLE ENCRYPT)
and migration rules, including entries reused exactly on the destination.

Apicurio native global-ID replay requires accessible, immutable versions and
accessible referenced versions. DISABLED, SUNSET and DRAFT versions, mutable
draft global IDs and unavailable references fail record preflight when the
selected framing needs them. Use immutable content-ID framing only when its
mapping and references remain resolvable, or perform a registry-only restore.
Unsupported cross-generation metadata and lifecycle states are rejected rather
than discarded. V2 may add a display name or description derived
from the schema where v3 had none; allocation accepts only these redundant
fields. Custom display metadata and archive preservation remain exact.

Shared source IDs that map to several Apicurio global-ID coordinates are
ambiguous. Select content-ID framing where the destination content mapping is
unique, or restrict the structured snapshot. No unmapped ID is silently retained.
Archived Kafka bytes are never edited; transformations happen after PITR/erasure
filtering and before partition selection and production. The transform's stable
fingerprint participates in checkpoint compatibility.

## Verification

Run `scripts/test-registry-restore.sh` against its isolated Compose fixture. It
captures resolved image digests, exact SDK dependency graphs and test logs under
`tmp/registry-compatibility/`, and removes only its own Compose project.
`REGISTRY_CONFLUENT_{SOURCE,TARGET}_VERSION` and
`REGISTRY_APICURIO_{SOURCE,TARGET}_VERSION` select exact versions.

The PR workflow runs CP 7.6.0/8.3.1 with Apicurio 2.6.13.Final/3.3.3. The nightly
and manual full workflow enumerates all 81 provider/version pairs: CP 6.2.0,
7.6.0, 7.9.2, 8.1.1, 8.3.1 and Apicurio 2.6.13.Final, 3.0.15, 3.1.7, 3.3.3.
Generation/GUID preservation exclusions are explicit rejection assertions.

The oracle is released Java SDK producers and consumers. Successful Kafka cases
consume with only the destination registry and check Avro (including top-level
bytes), JSON Schema, Protobuf, tombstones, raw keys and duplicate/null headers,
with normal and repartitioned replay and gzip/Snappy source compression. HTTP
mock tests establish failure contracts; they are not compatibility evidence.
See [the recorded compatibility evidence](schema-registry-compatibility.md).
