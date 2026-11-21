# Twenty CRM on Railway (pinned, worker + bucket)

Self-host [Twenty](https://github.com/twentyhq/twenty), the open-source CRM
(a Salesforce / HubSpot alternative), on Railway in one click.

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/REPLACE_WITH_TEMPLATE_CODE)

> **Unofficial community template.** It is not made, endorsed or supported by
> Twenty.com PBC. "Twenty" is used only to describe the software this template
> deploys. For problems with Twenty itself, use the
> [upstream repository](https://github.com/twentyhq/twenty).

**Current version:** Twenty **v2.60.1**, pinned by digest
`twentycrm/twenty@sha256:b190e54ca434ea1ee47fe73b8d76e77d0bf3c6f734fcec58e414108d6867939c`
(Updated Nov 2026).

## What you get

| Service | Image | Public | Storage |
|---|---|---|---|
| **Twenty** (web app + API) | `twentycrm/twenty:v2.60.1` (digest-pinned) | Yes, HTTPS domain, healthcheck `/healthz` | none |
| **Twenty Worker** (background jobs: imports, workflows, emails, sync, cron) | same image, `yarn worker:prod` | No | none |
| **Postgres** 18.6 | `ghcr.io/railwayapp-templates/postgres-ssl:18.6` (pinned) | No | volume |
| **Redis** 8.2.11 | `redis:8.2.11` (pinned) | No | volume |
| **Bucket** | Railway Storage Bucket (S3-compatible) | No (private) | attachments, avatars, files |

## Why this template

- **Pinned, never `:latest`.** Every image is pinned to an exact version (and digest
  where Railway accepts it). A redeploy never silently pulls an untested Twenty release.
- **Upgrade-tested.** Each version bump is deployed on top of the previous one with
  real data before the template is updated. The title shows `[Updated <month>]`.
- **Real worker.** Without the worker, imports, workflows, email and calendar sync,
  and scheduled jobs do not run. The worker reads every shared setting from the
  server service, so the two can't drift apart.
- **Files in a Railway bucket, not a local disk.** Railway volumes attach to one
  service only, so a local-disk setup breaks when the worker needs files. The bucket
  is shared by both and survives redeploys.
- **Secure defaults.** Random secrets (`ENCRYPTION_KEY`, `APP_SECRET`, database and
  Redis passwords), single-workspace mode (sign-up closes after the first workspace),
  workspace creation limited to server admins, only the web app is public,
  telemetry off.
- **Lean memory.** Node heap caps on server and worker. Cost estimate below.

## After deploy (first 5 minutes)

1. Wait until the **Twenty** service is healthy. The first boot creates the database
   and can take 3 to 5 minutes. The worker may restart a few times during this. That's normal.
2. **Open the Twenty URL right away** and create your account and workspace. The
   first person to finish setup becomes the admin. Don't share the URL before you do this.
3. Invite teammates from **Settings → Members** (copy the invite link if email is not set up).
4. Back up your `ENCRYPTION_KEY` (Twenty service → Variables) to a password manager.
   Without it, stored OAuth tokens and 2FA secrets can't be decrypted.

## Configuration

All settings live on the **Twenty** service. The worker references them
(`${{Twenty.VARIABLE}}`), so change them in one place and redeploy both services.

| Variable | Default in template | Notes |
|---|---|---|
| `SERVER_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | Must match the URL in the browser exactly. Change it when you add a custom domain. |
| `PORT` / `NODE_PORT` | `3000` | Twenty listens on `NODE_PORT`. |
| `PG_DATABASE_URL` | `${{Postgres.DATABASE_URL}}` | Private network. |
| `REDIS_URL` | `${{Redis.REDIS_URL}}` | Private network. Redis must keep `maxmemory-policy noeviction`. |
| `ENCRYPTION_KEY` | random (`${{secret()}}`) | Encrypts secrets in the DB. **Never change it** without the upstream key-rotation guide. |
| `APP_SECRET` | random | Legacy secret, kept for compatibility. |
| `STORAGE_TYPE` | `s3` | Files go to the Railway bucket. |
| `STORAGE_S3_NAME`, `STORAGE_S3_REGION`, `STORAGE_S3_ENDPOINT`, `STORAGE_S3_ACCESS_KEY_ID`, `STORAGE_S3_SECRET_ACCESS_KEY` | `${{Bucket.BUCKET}}`, `${{Bucket.REGION}}`, `${{Bucket.ENDPOINT}}`, `${{Bucket.ACCESS_KEY_ID}}`, `${{Bucket.SECRET_ACCESS_KEY}}` | From the Railway bucket. |
| `IS_MULTIWORKSPACE_ENABLED` | `false` | Single workspace. Sign-up closes after the first workspace. |
| `IS_WORKSPACE_CREATION_LIMITED_TO_SERVER_ADMINS` | `true` | Only server admins can create workspaces. |
| `EMAIL_DRIVER` | `logger` | Emails are written to the logs. See [Email](#email). |
| `EMAIL_FROM_ADDRESS` / `EMAIL_FROM_NAME` | `noreply@example.com` / `Twenty CRM` | Change when you enable email. |
| `TELEMETRY_ENABLED` | `false` | Set `true` to send anonymous usage data to Twenty. |
| `NODE_OPTIONS` | server `--max-old-space-size=1536`, worker `1024` | Heap cap. Raise if you see "heap out of memory". |
| `DISABLE_DB_MIGRATIONS` / `DISABLE_CRON_JOBS_REGISTRATION` | server `false`, worker `true` | Only the server migrates. Don't change. |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `60` (server) | Lets in-flight requests finish before an old deploy stops. |

Many other settings (Google/Microsoft sign-in, Gmail/Outlook sync, AI keys, SMTP)
can be set in Twenty's **Admin Panel → Config variables**. They're stored in the
database and used by both server and worker.

### Custom domain

1. Twenty service → Settings → Networking → add your domain and the DNS record.
2. Twenty service → Variables → set `SERVER_URL=https://crm.yourdomain.com`.
3. Redeploy **Twenty** and **Twenty Worker**.

## Email

Twenty v2.60.1 has only two email drivers: `logger` (nothing is sent; emails appear in
the logs) and `smtp`. It has **no HTTP-API mail driver**.

Railway **blocks outbound SMTP on Free, Trial and Hobby plans**. So:

- **Hobby:** keep `EMAIL_DRIVER=logger`. Login with email and password works without
  email. Invite teammates with the **copy invite link** button. Password-reset emails
  will not arrive. A workspace admin can help, or you can read the link from the
  worker or server logs.
- **Pro:** use an SMTP relay from a transactional email provider (Resend, Postmark,
  Brevo, Amazon SES, Mailgun). Example for Resend:

  ```env
  EMAIL_DRIVER=smtp
  EMAIL_SMTP_HOST=smtp.resend.com
  EMAIL_SMTP_PORT=465
  EMAIL_SMTP_USER=resend
  EMAIL_SMTP_PASSWORD=<your Resend API key>
  EMAIL_FROM_ADDRESS=crm@yourdomain.com
  EMAIL_FROM_NAME=Your Company CRM
  ```

  Set these on the Twenty service, then add `EMAIL_SMTP_*` references on the worker
  (`EMAIL_SMTP_HOST=${{Twenty.EMAIL_SMTP_HOST}}`, ...) or set them in the Admin Panel.
  Then redeploy both.

Gmail/Outlook **mailbox sync** uses Google/Microsoft APIs over HTTPS, not SMTP, so
it works on every plan once you add the OAuth credentials.

## Estimated monthly cost

Railway bills real usage: RAM $10/GB-month, CPU $20/vCPU-month, volumes
$0.15/GB-month, egress $0.05/GB, buckets $0.015/GB-month. Typical idle footprint
for a small team (estimate, to be confirmed by our test deploy):

| Service | RAM (typical) | Approx. cost/month |
|---|---|---|
| Twenty (server) | 0.6 to 0.9 GB | $6 to $9 |
| Twenty Worker | 0.3 to 0.5 GB | $3 to $5 |
| Postgres | 0.1 to 0.25 GB | $1 to $2.5 |
| Redis | ~0.02 GB | ~$0.2 |
| CPU (all, mostly idle) | 0.05 to 0.15 vCPU | $1 to $3 |
| Volumes + bucket + egress (small team) | 1 to 5 GB | < $1 |
| **Total** | **~1.0 to 1.7 GB** | **~$12 to $20** |

On the Hobby plan ($5/month, includes $5 of usage) that is about **$12 to $20 total
per month**. Heavy imports, many users or mailbox sync raise CPU and RAM. You can set
memory limits per service in Settings → Resource limits (template defaults: server
2 GB, worker 1.5 GB, Postgres 1 GB, Redis 256 MB). Limits are caps, not reservations:
you pay only for what's used.

## Upgrading Twenty

Twenty releases often (sometimes several times a week). This template bumps the pin
about once a month, after testing. To upgrade your own deploy:

1. **Back up first** (see below). Database migrations are one-way. There is no downgrade.
2. Check the [release notes](https://github.com/twentyhq/twenty/releases) and the
   [upgrade guide](https://docs.twenty.com/developers/self-hosting/upgrade-guide)
   for breaking changes. For example, v2.34+ needs PostgreSQL 15 or newer, and this
   template ships 18.
3. Change the image on **both** `Twenty` and `Twenty Worker` to the same new version,
   e.g. `twentycrm/twenty:v2.61.0` (or its digest). Never run different versions on
   server and worker.
4. Deploy **Twenty** first. On start it runs `upgrade` migrations automatically. Wait
   until it's healthy, then deploy **Twenty Worker**.
5. Check the server logs for `Successfully migrated DB!`. If you see "Upgrade completed
   with errors", restart the server once to retry. If it still fails, restore your
   backup and stay on the old version.

Upstream supports jumping straight from v1.23+ to the latest version. You don't need
to step through every release.

## Backup and restore

**Database** (most important):

```bash
# Railway CLI, linked to your project. Uses the Postgres service's variables.
railway run --service Postgres -- sh -c 'pg_dump "$DATABASE_PUBLIC_URL" -Fc -f twenty-$(date +%F).dump'
# Restore into an EMPTY database:
railway run --service Postgres -- sh -c 'pg_restore --no-owner -d "$DATABASE_PUBLIC_URL" twenty-YYYY-MM-DD.dump'
```

`DATABASE_PUBLIC_URL` needs Postgres public networking (TCP proxy) turned on. Turn it
off again afterwards. You need a local `pg_dump`/`pg_restore` version 18. If your plan
has Railway volume backups, schedule them on the Postgres volume too.

**Files** (bucket): Railway buckets have **no automatic backups**. Copy them with any S3 tool,
using the bucket's Credentials tab:

```bash
AWS_ACCESS_KEY_ID=... AWS_SECRET_ACCESS_KEY=... \
aws s3 sync s3://<BUCKET> ./bucket-backup --endpoint-url <ENDPOINT>
```

**Secrets:** save `ENCRYPTION_KEY` and `APP_SECRET`. A restored database without them
can't decrypt stored credentials.

## Troubleshooting

### First deploy was interrupted: `relation "core.…" does not exist`
The first boot creates the database schema (2–4 minutes). If it is interrupted (for example you click Redeploy while it is still running), the schema is left half-created and later boots fail with errors like `relation "core.keyValuePair" does not exist`. On a fresh install with no data, fix it by deleting the `Postgres` service volume (or the whole Postgres service and adding it again with the same name) and deploying `Twenty` once, then `Twenty Worker`. Do not redeploy during the first boot.

| Symptom | Fix |
|---|---|
| Login loops, or "Unable to reach backend" | `SERVER_URL` doesn't match the browser URL. Fix it, then redeploy both services. |
| Server never becomes healthy on first deploy | First boot can take several minutes. Check logs for DB init. The healthcheck timeout is 600 s. |
| `JavaScript heap out of memory` | Raise `NODE_OPTIONS=--max-old-space-size=...` and the memory limit on that service. |
| Uploads/attachments fail | Check the `STORAGE_S3_*` references and the bucket in the server logs. |
| Imports/workflows never finish | The worker isn't running, or uses different settings from the server. Check that its variables reference `Twenty`. |
| `ENOTFOUND redis.railway.internal` | The environment is old and IPv6-only. Append `?family=0` to `REDIS_URL`, or create a new environment. |
| Emails not delivered | Expected on Hobby (`logger` driver). See [Email](#email). |

## Local testing

`docker-compose.yml` mirrors the Railway setup (server, worker, Postgres, Redis, and
SeaweedFS as a local S3 bucket):

```bash
cp .env.example .env   # fill in the secrets as the comments show
docker compose up -d
open http://localhost:3000
```

## Credits and license

- **Twenty** is built by [Twenty.com PBC](https://twenty.com) and contributors:
  <https://github.com/twentyhq/twenty>. It is licensed under
  [AGPL-3.0](https://github.com/twentyhq/twenty/blob/main/LICENSE). Some
  enterprise-only files are under a separate commercial license. This template uses
  only community features. If you modify Twenty and offer it to others over a
  network, AGPL obligations apply to you.
- Files in **this repository** (docs and config) are under the [MIT License](LICENSE).
  The MIT license doesn't cover Twenty itself.
- Postgres, Redis and the Railway services are provided by their respective owners.
