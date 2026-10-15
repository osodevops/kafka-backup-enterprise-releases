# Fourteen-day Enterprise evaluation

The built-in evaluation needs no signup, card or network connection and does not automatically charge. Explicit `license activate-trial` or the first Enterprise operation starts exactly fourteen days. `license info`, help, version and licence-file verification do not start it.

Persist `/var/lib/kafka-backup` in Docker, or set `KAFKA_BACKUP_TRIAL_STATE_DIR` to durable writable storage. Retain the same directory or Helm trial PVC across restarts, upgrades and reinstalls. Do not replace evaluation state to restart a trial.

At expiry, requested Enterprise operations are refused and OSS functionality remains available. Existing permissions remain in place. Invalid explicitly supplied licences do not fall back to a trial. Offline integrity checks cannot prevent an administrator controlling the filesystem and clock from resetting local state.

Enterprise 0.8.0 rotates the commercial signing key. Re-download an existing licence through the portal and verify it with the new binary before upgrading. Reissuance preserves the original expiry and entitlements. New files require 0.8.0 or later; preserve matching old files if continuing to use an older binary. The built-in evaluation needs no signed licence file.

See the [evaluation guide](https://kafkabackup.com/enterprise/trial-evaluation/), [licensing guide](https://kafkabackup.com/enterprise/licensing/) and [commercial portal](https://enterprise.kafkabackup.com).
