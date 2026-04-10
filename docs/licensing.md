# Licensing

## How It Works

The enterprise binary has three modes of operation:

### 1. Auto-Trial (14 days, no signup)

On first run without a license, all enterprise features are enabled automatically for 14 days. No email, no form, no registration. Just install and use it.

```
$ kafka-backup-enterprise license info

  Status:       Enterprise Auto-Trial
  Days left:    14
  Features:     All enterprise features enabled
```

### 2. Licensed (30-day trial or annual)

Apply a license file to unlock enterprise features beyond the auto-trial:

```bash
kafka-backup-enterprise license apply --file license.lic
```

License files are cryptographically signed and verified **offline** — no internet required, works on air-gapped servers.

```
$ kafka-backup-enterprise license info

  License ID:   abc-123
  Customer:     Your Company (ops@company.com)
  Tier:         Enterprise
  Features:     encryption, schema_registry, rbac, audit, masking, wasm
  Expires:      2027-04-10 (365 days remaining)
  Status:       Valid
```

### 3. OSS Mode (no license, trial expired)

Without a license and after the trial expires, the binary continues to work as the full open-source edition. All OSS features (topic backup, restore, offset management) work indefinitely.

## Getting a License

- **30-day trial:** [kafkabackup.com/enterprise](https://kafkabackup.com/enterprise)
- **Annual license:** Contact [sales@osodevops.io](mailto:sales@osodevops.io)

## Applying a License

### CLI

```bash
kafka-backup-enterprise license apply --file license.lic
```

The license is saved to `~/.config/kafka-backup/license.lic` and loaded automatically on every subsequent run.

### Docker (environment variable)

```bash
docker run --rm \
  -e ENTERPRISE_LICENSE_KEY="$(base64 < license.lic)" \
  osodevops/kafka-backup-enterprise:v0.2.1 \
  backup --config /config.yaml
```

### Docker (mounted file)

```bash
docker run --rm \
  -v ./license.lic:/etc/kafka-backup/license.key \
  osodevops/kafka-backup-enterprise:v0.2.1 \
  backup --config /config.yaml
```

### Kubernetes (secret)

```bash
kubectl create secret generic kafka-backup-license \
  --from-literal=license-b64="$(base64 < license.lic)"
```

```yaml
env:
  - name: ENTERPRISE_LICENSE_KEY
    valueFrom:
      secretKeyRef:
        name: kafka-backup-license
        key: license-b64
```

## License Discovery Order

On startup, the binary checks these locations:

1. `ENTERPRISE_LICENSE_KEY` environment variable (base64-encoded content)
2. `ENTERPRISE_LICENSE_FILE` environment variable (file path)
3. `/etc/kafka-backup/license.key` (default container/K8s path)
4. `~/.config/kafka-backup/license.lic` (user config, Keygen.sh format)
5. `~/.config/kafka-backup/license.key` (user config, PEM format)
6. If none found: auto-trial (14 days)

## Offline Verification

License files are verified using Ed25519 cryptographic signatures. **No network calls are made during verification.** The public key is embedded in the binary at compile time.

This means:
- Licenses work on air-gapped servers
- No "phone home" or license server required
- No DNS lookups or HTTPS calls during startup
- The license file is completely self-contained

## Verifying a License

To verify a license without applying it:

```bash
kafka-backup-enterprise license verify --file license.lic
```

## Disabling the Auto-Trial

To disable the auto-trial (e.g., in CI where you want pure OSS mode):

```bash
KAFKA_BACKUP_NO_TRIAL=1 kafka-backup-enterprise backup --config backup.yaml
```
