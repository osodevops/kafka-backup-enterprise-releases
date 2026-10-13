# Recorded Schema Registry compatibility evidence

Enterprise 0.6.1 passed all **81 ordered provider/version pairs** and
**1297 real-registry/independent-SDK contracts** at commit
`ed01b0031f36bc825872b3806054ac4d5af2cb00`. Required workspace, security-audit and Docker CI
passed at the same commit. The release gate verifies the source SHA-256 manifest
against [the complete qualification report](schema-registry-release-qualification.json).

The report records exact image digests, all four pinned SDK dependency graphs,
contract hashes and expected rejections for every pair. Versions qualified are
Confluent 6.2.0, 7.6.0, 7.9.2, 8.1.1 and 8.3.1, and Apicurio
2.6.13.Final, 3.0.15, 3.1.7 and 3.3.3. Unsupported preservation and framing
combinations pass explicit rejection assertions; green pairs do not imply
those combinations are supported. The supported scope and limits below remain
part of the release contract.

## Earlier local qualification

Ten local runner invocations are retained in
[the historical evidence](schema-registry-compatibility-evidence.json). Some used
narrower fixtures than the final matrix. The complete release report above is
the authoritative evidence for the released source.

| Registry pair | Native preservation | SDK Kafka replay, both directions | Additional executed scope |
| --- | --- | --- | --- |
| Confluent 7.6.0 ↔ Apicurio 2.6.13.Final | Passed | Avro/bytes, JSON Schema, Protobuf | Expanded framing, ccompat, sparse IDs/shared content, checkpoint resume |
| Confluent 8.3.1 ↔ Apicurio 3.3.3 | Passed | Avro/bytes, JSON Schema, Protobuf | Expanded framing, ccompat, sparse IDs/shared content, checkpoint resume |
| Confluent 7.6.0 ↔ Apicurio 3.3.3 | Passed | Avro/bytes, JSON Schema, Protobuf | Expanded framing |
| Confluent 8.3.1 ↔ Apicurio 2.6.13.Final | Passed | Avro/bytes, JSON Schema, Protobuf | Expanded framing |
| Confluent 7.6.0 ↔ Apicurio 3.0.15 | Passed | Avro/bytes, JSON Schema, Protobuf | Numeric-prefix replay |
| Confluent 8.3.1 ↔ Apicurio 3.1.7 | Passed | Avro/bytes, JSON Schema, Protobuf | Numeric-prefix replay |
| Confluent 6.2.0 ↔ Apicurio 3.3.3 | Confluent expected rejection; Apicurio passed | Avro/bytes, JSON Schema, Protobuf | Inherited-mode IMPORT cleanup rejection |
| Confluent 7.9.2 ↔ Apicurio 3.3.3 | Passed | Avro/bytes, JSON Schema, Protobuf | Same-provider Confluent allocation |
| Confluent 8.1.1 ↔ Apicurio 3.3.3 | Passed | Avro/bytes, JSON Schema, Protobuf | Same-provider Confluent allocation |
| Apicurio 2.6.13.Final → 3.3.3, with Confluent 8.3.1 | Cross-generation Apicurio expected rejection | Cross-provider replay also passed | Same-provider Apicurio allocation, expanded framing and ccompat |

All recorded SDK replay runs include normal and repartitioned replay with
gzip/Snappy source compression. Expanded framing adds native ID headers,
content-ID headers, 8-byte prefixes and independently serialized keys. Confluent
GUID headers are used only with SDK/server versions supporting them. ccompat
contracts are explicitly named in the JSON evidence; they have not run for
every row. Earlier runs used narrower fixtures than the two final runs.

The final CP 7.6.0/Apicurio 2.6.13.Final and CP 8.3.1/Apicurio 3.3.3 fixtures
also verify native OpenAPI/GraphQL, metadata, disabled versions, empty groups
and supported empty artifacts, sparse IDs and shared content. Repeating registry
execution creates no entries; repeating Kafka replay from its checkpoint
produces no duplicate records and retains the transform fingerprint.

Released Java SDK producers and consumers serve as the independent oracle.
Each replay checks three decoded values and a tombstone, raw or serialized
keys, ordered duplicate headers and a null header. The final fixtures also
check null keys, and transform assertions check timestamps and offsets. Consumers use
only the destination registry. Maven classpaths are isolated by provider and
source/destination side, with serializer versions matching the selected image
tags. Snappy Java is pinned to 1.1.10.7. The evidence includes all four exact
dependency graphs as normalized artifact/version lists with SHA-256 hashes,
contract log hashes, image digests and architecture. These local image runs
used ARM64; CI records its own resolved images and architecture.

## Explicit limits and rejections

- CP 6.2.0 lacks mode-override deletion. Preservation into subjects with inherited
  modes rejects before mutation because exact IMPORT cleanup is unavailable.
- Native Apicurio ID-preserving archive import rejects across API generations;
  the executed v2-to-v3 rewrite case allocates destination IDs instead.
- GUID preservation and GUID destinations require CP 8.1+. Unsupported version
  combinations are rejection contracts in the matrix runner.
- The public Confluent images retained metadata but omitted submitted rule sets.
  Rule-preservation compatibility is unverified. New rule-bearing registrations
  are explicitly unsupported and reject before writes; exact verified reuse is
  permitted for registry-only restoration. Replay of encoding, domain
  (including CSFLE ENCRYPT) and migration rules rejects
  pending a verified executor/key integration.
- Unsupported metadata, states, settings, serializers and ambiguous mappings
  reject preflight. Legacy snapshots remain inspectable but cannot restore
  information absent from their manifests.
- Native Apicurio nested Protobuf and unqualified indexes-only/combined native
  framing reject before replay. The expanded oracle qualifies multiple top-level
  messages on native paths and nested message indexes on Confluent/compatible
  paths; an explicit native nested rejection contract exercises real records.

Eleven HTTP safety contracts also passed, covering unknown identifiers, missing
dependencies, collisions, populated IMPORT, legacy fidelity, generation
metadata, rule capability, cancellation after a committed write, cleanup/resume
and concurrent configuration changes, including rule-bearing exact-reuse replay
rejection. These establish failure behavior; real
registry and SDK contracts supply compatibility evidence.

## Resolved images

Only exact version tags were selected by the runner. The JSON may contain other
local tags attached to the same digest; those aliases were not version inputs.

| Exact image tag | Resolved digest |
| --- | --- |
| apache/kafka:3.7.1 | `sha256:ed74d7d115968d5e8b00ba6822ac6a384cbaaf54ca38991828647000d7089b68` |
| confluentinc/cp-schema-registry:6.2.0 | `sha256:29a044bffee7e2267923464221f13e5c54667bdb742ec7dc46772eb59a0b1f14` |
| confluentinc/cp-schema-registry:7.6.0 | `sha256:d23d8440708bb44daad5cba4195c070b9713c126375b355afb5ffedf7f1b94e7` |
| confluentinc/cp-schema-registry:7.9.2 | `sha256:d92bd8a12290d131f77eca1daeb524d835728b597b698617a496904f40b16fc9` |
| confluentinc/cp-schema-registry:8.1.1 | `sha256:ba3353d723c3f0b5c5d84d63b5dfd864e2b6b271dd34d569aae7f6e7beaf84c7` |
| confluentinc/cp-schema-registry:8.3.1 | `sha256:f0cfd047a839c1ace54d93b92e3459f0d03dc3b5c9db1192a2246fd79b4f44c4` |
| quay.io/apicurio/apicurio-registry-mem:2.6.13.Final | `sha256:9821c5a9e57495647515303dde3f8670d9afe79150f848fd513383a60a2a934e` |
| quay.io/apicurio/apicurio-registry:3.0.15 | `sha256:81bac63f05c2818a9ffa609bdf5ef947079c15e5d2d7798c6e8215411b95cbb6` |
| quay.io/apicurio/apicurio-registry:3.1.7 | `sha256:3ff121c9f744d535ef770b80ff95693bc95063295316a5864b56312b6edfb4e2` |
| quay.io/apicurio/apicurio-registry:3.3.3 | `sha256:c9cae4c90ce46538abf673c68eb591345f23bcc6c2aa6bfd47189adcf609d8f1` |

The PR workflow runs the four CP 7.6.0/8.3.1 and Apicurio
2.6.13.Final/3.3.3 combinations. Nightly, qualification-branch and full manual
workflows enumerate all 81 ordered provider/version pairs. The full release
matrix passed; its results apply to the executed fixtures and explicit limits
above. They do not establish compatibility for unqualified custom serializers,
rule executors or other excluded semantic combinations.

## Published artifacts

The [artifact verification report](schema-registry-release-artifacts.json) records
all five binary checksums, execution and licence-key checks, the public Docker
digest, OCI chart verification, the public Helm index and Homebrew checksums.
Linux standalone binaries require glibc 2.39+; the Docker image supplies its own
runtime. The public Helm chart 0.6.2 and OCI chart 0.6.1 both install application
0.6.1. Chart 0.6.2 includes the source licence smoke-test hook.
