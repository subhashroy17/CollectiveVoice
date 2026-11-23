# CollectiveVoice

Live meeting Q&A with accounts, guest participation, question grouping, voting, polls, moderation, reports, and an admin panel. Built with Node.js, WebSockets, Prisma, and browser JavaScript modules.

## Run locally

Requires Node.js 18 or newer and npm.

```powershell
npm install
Copy-Item .env.example .env   # Only if you do not already have .env
npm run db:local
npm run db:seed:local        # Optional demo content
npm run dev
```

Open http://localhost:3000. The demo login is `ananya.sharma@email.com` / `password123`; live meeting code: `482916`.

`npm run dev` always uses `.local/dev.db`. It does not connect to the PostgreSQL URL in your `.env`. `npm start` uses the configured database instead. To use a different local port, set `$env:PORT='3100'` before starting.

The SQLite schema and client are generated from `prisma/schema.prisma`; do not switch the source provider manually. Re-run `npm run db:local` after schema changes, with the local server stopped. Generated clients and databases are ignored by Git. Seeding is optional; normal sign-up and meeting creation work with an empty database.

## Verify

```powershell
npm run check
npm run build
npm test
```

Tests create an isolated SQLite database for each suite, with demo fixtures. They never use your configured application database. Coverage includes accounts, refresh rotation and revocation, meeting APIs, database transactions, report permissions/exports, concurrent voting, WebSocket identity and meeting isolation, and clustering fallback behavior.

The default suite disables model downloads. Real MiniLM checks are opt-in and require network access or a cached model:

```powershell
npm run test:nlp:semantic
npm run test:clustering:integration
npm run test:e2e:integration
```

`NLP_DISABLED=1` explicitly enables the lightweight text-similarity fallback. Otherwise the app attempts to load MiniLM on first use and falls back if unavailable.

## Features and routes

- Accounts: `#/login`, `#/signup`, `#/forgot`, `#/reset`, `#/settings`.
- Meetings: `#/meetings`, `#/meetings/create`, `#/join` (code or camera QR scan).
- Live views: `#/audience`, `#/moderator`, `#/speaker`.
- Activity and reports: `#/activity`, `#/report/<meetingId>`.
- Admin: `/admin.html`, requiring an admin or superadmin account.

Report CSV and JSON exports download stored meeting data with owner/admin authorization. PDF uses the browser print dialog: choose **Save as PDF**. Reports are visible to authenticated meeting members. Settings lists sign-in sessions and can revoke renewal; already-issued access tokens expire after their configured lifetime (one hour by default). Native two-factor authentication is not implemented and is not presented as an available toggle.

## Optional services

Google sign-in requires real `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, and `GOOGLE_CALLBACK_URL` values. Leave them empty to disable it. Password-reset email requires SMTP settings and `APP_URL`; development logging is not an email delivery service. Microsoft and GitHub sign-in are not implemented.

`SUPERADMIN_EMAIL` promotes the matching account through the existing authentication flow. Use your own email for deployment. Never use demo credentials in a production database.

## PostgreSQL deployment

Set `NODE_ENV=production`, `DATABASE_URL`, `DIRECT_URL`, `JWT_SECRET`, `ALLOWED_ORIGINS`, `APP_URL`, and optionally `SUPERADMIN_EMAIL` in the deployment environment. Use a strong unique JWT secret. `DIRECT_URL` is the direct migration connection; `DATABASE_URL` may be pooled. Neon hosts use the serverless adapter; ordinary PostgreSQL uses Prisma's native connection.

For a **new, empty PostgreSQL database**:

```powershell
npm ci
npm run build
npx prisma migrate deploy
npm start
```

The Render blueprint runs generation at build time and applies migrations before startup. `/health` is its health check. No deployment or remote database migration is performed by local development or tests.

**Existing databases created with `db push`:** do not reset them or blindly apply the initial migration. Back up the database and compare it with `prisma/schema.prisma`. After reconciling the schema, mark `20261025000000_postgres_baseline` as applied using `prisma migrate resolve --applied 20261025000000_postgres_baseline`, then use `migrate deploy` for later migrations. If the database already has migration history, reconcile that history before switching to the new baseline. The original SQLite SQL files are preserved in `prisma/legacy-sqlite-migrations` for reference; they cannot be applied to PostgreSQL.

Production PostgreSQL, external OAuth, SMTP delivery, and downloaded-model behavior need verification in the target environment. The local test suite does not substitute for those integration checks.
