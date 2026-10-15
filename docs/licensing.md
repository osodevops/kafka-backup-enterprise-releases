# Enterprise licensing in 0.8.0

The binary includes a fourteen-day evaluation of shipped Enterprise features with no signup, card or automatic charge. See [evaluation and persistence](evaluation.md) before running containers or scheduled jobs. `license info` is read-only; `license activate-trial` or the first operational command in the Enterprise binary starts the evaluation.

Commercial licences are purchased through [enterprise.kafkabackup.com](https://enterprise.kafkabackup.com). Runtime validation is offline using an embedded Ed25519 public key. Keep the licence access controlled; the commercial signing private key is not distributed with the software.

```bash
kafka-backup license verify --file license.key
kafka-backup license apply --file license.key
kafka-backup license info --format json
```

Homebrew uses the executable name `kafka-backup-enterprise`.

Discovery order is `ENTERPRISE_LICENSE_KEY` (base64 licence content), `ENTERPRISE_LICENSE_FILE`, `/etc/kafka-backup/license.key`, then `~/.config/kafka-backup/license.key`. An explicitly configured missing or invalid licence is authoritative and does not fall back to evaluation. Unset an inline override before applying a replacement file.

The `rbac` entitlement covers MDS capture and authorization recovery. Other shipped features use their documented entitlements, including `schema_registry`, `oauth`, `erasure`, `encryption` for CSFLE metadata, and migration entitlements. Licence flags for planned features do not make those features available.

At expiry, requested Enterprise operations fail. Authorization operations refuse with exit code 2 before transfer or permission mutation. Already created permissions remain in place; no automatic rollback occurs. OSS operations and licence inspection remain available.

Enterprise 0.8.0 rotates commercial signing trust. Existing licence files require reissuance for the new binary without changing their original expiry or entitlements. New files require 0.8.0 or later. The built-in evaluation needs no signed file.

The [public licensing reference](https://kafkabackup.com/enterprise/licensing/) is the maintained source for entitlement and upgrade details.
