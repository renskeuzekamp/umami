# Umami fork: Pass 3 findings (dependencies and configuration)

Audit date: 2026-11-03. Repo: `renskeuzekamp/umami` (public GitHub fork), default branch `master` at `729c0cf`.
Deployed as Vercel project `umami-analytics` (team Domundos, `prj_qAa8GhrImG71bLDaaGaADPVcTPmX`) at `analytics.rensranks.dev`.
No request was sent to the live hostname. Vercel and Supabase facts come from their management APIs; code facts from this checkout; dependency facts from `pnpm install --frozen-lockfile` plus `pnpm audit --prod --json` run in the audit container.

Totals: critical 1, high 2, medium 9, low 4, info 6 (22 findings).

### UMAMI-P3-001  Fork is upstream v3.3.1 plus one local commit; two releases and 115 commits behind
Severity: high
Effort:   S (half day)
Confidence: confirmed (reproduced)
Where:    package.json:3, biome.json:3-7, git history (`git log origin/master`)
Evidence: `ca661c7` is tagged `v3.3.1` upstream (2026-08-20). The only local commit is `729c0cf` (2026-08-29), which flips `vcs.enabled` and `vcs.useIgnoreFile` to `false` in `biome.json` (2 lines) so Vercel CLI builds without a `.git` directory do not crash Biome. No other file differs from v3.3.1. Nothing local touches auth, `/api/send`, or the share pages. Upstream `umami-software/umami` master is at `v3.4.1` (2026-10-23); `git rev-list --count v3.3.1..v3.4.1` = 115 commits, 552 files changed, 2 new Prisma migrations (`25_add_annotation`, `26_add_api_key`). No `upstream` remote is configured in the clone. The fork repo is public on GitHub (`private: false`); no `.env` file is committed.
Impact:   The deployed build misses the v3.4.0 auth hardening (partial-2FA-token rejection, deleted-team access checks, report website reassignment fix) and the v3.4.1 security hotfix (Next.js 16.4.2, React 19.3.1, dependency floors). Because the local change is 2 lines of Biome config, a rebase or merge onto v3.4.1 carries no code-conflict risk.
Fix:      Add `git remote add upstream https://github.com/umami-software/umami.git`, merge `v3.4.1` into `master` (see UMAMI-P3-020 for the full plan), redeploy.
Related:  UMAMI-P3-002, UMAMI-P3-003, UMAMI-P3-020

### UMAMI-P3-002  Next.js 16.3.0 carries five critical RCE advisories fixed in 16.3.3 to 16.4.1
Severity: critical
Effort:   S (half day)
Confidence: confirmed (reproduced)
Where:    package.json:103 (`"next": "16.3.0"`), pnpm-lock.yaml
Evidence: `pnpm audit --prod --json` on the frozen lockfile reports for `next@16.3.0`: GHSA-2464-fhr4-w9xw (unauthenticated RCE, Image Optimization with SVG, fixed >=16.4.1), GHSA-2xp9-vwfh-vxw4 (unauthenticated RCE, Image Optimization with AVIF, fixed >=16.3.3), GHSA-673q-mjm6-r5xr (RCE via `use cache` typed arrays, fixed >=16.4.1), GHSA-vcvr-r3jv-pc5j (RCE in `next/og`, fixed >=16.3.6), GHSA-p293-qw3h-jr36 / CVE-2026-75604 (RCE on Windows hosts, fixed >=16.3.3); high: CVE-2026-94483 SSRF in Image Optimization (fixed >=16.3.8), CVE-2026-94482 DoS via Server Actions (fixed >=16.4.1); moderate: four cache-poisoning / cache-leak issues (CVE-2026-94543/94544/94484/103004, fixed >=16.3.8), CVE-2026-94485 metadata image info disclosure; low: CVE-2026-94486 (dev server only). Upstream v3.4.1 release notes: "Upgraded Next.js to 16.4.2 ... fixes a critical RCE in next/og, SSRF in image optimization, cache poisoning, and an information disclosure issue. All self-hosted users should upgrade." The deployed Vercel build (`dpl_J3Sb2uXKTYFJiwA8SwHnVT6duQbw`, 2026-08-29) was built from this lockfile.
Impact:   Reachability on the public hostname: `next.config.ts` sets no `images` option, so the `/_next/image` optimizer endpoint is served unauthenticated by default (the SVG/AVIF RCE and the SSRF apply to that endpoint; Vercel's build uses `sharp@0.35.3`, itself flagged for libheif/librsvg CVEs). `next/og` and `ImageResponse` are not imported anywhere in `src`, so that RCE is not reachable. The `use cache` and Server Actions issues depend on Next internals rather than app code; treat as reachable until upgraded. The Windows RCE does not apply to Vercel Linux functions. A compromise of this function reads `DATABASE_URL` and `APP_SECRET` (which signs every login, share and cache token) and reaches analytics data for every portfolio site.
Fix:      Take upstream v3.4.1 (Next 16.4.2) via UMAMI-P3-020. If the merge must wait, the minimal stopgap is `pnpm add next@16.4.2 react@19.3.1 react-dom@19.3.1` on the fork, rebuild, redeploy, and add `images: { unoptimized: true }` to `next.config.ts` (Umami never uses `next/image`) so the optimizer endpoint is removed outright.
Related:  UMAMI-P3-001, UMAMI-P3-004, UMAMI-P3-020

### UMAMI-P3-003  Fork accepts the 5-minute partial 2FA token as a full session (fixed upstream in v3.4.0)
Severity: medium
Effort:   XS (<1h)
Confidence: likely (read in code, not run)
Where:    src/lib/auth.ts:33-45, src/app/api/auth/login/route.ts:47-50
Evidence: Login issues `createSecureToken({ userId, type: 'partial-auth' }, secret(), { expiresIn: '5m' })` when a user has 2FA enabled. `checkAuth()` in the fork parses any secure token, reads `userId`, loads the user and accepts it; it never inspects `type`. Upstream v3.4.1 `src/lib/auth.ts` adds `if (payload?.type === PARTIAL_AUTH_TOKEN_TYPE) return null;` and the v3.4.0 notes list "Reject partial 2FA tokens for authenticated API access". Currently latent: `TWO_FACTOR_ENCRYPTION_KEY` is not set on Vercel (see UMAMI-P3-009), so no account can have 2FA enabled and no partial token is ever minted.
Impact:   If 2FA is switched on without first upgrading, the password step alone yields a token that bypasses the second factor for 5 minutes. Fixing UMAMI-P3-009 before this finding would reintroduce the hole.
Fix:      Upgrade to v3.4.1 (UMAMI-P3-020). Only enable 2FA after that lands.
Related:  UMAMI-P3-001, UMAMI-P3-009, UMAMI-P3-020

### UMAMI-P3-004  Other production-tree advisories: 72 in 29 packages, almost all build-time or admin-UI only
Severity: medium
Effort:   S (half day)
Confidence: confirmed (reproduced)
Where:    pnpm-lock.yaml; `pnpm audit --prod --json` (critical 6, high 41, moderate 21, low 6; the full tree incl. dev adds 37 more)
Evidence: Reachability by public surface (collect endpoint `/api/send`, `/script.js`, `/share/<slug>` are unauthenticated; admin is behind login):
- `zod@4.4.3` GHSA-mrhx-c7xc-m3w5 (high, memory exhaustion on oversized constrained collections, fixed >=4.6.6). Reachable: `/api/send` validates attacker-controlled JSON with zod (`src/app/api/send/route.ts:33`). Vercel caps request bodies at 4.5 MB, which bounds the blast radius to one function invocation per request.
- `shell-quote@1.8.3` (critical CVE-2026-9277, high CVE-2026-13311) via `npm-run-all`: build scripts only, never runs in the deployed function. Not reachable.
- `sharp@0.35.3` (two high, libheif/librsvg) via `next`: reachable only through the image optimizer (see UMAMI-P3-002).
- `@dicebear/initials@9.4.2` (moderate SVG injection) and `colord@2.9.3` (moderate ReDoS): avatar/colour rendering in the logged-in UI; attacker needs an account.
- `fflate@0.4.8` (moderate, infinite loop on malformed ZIP64) via `rrweb-player`: session-replay viewer in the admin UI; replays are recorded by the site's own recorder, so an attacker would need to inject a crafted replay via the public `/api/record` endpoint. Low likelihood.
- `d3-color@2.0.0` (high ReDoS) via `react-simple-maps`: client-side map on dashboards and share pages; input is country codes, not user strings. Not practically reachable.
- `mysql2@3.15.3`, `fast-uri@3.1.4`, `deepmerge-ts@7.1.5` via `prisma` / `@prisma/dev`: Prisma CLI and dev tooling, not loaded at runtime (the app uses `@prisma/adapter-pg`). Not reachable.
- `minimatch`, `brace-expansion`, `browserslist`, `svgo`, `js-yaml`, `source-map-js`, `nanoid`, `@babel/core`, `deepmerge` via `@svgr/cli`, `npm-run-all`, `postcss`: build-time only.
Upstream v3.4.1 "raised the security floors for vite, undici, sharp, shell-quote, brace-expansion, fast-uri, postcss-selector-parser, and source-map-js" through pnpm overrides, which clears most of this list.
Impact:   Beyond Next.js, the only runtime-reachable item on the public surface is the zod memory-exhaustion DoS on the collect endpoint. The rest is noise that will keep failing `pnpm audit` and hide real regressions.
Fix:      Take v3.4.1 (brings the overrides). Add `zod` to `pnpm.overrides` at `>=4.6.6` if the merged lockfile still resolves below that (`pnpm why zod` after merge).
Related:  UMAMI-P3-002, UMAMI-P3-020

### UMAMI-P3-005  Published Umami advisories: GHSA-3xx8-742v-m4mq (cache token as share token) is mitigated in this code; verify no other gap
Severity: info
Effort:   XS (<1h)
Confidence: likely (read in code, not run)
Where:    src/lib/auth.ts:112-128 (`parseShareToken`), src/app/api/send/route.ts:363 (cache token `type: CACHE_TOKEN_TYPE`)
Evidence: The upstream advisory list shows two entries: GHSA-8www-cffh-4q98 (critical, share link could reset website data, affected <2.3.0, patched 2.3.1) and GHSA-3xx8-742v-m4mq (high, published 2026-06-30, "Cache token from tracking endpoint reusable as share token grants full analytics access", listed as affecting "current (>= 2.3.1)" with "Patched versions: none"). In this checkout `parseShareToken()` rejects any token whose `type !== SHARE_TOKEN_TYPE`, with the comment "prevents other tokens signed with the same secret (e.g. the cache token from /api/send) from being replayed as share tokens". The `/api/send` cache token is minted with `type: CACHE_TOKEN_TYPE`. So v3.3.1 already carries the fix even though the advisory page was never updated. Between v3.3.1 and v3.4.1 no further repo advisory was published; the v3.4.0 security notes (partial 2FA tokens, password fingerprint binding, deleted-team checks, report reassignment) and the v3.4.1 Next.js hotfix are the relevant items and are covered by UMAMI-P3-002 and UMAMI-P3-003. Password fingerprint binding (`pwd: hash(user.password)` in login, checked in `checkAuth`) is already present in the fork.
Impact:   None outstanding from this advisory in the deployed code, provided the deployed build matches `729c0cf` (Vercel deployment meta confirms that SHA).
Fix:      No code change. After upgrading, run `src/lib/auth.test.ts` to keep the share-token type check covered.
Related:  UMAMI-P3-001, UMAMI-P3-013

### UMAMI-P3-006  Configuration inventory: Vercel holds only DATABASE_URL, DIRECT_DATABASE_URL and APP_SECRET
Severity: info
Effort:   XS (<1h)
Confidence: confirmed (reproduced)
Where:    Vercel dashboard > umami-analytics > Settings > Environment Variables; next.config.ts; scripts/check-env.js; src/lib/*
Evidence: `filter_project_envs` returns exactly three variables, all `sensitive`/`secret`, target `production` only, created 2026-08-29 by renskeuzekamp: `APP_SECRET`, `DATABASE_URL`, `DIRECT_DATABASE_URL`. Every variable the code reads (grep of `process.env.*` in `src`, `scripts`, `next.config.ts`) and what it does to security or data:
- `DATABASE_URL` (required; `check-env` fails the build without it): Postgres connection for the app. Also the fallback signing secret if `APP_SECRET` is absent (`src/lib/crypto.ts:57`). Set.
- `DIRECT_DATABASE_URL`: non-pooled connection used only by `prisma migrate deploy` at build (`scripts/check-db.js:84`). Set.
- `APP_SECRET`: HMAC key for login JWTs, share tokens, collect cache tokens and the `uuid()` session hash. Rotating it logs everyone out, invalidates all share tokens and changes every session hash going forward. Set.
- `DISABLE_LOGIN`: hides the login pages (`src/app/login/page.tsx:7`); `/api/auth/login` still exists. Unset (correct).
- `CLOUD_MODE`, `CLOUD_URL`: Umami Cloud behaviour (trusts `x-umami-client-*` headers for IP and geo, rewrites `/script.js` to cloud.umami.is, skips 2FA). Unset (correct; setting it would let any client spoof its IP).
- `PRIVATE_MODE`: disables telemetry pixel and update checks. Unset.
- `DISABLE_TELEMETRY`, `DISABLE_UPDATES`: see UMAMI-P3-021. Unset.
- `TRACKER_SCRIPT_NAME`: extra aliases for `/script.js` (ad-block evasion). Unset, so sites must load `/script.js`.
- `COLLECT_API_ENDPOINT`: alias path rewritten to `/api/send`. Unset, so the tracker posts to `/api/send`.
- `TWO_FACTOR_ENCRYPTION_KEY`: see UMAMI-P3-009. Unset.
- `REDIS_URL`: server-side sessions, rate limiting, caching. Unset (see UMAMI-P3-010, UMAMI-P3-011).
- `REMOVE_TRAILING_SLASH`: path normalisation at collect time only. Unset.
- `SALT_ROTATION`: `day|week|month`, how often the session-hash salt changes (default `month`). Unset.
- `IGNORE_IP`: comma list / CIDR to drop (e.g. office IPs). Unset.
- `DISABLE_BOT_CHECK`: keeps bot traffic. Unset (correct).
- `CLIENT_IP_HEADER`, `SKIP_LOCATION_HEADERS`, `GEOLITE_DB_PATH`, `GEO_DATABASE_URL`, `MAXMIND_LICENSE_KEY`, `SKIP_BUILD_GEO`: IP and geolocation sources. Unset; on Vercel the `x-vercel-ip-*` headers are used first, MaxMind file as fallback.
- `FORCE_SSL`: adds HSTS from the app. Unset; Vercel already sends HSTS on every deployment, so no gap.
- `ALLOWED_FRAME_URLS`: CSP `frame-ancestors` for embedding share pages. Unset (`'self'` only, correct).
- `API_URL`, `BASE_PATH`, `CORS_MAX_AGE`, `DEFAULT_LOCALE`, `DEFAULT_CURRENCY`, `FAVICON_URL`, `LINKS_URL`, `PIXELS_URL`, `TRACKER_SCRIPT_URL`, `UMAMI_SELF_TRACK`, `UMAMI_SELF_RECORD`, `USE_UUIDV7`, `LOG_QUERY`, `ENABLE_TEST_CONSOLE`, `DISABLE_UI`, `SKIP_DB_CHECK`, `SKIP_DB_MIGRATION`, `DATABASE_TYPE`, `DATABASE_REPLICA_URL`, `CLICKHOUSE_URL`, `KAFKA_*`: cosmetic, build, or alternative-backend options; all unset, none needed.
`DISABLE_LOGIN=1` would not protect the admin: it only removes the UI; API login keeps working. A default admin password (UMAMI-P3-008) is the real exposure.
Impact:   The must-set list for Vercel is exactly what is set. Everything else is hardening (UMAMI-P3-009, -010, -011, -021) rather than breakage.
Fix:      Keep as is; add the variables named in the related findings.
Related:  UMAMI-P3-008, UMAMI-P3-009, UMAMI-P3-010, UMAMI-P3-011, UMAMI-P3-021

### UMAMI-P3-007  The umami.is docs could not be read from the audit container
Severity: info
Effort:   XS (<1h)
Confidence: confirmed (reproduced)
Where:    audit environment network policy
Evidence: `getaddrinfo ENOTFOUND umami.is` when fetching `https://umami.is/docs/environment-variables`. The env-var inventory in UMAMI-P3-006 was therefore built from the code (`grep process.env` across `src`, `scripts`, `next.config.ts`) rather than the docs. GitHub (release notes, advisories) was reachable.
Impact:   Any env var the docs describe but the code no longer reads would be missed; the code is the authority for what the deployed build honours, so this is low risk.
Fix:      None required. Cross-check the list once against https://umami.is/docs/environment-variables when convenient.
Related:  UMAMI-P3-006

### UMAMI-P3-008  Default `admin` / `umami` account is seeded by the schema; whether the password was changed is unverified
Severity: high
Effort:   XS (<1h)
Confidence: suspected
Where:    prisma/migrations/01_init/migration.sql:199; dashboard: analytics.rensranks.dev > Settings > Users
Evidence: The initial migration inserts user `admin` (id `41e2b680-648e-4b09-bcd7-3e2b10c06264`, role `admin`) with the bcrypt hash of the documented default password `umami`. Nothing in the fork alters this seed or adds a first-run forced change. There is no self-registration: `POST /api/users` requires `canCreateUser()` which is `user.isAdmin` (`src/permissions/user.ts:3`), and no signup page exists. `/api/auth/login` is public and unthrottled (UMAMI-P3-011). The audit did not attempt a login against production, so the current password is unknown.
Impact:   If the seed password is still in place, anyone can log in as admin, read analytics for every portfolio site, create share links, delete websites or reset data. Because the instance is shared, this is a portfolio-wide exposure.
Fix:      Log in to the dashboard, open Settings > Profile and confirm the admin password was changed (or run `select username, updated_at from "user"` against the database and check `updated_at` differs from the migration time). If not changed, change it now, then create a second admin under a personal username and keep `admin` only as a break-glass account.
Related:  UMAMI-P3-009, UMAMI-P3-011

### UMAMI-P3-009  Two-factor auth cannot be enabled: TWO_FACTOR_ENCRYPTION_KEY is not set
Severity: medium
Effort:   XS (<1h)
Confidence: likely (read in code, not run)
Where:    src/lib/two-factor/crypto.ts; src/app/api/auth/login/route.ts:42-44; Vercel env list
Evidence: 2FA secrets are encrypted at rest with `TWO_FACTOR_ENCRYPTION_KEY` (64-hex, per `docker-compose.yml:12`). The variable is absent from the Vercel project. `isTwoFactorConfigured()` is false, so setup returns an error and login for a 2FA user returns 503. The admin account is therefore protected by a single password on an internet-facing login with no rate limit.
Impact:   Shared infrastructure for five properties guarded by one factor.
Fix:      Generate `openssl rand -hex 32`, add it as `TWO_FACTOR_ENCRYPTION_KEY` (production, sensitive) on Vercel, redeploy, enrol the admin account(s). Do this after the v3.4.1 upgrade (UMAMI-P3-003), not before.
Related:  UMAMI-P3-003, UMAMI-P3-008, UMAMI-P3-011

### UMAMI-P3-010  Sessions are stateless JWTs with no server-side revocation (no Redis)
Severity: low
Effort:   S (half day)
Confidence: likely (read in code, not run)
Where:    src/app/api/auth/login/route.ts:52-58; src/lib/auth.ts:33-46; src/app/api/auth/logout/route.ts
Evidence: Without `REDIS_URL`, login returns `createSecureToken({ userId, role, pwd }, secret())` with no expiry (`createSecureToken` is called without `expiresIn`). `checkAuth` validates by signature and by `pwd === hash(user.password)`. Logout only clears the client. A token is valid until the user's password changes or `APP_SECRET` rotates.
Impact:   A token captured from a laptop or browser stays usable indefinitely; the only remediation is a password change or secret rotation (which also kills every share link and cache token).
Fix:      Smallest: rotate the admin password after any suspected exposure, and treat a password change as the revocation procedure. Better: provision a small Redis (Upstash via the Vercel marketplace), set `REDIS_URL`; sessions then live in Redis with `saveAuth()` and can expire and be deleted.
Related:  UMAMI-P3-008, UMAMI-P3-011

### UMAMI-P3-011  No rate limiting on /api/auth/login and no Vercel firewall rules
Severity: medium
Effort:   XS (<1h)
Confidence: likely (read in code, not run)
Where:    src/app/api/auth/login/route.ts; src/lib/redis.ts:72; Vercel dashboard > umami-analytics > Firewall
Evidence: Login compares bcrypt and returns 401 on failure with no counter. The only rate limiter in the fork is for 2FA verification (`src/lib/two-factor/rate-limit.ts`) and the generic `redis.rateLimit()` which is inert without Redis. `get_firewall_config` for the project returns `404 not_found` (no custom WAF configuration). CORS on `/api/*` is `Access-Control-Allow-Origin: *` (`next.config.ts:70`), which is required for `/api/send` but also applies to login; harmless because auth is bearer-token not cookie, but it means a browser from any origin can drive the login endpoint.
Impact:   Online password guessing against `admin` is bounded only by Vercel function throughput, which the attacker also pays for in your invocation quota.
Fix:      In Vercel Firewall add a rate-limit rule: path `/api/auth/login`, 10 requests per minute per IP, action deny; optionally a second rule for `/api/2fa/*`. Keep `/api/send` unthrottled or very generous (legit sites post several events per page view).
Related:  UMAMI-P3-008, UMAMI-P3-009, UMAMI-P3-010

### UMAMI-P3-012  Account model: admin-only user creation, teams unused, no self-registration
Severity: info
Effort:   XS (<1h)
Confidence: likely (read in code, not run)
Where:    src/permissions/user.ts:3, src/permissions/team.ts:18, src/app/api/users/route.ts; dashboard: Settings > Users, Settings > Teams
Evidence: Users are created only by an admin via `POST /api/users` (password min 8 chars, usernames lower-cased since migration 24). Teams exist (`canCreateTeam` for admins or roles with `teamCreate`); team join codes are shareable links that add members. Whether any teams or extra users exist in production could not be checked without database access.
Impact:   Correct defaults. A stale team join code or a forgotten extra user would be the only drift.
Fix:      In the dashboard confirm Settings > Users lists only expected accounts and Settings > Teams is empty or expected; regenerate any team access code you do not recognise.
Related:  UMAMI-P3-008

### UMAMI-P3-013  Share links expose a website's full stats to anyone with the URL; inventory unverified
Severity: low
Effort:   XS (<1h)
Confidence: suspected
Where:    src/app/api/share/[slug]/route.ts:152-205; src/permissions/website.ts:13-20; dashboard: each website > Settings > Share
Evidence: `GET /api/share/<slug>` is unauthenticated and returns a signed share token for the website; the token then satisfies `canViewWebsite` for all read endpoints in the share context. Slugs are 16 random characters by default (`getRandomChars(16)`) but a user can choose a custom slug (`slug: z.string().max(100).optional()`). Shares persist until deleted and have no expiry. Migration `15_add_share` means the table exists; row count unknown.
Impact:   A guessable or leaked slug publishes traffic, referrers, countries and custom events for that site. Not sensitive personal data, but commercially sensitive (traffic volumes per property site, campaign performance).
Fix:      Open each website's Settings > Share and delete any share you did not create deliberately. Prefer the random slug; never use a dictionary slug. Consider `ALLOWED_FRAME_URLS` unset (it is) so share pages cannot be framed elsewhere.
Related:  UMAMI-P3-005

### UMAMI-P3-014  Database host, hosting tier and backup policy could not be identified
Severity: medium
Effort:   XS (<1h)
Confidence: suspected
Where:    Vercel env `DATABASE_URL` / `DIRECT_DATABASE_URL` (secret, not decrypted by the audit); Vercel dashboard > Storage / Integrations
Evidence: The engine is Postgres (the only Prisma schema and migrations in the repo are `prisma/migrations` for PostgreSQL; `check-db.js` queries `server_version_num` and requires >= 9.4). The presence of both a pooled `DATABASE_URL` and a `DIRECT_DATABASE_URL` matches the pattern of Neon or Supabase pooler plus direct connection. None of the three Supabase projects on the account (`domundos-agents`, `seo-platform`, `Beyond Agents`, all eu-west-1) contains Umami tables (`website`, `session`, `website_event`); `seo-platform` only has `umami_daily_stats` / `umami_daily_metrics`, which are downstream aggregates. The Vercel integrations list returned 403 for the token used, so a Neon/Vercel Postgres attachment could not be confirmed. Migrations in repo: 24 (`01_init` to `24_lowercase_username`); whether all 24 are applied is unknown (the build would have failed otherwise, so likely yes).
Impact:   Without knowing the provider, nothing can be said about backups, point-in-time recovery, region, or what happens when a free-tier instance is suspended for inactivity. Losing the database loses all analytics history for every site.
Fix:      In Vercel, open the project's Storage tab (or decrypt `DATABASE_URL` locally with `vercel env pull`) and record the provider, plan, region and backup retention in `docs/audits/`. If it is a free tier with no automated backups, either upgrade to a plan with daily backups or add a weekly `pg_dump` to object storage.
Related:  UMAMI-P3-015, UMAMI-P3-017, UMAMI-P3-020

### UMAMI-P3-015  Vercel build runs `prisma migrate deploy` against production; a deployment rollback does not roll back the schema
Severity: medium
Effort:   S (half day)
Confidence: likely (read in code, not run)
Where:    package.json:14 (`build` script), scripts/check-db.js:79-90
Evidence: `pnpm build` = `check-env build-db check-db ...`; `check-db` connects to `DATABASE_URL`, checks the version, then `execSync('prisma migrate deploy')` using `DIRECT_DATABASE_URL` unless `SKIP_DB_MIGRATION` is set. Every Vercel build therefore migrates production during the build step. Env vars are production-only, so a preview build currently fails at `check-env`, which is a useful accidental guard: adding `DATABASE_URL` to the preview target would make preview builds migrate production.
Impact:   Rolling back on Vercel to the previous deployment ("Instant Rollback") leaves the newer schema in place. For v3.4.1 the two new migrations are additive (new `annotation` and `api_key` tables), so the old code keeps working, but future upstream migrations may not be.
Fix:      Before each upgrade take a database snapshot or `pg_dump` (UMAMI-P3-014). Never add `DATABASE_URL` to the preview environment; if previews are wanted, point them at a separate database. Consider `SKIP_DB_MIGRATION=1` at build and running `pnpm update-db` (or `db:migrate` after upgrade) deliberately from a laptop with `DIRECT_DATABASE_URL`, so migration and deploy are separate steps you control.
Related:  UMAMI-P3-014, UMAMI-P3-020

### UMAMI-P3-016  Data retention: events are kept forever; no purge, no retention setting
Severity: medium
Effort:   S (half day)
Confidence: likely (read in code, not run)
Where:    prisma/schema.prisma (`session`, `website_event`, `event_data`, `session_data`); src/queries/sql/*
Evidence: No code path deletes events by age. Deletion exists only per website (reset / delete website) and per session (`sessionDeletionEnabled` when relational-only). Umami has no scheduler, and no Vercel cron is configured (no `vercel.json` in the fork; upstream's only sets the install command). The table set is pseudonymous (see UMAMI-P3-022) but still "personal data" under GDPR in the sense that `session_id` links page views within a month.
Impact:   Unbounded growth of the Postgres instance (cost, query latency on dashboards) and no defensible storage-limitation period for the privacy policy.
Fix:      Decide a retention period (e.g. 26 months, GA4's maximum, or 14 months) and run a monthly SQL job: `delete from website_event where created_at < now() - interval '26 months'`, then `event_data`, `session_data`, and orphaned `session` rows. Run it from n8n or a Vercel cron hitting a small protected route; document the period in the sites' privacy notices.
Related:  UMAMI-P3-014, UMAMI-P3-022

### UMAMI-P3-017  Production is a single 66-day-old CLI deploy from a laptop; no Git-triggered builds
Severity: medium
Effort:   XS (<1h)
Confidence: likely (read in code, not run)
Where:    Vercel dashboard > umami-analytics > Deployments and Settings > Git
Evidence: `list_deployments` shows exactly three production deployments, all on 2026-08-29 between 14:04 and 14:08 UTC, all `source: cli`, creator renskeuzekamp. The first two (`ca661c7`) are `ERROR` (the Biome VCS crash the local commit fixes), the third (`729c0cf`) is `READY` and current. Nothing has been deployed since. `get_project` shows framework `nextjs`, Node `24.x`, region `iad1`, no custom build or install command (defaults: `pnpm install` with the committed `pnpm-lock.yaml` v9, `next build`). Upstream v3.4.1 adds `vercel.json` pinning `pnpm@12.3.4` and an `engines.pnpm` field, which the fork does not yet have. SSO protection is enabled for all non-custom-domain URLs (preview and `*.vercel.app` need a Vercel login; `analytics.rensranks.dev` is public). Password protection and trusted IPs are off (correct for a public collect endpoint).
Impact:   Security patches depend on someone remembering to run `vercel --prod` from a machine with the repo checked out at the right commit. The deployed artefact is whatever was in that working tree, not necessarily what is on `master`.
Fix:      Connect the Vercel project to the GitHub repo (Settings > Git) with production branch `master`, so a merge deploys. Keep deploy previews off or without database variables (UMAMI-P3-015). After the upgrade, Node 22.x matches upstream's pinned runtime; 24.x also works but is untested upstream.
Related:  UMAMI-P3-015, UMAMI-P3-020

### UMAMI-P3-018  Every portfolio site depends on this one project; outage behaviour on the sites is unverified
Severity: medium
Effort:   S (half day)
Confidence: suspected
Where:    Vercel dashboard > Domundos team > Settings > Billing (spend management); each site's tracker snippet
Evidence: The team's billing API returned `costs_not_found`, so plan and spend limit could not be read; Umami itself has no cron and no background jobs, so cost is driven purely by function invocations from `/api/send` plus bandwidth for `/script.js` (cached 24h by `Cache-Control: public, max-age=86400`). If the project is paused (manually, by a spend limit, or on Hobby by exhausted quota), Vercel returns an error page for `/script.js` and `/api/send`. The tracker is a `<script defer src=...>`; a failed load fires `onerror` and does not block rendering, but `window.umami` is then undefined, so any site code calling `umami.track(...)` without a guard throws. The tracker itself swallows fetch failures (`src/tracker/index.ts:390-420` catches). None of the five sites' snippets were inspected in this pass.
Impact:   Best case a paused Umami silently drops analytics. Worst case a site's custom-event call throws in a form handler or checkout. Either way nobody is alerted.
Fix:      On the sites, wrap custom calls as `window.umami?.track(...)`. On Vercel set a spend limit with "pause" off for this project (or notify-only) and add an uptime check on `https://analytics.rensranks.dev/api/heartbeat` (public, returns `{ok:true}`). Put the tracker on a dedicated hostname already (done: `analytics.rensranks.dev`) so a move to another host is a DNS change.
Related:  UMAMI-P3-017

### UMAMI-P3-019  Function region is iad1 (US East) while the audience, the sites and the other databases are in Europe
Severity: low
Effort:   XS (<1h)
Confidence: likely (read in code, not run)
Where:    Vercel dashboard > umami-analytics > Settings > Functions > Region; deployment `dpl_J3Sb2uXKTYFJiwA8SwHnVT6duQbw` `regions: ["iad1"]`
Evidence: The only deployment runs in `iad1`. The portfolio's Supabase projects are all `eu-west-1`. If the Umami database is also in Europe (UMAMI-P3-014), every `/api/send` call does a transatlantic round trip to Postgres, and visitor IPs from EU users are processed in the US before being hashed.
Impact:   Added latency per event (not user-visible, the tracker is async) and a GDPR transfer footnote: the raw IP is processed, not stored, in a US function. Low risk, easy to remove.
Fix:      Set the function region to `fra1` or `dub1` to match the database region, redeploy.
Related:  UMAMI-P3-014, UMAMI-P3-022

### UMAMI-P3-020  Upgrade plan to upstream v3.4.1 with rollback
Severity: info
Effort:   S (half day)
Confidence: likely (read in code, not run)
Where:    repo; Vercel project; database
Evidence: Local delta is 2 lines in `biome.json`; upstream `biome.json` at v3.4.1 still has `vcs.enabled: true`, so the merge will conflict only if upstream touched those lines (it did not: `git diff v3.3.1 v3.4.1 -- biome.json` is empty apart from unrelated keys; verify at merge time). Upstream changes that affect this deployment: Next 16.4.2 / React 19.3.1; Prisma 7.10; pnpm pinned to 12.3.4 (`engines.pnpm`, `vercel.json` install command `npx --yes pnpm@12.3.4 install --frozen-lockfile`); npm scripts renamed from `build-db` style to `build:db` style; new build steps `build:openapi` and `build:packages`; migrations `25_add_annotation` and `26_add_api_key` (additive); MCP endpoint off by default (`MCP_ENABLED` unset keeps `/mcp` disabled); API keys feature (new `api_key` table, keys hashed).
Impact:   Steps:
1. Snapshot the database (provider snapshot or `pg_dump` with `DIRECT_DATABASE_URL`). Note the current deployment id `dpl_J3Sb2uXKTYFJiwA8SwHnVT6duQbw` as the rollback target.
2. `git remote add upstream https://github.com/umami-software/umami.git && git fetch upstream --tags`.
3. `git checkout -b upgrade/v3.4.1 master && git merge v3.4.1`. Expect no conflicts; if `biome.json` conflicts, keep `vcs.enabled: false` and `useIgnoreFile: false` (the Vercel CLI fix). Alternatively drop the local change entirely once deploys come from Git (UMAMI-P3-017), since a Git-integrated build has `.git`.
4. `pnpm install --frozen-lockfile` with pnpm 12.3.4 (`corepack use pnpm@12.3.4`), `pnpm audit --prod`, `pnpm test`, and `pnpm build` locally with `SKIP_DB_CHECK=1 SKIP_BUILD_GEO=1` to confirm the build passes without touching the database.
5. On Vercel, confirm the install command is now driven by `vercel.json`, set Node to 22.x to match upstream's pin, and add `TWO_FACTOR_ENCRYPTION_KEY` (UMAMI-P3-009).
6. Merge to `master`; deploy (Git-triggered, or `vercel --prod` from the clean checkout). The build applies migrations 25 and 26.
7. Smoke test: dashboard login, one site's realtime view shows new page views, `/script.js` returns the new tracker, a share link still opens.
Rollback: Vercel > Deployments > Instant Rollback to `dpl_J3Sb2uXKTYFJiwA8SwHnVT6duQbw`. The two new tables can stay (v3.3.1 ignores them); only restore the snapshot if data was damaged. Re-pin Node and the install command if they were changed.
Fix:      Execute the plan above as a single short change window; nothing on the sites needs to change (tracker API is unchanged between 3.3 and 3.4).
Related:  UMAMI-P3-001, UMAMI-P3-002, UMAMI-P3-015, UMAMI-P3-017

### UMAMI-P3-021  Telemetry and update checks phone home to umami.is from builds and from the admin UI
Severity: low
Effort:   XS (<1h)
Confidence: likely (read in code, not run)
Where:    scripts/postbuild.js:5, scripts/telemetry.js:10, src/app/api/scripts/telemetry/route.ts, src/store/version.ts:20, src/lib/constants.ts:18-19
Evidence: With `DISABLE_TELEMETRY` unset: every build POSTs version, Node, OS and `is_ci` to `https://api.umami.is/v1/telemetry`; the dashboard loads `/telemetry.js`, which embeds a pixel from `https://i.umami.is/a.png?v=<version>`; and the UI fetches `https://api.umami.is/v1/updates?v=<version>` unless `DISABLE_UPDATES` is set. No site visitor data is included; the tracker script served to the sites does not call umami.is.
Impact:   Minor metadata leak (that this instance exists and which version it runs) and one more third-party host in the admin's browser. Visitors are unaffected.
Fix:      Set `DISABLE_TELEMETRY=1` (keep update checks on, they are useful for noticing releases like v3.4.1).
Related:  UMAMI-P3-006

### UMAMI-P3-022  Privacy posture: cookie-less and IP-free as shipped; nothing in the fork changes it; session replay must stay off
Severity: info
Effort:   XS (<1h)
Confidence: likely (read in code, not run)
Where:    src/tracker/index.ts (no `document.cookie` use; `localStorage` read only for the `umami.disabled` opt-out at :379), src/app/api/send/route.ts:146-160, src/lib/crypto.ts:56-80, prisma/schema.prisma (`session`, `website_event`), prisma/schema.prisma:94 (`recorder_enabled` default false)
Evidence: The tracker sets no cookies and writes nothing to storage; it only reads `localStorage.getItem('umami.disabled')` to honour an opt-out and `navigator.doNotTrack` when the snippet carries `data-do-not-track`. The collect endpoint derives `session_id = uuidv5(websiteId + ip + userAgent + salt)`, where the salt is `hash(startOfMonth(now))` by default (`SALT_ROTATION` unset = `month`), and `visit_id` rotates hourly. Neither `session` nor `website_event` has an IP column; the raw IP is used in memory for the hash and for geolocation (country, region, city from `x-vercel-ip-*` headers, MaxMind fallback) and discarded. `REMOVE_TRAILING_SLASH` is unset (affects path normalisation only). The fork's single local commit does not touch any of this. Two features would change the picture if enabled per website in the dashboard: session replay (`recorder.js`, stores DOM recordings; `recorder_enabled` defaults false) and the `identify()` call with `distinct_id` (links sessions to an identifier the site supplies; the agents portal is the only candidate). Country-level geo and a monthly-rotating pseudonymous id are the common basis for running Umami without a consent banner; a 24-hour or daily salt (`SALT_ROTATION=day`) is the stricter setting some DPAs prefer.
Impact:   The cookie-consent configuration on the sites can continue to classify Umami as a cookie-less, no-personal-data tool provided replay stays off and `identify()` is not used with real user ids on consumer sites.
Fix:      Document in the sites' privacy notices: hashed session id with monthly salt, no IP stored, no cookies, data retention per UMAMI-P3-016. Optionally set `SALT_ROTATION=day` if the sites' consent tool treats monthly linkage as needing consent. Check each website's Settings in the dashboard to confirm "Session replay" is off.
Related:  UMAMI-P3-016, UMAMI-P3-019
