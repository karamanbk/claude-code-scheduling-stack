# 17 — Analytics & Reporting

This guide adds the metrics that tell you whether the platform is actually generating and converting leads. Most of it is read-only aggregation over data you already store.

## What to measure

| Metric | How it's derived |
|--------|------------------|
| **Total bookings** | count of `Booking` |
| **Bookings this week/month** | count where `createdAt` in range |
| **Confirmed / cancelled** | count by `status` |
| **No-show rate** | bookings marked no-show ÷ past confirmed bookings |
| **Conversion rate** | bookings ÷ routing-form starts (or page views) |
| **Bookings by event type** | group by `eventTypeId` |
| **Lead source (UTM)** | group `Contact` by `utmSource` / `utmCampaign` |
| **Routing outcomes** | how many leads each rule sent where |

## Tracking the denominators

Counts of bookings are easy — you already have them. The valuable ratios need a **denominator** you must capture deliberately:

- **Form starts** — to compute conversion, record when someone *starts* a routing/booking form, not just when they finish. A lightweight `FormStart` row (or an event log) keyed by rule/event type + timestamp + UTM.
- **No-shows** — add a way to mark a past booking as a no-show (manual toggle, or infer from a CRM/calendar signal). Without it, no-show rate is unknowable.

```prisma
model FormStart {
  id          String   @id @default(uuid())
  eventTypeId String?
  routingRuleId String?
  utmSource   String?
  utmCampaign String?
  createdAt   DateTime @default(now())
}
```

## Computing metrics

Keep aggregation in the database where possible — `groupBy` and `count` are far faster than pulling rows into Node.

```ts
// bookings by event type
const byType = await prisma.booking.groupBy({
  by: ["eventTypeId"],
  _count: { _all: true },
  where: { createdAt: { gte: start, lte: end } },
});

// lead source breakdown
const bySource = await prisma.contact.groupBy({
  by: ["utmSource"],
  _count: { _all: true },
  where: { createdAt: { gte: start, lte: end } },
});
```

For ratios, fetch the numerator and denominator counts and divide in the handler. Guard against divide-by-zero.

## The analytics page

A dashboard page (admin or per-host) that shows:

- Headline cards: total bookings, this week, conversion rate, no-show rate.
- A bookings-over-time chart (group by day/week).
- A table of bookings by event type.
- A UTM source/campaign breakdown.
- Optionally, per-host performance for round-robin teams.

Add a date-range filter; default to last 30 days. Scope by host for non-admins if you want per-rep views.

## Attribution: make UTMs flow end-to-end

Analytics is only as good as the data captured. The booking, routing, and embed guides all stress forwarding `utm_*` and other tracking params from the first touch through to the `Contact`. Confirm the chain:

```
landing page (?utm_source=...) → embed script forwards params
  → iframe/booking page reads params → Contact stores utmSource/Medium/Campaign
  → analytics groups by them, CRM syncs them
```

If a refactor breaks any hop, attribution silently goes blank — test it end-to-end (the [Debugging guide](./12-debugging-with-claude-code.md) covers this exact failure mode).

## Performance

- Add DB indexes on the columns you filter/group by most (`Booking.createdAt`, `Booking.eventTypeId`, `Contact.utmSource`).
- For large datasets, precompute daily rollups in a nightly cron rather than aggregating raw rows on every page load.

## Exporting

Offer CSV export of bookings/contacts for a date range — teams will want to pull data into spreadsheets or BI tools. Stream it from a server route; don't build the whole file in memory for large exports.

## Working with Claude Code

Start with the headline counts (they need no new tables). Add the `FormStart` capture and no-show flag before promising conversion/no-show ratios. Push aggregation into Prisma `groupBy`. Add indexes once you see which queries the analytics page actually runs (ask Claude Code to check the generated SQL or add `prisma` query logging).
