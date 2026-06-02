# 12 — Debugging with Claude Code

Building the platform is one thing; keeping it working is another. This guide is a practical workflow for finding and fixing bugs with Claude Code, drawn from the kinds of issues this stack actually produces.

## The golden rule: find the real error first

Most wasted debugging time comes from guessing. Before changing any code, get the **actual error text** — the HTTP status, the stack trace, the rejected payload. Then hand that to Claude Code.

A weak prompt: *"the CRM sync isn't working, fix it."*

A strong prompt: paste the real error —

```
CRM sync failed (400): Property values were not valid:
[{ "error": "INVALID_URL", "name": "meeting_link" }]
Attempted properties: { "meeting_link": "2026-04-23T14:15:00.000Z", ... }
```

The second prompt is solvable in one shot: a datetime was mapped to a URL field. The first leads to flailing.

> **Make your code tell you the truth.** When an external call fails, log the status, the full response body, and the payload you sent. The CRM and calendar guides both stress this — those error bodies name the exact offending field.

## A repeatable loop

1. **Reproduce.** Get a deterministic way to trigger the bug (a specific URL, a specific record, a curl command). Intermittent bugs become tractable once reproducible.
2. **Locate.** Ask Claude Code to trace the flow: *"trace how a booking's redirect URL is built, from the API down to where it's sent."* Let it map the path before proposing fixes.
3. **Diagnose.** Share the real error + the relevant code. Ask for the **root cause**, not just a patch. ("Why is this empty?" beats "make this not empty.")
4. **Fix the smallest thing.** Prefer a targeted change over a rewrite. Ask Claude Code to change one thing and explain why.
5. **Verify.** Run the type checker, run the app, reproduce the original trigger. Confirm the fix and that nothing nearby broke.
6. **Commit** the working fix with a message that records the root cause.

## Prompts that work well

- *"Trace where X comes from and where it's used. Don't change anything yet — just report the path with file:line."* (Use a read-only exploration first; decide after.)
- *"Here's the exact error and the payload. What's the root cause?"*
- *"Fix only the failing property; don't touch the rest of the flow."*
- *"This worked last week. Show me the commits that touched this file since then."* (Regressions often have a culprit commit.)
- *"Add logging that prints the status, response body, and the request payload, so we can see what the API rejected."* (Then reproduce, read the log, fix.)

## Stack-specific gotchas (and how to spot them)

These recur in this exact architecture. Knowing them turns a long hunt into a quick check.

### Prisma client out of sync
**Symptom:** `Unknown argument 'newField'` even though the column exists.
**Cause:** schema pushed to the DB but the generated client wasn't regenerated, or the dev server cached the old client at boot.
**Fix:** `prisma generate`, then **restart the dev server**. Ask Claude Code to confirm the field exists in the generated types.

### OAuth token not refreshed
**Symptom:** an integration works for ~an hour, then 401s.
**Cause:** an endpoint reads the stored access token directly instead of going through the refreshing helper.
**Fix:** route every external call through `getValidToken`. Search for direct token reads.

### Cross-origin / www cookie mismatch
**Symptom:** logged-in users get bounced to login, or the session "doesn't stick" on `www.`.
**Cause:** the session cookie is set on the apex domain but the request came in on `www.` (or vice versa).
**Fix:** redirect `www` → apex in middleware; keep all callback/redirect URLs on one canonical host.

### Params lost in an embed
**Symptom:** UTMs/query params don't reach the redirect when embedded.
**Cause:** code tries to read the parent URL from inside the iframe (cross-origin blocked), or a refactor stopped forwarding params.
**Fix:** the embed script (on the parent) must forward params into the iframe `src`; the iframe reads them from its **own** `window.location.search`. Verify each hop.

### Whole external upsert fails for one bad field
**Symptom:** nothing syncs to the CRM; one property is invalid.
**Cause:** CRMs reject the entire payload if any property is invalid.
**Fix:** parse the bad property from the error, drop it, retry (bounded). Don't fall back to "standard fields only" — that silently drops the fields you care about.

### Serverless fire-and-forget gets killed
**Symptom:** side effects (emails, sync) run locally but not in production.
**Cause:** async work wasn't awaited before the response returned; the function froze.
**Fix:** await side effects, or hand them to a durable queue/cron.

### Timezone / DST slot bugs
**Symptom:** slots off by an hour, or missing/duplicated around DST.
**Cause:** mixing local and UTC, or naive date math across a DST boundary.
**Fix:** store UTC, convert at the edges, and unit-test `computeSlots` across DST and across visitor/host timezone gaps.

## When Claude Code proposes a fix

- Ask it to explain the root cause in one or two sentences. If it can't, it's probably guessing — get more error detail.
- Prefer fixes that add a guard or fix the source over fixes that paper over a symptom downstream.
- Watch for "retry forever" or "swallow the error" patterns; bound retries and keep logging.
- After the fix, ask: *"what else calls this code path that might be affected?"*

## Keeping future debugging cheap

- Log external failures with full context (status, body, payload).
- Keep pure logic (slot math, routing decisions) in testable functions separate from I/O.
- Commit small, working increments so `git bisect` / "what changed" is meaningful.
- Write the root cause into the commit message — future-you will search for it.

## Working with Claude Code

Treat Claude Code as a fast, tireless investigator: have it trace and explain before it edits, feed it real errors rather than descriptions, and make it fix one root cause at a time. The better your logging, the shorter every one of these loops gets.
