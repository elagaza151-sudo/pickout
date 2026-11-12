# PICK & OUT — Cloudflare R2 daily backup

Status: **prepared, not active until GitHub Actions secrets and R2 bucket are configured and a manual run succeeds.**

## Setup

1. Cloudflare Dashboard → R2 → create a **private** bucket (suggested name: `pickout-backups`). Do not enable public access.
2. R2 → Manage R2 API tokens → create a bucket-scoped **Object Read & Write** token. Never paste its credentials into chat or commit them.
3. Supabase Pickout project → Connect → copy **Direct connection** URI (preferred for pg_dump). GitHub-hosted runners may not have working IPv6; if direct connection fails, use **Session pooler, port 5432**, not transaction pooler. Include database password and use SSL.
4. GitHub repository Settings → Secrets and variables → Actions → add these **repository secrets**:
   - `PICKOUT_BACKUP_DATABASE_URL`: full PostgreSQL connection URI
   - `PICKOUT_R2_ACCOUNT_ID`: Cloudflare account ID
   - `PICKOUT_R2_BUCKET`: private bucket name
   - `PICKOUT_R2_ACCESS_KEY_ID`: R2 access key
   - `PICKOUT_R2_SECRET_ACCESS_KEY`: R2 secret access key
   - `PICKOUT_BACKUP_PASSPHRASE`: long random encryption passphrase, stored separately in a password manager
5. GitHub → Actions → **PICK & OUT encrypted daily database backup to R2** → Run workflow. Confirm green run and encrypted object plus `.sha256` in R2.
6. Cloudflare R2 bucket → Settings → Object lifecycle rules → set **Delete objects after 30 days** for prefix `pickout/postgres/`. Verify this rule in dashboard. This is not set automatically by the workflow.

## What is backed up?

A PostgreSQL custom-format logical dump of the database, including public POS, inventory, sales, shifts, staff and related tables, and other non-system schemas included by pg_dump. This is **not** a full Supabase project snapshot: Auth service configuration, Storage object bytes, external services, secrets and R2 settings are not included. Some managed Supabase objects may require special restore procedures.

## Schedule and protection

GitHub Actions cron: **02:15 UTC daily** (04:15 Egypt standard time); GitHub schedules may be delayed. The workflow validates the dump format, encrypts with AES-256-CBC/PBKDF2, uploads to private R2 and verifies remote object size. The sidecar SHA-256 can be used to check downloaded encrypted data. The workflow does **not** automatically restore anything.

## Recovery (manual, administrator only)

1. Select an R2 object and its matching `.sha256`, download privately using AWS CLI configured for R2.
2. Verify `sha256sum -c database.dump.enc.sha256`.
3. Decrypt with `openssl enc -d -aes-256-cbc -pbkdf2 -iter 200000 -in database.dump.enc -out database.dump -pass file:passphrase`.
4. Run `pg_restore --list database.dump` and **test restore to an isolated staging database** using PostgreSQL 17 tools. Review schema/roles/permissions and dependencies.
5. Only after written approval and a verified staging test, plan production restoration and downtime. **Never restore directly to live production without review.**

## Limitations

No backup is guaranteed until the workflow completes successfully and a restore test is performed. Keep the encryption passphrase separate from GitHub. A single backup may contain customer and staff personal data; limit R2 access. For stronger tamper protection, use a separate read-only recovery credential and consider R2 retention controls.
