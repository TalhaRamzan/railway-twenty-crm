# Railway template checklist: Twenty CRM (pinned, worker + bucket)

Step-by-step guide to build this template in the Railway dashboard. Follow it in
order. Service names matter, because reference variables use them
(`${{Twenty.SERVER_URL}}`, `${{Postgres.DATABASE_URL}}`, `${{Bucket.BUCKET}}`...).

Pinned versions (researched 2026-11-21):

| Service | Image | Digest |
|---|---|---|
| Twenty, Twenty Worker | `twentycrm/twenty:v2.60.1` | `sha256:b190e54ca434ea1ee47fe73b8d76e77d0bf3c6f734fcec58e414108d6867939c` |
| Postgres | `ghcr.io/railwayapp-templates/postgres-ssl:18.6` | `sha256:4fc833b43d2aaf6ecc9b18615f5533602b723a32d0e2efe2dae8ca0f7217c863` |
| Redis | `redis:8.2.11` | `sha256:b0b01167ff47bc3868e56a967895a26be4230891a1f42a78ab61cdd227accc8d` |
| Bucket | Railway Storage Bucket | (managed, no image) |

Fallback if v2.60.1 fails the test: `twentycrm/twenty:v2.59.3`
(`sha256:1a1e8a8db27fef92227e2817678968a17795d586c241be257c3c6305c65706a6`).

**Image field format.** Prefer the digest form, for example
`twentycrm/twenty@sha256:b190e54c...939c`, so the image can never change under
the same name. If the Railway image field does not accept a digest, use the tag
(`twentycrm/twenty:v2.60.1`). Docker Hub version tags are not normally
re-pushed, but only a digest guarantees it. Record which form worked in TEST.md.

---

## 0. Before you start

- [ ] Use a **new, separate Railway project** for the test. Do not touch any other project.
- [ ] Re-check the latest Twenty release (Docker Hub `twentycrm/twenty` tags, GitHub
      releases `twenty/vX.Y.Z`). If you bump the version, update the digest everywhere
      (this file, README.md, docker-compose.yml, TEST.md).
- [ ] Open **Workspace Settings → Templates → New Template** (from scratch), or build
      a project first and use **Project Settings → Generate Template from Project**.
      The second way is easier to test before you publish.

## 1. Postgres

1. Add **Database → PostgreSQL** (Railway's managed Postgres). Keep the name **`Postgres`**.
2. Settings → Source → Image: change it to the pinned image
   `ghcr.io/railwayapp-templates/postgres-ssl:18.6` (or its digest form).
   - Twenty requires PostgreSQL 15 or newer (since v2.34). 18 is Railway's current default.
3. Keep Railway's default variables (`POSTGRES_PASSWORD=${{secret(32)}}`, `PGDATA`,
   `DATABASE_URL`, ...) and its volume at `/var/lib/postgresql/data`.
4. Public networking: **off** (no TCP proxy). Twenty connects over the private network.
5. Memory limit (Settings → Resource limits): **1 GB**.

## 2. Redis

1. Add **Database → Redis**. Keep the name **`Redis`**.
2. Settings → Source → Image: `redis:8.2.11` (or its digest form).
3. Keep Railway's default start command, password (`${{secret(32)}}`), `REDIS_URL`
   and volume (`/data`).
4. Do **not** set a `maxmemory-policy` other than `noeviction` (the default).
   Twenty's job queue (BullMQ) needs `noeviction`.
5. Public networking: **off**.
6. Memory limit: **256 MB**.

## 3. Bucket (file storage)

1. Add **Bucket** (Create → Bucket). Name it **`Bucket`**. Pick the region closest
   to the services.
2. Nothing else to set. It provides `BUCKET`, `ACCESS_KEY_ID`, `SECRET_ACCESS_KEY`,
   `REGION`, `ENDPOINT`.
3. Note: Twenty's S3 client always uses **path-style** URLs (`forcePathStyle: true`
   in the server source). Railway says new buckets use virtual-hosted style. Whether
   path-style requests also work is the **#1 thing to verify in TEST.md**.

Why a bucket: Railway volumes attach to one service only, but the server and the
worker both read and write files. Upstream's docker-compose shares one local
volume between them, which Railway can't do. A bucket works for both.

## 4. Twenty (server, public)

1. Add **Docker Image** service: `twentycrm/twenty:v2.60.1` (or digest form).
   Name it **`Twenty`**.
2. **Start command:** leave **empty**. The image's entrypoint (`/app/entrypoint.sh`)
   creates the DB schema on first boot, runs `upgrade` migrations on every start,
   flushes the cache and registers cron jobs. Then it starts `node dist/main`.
3. **Public networking:** Generate Domain → target port **3000**.
4. **Healthcheck path:** `/healthz` (upstream `ApiPath.Health = 'healthz'`; upstream
   compose uses `curl --fail http://localhost:3000/healthz`).
5. **Healthcheck timeout:** **600** seconds. The first boot (DB init) and upgrades
   can take several minutes.
6. **Restart policy:** On failure, max 10 retries.
7. **Volume:** none (files go to the bucket).
8. **Memory limit:** **2 GB**. CPU: 1 to 2 vCPU is plenty.
9. **Variables** (Raw Editor, paste exactly):

```env
PORT=3000
NODE_PORT=${{PORT}}
SERVER_URL=https://${{RAILWAY_PUBLIC_DOMAIN}}
PG_DATABASE_URL=${{Postgres.DATABASE_URL}}
REDIS_URL=${{Redis.REDIS_URL}}
ENCRYPTION_KEY=${{secret(43, "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/")}}=
APP_SECRET=${{secret(43, "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/")}}=
STORAGE_TYPE=s3
STORAGE_S3_NAME=${{Bucket.BUCKET}}
STORAGE_S3_REGION=${{Bucket.REGION}}
STORAGE_S3_ENDPOINT=${{Bucket.ENDPOINT}}
STORAGE_S3_ACCESS_KEY_ID=${{Bucket.ACCESS_KEY_ID}}
STORAGE_S3_SECRET_ACCESS_KEY=${{Bucket.SECRET_ACCESS_KEY}}
IS_MULTIWORKSPACE_ENABLED=false
IS_WORKSPACE_CREATION_LIMITED_TO_SERVER_ADMINS=true
EMAIL_DRIVER=logger
EMAIL_FROM_ADDRESS=noreply@example.com
EMAIL_FROM_NAME=Twenty CRM
TELEMETRY_ENABLED=false
DISABLE_DB_MIGRATIONS=false
DISABLE_CRON_JOBS_REGISTRATION=false
NODE_OPTIONS=--max-old-space-size=1536
RAILWAY_DEPLOYMENT_DRAINING_SECONDS=60
```

Notes on each variable:

| Variable | Why |
|---|---|
| `PORT` / `NODE_PORT` | Twenty listens on `NODE_PORT` (default 3000), not `PORT`. We pin both to 3000 so Railway's domain target port and the app agree. |
| `SERVER_URL` | Must be the exact public URL. Wrong value = login loops and broken attachments. If you add a custom domain, change it **here only** (the worker references it). |
| `PG_DATABASE_URL` | Railway Postgres private URL. |
| `REDIS_URL` | Railway Redis private URL (has the password). Environments created after Oct 2025 have IPv4 and IPv6 private networking. If you see `ENOTFOUND redis.railway.internal` in an old IPv6-only environment, try appending `?family=0` (ioredis option). That is not verified with Twenty's second Redis client (node-redis). |
| `ENCRYPTION_KEY` | Primary key for at-rest encryption (Twenty v2.5+). 43 random base64 chars + `=` matches the `openssl rand -base64 32` format. **Users must back it up.** |
| `APP_SECRET` | Legacy secret. Twenty falls back to it when `ENCRYPTION_KEY` is unset. Upstream compose still passes it, so we set it too. |
| `STORAGE_*` | `s3` is converted to `S_3` by Twenty. All values come from the Railway bucket. |
| `IS_MULTIWORKSPACE_ENABLED=false` | Single-workspace mode: sign-up closes after the first workspace exists. |
| `IS_WORKSPACE_CREATION_LIMITED_TO_SERVER_ADMINS=true` | Belt and braces: no new workspaces without a server admin, and no sign-up without an invite or approved domain. Upstream says it is ignored during the very first setup. |
| `EMAIL_DRIVER=logger` | Twenty v2.60.1 only has `logger` and `smtp` drivers. Railway Hobby blocks SMTP. See README → Email. |
| `TELEMETRY_ENABLED=false` | Privacy default for a self-hosted CRM. Users can turn it on. |
| `DISABLE_*=false` | Explicit: the server owns migrations and cron registration. |
| `NODE_OPTIONS` | Caps the Node heap so the server stays inside the 2 GB limit (heap + ~300 MB overhead). |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | Gives in-flight requests time to finish before the old deploy stops (upstream uses a 120 s stop grace period). |

## 5. Twenty Worker (private)

1. Add **Docker Image** service with the **same image** as `Twenty`. Name it **`Twenty Worker`**.
2. **Start command:** `/app/entrypoint.sh yarn worker:prod`
   - Railway's custom start command replaces the image ENTRYPOINT. We call the
     entrypoint ourselves. With both `DISABLE_*` flags set to `true` it skips
     migrations and cron registration and just `exec`s the worker. This also works
     if Railway keeps the entrypoint (it would only run twice, doing nothing extra).
3. **Public networking:** **off**. No domain.
4. **Healthcheck:** none (the worker has no HTTP port).
5. **Restart policy:** Always (or On failure, max 10). On a fresh deploy the worker
   may restart a few times until the server finishes the first DB init. That is expected.
6. **Volume:** none.
7. **Memory limit:** **1.5 GB**.
8. **Variables** (Raw Editor):

```env
SERVER_URL=${{Twenty.SERVER_URL}}
PG_DATABASE_URL=${{Twenty.PG_DATABASE_URL}}
REDIS_URL=${{Twenty.REDIS_URL}}
ENCRYPTION_KEY=${{Twenty.ENCRYPTION_KEY}}
APP_SECRET=${{Twenty.APP_SECRET}}
STORAGE_TYPE=${{Twenty.STORAGE_TYPE}}
STORAGE_S3_NAME=${{Twenty.STORAGE_S3_NAME}}
STORAGE_S3_REGION=${{Twenty.STORAGE_S3_REGION}}
STORAGE_S3_ENDPOINT=${{Twenty.STORAGE_S3_ENDPOINT}}
STORAGE_S3_ACCESS_KEY_ID=${{Twenty.STORAGE_S3_ACCESS_KEY_ID}}
STORAGE_S3_SECRET_ACCESS_KEY=${{Twenty.STORAGE_S3_SECRET_ACCESS_KEY}}
IS_MULTIWORKSPACE_ENABLED=${{Twenty.IS_MULTIWORKSPACE_ENABLED}}
IS_WORKSPACE_CREATION_LIMITED_TO_SERVER_ADMINS=${{Twenty.IS_WORKSPACE_CREATION_LIMITED_TO_SERVER_ADMINS}}
EMAIL_DRIVER=${{Twenty.EMAIL_DRIVER}}
EMAIL_FROM_ADDRESS=${{Twenty.EMAIL_FROM_ADDRESS}}
EMAIL_FROM_NAME=${{Twenty.EMAIL_FROM_NAME}}
TELEMETRY_ENABLED=${{Twenty.TELEMETRY_ENABLED}}
DISABLE_DB_MIGRATIONS=true
DISABLE_CRON_JOBS_REGISTRATION=true
NODE_OPTIONS=--max-old-space-size=1024
```

Every shared setting points at the server, so users change settings in **one place**
(the `Twenty` service). This fixes the most common bug in other templates: the
server and worker drifting apart (different secrets or `SERVER_URL`).

## 6. Optional variables (do not add to the template, document only)

Users add these on **`Twenty`** and reference them from **`Twenty Worker`**
(`${{Twenty.NAME}}`), or set them in Twenty's **Admin Panel → Config variables**
(`IS_CONFIG_VARIABLES_IN_DB_ENABLED` is `true` by default, so both server and worker
read them from the database).

| Variable | Use |
|---|---|
| `EMAIL_DRIVER=smtp`, `EMAIL_SMTP_HOST`, `EMAIL_SMTP_PORT`, `EMAIL_SMTP_USER`, `EMAIL_SMTP_PASSWORD`, `EMAIL_SMTP_NO_TLS` | Sending email. Railway **Pro** only (Hobby blocks SMTP). |
| `AUTH_GOOGLE_*`, `MESSAGING_PROVIDER_GMAIL_ENABLED`, `CALENDAR_PROVIDER_GOOGLE_ENABLED` | Google sign-in, Gmail and Calendar sync. |
| `AUTH_MICROSOFT_*`, `MESSAGING_PROVIDER_MICROSOFT_ENABLED`, `CALENDAR_PROVIDER_MICROSOFT_ENABLED` | Microsoft sign-in, Outlook sync. |
| `FALLBACK_ENCRYPTION_KEY` | Only during an encryption key rotation. |
| `PG_POOL_MAX_CONNECTIONS` | Default 10 per process. Leave it. |

## 7. Template metadata (publish step)

- **Title:** `Twenty CRM | Open-Source Salesforce Alternative, Pinned & Upgrade-Tested [Updated Nov 2026]`
- **Short description:** `Self-hosted Twenty v2.60.1 CRM: server + background worker, Postgres, Redis and a Railway bucket for files. Pinned image digests, sign-up locked, upgrade guide and cost estimate.`
- **Category:** CRM / Business tools. **Tags:** `crm`, `twenty`, `salesforce-alternative`, `hubspot-alternative`, `self-hosted`, `open-source`, `sales`.
- **Icon:** a generic CRM/contacts icon. Do **not** use the Twenty logo (see PLAN section 5, rule 4).
- **README / overview:** paste README.md (replace the deploy-button placeholder with the real template URL).
- Turn on **Template Queue email notifications** (answering questions raises the kickback from 15% to 25%).

## 8. Monthly maintenance

1. Check the new Twenty release (Docker Hub tags + GitHub `twenty/v*` release notes).
2. Read the upgrade notes for breaking changes (Postgres version bumps, removed env vars).
3. In a test project: deploy the **current** template, create data, then bump both
   Twenty services to the new digest and run TEST.md section C (upgrade test).
4. Update the digest in this file, README.md, docker-compose.yml and TEST.md.
   Change the title suffix to `[Updated <Month YYYY>]`.
5. Delete the test project.
