# Operating this Umami fork

Self-hosted analytics for the portfolio sites, deployed from this repository to
Vercel at `analytics.rensranks.dev`. Every site loads the tracker from it, so an
outage or a compromise here touches all of them at once. This page covers the
few things that need a human on a schedule. It was written for the November 2026
audit (findings UMAMI-P3-008, 009, 010, 013, 014, 016, 019, 020, 021).

## 1. Check that the default admin password was changed

The first migration (`prisma/migrations/01_init/migration.sql`) seeds a user
`admin` with the documented default password `umami`. Nothing forces a change
on first login and there is no signup page, so this one account is the front
door for the whole portfolio.

**Definitive check (SQL).** Connect with the direct URL (see section 3) and run:

```sql
select username, role, created_at, updated_at,
       password = '$2b$10$BUli0c.muyCW1ErNJc3jL.vFRFtFJWrT8/GcR4A.sUdCznaXiqFXa' as is_default_password
from "user"
where username = 'admin';
```

`is_default_password = true` means the seed password is still live. Fix it
before anything else. A non-null `updated_at` is a weaker hint that the row was
edited at some point; the hash comparison is the one to trust.

**Dashboard check.** Log in as `admin`, open Settings > Profile and set a new
password. If you can log in with `umami`, the password was never changed.

Changing the admin password also invalidates every existing login token for
that account (tokens carry a hash of the password), so it doubles as the
"log everyone out" procedure. See section 6.

## 2. Create a personal admin and demote the seeded account

Keep `admin` only as a break-glass account, and use a personal login day to day.

1. Log in as `admin`, go to Settings > Users > Create user. Pick a personal
   username, a long password, and the role **Administrator**.
2. Log out, log in as the new user and confirm you can see every website and
   Settings > Users.
3. Back in Settings > Users, edit `admin` and set its role to **View only**
   (or delete it outright if you are confident the new account works and its
   password is stored in a password manager). Roles are `admin`, `user` and
   `view-only`.

The same thing from the API, if you prefer (`$TOKEN` is the value returned by
`POST /api/auth/login`; the base URL is the deployment, written here as `$APP`):

```bash
# create the personal admin
curl -X POST "$APP/api/users" -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"username":"rens","password":"<long password>","role":"admin"}'

# find the seeded account's id, then demote it
curl "$APP/api/users" -H "Authorization: Bearer $TOKEN"
curl -X POST "$APP/api/users/41e2b680-648e-4b09-bcd7-3e2b10c06264" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"role":"view-only"}'
```

(`41e2b680-648e-4b09-bcd7-3e2b10c06264` is the id the migration gives `admin`.)

Once the deployment runs v3.4.0 or later and `TWO_FACTOR_ENCRYPTION_KEY` is
set on Vercel (see `.env.example`), enrol the personal admin in two-factor auth
under Settings > Profile. Do not enable 2FA on an older build: before v3.4.0 the
five-minute partial token issued after the password step was accepted as a full
session.

## 3. Where the database lives

The app only knows the database through two environment variables on Vercel:

| Variable | Used by | Purpose |
| --- | --- | --- |
| `DATABASE_URL` | the running functions, `prisma.config.ts` | pooled connection (PgBouncer-style). Fine for queries, not for migrations or dumps. |
| `DIRECT_DATABASE_URL` | `scripts/check-db.js` at build time | direct connection. `prisma migrate deploy` runs against this one. Use it for backups too. |

The engine is PostgreSQL (the only Prisma schema and migration set in the repo
is the Postgres one, and the build refuses anything below Postgres 9.4). The
pooled-plus-direct pair is the pattern of Neon, Supabase or Vercel Postgres.
The hostname tells you which:

```bash
vercel env pull .env.vercel --environment=production   # then read the host
grep -E 'DATABASE_URL' .env.vercel | sed -E 's#://[^@]*@#://***@#'
rm .env.vercel
```

`*.neon.tech` is Neon, `*.pooler.supabase.com` / `*.supabase.co` is Supabase,
`*.postgres.vercel-storage.com` is Vercel Postgres. Once known, record the
provider, plan, region and backup retention in `docs/audits/`, and make sure the
region matches the function region in `vercel.json` (currently `dub1`, Dublin;
switch to `fra1` if the database turns out to be in Frankfurt).

As of the audit the three Supabase projects on the account do not hold the Umami
tables, so the provider is still to be confirmed from the Vercel Storage tab.

## 4. Backups

Take one before every upgrade and once a week if the provider has no automatic
daily backup. Always use the direct URL; poolers in transaction mode break
`pg_dump`.

```bash
# dump (custom format: compressed, restorable table by table)
pg_dump "$DIRECT_DATABASE_URL" --format=custom --no-owner --no-privileges \
  --file "umami-$(date +%F).dump"

# verify the file is readable
pg_restore --list "umami-$(date +%F).dump" | head

# restore into an empty database (or over the existing one with --clean)
pg_restore --dbname "$DIRECT_DATABASE_URL" --no-owner --no-privileges \
  --clean --if-exists "umami-YYYY-MM-DD.dump"
```

Use the `pg_dump` from a client at the same major version as the server
(`psql "$DIRECT_DATABASE_URL" -c 'show server_version'`). Store dumps off the
database provider, for example an S3 or Backblaze bucket, and keep the last
four weekly files plus one per month for a year. If the provider offers
point-in-time recovery on a paid tier, that is simpler than a cron job and worth
the few euros, because losing this database loses the analytics history of every
site at once.

Rollback target for a deploy that goes wrong is Vercel > Deployments > Instant
Rollback to the previous production deployment. Only restore a dump if data was
actually damaged; new migrations are additive and the old build ignores the
extra tables.

## 5. Monthly upstream check

Umami ships security fixes as point releases and Next.js carries most of the
risk. Do this at the start of each month, or sooner when a Next.js advisory
lands.

```bash
git fetch origin
git fetch upstream --tags          # upstream = https://github.com/umami-software/umami
git tag --sort=-v:refname | head -3
git rev-list --count master..upstream/master   # commits behind
```

If there is a new release:

```bash
git checkout -b upgrade/vX.Y.Z master
git merge vX.Y.Z                   # keep biome.json vcs.enabled=false on conflict
npx --yes pnpm@12.3.4 install --frozen-lockfile
npx --yes pnpm@12.3.4 audit --prod # no critical, nothing in next
DATABASE_URL=postgresql://u:p@localhost:5432/dummy \
  npx --yes pnpm@12.3.4 test
SKIP_DB_CHECK=1 SKIP_BUILD_GEO=1 DISABLE_TELEMETRY=1 \
DATABASE_URL=postgresql://u:p@localhost:5432/dummy \
  npx --yes pnpm@12.3.4 build
```

Then open a PR, merge, take a backup (section 4), redeploy, and smoke test:
dashboard login, one site's realtime view shows fresh page views, `/script.js`
serves the new tracker, one share link still opens. Record the new version in
`docs/audits/`.

The fork carries exactly one local change on top of upstream: `biome.json`
sets `vcs.enabled` and `vcs.useIgnoreFile` to `false` so Vercel CLI builds,
which have no `.git` directory, do not crash Biome. If deploys move to the Git
integration this change can be dropped.

With `DISABLE_UPDATES=1` set, the dashboard no longer shows the "new version"
banner, so this routine is the only reminder. If you would rather keep the
banner, leave `DISABLE_UPDATES` unset; it is a single request to
`api.umami.is` from the admin's browser and sends only the version number.

## 6. Sessions, share links and retention (while you are in there)

**Sessions.** Without `REDIS_URL`, logins are stateless tokens with no expiry
and no server-side revocation. Logging out only clears the browser. If a laptop
or browser session may have leaked, change that user's password: tokens embed
the password hash, so the change kills them all. Rotating `APP_SECRET` also
works but takes every share link and cache key with it.

**Share links.** `GET /share/<slug>` is public and never expires. Once a month
open each website > Settings > Share and delete anything you did not create on
purpose. Prefer the random 16-character slug over a custom one.

**Retention.** Events are kept forever and nothing purges them. Decide a
period (26 months matches GA4's maximum; 14 months is a common choice), state
it in the sites' privacy notices, then run this monthly against the direct URL:

```sql
begin;
delete from event_data    where created_at < now() - interval '26 months';
delete from session_data  where created_at < now() - interval '26 months';
delete from website_event where created_at < now() - interval '26 months';
delete from session s
 where not exists (select 1 from website_event e where e.session_id = s.session_id);
commit;
```

Run it from n8n or any scheduler that can reach the database; the app itself has
no cron. Check row counts with a `select count(*)` first the first time.

## 7. Environment variables at a glance

See `.env.example` for the full list with comments. The ones that matter for
this deployment: `DATABASE_URL`, `DIRECT_DATABASE_URL`, `APP_SECRET`,
`DISABLE_TELEMETRY=1`, `DISABLE_UPDATES=1`, and, once on v3.4+,
`TWO_FACTOR_ENCRYPTION_KEY`. They are set in Vercel > Project > Settings >
Environment Variables; the repo never contains real values.
