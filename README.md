# Building a Lead-Generation Scheduling Platform with Claude Code

This is a step-by-step guide series for building a self-hosted, **single-tenant** lead-generation scheduling platform from scratch using [Claude Code](https://claude.com/claude-code). Think of it as your own internal alternative to Calendly + a lightweight lead router, fully owned by your company.

> **Single-tenant by design.** Every guide assumes one company runs one deployment. There is no billing, no subscription tiers, and no multi-workspace/tenant management. If you later need multi-tenancy, you can layer it on — but starting single-tenant keeps the data model and auth dramatically simpler.

## What you'll build

- A booking platform where leads pick a time and book a meeting
- Calendar sync (Google Calendar, Microsoft Outlook) with conferencing (Google Meet, Zoom, Teams)
- Availability rules, buffers, round-robin assignment
- A lead **routing** engine (rule-based and AI-assisted) that sends the right lead to the right person or event type
- **Smart forms** that validate submissions and enrich contact data
- **Workflows** that send reminders and follow-ups by email/SMS
- **CRM sync** to push contacts and meetings into your CRM
- **Embeddable widgets** so you can drop the booking flow onto any website

## The guides

**New here? Start with [00 — Getting Started](./00-getting-started.md)** — it's the linear, do-this-then-that path through everything below.

| # | Guide | What it covers |
|---|-------|----------------|
| 00 | [Getting Started](./00-getting-started.md) | The quickstart path + MVP definition of done |
| 01 | [Architecture & Stack](./01-architecture-and-stack.md) | The big picture, tech choices, folder layout |
| 02 | [Project Setup](./02-project-setup.md) | Bootstrapping Next.js, Prisma, the database, env vars |
| 03 | [Data Model](./03-data-model.md) | The Prisma schema, single-tenant simplifications |
| 04 | [Authentication](./04-authentication.md) | NextAuth with Google, Microsoft, and email/password |
| 05 | [Event Types & Availability](./05-event-types-and-availability.md) | Bookable meeting templates and availability rules |
| 06 | [Calendar & Video Integrations](./06-calendar-and-video-integrations.md) | OAuth, free/busy lookups, conferencing links |
| 07 | [The Booking Flow](./07-booking-flow.md) | Slot generation, the booking page, confirmations |
| 08 | [Lead Routing & Smart Forms](./08-lead-routing-and-smart-forms.md) | Rule-based + AI routing, form validation/enrichment |
| 09 | [Workflows & Notifications](./09-workflows-and-notifications.md) | Scheduled emails/SMS, cron jobs |
| 10 | [CRM Sync](./10-crm-sync.md) | Pushing contacts and meetings to a CRM |
| 11 | [Embeddable Widgets](./11-embeddable-widgets.md) | The embed script, iframes, passing params |
| 12 | [Debugging with Claude Code](./12-debugging-with-claude-code.md) | A practical workflow for finding and fixing bugs |
| 13 | [Integrations Setup](./13-integrations-setup.md) | Getting credentials: Google, Microsoft, Zoom, Twilio, SMTP, LLM |
| 14 | [Deployment](./14-deployment.md) | Shipping to production on Vercel + Supabase, cron, domains |
| 15 | [Branding & White-Labeling](./15-branding-and-white-labeling.md) | Logo, colors, fonts, custom thank-you, branded emails |
| 16 | [Security & Data Handling](./16-security-and-data-handling.md) | Token encryption, endpoint auth, rate limiting, PII |
| 17 | [Analytics & Reporting](./17-analytics-and-reporting.md) | Conversion, no-show, UTM attribution, exports |
| 18 | [Testing Strategy](./18-testing-strategy.md) | What to unit/integration/E2E test, and what to skip |

## How to use this with Claude Code

Each guide is written so you can hand a section to Claude Code and say *"implement this."* The recommended loop:

1. Read the guide section yourself first so you understand the intent.
2. Ask Claude Code to implement one slice at a time (a model, an endpoint, a page) — not the whole guide at once.
3. After each slice, run the type checker and the app, and verify behavior.
4. Commit working increments frequently.

Treat these guides as the *spec*, and Claude Code as the implementer. The smaller and more concrete each request, the better the result.

## Prerequisites

- Node.js 20+ and a package manager (npm/pnpm)
- A PostgreSQL database (a managed Postgres provider works well)
- Accounts for any integrations you want: Google Cloud (OAuth + Calendar), Microsoft Entra (OAuth + Calendar), a video provider, an LLM API key for AI features, and an SMTP provider for email
