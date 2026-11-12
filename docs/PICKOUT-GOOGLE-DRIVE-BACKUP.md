# PICK & OUT — Google Drive backup setup

**Status: code prepared, NOT active until secrets are configured and the first run passes.** R2 schedule has been disabled.

## Requirements
- Private Google Drive account with available quota.
- GitHub Actions repository secrets (never put credentials in code or chat).
- `rclone` on your local computer to authorize Google Drive.

## Configure
1. Install rclone locally: https://rclone.org/downloads/
2. Run `rclone config`; create a remote named exactly **pickoutdrive**, select Google Drive, and complete browser OAuth login. Use your own OAuth client if Google's default app quota or consent policies require it. Ensure this remote can create files in your personal Drive. Keep `rclone.conf` private.
3. Open the local rclone config path shown by `rclone config file`. Encode the **entire config file** as base64, e.g. PowerShell: `[Convert]::ToBase64String([IO.File]::ReadAllBytes("$env:APPDATA\\rclone\\rclone.conf"))`. Save the result as secret `PICKOUT_DRIVE_RCLONE_CONFIG_B64`. This contains OAuth credentials; do not share it.
4. Supabase Pickout → Connect → copy a PostgreSQL URI suitable for `pg_dump`. Prefer direct connection if IPv6 works, otherwise session pooler port 5432. Put URI (including password and SSL) into secret `PICKOUT_BACKUP_DATABASE_URL`.
5. Generate a strong random passphrase, keep an offline copy in a password manager, and set secret `PICKOUT_BACKUP_PASSPHRASE`.
6. GitHub → Settings → Secrets and variables → Actions → New repository secret. Add all three secrets.
7. GitHub → Actions → **PICK & OUT encrypted daily backup to Google Drive** → Run workflow. Verify successful run and two files in the private `PICKOUT-Backups` Drive folder.
8. Run a separate restore test to a staging database before relying on backups.

## Coverage and retention
Daily 02:15 UTC. `pg_dump` includes database tables (orders, POS, inventory, shifts, staff, etc.) but **not** Supabase Storage object contents, Auth provider settings, or external secrets. Backups are AES-256-CBC/PBKDF2 encrypted and SHA-256 sidecars are uploaded. Files older than 30 days matching `pickout-*.dump.enc*` in the dedicated folder are deleted after successful upload. Drive quota and GitHub Actions usage limits apply. Scheduled GitHub workflows can be delayed or disabled after prolonged inactivity.

## Restore
Download the matching encrypted dump and `.sha256`. Verify with `sha256sum -c <filename>.sha256`. Decrypt with `openssl enc -d -aes-256-cbc -pbkdf2 -iter 200000 -in <filename>.dump.enc -out database.dump -pass file:passphrase`. Inspect `pg_restore --list database.dump` and **restore only to a separate staging PostgreSQL database first**. A production restore requires administrator approval and a plan; the workflow never automatically restores or modifies production.

**Security:** Drive OAuth tokens are powerful; restrict GitHub repository Actions secrets access, enable 2FA and protect the GitHub repository. Do not store unencrypted database dumps on Drive.
