# TEST.md: Twenty CRM template test checklist

Run this in a **new, throwaway Railway project**. The stack uses about 1.5 GB of RAM,
so keep the test **to a few hours**, then **delete the whole project** (PLAN sections 2 and 7).

Pinned under test: `twentycrm/twenty:v2.60.1`
`sha256:b190e54ca434ea1ee47fe73b8d76e77d0bf3c6f734fcec58e414108d6867939c`.
Upgrade-from version: `twentycrm/twenty:v2.59.3`
`sha256:1a1e8a8db27fef92227e2817678968a17795d586c241be257c3c6305c65706a6`.

## Already verified statically (2026-11-21, no Docker on the build machine)

| Check | Result |
|---|---|
| `twentycrm/twenty:v2.60.1` exists; digest via Docker Hub API **and** registry manifest | OK `sha256:b190e54c...939c` (amd64 + arm64) |
| `redis:8.2.11` digest (Hub API + registry) | OK `sha256:b0b01167...cc8d` |
| `ghcr.io/railwayapp-templates/postgres-ssl:18.6` digest (GHCR registry) | OK `sha256:4fc833b4...c863` (same as tag `18`) |
| `chrislusf/seaweedfs:4.54`, `amazon/aws-cli:2.41.2` digests (Hub API, local compose only) | OK |
| `docker-compose.yml` parses as YAML; every service image has tag + digest | OK (python3 yaml) |
| Every env var in compose/checklist exists in Twenty v2.60.1 `config-variables.ts` (or is read by `entrypoint.sh`) | OK |
| Healthcheck path `/healthz` (`ApiPath.Health = 'healthz'`, upstream compose `curl --fail .../healthz`) | OK |
| Worker command `yarn worker:prod` = `node dist/queue-worker/queue-worker` (twenty-server package.json) | OK |
| Entrypoint: `database:init:prod` on an empty DB, then `upgrade` + cache flush on every start; skipped with `DISABLE_DB_MIGRATIONS=true` | OK (read `packages/twenty-docker/twenty/entrypoint.sh` at tag `twenty/v2.60.1`) |
| `EMAIL_DRIVER` options: only `LOGGER`, `SMTP` | OK (email-driver.enum.ts) |
| `STORAGE_TYPE=s3` becomes `S_3` (lodash snakeCase + upper) | OK |
| Twenty S3 client uses `forcePathStyle: true` | Confirmed in source. **Railway compatibility NOT verified** (see B3). |

## A. Local run (needs Docker, not done yet)

- [ ] `cp .env.example .env`, fill secrets, `docker compose config` shows no errors.
- [ ] `docker compose up -d`. `bucket-init` exits 0, `twenty` turns healthy, `twenty-worker` stays up.
- [ ] http://localhost:3000 → create account + workspace.
- [ ] Upload a profile picture and a file attachment on a company record. Then
      `aws --endpoint-url http://localhost:8333 s3 ls s3://twenty-files --recursive`
      (port 8333 is not published by default; run it via `docker compose exec` or publish it temporarily) shows objects.
- [ ] `docker compose restart` → data and files still there.
- [ ] Worker logs show jobs being processed (e.g. after a CSV import).

## B. Railway deploy (fresh)

1. [ ] Build the project following `railway-template.md` (or deploy the draft template).
       Note whether the image field accepted the `@sha256:` digest form: ______
2. [ ] Server deploy log shows `Database appears to be empty, running migrations.` then
       `Successfully migrated DB!` and `Successfully registered all background sync jobs!`.
       Time to first healthy: ______ min (healthcheck timeout is 600 s).
3. [ ] **Bucket / path-style (highest risk):** upload an avatar and an attachment.
       They display after a page reload, and objects appear in the bucket (Bucket → Data/Files tab).
       - If uploads fail with signature/NoSuchBucket/host errors: Railway's bucket does
         not accept path-style requests. Options: (a) ask on Railway Station whether
         path-style is supported for new buckets, (b) use an external S3 provider that
         supports path-style (Cloudflare R2, Backblaze B2, AWS S3), (c) last resort:
         `STORAGE_TYPE=local` + a volume on the server only (worker file jobs will break).
         Record the result: ______
4. [ ] Worker logs show it connected to Redis and is processing queues (no crash loop
       after the server is healthy).
5. [ ] Reference variables resolved: worker `SERVER_URL` shows the real https URL
       (nested reference `${{Twenty.SERVER_URL}}` → `https://${{RAILWAY_PUBLIC_DOMAIN}}`).
6. [ ] `ENCRYPTION_KEY` / `APP_SECRET` generated: 44 chars, ending with `=`, different values.
7. [ ] Open the app → create admin + workspace. Create 1 company, 1 person, 1 note, 1 task.
8. [ ] **Sign-up locked:** in a private window, try to sign up with a new email.
       It must be refused (no new workspace). Invite link from Settings → Members works.
9. [ ] Email logger: trigger an invite or password reset. The email body appears in
       the logs (server or worker), and nothing crashes.
10. [ ] CSV import of ~50 contacts completes (proves the worker works).
11. [ ] Redeploy **Twenty** and **Twenty Worker** → all records and files still there.
12. [ ] Restart Postgres → app recovers without manual action.
13. [ ] Redis: `redis-cli CONFIG GET maxmemory-policy` returns `noeviction`.
14. [ ] Memory graph after ~1 h idle: server ____ MB, worker ____ MB, Postgres ____ MB,
        Redis ____ MB. Update the README cost table if it's very different.
        Check that no service hits its memory limit and nothing logs `heap out of memory`.
15. [ ] Healthcheck: `curl -i https://<domain>/healthz` returns 200.
16. [ ] Only `Twenty` has a public domain. Postgres/Redis/worker have no public networking.

## C. Upgrade test (this is what "upgrade-tested" means)

1. [ ] Fresh project, both Twenty services on **v2.59.3** (digest above). Complete steps B2, B7 and B3 (data + files).
2. [ ] Change both images to **v2.60.1** digest. Deploy Twenty first, then the worker.
3. [ ] Server log: `upgrade` ran, `Successfully migrated DB!`, no "Upgrade completed with errors".
4. [ ] All records and files from step 1 are still there. Login still works without a forced logout,
       or note it if one happens: ______
5. [ ] Worker processes a new CSV import after the upgrade.

## D. Backup / restore drill (optional, recommended once)

- [ ] `pg_dump` via `DATABASE_PUBLIC_URL` works (TCP proxy on, then off again).
- [ ] `aws s3 sync` of the bucket with the bucket credentials works.

## E. Cleanup

- [ ] Delete the **entire** test project (services, volumes, bucket).
- [ ] Record the date, version and results here, then publish (or fix and retest).

## Results log

| Date | Version | Fresh deploy | Bucket | Upgrade from | Idle RAM total | Notes |
|---|---|---|---|---|---|---|
| | v2.60.1 | | | v2.59.3 | | |

## Local run results (2026-11-21, OrbStack, Docker 29.8.2, Apple x86_64 host)
- PASS: `docker compose config`, `bucket-init` exit 0, server healthy in ~70 s on first boot, worker up and processing BullMQ jobs.
- PASS: sign up → create workspace → profile → companies view loads with seed data.
- PASS: S3 storage: objects written to the bucket (SeaweedFS stand-in) on workspace creation.
- PASS: `docker compose restart` → healthy again, user + workspace still in Postgres.
- Measured RAM (idle, 1 workspace): server ~0.85–0.9 GB, worker ~0.75–1.1 GB, Postgres ~60–115 MB, Redis ~20 MB → **~1.7–2.2 GB total**, i.e. roughly $17–22/month RAM on Railway before CPU. Update README cost table if Railway numbers agree.
- Not tested locally: Railway bucket path-style access (B3), file upload via UI, upgrade v2.59.3 → v2.60.1.

## Local upgrade test results (2026-11-21)
- PASS: fresh stack on v2.59.3 → sign up, workspace "Upgrade Co", seed companies; renamed one company to `UpgradeMarker`.
- PASS: switched server to v2.60.1 → healthy; upgrade-metadata step ran, cron registration 32 OK / 0 failed; then worker on v2.60.1 → running, no errors.
- PASS: all 5 companies incl. `UpgradeMarker` still there (DB and UI); browser session stayed logged in, no forced logout.

## Railway deploy results (2026-11-21, Hobby plan, test project deleted after ~28 min)
- PASS: image field accepts `twentycrm/twenty:v2.60.1@sha256:…`.
- PASS: references resolve, including nested worker references (`${{Twenty.SERVER_URL}}`, `${{Twenty.STORAGE_S3_*}}`) and the bucket's `${{Bucket.*}}` variables. `ENCRYPTION_KEY` secret format resolves to 44 chars.
- PASS (B3, the #1 risk): Railway bucket (endpoint `https://t3.storageapi.dev`) accepts **path-style** S3 requests (put/list/get/delete verified with aws-cli `addressing_style = path`). Twenty itself wrote its app-registration files into the bucket on boot.
- PASS: server healthy (`/healthz` 200, `/welcome` 200), "Successfully migrated DB!"; worker (start command `/app/entrypoint.sh yarn worker:prod`) processes BullMQ cron jobs over Railway Redis.
- Measured RAM on Railway (idle, no workspace yet): Twenty ~860 MB, Worker ~1.03 GB, Postgres ~75 MB, Redis ~19 MB → ~2 GB ≈ $20/month RAM.
- FOUND: if the very first boot is interrupted (e.g. a redeploy while `database:init:prod` is running), the `core` schema exists but is incomplete; later boots skip init and fail with `relation "core.keyValuePair" does not exist`. Fix: reset the database (new Postgres or drop the `core` schema) and deploy once without interrupting. Added to README troubleshooting.
- Not tested on Railway: sign-up/workspace in the browser and file upload from the UI (verified locally).
