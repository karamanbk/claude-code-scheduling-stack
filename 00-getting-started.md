# 00 — Getting Started

**Start here.** This is the linear path from an empty folder to a running, deployed scheduling platform. Each step links to the deep-dive guide for that topic. Do them in order — later steps assume earlier ones work.

## The 30-second overview

You're building a single-tenant booking platform: leads pick a time, a meeting gets created on a host's calendar with a video link, and the lead is captured, routed, messaged, and synced to your CRM. One company, one deployment, no billing.

## Before you write code

Create accounts (all have free tiers to start):

- **A PostgreSQL host** — e.g. Supabase (also gives you file storage). See [Deployment](./14-deployment.md).
- **A hosting platform** — e.g. Vercel (also runs the cron jobs). See [Deployment](./14-deployment.md).
- **Google Cloud project** — OAuth login + Google Calendar/Meet.
- **Microsoft Entra app** — OAuth login + Outlook Calendar/Teams.
- **A video provider** (optional) — e.g. Zoom.
- **An SMS provider** (optional) — e.g. Twilio, for SMS/WhatsApp workflows.
- **An SMTP provider** — transactional email (confirmations, reminders, verification).
- **An LLM API key** — AI routing, smart-form validation, enrichment.

Credential-by-credential setup is in [Integrations Setup](./13-integrations-setup.md). You don't need them all on day one — wire them as you reach each feature.

## The build order

| Step | Do this | Guide |
|------|---------|-------|
| 1 | Read the big picture | [01 — Architecture & Stack](./01-architecture-and-stack.md) |
| 2 | Bootstrap Next.js + Prisma + DB, get `/api/health` green | [02 — Project Setup](./02-project-setup.md) |
| 3 | Define the schema, migrate | [03 — Data Model](./03-data-model.md) |
| 4 | Add login (credentials first, then Google/Microsoft) | [04 — Authentication](./04-authentication.md) + [13 — Integrations Setup](./13-integrations-setup.md) |
| 5 | Build event types + availability + the slot engine | [05 — Event Types & Availability](./05-event-types-and-availability.md) |
| 6 | Connect a calendar, create real events | [06 — Calendar & Video](./06-calendar-and-video-integrations.md) + [13](./13-integrations-setup.md) |
| 7 | Build the booking page + `POST /api/bookings` | [07 — Booking Flow](./07-booking-flow.md) |
| 8 | Add routing + smart forms | [08 — Routing & Smart Forms](./08-lead-routing-and-smart-forms.md) |
| 9 | Add workflows + the cron job + SMTP/SMS | [09 — Workflows](./09-workflows-and-notifications.md) + [13](./13-integrations-setup.md) |
| 10 | Add CRM sync | [10 — CRM Sync](./10-crm-sync.md) |
| 11 | Make it embeddable | [11 — Embeddable Widgets](./11-embeddable-widgets.md) |
| 12 | Deploy to production | [14 — Deployment](./14-deployment.md) |
| — | When something breaks | [12 — Debugging with Claude Code](./12-debugging-with-claude-code.md) |

**Once the core works, layer in the polish guides as needed:**

| Topic | Guide |
|-------|-------|
| Make it look like your brand | [15 — Branding & White-Labeling](./15-branding-and-white-labeling.md) |
| Lock it down (tokens, PII, rate limits) | [16 — Security & Data Handling](./16-security-and-data-handling.md) |
| Measure conversion & attribution | [17 — Analytics & Reporting](./17-analytics-and-reporting.md) |
| Test the danger zones | [18 — Testing Strategy](./18-testing-strategy.md) |

## Seeing what's going on while you develop

When you start coding, set up these windows so you're never guessing what the app is doing. Keep them open from step 2 onward.

- **The dev server + its console.** Run `npm run dev` and watch the terminal. Server component errors, API route errors, and your `console.log`/`console.error` lines all print here. This is your primary feedback loop — most backend bugs show up in this terminal first.
- **A type-check watcher.** In a second terminal run `npx tsc --noEmit --watch`. It catches schema/type drift the instant it happens — far faster than discovering it at runtime. Treat a red type check as a failing build.
- **The database, visually.** `npx prisma studio` opens a browser UI to inspect and edit rows. After every action (sign up, create event type, book), look at the tables to confirm the data is what you expect. Indispensable while building bookings/contacts.
- **The browser DevTools.** The **Network** tab shows each API call, its status, and its response body — the fastest way to see *why* a request failed. The **Console** shows client-side errors and `postMessage` traffic (critical for debugging embeds).
- **The health check.** `GET /api/health` returning `{ ok: true }` is your "is the app + DB alive" canary. Hit it whenever something feels wrong before digging deeper.
- **Verify each milestone, don't assume it.** After each step in the table above, *prove* it works: see the row in Prisma Studio, see the calendar event actually appear, see the email land, see the reminder fire. The [definition of done](#a-definition-of-done-for-the-mvp) below is the checklist.

> When something breaks, copy the **real error** from the dev-server terminal or the Network tab and hand it to Claude Code — not a paraphrase. The [Debugging guide](./12-debugging-with-claude-code.md) is built around this: find the true error first, fix the root cause, verify.

## A "definition of done" for the MVP

You can call it working when:

- [ ] A team member can log in and create an event type.
- [ ] They can connect a calendar and set availability.
- [ ] A visitor can open a booking link, see correct slots in their timezone, and book.
- [ ] The booking appears on the host's calendar with a video link.
- [ ] The visitor and host get a confirmation email.
- [ ] A reminder fires before the meeting (cron working).
- [ ] The booking can be rescheduled and cancelled (calendar updates/deletes).
- [ ] The flow works embedded in an iframe on another site, with UTMs preserved.

## How to drive this with Claude Code

- Hand Claude Code **one guide section at a time**, not a whole guide.
- After each slice: run `tsc --noEmit`, run the app, verify behavior, commit.
- Keep the [Debugging guide](./12-debugging-with-claude-code.md) handy — feed real errors, not descriptions.
- Don't wire every integration before the core works. Get booking working with one calendar provider, then expand.

Next: [01 — Architecture & Stack](./01-architecture-and-stack.md).
