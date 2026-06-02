# 01 — Architecture & Stack

This guide describes the overall shape of the application before you write any code. Understanding the architecture first makes every later guide easier to implement with Claude Code.

## Tech stack

| Layer | Choice | Why |
|-------|--------|-----|
| Framework | **Next.js (App Router)** | Server components for data-heavy pages, API routes for the backend, one deployable |
| Language | **TypeScript** | Type safety across the data model and API |
| ORM | **Prisma** | Declarative schema, generated client, easy migrations |
| Database | **PostgreSQL** | Relational data fits scheduling well (bookings, contacts, relations) |
| Auth | **NextAuth / Auth.js** | OAuth providers + credentials, JWT sessions |
| Styling | **Tailwind CSS** | Fast, consistent UI |
| Background jobs | **Cron + a queue** | Reminders and scheduled workflows |
| LLM | **An LLM API** | AI routing, smart-form validation, contact enrichment |

## High-level architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Next.js app                          │
│                                                          │
│  ┌────────────────┐   ┌──────────────┐  ┌────────────┐  │
│  │  Dashboard     │   │ Public        │  │  API routes │ │
│  │  (admin UI)    │   │ booking pages │  │  (/api/*)   │ │
│  │  server comps  │   │ + embeds      │  │             │ │
│  └────────────────┘   └──────────────┘  └─────┬───────┘ │
│                                                 │         │
└─────────────────────────────────────────────────┼───────┘
                                                  │
        ┌──────────────┬──────────────┬──────────┴───────┐
        │              │              │                   │
   ┌────▼────┐   ┌─────▼─────┐  ┌─────▼──────┐    ┌───────▼──────┐
   │ Postgres │   │ Calendar  │  │   CRM      │    │  LLM / SMTP  │
   │ (Prisma) │   │ providers │  │ provider   │    │  / SMS       │
   └──────────┘   └───────────┘  └────────────┘    └──────────────┘
```

## The two surfaces

There are two distinct UI surfaces, and keeping them separate matters:

1. **The dashboard** — authenticated admin UI where your team configures event types, availability, routing, workflows, and integrations. Mostly server components reading from the database.

2. **The public booking surface** — unauthenticated pages that leads actually interact with: the booking page, the routing form, and the embeddable widget. These must be fast, work cross-origin in an iframe, and never leak admin data.

In the Next.js App Router, model this with route groups:

```
src/app/
  (dashboard)/        # authenticated admin pages
  (booking)/          # public booking + routing + embed pages
  (auth)/             # login, signup, password reset
  api/                # all backend endpoints
```

## Request flow for a booking

1. A lead opens a booking page (direct link or embedded iframe).
2. The page loads the event type config and computes **available slots** (availability rules minus existing bookings minus calendar busy times).
3. The lead picks a slot and submits the form.
4. The API creates a `Booking`, creates the calendar event (with a conferencing link), and records the `Contact`.
5. Post-booking side effects fire: confirmation email, workflow scheduling, CRM sync, optional enrichment.

## Single-tenant simplification

A multi-tenant version of this app would have organizations, members, roles, invitations, billing, and per-tenant isolation on every query. **We drop all of that.**

For single-tenant:

- There is **one** workspace. Store global config in a single `Settings` row (or environment variables).
- `User` records are just the people on your team who can host meetings and log into the dashboard.
- Every query is implicitly scoped to "the one workspace," so you don't thread a `tenantId` through everything.
- No billing, plans, trials, or seat counting.

This removes a large amount of incidental complexity. The next guide sets up the project; the data-model guide shows the simplified schema.

## A note on working with Claude Code

When you start implementing, give Claude Code this architecture doc as context and ask it to scaffold the route groups and a health-check endpoint first. Verify the app boots before adding the data model. Build outward from a running skeleton.
