# 14 — Deployment

This guide deploys the platform to production using **Vercel** (hosting + cron) and **Supabase** (Postgres + file storage). Other hosts/databases work too; the concepts transfer.

## Database & storage: Supabase

1. Create a **Supabase** project. Pick a region close to your users/host.
2. **Database → Connection string.** You get two URLs you care about:
   - A **pooled** connection (for the running app — serverless functions open many short-lived connections).
   - A **direct** connection (for migrations).
3. Map them to env:

```bash
DATABASE_URL=<pooled connection string>
DIRECT_URL=<direct connection string>
```

   Use the pooled URL as `DATABASE_URL` and the direct URL as `DIRECT_URL`. Migrations need the direct connection; the app should use the pooled one to avoid exhausting connections.
4. **Run migrations against production** from your machine (or CI):

```bash
npx prisma migrate deploy
```

   Use `migrate deploy` (not `migrate dev`) in production — it applies existing migrations without prompting.

### File storage (logos, avatars)

If you store uploads (company logo, user avatars):

1. Create a **Storage bucket** in Supabase (public for logos/avatars, or signed URLs for private files).
2. Add the Supabase URL + service key to env and upload from a server route.

```bash
SUPABASE_URL=https://YOUR-PROJECT.supabase.co
SUPABASE_SERVICE_KEY=...        # server-side only, never expose to the client
```

### Auth URL settings (if you use Supabase Auth anywhere)

If you rely on Supabase for any auth-issued links/redirects, set **Site URL** to `https://yourdomain.com` and add redirect URLs `https://yourdomain.com/**` and `http://localhost:3000/**`. Note: if you use NextAuth for login (recommended in this build), Supabase is just your database/storage and these settings only matter for Supabase-issued flows.

---

## Hosting & cron: Vercel

1. Push your repo to GitHub and **import the project into Vercel.**
2. **Environment Variables:** add every var from `.env.local` to the Vercel project (Production + Preview). At minimum:
   - `NEXT_PUBLIC_APP_URL=https://yourdomain.com`
   - `NEXTAUTH_URL=https://yourdomain.com`
   - `NEXTAUTH_SECRET`, `DATABASE_URL`, `DIRECT_URL`
   - `CRON_SECRET`
   - All provider keys you've configured (Google, Microsoft, Zoom, Twilio, SMTP, LLM).
3. Deploy. Vercel builds and hosts the Next.js app.

### Make Prisma generate on build

Ensure the Prisma client is generated during the build (otherwise the deployed app uses a stale/missing client). Add it to the build:

```jsonc
// package.json
{
  "scripts": {
    "build": "prisma generate && next build",
    "postinstall": "prisma generate"
  }
}
```

### Cron jobs

Vercel runs scheduled functions defined in `vercel.json`. Register the workflow runner and any token/webhook refreshers:

```jsonc
// vercel.json
{
  "crons": [
    { "path": "/api/cron/run-workflows", "schedule": "* * * * *" },
    { "path": "/api/cron/refresh-tokens", "schedule": "*/30 * * * *" },
    { "path": "/api/cron/renew-webhooks", "schedule": "0 */6 * * *" }
  ]
}
```

Protect every cron endpoint with the `CRON_SECRET` bearer check (see the workflows guide). Vercel sends the configured secret; reject anything else so the endpoints can't be triggered publicly.

> **Heads-up on cron frequency and serverless limits:** very frequent crons and long-running jobs can hit plan limits and function timeouts. Keep each cron run bounded (process N due jobs per tick, not all of them) so a backlog can't time out.

### Long-running / reliable background work

Serverless functions are short-lived. For work that must not be dropped (e.g. precise reminders, retries), prefer the **cron + a `ScheduledWorkflowJob` table** pattern (durable: state lives in Postgres, the cron just drains due rows). If you need finer-grained delays than cron, add a managed queue/scheduler that calls your endpoints — but the DB-backed cron pattern covers most needs.

---

## Domains & DNS

1. Add your domain in Vercel and follow its DNS instructions.
2. **Pick one canonical host** — apex (`yourdomain.com`) or `www`, not both. This build redirects `www` → apex in middleware so session cookies (set on the apex) always match. Keep all OAuth redirect URIs and `NEXTAUTH_URL` on that same canonical host.
3. Verify HTTPS is active before testing OAuth — providers reject non-HTTPS redirect URIs in production.

## Post-deploy checklist

- [ ] `https://yourdomain.com/api/health` returns `{ ok: true }`.
- [ ] Login works for credentials + each OAuth provider (redirect URIs match production exactly).
- [ ] A real booking creates a calendar event with a working video link.
- [ ] Confirmation email arrives (and links point to the production URL, not localhost).
- [ ] A scheduled reminder actually fires (cron is running — check function logs).
- [ ] `www` redirects to apex; sessions persist across navigations.
- [ ] Embedding on a separate test page works and preserves UTMs.
- [ ] Robots/SEO: add a `robots.txt` that disallows dashboard/api/booking-internals if you don't want them indexed.

## Updating production safely

- Run `prisma migrate deploy` as part of (or just before) each deploy that changes the schema.
- After adding a Prisma field, the build's `prisma generate` keeps the client in sync — but if you ever see `Unknown argument` errors, that step was skipped.
- Roll out behind a branch/preview deploy first; Vercel preview URLs let you test against production data settings without touching the live domain.

## Working with Claude Code

Ask Claude Code to: add the `prisma generate` build/postinstall scripts, write `vercel.json` with the cron entries, and add the `CRON_SECRET` guard to each cron route. Then deploy and walk the post-deploy checklist together — paste any failing logs straight into Claude Code (see the [Debugging guide](./12-debugging-with-claude-code.md)).
