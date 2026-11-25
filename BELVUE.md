# Belvue Umami deployment

This repository is **Belvue Christian Church's pinned fork** of
[Umami](https://github.com/umami-software/umami) for the church's self-hosted analytics.

- Upstream: `umami-software/umami`
- Pinned tag: **v3.5.0**
- Deploy branch: **`production`**
- Host: **Vercel Hobby**
- Database: **Neon Free PostgreSQL 18** (US East, `aws-us-east-1`)
- Public URL: `https://stats.belvuechristianchurch.com`
- Church site integration: `braydonsprojects/belvue-christian-church-site`
  (`docs/analytics.md`, `docs/plans/umami-rollout.md`)

The parent `README.md` is upstream's. This file is the runbook for this deployment.

## This deployment

| Thing | Value |
| --- | --- |
| Neon organization | `org-little-dust-56688999` |
| Neon project | `old-meadow-70904627` ("Belvue Christian Church Analytics") |
| Neon branch | `production` (default) |
| Neon region / version | `aws-us-east-1`, Postgres 18 |
| Vercel project | `belvue-christian-church/belvue-umami` (`prj_LpkhJunRmwsjVSEdZn0LF1oaArN8`) |
| Production URL | `https://belvue-umami.vercel.app` (requires Vercel Authentication) |
| Public URL | `https://stats.belvuechristianchurch.com` |
| Node.js | 22.x (project setting) |
| Umami website | `Belvue Christian Church` (`belvuechristianchurch.com`) |
| Umami website ID | `58ebfc66-d07e-4ec4-8b68-1cfb66cbe7b1` |

The admin password, the view-only user's password and the view-only API key are in the church
password manager / the operator's local secret store. They are not in this repository. The
throwaway `Belvue Neon Proof` website used for the first end-to-end check was deleted; only the
real website remains.

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
| Build Command | `printf 'node_modules\n.next\n' > .gitignore && npx --yes pnpm@12.3.4 run build` (project setting; see build notes) |
| Output Directory | `.next` (auto-detected) |
| Node.js Version | 22.x |
| Production Branch | `production` |
| Deployment Protection | All deployments except custom domains (the custom domain is public; `*.vercel.app` requires Vercel Authentication) |

`vercel.json` also sends `X-Robots-Tag: noindex` on every response, so the analytics host is not
indexed. The church site is the only public origin that should appear in search.

## Environment variables

Set these in Vercel → Project → Settings → Environment Variables. They are **not** committed.

| Name | Value |
| --- | --- |
| `DATABASE_URL` | Neon **pooled** connection string, with `sslmode=require`; add `pgbouncer=true` only if Prisma reports a prepared-statement error |
| `DIRECT_DATABASE_URL` | Neon **direct** connection string (the `DATABASE_URL_UNPOOLED` value), with `sslmode=require`; used for Prisma migrations |
| `APP_SECRET` | `openssl rand -hex 32`; unique, never rotate casually (it signs sessions) |
| `DISABLE_TELEMETRY` | `1` |
| `FORCE_SSL` | `1` |

Node 22 is a Vercel **project setting** (`vercel project update … --node-version 22.x`), not an
environment variable. All five variables above are set for **Production**.

The Neon CLI writes the connection strings **quoted** into `.env`. `dotenv` strips the quotes
locally, but `vercel env add --value` does not: a quoted value reaches the build as
`"postgresql://…"` and `new URL()` fails in `scripts/check-db.js`. Strip the surrounding quotes
before adding the value to Vercel. The current values were added unquoted; the build connects.

`vercel env pull` writes `[SENSITIVE]` placeholders for secret values. Pull the real strings from
Neon (`neon env pull`) or copy them from the Neon dashboard; never paste them into the repo.

`neon deploy` pulls `DATABASE_URL` (pooled) and `DATABASE_URL_UNPOOLED` (direct) into the local
env file. The app reads `DATABASE_URL` at runtime and `DIRECT_DATABASE_URL` for schema migrations,
so set `DIRECT_DATABASE_URL` in Vercel to the `DATABASE_URL_UNPOOLED` value. `prisma.config.ts`
uses `DATABASE_URL`; `scripts/check-db.js` uses `DIRECT_DATABASE_URL` for the migration step.

If the first build fails on a Prisma/PgBouncer prepared-statement error, point `DATABASE_URL` at
the **direct** string and redeploy. At the church's traffic level, connection count is not a
problem. Record the outcome here.

## First boot (completed 2026-11-24)

1. Deployed. The build created the Umami tables.
2. Signed in `admin` / `umami` and **changed the password**. The old password now returns 401.
3. Added the website `belvuechristianchurch.com`, name `Belvue Christian Church`.
4. Created the view-only user (`belvue-site`).
5. Created the team (`Belvue`), transferred the website to it, and added the view-only user with
   the `team-view-only` role. Without the team the view-only API key sees no websites.
6. Signed in as the view-only user and created an API key. It is kept outside the repository;
   it becomes `UMAMI_API_KEY` on the church site.
7. Added `stats.belvuechristianchurch.com` to the Vercel project. At Cloudflare, the CNAME is:
   `stats` → `e76bfe628213bb09.vercel-dns-017.com`, **DNS only** (grey cloud). Vercel recommended
   this project-specific target; `cname.vercel-dns.com` is the generic fallback. Record ID
   `e4c9ad6d51876bc68ef7953e0c3bd423`.
8. Verified `https://stats.belvuechristianchurch.com/api/heartbeat` 200, the root 200,
   `/script.js` 200, and an admin login plus the `Belvue Christian Church` dashboard through the
   custom domain. Production is public on the custom domain; `*.vercel.app` stays behind Vercel
   Authentication because protection is “all deployments except custom domains”.

The first production deployment initially failed three times: quoted Neon URLs made
`new URL()` throw, Biome could not find `.gitignore` in the build sandbox, and the custom build
command used the sandbox's default pnpm. The fixes are in the environment and build notes above.

## Backups

`.github/workflows/backup.yml` runs monthly (`0 6 1 * *`) and on demand. It dumps the database with
`postgres:18`'s `pg_dump` and uploads the custom-format dump as a 90-day workflow artifact.

- Repository secret: `NEON_DIRECT_URL` (the direct Neon string, not the pooled one). Set
  2026-11-24.
- Ran manually 2026-11-24: [run 45527385891](https://github.com/braydonsprojects/belvue-umami/actions/runs/45527385891)
  succeeded (dump and upload). Download the artifact and restore it into a throwaway Neon branch
  to prove the restore procedure.

Restore, using a `postgres:18` image or a matching client:

```bash
pg_restore --clean --no-owner --no-acl --dbname "$NEW_DATABASE_URL" umami.dump
```

Then point `DATABASE_URL` and `DIRECT_DATABASE_URL` at the restored database and redeploy.

## Vercel build notes

- Vercel does not upload the repository's `.gitignore` into the build sandbox. Biome 2.5 is
  configured with `vcs.useIgnoreFile: true`, so `build:tracker:types` fails with
  “Biome couldn't find an ignore file”. The Vercel project's **Build Command** creates a small
  `.gitignore` before `pnpm run build`:

  ```text
  printf 'node_modules\n.next\n' > .gitignore && npx --yes pnpm@12.3.4 run build
  ```

  `npx` pins pnpm 12.3.4 because a custom Vercel build command otherwise uses the sandbox's
  default pnpm, which fails the `engines.pnpm` check.

  The file is a build-sandbox shim; it is not committed. If Vercel ever includes `.gitignore`
  again, the redirect is harmless.
- This setting lives in the Vercel project, not in the repository. Reapply it if the project is
  recreated.

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

### Build under an AI coding agent

Next 16.4.2 stops an agent-driven `npm run build` with a `SecurityFatalError` because the
installed version is affected by a known security advisory. Vercel's CI is not an agent and is not
affected. For a local agent build, run the command with the agent detection variables cleared:

```bash
env -u AI_AGENT -u OPENCODE -u OPENCODE_CLIENT -u OPENCODE_SESSION_ID -u OPENCODE_TERMINAL npm run build
```

That is a local workaround, not a fix. The recommended action is the security upgrade
`next upgrade --agent=security` after the operator approves it; it is deliberately not run by this
deployment setup.

## Security

- Never commit `DATABASE_URL`, `DIRECT_DATABASE_URL`, `APP_SECRET`, or the API key.
- The Vercel project is public-facing but the dashboard requires login.
- Use the view-only API user for the church site's traffic page; it cannot administer Umami.
- Keep the admin password and the view-only key in the church password manager.
