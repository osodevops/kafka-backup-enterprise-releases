# Changelog

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
