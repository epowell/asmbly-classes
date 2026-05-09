## Plan: Integrate Pottery Shelves into Asmbly Classes Site

Rewrite the pottery-shelves Next.js/React app as a SvelteKit sub-route (`/pots`) within the asmbly-classes project. The pottery data lives in a **separate PostgreSQL database** on the same server (full isolation). All ~15 API routes and 3 pages are rewritten as Svelte components and SvelteKit endpoints. Auth reuses the existing Neon CRM flow; styles adopt Tailwind v3 + DaisyUI conventions.

### Steps

1. **Create a second Prisma client for pottery data isolation.** Add a second Prisma schema at `prisma/pots/schema.prisma` defining `shelves`, `assignments`, `current_assignments` (as a view), `settings`, and `change_log` tables—mirroring the Supabase schema and its migrations. Create a new `src/lib/pots/postgres.js` exporting a dedicated `PrismaClient` configured via a `POTS_DATABASE_URL` env var, keeping it fully separate from the classes `prisma` instance.

2. **Add the `/pots` route group with its own layout.** Create `src/routes/(pots-app)/pots/+layout.svelte` with a simplified nav (ceramics wordmark, back-to-classes link) using DaisyUI classes. This layout **does not import or reference** any classes data models. Add sub-routes: `+page.svelte` (login), `member/+page.svelte`, and `admin/+page.svelte`, rewriting the React components (`MemberFlow.tsx`, `AdminDashboard.tsx`) as Svelte with reactive `$:` blocks replacing `useState`/`useMemo`.

3. **Rewrite API routes as SvelteKit server endpoints.** Create `src/routes/(pots-app)/pots/api/` mirroring the pottery API structure (`shelves/+server.js`, `claim/+server.js`, `vacate/+server.js`, `check-member/+server.js`, `member/*/+server.js`, `admin/*/+server.js`). Each imports only from `$lib/pots/postgres.js` and `$lib/pots/` helpers—never from `$lib/postgres.js`. Port the Neon CRM lookup from `lib/neon.ts` into `$lib/pots/neon.js`, reusing env vars already available to the classes app.

4. **Port email and changelog helpers.** Add `resend` to `package.json` dependencies. Rewrite `lib/email.ts` and `lib/changelog.ts` into `$lib/pots/email.js` and `$lib/pots/changelog.js`, using the pots Prisma client for changelog inserts and removing all Supabase/Next.js imports.

5. **Integrate auth with existing Neon session system.** Pottery admin auth should reuse the classes app's `validateSessionToken` from `hooks.server.js`. Member auth (email-only Neon CRM check) needs no session—just the API call. Add an admin allow-list in the pots `settings` table. Update `hooks.server.js` to pass `event.locals` through to `/pots` routes without granting access to class Prisma models.

6. **Adapt styles to the classes site design system.** Replace pottery's zinc-950 dark theme and `#b24a9a` accent with DaisyUI's `ceramics` color token (`#b34a9a` already defined in `tailwind.config.js`) and `base-100`/`base-content` semantic classes. Use DaisyUI `btn`, `card`, `input`, and `alert` components instead of raw Tailwind utility classes.

### Further Considerations

1. **Dependency version conflicts**: The classes app uses Tailwind v3 and ESLint v8; pottery uses Tailwind v4 and ESLint v9. Since we're rewriting into the classes stack, no new Tailwind or ESLint dependencies change—only `resend`, a second `@prisma/client` output (via `--output` flag), and optionally `@prisma/client` generator renaming are needed.

2. **Database provisioning in Docker Compose**: The existing `compose.yaml` Postgres container serves one database. Add an init script (`docker-entrypoint-initdb.d`) to create the `pots` database with a restricted user that has **no access** to the classes database, and expose `POTS_DATABASE_URL` as a separate env var.

3. **Cron jobs for pottery (audit, scheduled exports)**: The classes app has a `cron-service` container. Should pottery cron tasks (membership audit, roster export) be added there as additional jobs, or run as SvelteKit API cron endpoints hit by an external scheduler? Adding to cron-service is simpler but requires it to also have `POTS_DATABASE_URL`.

---

### Pottery Cron Jobs

The pottery-shelves app has three data-refresh jobs that mirror the classes cron-service pattern. All three are added **inside the existing `cron-service`** as a `pots/` subdirectory, using a **separate Prisma schema and client** pointed at `POTS_DATABASE_URL`. They never touch the classes database.

#### New jobs

| Job | Schedule | Trigger source |
|---|---|---|
| `potsMembershipAudit` | Configurable (daily/weekly/monthly, per `app_settings`) | AWS EventBridge rule, `cronType: "potsMembershipAudit"` |
| `potsRosterExport` | Configurable (weekly/monthly, per `app_settings`) | AWS EventBridge rule, `cronType: "potsRosterExport"` |
| `potsAutoVacateLapsed` | Weekly (fixed, Sunday 08:00 CT) | AWS EventBridge rule, `cronType: "potsAutoVacateLapsed"` |

#### `potsMembershipAudit` (`cron-service/pots/membershipAudit.js`)

Mirrors the existing `/api/admin/audit` endpoint logic, but runs unattended:

1. Read `app_settings` row — skip execution if `audit_auto_run` is `false`.
2. Check whether today matches the configured `audit_frequency` + `audit_day_of_week`/`audit_day_of_month` to avoid running on off-days when the EventBridge trigger fires daily.
3. Query `current_assignments` for all non-vacated rows; deduplicate by `member_email`.
4. Call the Neon CRM API (`getMemberByEmail`) for each unique email — using the same `NEON_API_KEY` / `NEON_ORG_ID` SSM params already fetched for the classes Lambda, adding `/pots-db/dsn` as a new SSM path.
5. Write the resolved `neon_membership_status` back to the `assignments` table via the pots Prisma client.
6. Update `app_settings.last_audit_at`.
7. If `audit_auto_email` is `true` and `audit_recipients` is non-empty, build a CSV of the full roster and send it via Resend (reusing `RESEND_API_KEY` from SSM; add `/resend/api_key` as a new SSM path).

#### `potsRosterExport` (`cron-service/pots/rosterExport.js`)

1. Read `app_settings` — skip if `export_auto_email` is `false`.
2. Check `export_frequency` + `export_day_of_week`/`export_day_of_month` against today.
3. Query the full `shelves` table joined with `current_assignments` view.
4. Build a CSV (unit, slot, type, status, member name, email, claimed date, membership status).
5. Email the CSV to `export_recipients` via Resend; update `app_settings.last_export_at`.

#### `potsAutoVacateLapsed` (`cron-service/pots/autoVacateLapsed.js`)

Handles automatic cleanup of shelves held by members whose membership has lapsed beyond a configurable grace period (stored as `lapsed_grace_days` in `app_settings`, default 5):

1. Query all current assignments where `neon_membership_status IN ('Lapsed', 'Inactive')` and `last_audit_at` is older than `lapsed_grace_days` days ago.
2. For each matching assignment, mark it vacated (`vacated_at = NOW()`, `vacated_by = 'system'`).
3. Insert a `change_log` row recording the auto-vacate action.
4. Send a `sendLapsedNotice` email to the member via Resend.
5. Log a summary of vacated slots to stdout (visible in AWS CloudWatch via the existing log group).

#### Infrastructure changes

**`cron-service/pots/prismaClient.js`** — a second Prisma client reading `POTS_DATABASE_URL`:

```js
import { PrismaClient } from '@prisma/pots-client'  // separate generator output
export const potsPrisma = new PrismaClient()
```

**`cron-service/prisma/pots/schema.prisma`** — mirrors the pottery Prisma schema (shelves, assignments, app_settings, change_log) with `output = "../../../node_modules/@prisma/pots-client"`.

**`cron-service/package.json`** — add `resend` dependency; add a second `prisma generate --schema prisma/pots/schema.prisma` step to the Dockerfile `RUN` layer.

**`cron-service/handler.js`** — add three new `cronType` branches and fetch `/pots-db/dsn` + `/resend/api_key` from SSM alongside the existing parameters.

**`cron-service/crontab.txt`** (Docker Compose / local dev mode only) — add daily trigger lines; in production these are EventBridge rules:

```
0 6 * * * node /usr/src/app/pots/membershipAudit.js > /proc/1/fd/1 2>&1
0 6 * * * node /usr/src/app/pots/rosterExport.js > /proc/1/fd/1 2>&1
0 8 * * SUN node /usr/src/app/pots/autoVacateLapsed.js > /proc/1/fd/1 2>&1
```

**AWS SSM** — add two new Parameter Store paths:
- `/pots-db/dsn` — connection string for the `pots` Postgres database
- `/resend/api_key` — Resend API key (if not already shared with the SvelteKit app env)

**Isolation guarantee** — the pots Prisma client is generated into a distinct output path (`@prisma/pots-client`) so the classes `PrismaClient` import (`@prisma/client`) and the pots one can never be confused at the module level. The pots database user in Postgres has `CONNECT` privilege only on the `pots` database and no grants on the classes database schema.


