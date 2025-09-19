# Stripe Churn Deflection

Recover lost revenue from failed payments and reduce involuntary churn with an opinionated, deployable kit for Stripe-powered apps — automated dunning, checkout & billing portal wiring, and an admin console for triage and recovery.

See `docs/E2E_AND_SELLING.md` for local e2e instructions and selling/pricing notes.

## What’s inside
- Next.js 14 + TypeScript
- Stripe webhooks (invoice.* events), Checkout and Billing Portal endpoints
- Admin dashboard with secure cookie auth (JWT), CSRF for POST, rate limiting
- Audit log model and APIs (list, CSV export, retention)
- Dunning processor with dry‑run and Safe Mode
- Health `/api/health`, Readiness `/api/ready`, Version `/api/version`
- Prisma ORM (SQLite for dev, Postgres recommended in prod)
- CI workflow + Jest tests

[![CI](https://github.com/ShamWuo/stripe-micro-saas/actions/workflows/ci.yml/badge.svg)](https://github.com/ShamWuo/stripe-micro-saas/actions/workflows/ci.yml)

## Setup (Windows PowerShell)
1) Copy env vars
	- Copy `.env.example` to `.env.local` and fill values.

2) Install deps
```
npm install
```

3) Init database
```
npx prisma generate
npx prisma db push
```

Windows/OneDrive note:
- On Windows, developer tools that generate native binaries (Prisma query engine) can hit file locking or EPERM errors when your project lives inside OneDrive. If you see errors during `npx prisma generate` like "EPERM: operation not permitted, rename ... query_engine-windows.dll.node.tmp", try one of the following:
	- Run the generate command from a different folder outside OneDrive and copy the generated `node_modules/.prisma` folder back into the project.
	- Use the Prisma generator setting `engineType = "library"` (already applied in this repo) to reduce binary rename operations.
	- Move the project outside OneDrive while running `npx prisma generate`.
	- Run your dev work in WSL or a Linux/macOS environment for a smoother dev experience.

4) Run dev server
```
npm run dev
```

## Ready for selling — quick checklist

Before you market or accept paid customers, make sure you complete these production steps:

- Use Postgres in production and run `npx prisma migrate deploy` as part of your deploy pipeline.
- Store secrets (DATABASE_URL, STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET, SENTRY_DSN, CRON_SECRET) in a secure secret manager; do not commit them.
- Enable HTTPS and HSTS for your domain and configure your platform to call `/api/ready` as the readiness check.
- Configure billing: create Stripe products/prices, wire webhooks, and provide a trials/plan matrix.
- Add data export and deletion endpoints for GDPR/CCPA compliance and a documented retention policy.

See `PRODUCTION_CHECKLIST.md` for a concise, ordered checklist to prepare a production deploy.

Release checklist (quick)
-------------------------

- Ensure all unit tests and e2e smoke tests pass in CI (`npm run ci` then `npm run e2e` or the Playwright workflow).
- Verify `npx prisma generate` runs in CI (use the Playwright workflow template as a guide).
- Confirm Stripe keys and webhook secret are set in staging and run a full dunning scenario.
- Update `CHANGELOG.md` with release notes and bump `package.json` version.

CHANGELOG
---------

Add a `CHANGELOG.md` file and follow Keep a Changelog conventions. Start with an Unreleased section and list the main productization changes (admin safety, export, e2e, CI).

Developer convenience (Windows PowerShell)

For local E2E runs on port 3100 (helps avoid OneDrive file locking on Windows):

```powershell
$env:ADMIN_SECRET='secret123'; $env:CSRF_SECRET='csrf123'; $env:NEXT_TELEMETRY_DISABLED='1'; npm run dev -- -p 3100
```

Quick Windows helper
--------------------

If you're on Windows and want a small convenience wrapper that sets the recommended
env vars and starts Next on a fixed port, use the PowerShell helper included in
this repo:

```powershell
.\scripts\start-dev-win.ps1 -Port 3100 -AdminSecret 'secret123' -CsrfSecret 'csrf123'
```

OneDrive `.next` / readlink errors
----------------------------------

If Next fails during startup with errors like `EINVAL: invalid argument, readlink` or
Prisma shows `EPERM` while generating binaries, it's often caused by OneDrive
file sync interference. Quick remedies:

- Delete the `.next` folder and restart (the helper above does not remove it).
- Move the repo outside OneDrive while doing first-time builds or run in WSL.
- Run `npx prisma generate` from a non-OneDrive folder and copy `node_modules/.prisma`
	back into place.

These steps are harmless and only affect local developer convenience on Windows.

See `.github/CI_SECRETS.md` for CI and scheduled job secrets required by GitHub Actions.

5) Stripe webhook (optional, for local testing)
	- In another terminal:
```
stripe listen --forward-to localhost:3000/api/stripe/webhook
```

6) Try it
	- Open http://localhost:3000
	- Click the Buy button (requires `STRIPE_PRICE_ID`)
	- Or enter a Stripe customer ID (cus_...) and open the billing portal
	- Admin: visit http://localhost:3000/admin-login, log in using `ADMIN_USER`/`ADMIN_PASS` (or legacy `ADMIN_SECRET`), then go to `/admin`

Admin security notes:
- Admin APIs require an HttpOnly cookie `admin_token` (JWT signed with `ADMIN_SECRET`).
- CSRF is enforced on admin POST endpoints; the admin UI fetches a token from `/api/admin/csrf` and sends it in `x-csrf-token`.
- Rate limiting protects admin endpoints; adjust in `lib/rateLimit.ts` if needed.

Readiness and health:
- `/api/health` returns `{ ok: true }`.
- `/api/ready` checks DB connectivity, Stripe key presence, SMTP config; returns 200 when ready, 503 otherwise.
- `/api/version` returns `{ version, commit }` for release diagnostics.

Dunning:
- The processor lives at `/api/cron/dunning` and is callable securely with `CRON_SECRET`.
- Safe Mode: set `SAFE_MODE=true` to prevent any emails/charges during testing.
- You can also trigger a run from the admin UI and choose Dry Run.

 - Analytics API supports CSV: `/api/analytics/metrics?format=csv` and HEAD requests for cache validation.
 - Update `public/robots.txt`, `public/sitemap.xml`, and `public/.well-known/security.txt` with your real domain and contacts.

## Notes
- Map your real users to `User.stripeCustomerId` for production
- Secure the cron route (HMAC via `CRON_SECRET`) and configure email/Slack for dunning
- Extend webhook handling to create recovery offers and measure recovered revenue

## Admin login

Use `/admin-login` to set the `admin_token` cookie. It accepts `ADMIN_USER`/`ADMIN_PASS` (recommended) or, if not set, a legacy flow where `ADMIN_SECRET` is the password. The cookie lasts 2 hours and is HttpOnly with SameSite=Lax.

## Environment variables

See `.env.example` for all variables, including admin auth, CSRF, Stripe, SMTP/Postmark, Slack, and cron. Optional: set `REDIS_URL` (e.g., redis://localhost:6379) to enable distributed rate limiting/caching for analytics and admin endpoints. For production, set at minimum:

- `DATABASE_URL` (Postgres recommended)
- `ADMIN_SECRET`, `ADMIN_USER`, `ADMIN_PASS`, `CSRF_SECRET`
- `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_ID`
- Email settings (SMTP or Postmark) if you want to send real dunning emails

## CI & tests

## CI & tests

This repo includes a small test suite and a template precompile step.

Local commands:

```powershell
npm install
npm run precompile-templates
npm test
```

On CI we run unit tests and Playwright e2e.
Note: Playwright billing tests are skipped unless Stripe secrets are provided in CI. To enable them, add `STRIPE_SECRET_KEY` and `STRIPE_WEBHOOK_SECRET` as GitHub Actions secrets and set `PLAYWRIGHT_RUN_BILLING=true` in the workflow env.
### End-to-end tests (Playwright)

Local run (stable on Windows OneDrive: run against dev server on port 3100):

```powershell
$env:ADMIN_SECRET='secret123'; $env:CSRF_SECRET='csrf123'; $env:NEXT_TELEMETRY_DISABLED='1'; npm run dev -- -p 3100
# new terminal
$env:E2E_BASE_URL='http://localhost:3100'; $env:E2E_COMMAND='npm run dev -- -p 3100'; npm run e2e:run
```

Headed mode:

```powershell
$env:ADMIN_SECRET='test_admin_secret'
$env:E2E_COMMAND='npm run e2e:start'
npm run e2e:headed
```

Notes:
- `ADMIN_SECRET` enables the admin login e2e via the legacy password flow.
- On Windows/OneDrive, `rimraf`/unlink can fail during builds. Use the split/start-only flow or run against `npm run dev` on a fixed port (e.g., 3100) as above.
- In CI we build, then Playwright starts with `npm run start`.

### CSP hardening
- In production, style inline is disabled (`style-src 'self'`). Scripts use a per-request nonce set by middleware and applied in `_document.tsx`. Remove any inline script usage or add proper nonces.


Environment variables are listed in `.env.example`. Important: set `STRIPE_PRICE_ID` to your created price id (for Checkout) and `STRIPE_WEBHOOK_SECRET` for webhook verification.

## Production notes: Postgres & migrations

This project uses Prisma. For production you should use Postgres (not SQLite).

Quick steps:

1. Provision Postgres and set environment variables:

	 - PRISMA_DB_PROVIDER=postgresql
	 - DATABASE_URL=postgresql://USER:PASS@HOST:5432/DBNAME?schema=public

2. Generate the client and create migrations locally (development):

```powershell
npx prisma generate
npx prisma migrate dev --name init
```

3. Commit the generated migration SQL files, push to your repo, and in
	 production run:

```powershell
npx prisma generate
npx prisma migrate deploy
```

4. Ensure `npx prisma generate` runs as part of your CI/deploy so the
	 generated Prisma client exists in the deployed artifact.

Notes:
- Do not run the app on SQLite in production; it's not safe for concurrent
	writes or backups.
- Protect webhooks and cron endpoints (set `STRIPE_WEBHOOK_SECRET` and
	`CRON_SECRET`) and use `/api/ready` for platform readiness checks.
- Consider setting `SAFE_MODE=true` for the first production deploy while
	you validate behavior, then disable it.

If you want, I can help create a minimal Vercel/Render deploy config and show
how to wire a managed Postgres instance.

## Docker

Build and run locally:

```powershell
docker build -t stripe-churn .
docker run --rm -p 3000:3000 --env-file .env.local stripe-churn
```

CI smoke: a GitHub Actions workflow `docker-build-smoke.yml` will build the production image and run basic readiness/health checks on merges to `main`.

## Postgres & Redis (optional)

Spin up Postgres and Redis locally with docker-compose:

```powershell
docker compose up -d db redis
# Then set DATABASE_URL=postgresql://app:app@localhost:5432/app
# And optionally set REDIS_URL=redis://localhost:6379
npx prisma generate
npx prisma migrate dev --name init
```

Quick docker-compose dev helper

This repo includes `docker-compose.dev.yml` which provides a simple local
Postgres + Redis setup for development. To use it:

```powershell
docker compose -f docker-compose.dev.yml up -d
$env:PRISMA_DB_PROVIDER='postgresql';
$env:DATABASE_URL='postgresql://app:app@localhost:5432/app?schema=public';
npx prisma generate
npx prisma migrate dev --name init
npm run dev
```

When finished:

```powershell
docker compose -f docker-compose.dev.yml down -v
```

## Backfill recovered revenue (optional)

Fetch paid invoices from Stripe and create RecoveryAttribution rows (dry-run by default):

```powershell
set STRIPE_SECRET_KEY=sk_test_...
npm run backfill:recovery
# To write changes:
set BACKFILL_DRY_RUN=false
npm run backfill:recovery
# Filter by date:
set BACKFILL_SINCE=2024-01-01
npm run backfill:recovery
```

