# Belvue Umami deployment

This repository is **Belvue Christian Church's pinned fork** of
[Umami](https://github.com/umami-software/umami) for the church's self-hosted analytics.

- Upstream: `umami-software/umami`
- Pinned tag: **v3.5.0**
- Deploy branch: **`production`**
- Host: **Vercel Hobby**
- Database: **Neon Free PostgreSQL** (US East, `aws-us-east-1`)
- Public URL: `https://stats.belvuechristianchurch.com`
- Church site integration: `braydonsprojects/belvue-christian-church-site`
  (`docs/analytics.md`, `docs/plans/umami-rollout.md`)

The parent `README.md` is upstream's. This file is the runbook for this deployment.

**Do not edit Umami application code in this fork.** The fork exists so the deployed version is
pinned and so the database backup workflow lives next to the database. Upgrades are upstream tag
merges; anything else makes the next merge harder.

## Vercel project settings

Import `braydonsprojects/belvue-umami` as a Vercel project with:

| Setting | Value |
| --- | --- |
| Framework Preset | Next.js |
| Root Directory | `.` |
| Install Command | `npx --yes pnpm@12.3.4 install --frozen-lockfile` (already in `vercel.json`) |
| Build Command | `pnpm build` (auto-detected) |
| Output Directory | `.next` (auto-detected) |
| Node.js Version | 22.x |
| Production Branch | `production` |

`vercel.json` also sends `X-Robots-Tag: noindex` on every response, so the analytics host is not
indexed. The church site is the only public origin that should appear in search.

## Environment variables

Set these in Vercel → Project → Settings → Environment Variables. They are **not** committed.

| Name | Value |
| --- | --- |
| `DATABASE_URL` | Neon **pooled** connection string, with `sslmode=require&pgbouncer=true` |
| `DIRECT_DATABASE_URL` | Neon **direct** connection string, with `sslmode=require`; used for Prisma migrations |
| `APP_SECRET` | `openssl rand -hex 32`; unique, never rotate casually (it signs sessions) |
| `DISABLE_TELEMETRY` | `1` |
| `FORCE_SSL` | `1` |
| `NODE_VERSION` | `22` |

If the first build fails on a Prisma/PgBouncer prepared-statement error, point `DATABASE_URL` at
the **direct** string and redeploy. At the church's traffic level, connection count is not a
problem. Record the outcome here.

## First boot

1. Deploy. The build creates the Umami tables.
2. Open the deployment URL. Sign in `admin` / `umami`; **change the password immediately**.
3. Add the website `belvuechristianchurch.com`.
4. Create a view-only user (`belvue-site`).
5. Create a team (`Belvue`), transfer the website to it, and add the view-only user with the
   `team-view-only` role. Without the team the view-only API key sees no websites.
6. Sign in as the view-only user and create an API key. Store it in the church password manager;
   it becomes `UMAMI_API_KEY` on the church site.
7. Add `stats.belvuechristianchurch.com` to the Vercel project. At the DNS host:
   CNAME `stats` → `cname.vercel-dns.com`, **DNS only** (grey cloud at Cloudflare).
8. Confirm `https://stats.belvuechristianchurch.com/api/heartbeat` is 200 and the root is 200, not
   401. Production must not be behind Vercel Authentication.

## Backups

`.github/workflows/backup.yml` runs monthly (`0 6 1 * *`) and on demand. It dumps the database with
`postgres:17`'s `pg_dump` and uploads the custom-format dump as a 90-day workflow artifact.

- Repository secret: `NEON_DIRECT_URL` (the direct Neon string, not the pooled one).
- Run it once manually after setup, download the dump, and restore it into a throwaway Neon branch
  to prove the procedure.

Restore, using a `postgres:17` image or a matching client:

```bash
pg_restore --clean --no-owner --no-acl --dbname "$NEW_DATABASE_URL" umami.dump
```

Then point `DATABASE_URL` and `DIRECT_DATABASE_URL` at the restored database and redeploy.

## Upgrades

1. Take a backup (run the workflow or `pg_dump`).
2. Fetch the upstream tag and merge it into `production`:

   ```bash
   git fetch upstream --tags
   git merge vX.Y.Z
   ```

3. Push `production`. Vercel builds a preview.
4. Against a Neon branch (or with the dump ready), check `/api/heartbeat`, sign in, record one
   pageview, and open one dashboard report.
5. Promote the preview to production. Record the new tag here and in the commit message.
6. If it fails, promote the previous Vercel deployment. Restore the dump only if a migration
   changed data.

Scheduled GitHub workflows are disabled after 60 days without repository activity. If the monthly
backup stops, run it manually and re-enable the workflow in the Actions tab.

## Local verification

The church site repository has a local Umami stack (`docker-compose.umami.yml`) for testing the
tracker and API without touching this deployment. See its `docs/plans/umami-rollout.md`.

## Security

- Never commit `DATABASE_URL`, `DIRECT_DATABASE_URL`, `APP_SECRET`, or the API key.
- The Vercel project is public-facing but the dashboard requires login.
- Use the view-only API user for the church site's traffic page; it cannot administer Umami.
- Keep the admin password and the view-only key in the church password manager.
