# Authorization recovery in Enterprise 0.8.0

Use the [public rehearsal guide](https://kafkabackup.com/enterprise/authorization-recovery/) and [Platform configuration example](../examples/authorization-platform.yaml).

Recovery is additive: create missing native assignments, recognize exact existing assignments and preserve target-only access. It does not create identities, recover credentials, delete permissions or automatically roll changes back.

1. Capture a fresh, complete authorization snapshot. Legacy unversioned MDS exports cannot establish completeness.
2. Restore and independently verify data, schemas and requested consumer offsets with externally provisioned administration access.
3. Map every selected principal, native scope and resource explicitly. Include the matching native cluster scope for every Kafka ACL scope, even for ACL-only recovery.
4. Review the plan's create/existing/target-only/blocked decisions and checksum. Administrator bindings require explicit selection and `allow_privileged_bindings`.
5. Apply the approved plan, resume it after a failure if needed, then re-enumerate assignments and independently probe allowed and denied operations.
6. Repeat apply and require zero additional writes. Verify target-only sentinels and retain redacted evidence.

Planning and verification perform no permission writes. Apply journals intent before writes, reconciles uncertain outcomes and stops on failure with completed changes retained. Keep its storage durable and serialize recovery across hosts externally.

Platform qualification covers CP 7.6.0 and 8.3.1, including secured Schema Registry, Connect and ksqlDB. Cloud qualification covers service-account principals in two environments within one organisation. Cloud user/group/identity-pool and wildcard principals, cross-organisation recovery, operator CRDs and complete recovery sets are excluded. Kafka ACL writes require SASL/PLAIN over SASL_SSL or SASL_PLAINTEXT.

Assignment verification does not prove universal effective-access equivalence: external identity policies and target-only permissions can broaden access. Bootstrap administration remains externally provisioned.
