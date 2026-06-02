# 07 — The Booking Flow

This guide builds the public booking experience and the API that creates bookings.

## The booking page

A public, unauthenticated page at `/book/[eventTypeId]` (or by slug). It:

1. Loads the event type (404 if inactive).
2. Shows host info, title, duration, location.
3. Fetches slots from `/api/availability` and renders a calendar + time picker.
4. After a slot is chosen, shows the form (`formFields`).
5. On submit, calls `POST /api/bookings`.

Keep it a server component for the initial load (fast, SEO-able) and hydrate the interactive calendar/form as client components.

### Pre-filling and UTM capture

Read query params on load and pre-fill matching form fields. Capture `utm_*` (and other tracking params) so they flow into the `Contact` and, later, the CRM. Preserve these through the whole flow so they survive to the thank-you/redirect step.

## Creating a booking

`POST /api/bookings` is the critical path. Do this in a careful order:

```ts
// pseudo-flow
1. Validate input (slot still available? required fields present?)
2. Re-check availability server-side (never trust the client's slot)
3. Pick the host (single, or round-robin selection)
4. Create the Booking row (status CONFIRMED or PENDING)
5. Create the calendar event + conferencing link
6. Persist meetingUrl / provider / calendar event id on the Booking
7. Upsert the Contact and link via BookingInvitee
8. Fire side effects (see below)
9. Return the booking + redirect/thank-you info
```

### Side effects (do them after the booking is safely created)

- **Confirmation email** to the invitee (and optionally the host).
- **Schedule workflows** (reminders, follow-ups) — see the workflows guide.
- **CRM sync** — push the contact + meeting (see the CRM guide).
- **Enrichment** — optionally enrich the contact via the LLM.

Run side effects so that a failure in one doesn't roll back the booking. On serverless platforms, make sure async work is **awaited** (or handed to a durable queue) — fire-and-forget often gets killed when the response returns.

```ts
try {
  await syncToCrm(booking.id, contact.id);
} catch (err) {
  console.error("CRM sync failed (non-fatal):", err);
}
```

## Confirmation, reschedule, cancel

- **Confirmation screen:** show the meeting details + add-to-calendar.
- **Reschedule:** `/book/[eventTypeId]/reschedule/[bookingId]?token=...` reuses the picker; on submit it updates the existing calendar event and booking.
- **Cancel:** `/cancel/[bookingId]?token=...` sets status `CANCELLED`, deletes the calendar event, and cancels any pending workflow jobs.

Always validate the `rescheduleToken` before allowing changes — it's the only auth the public has.

## The thank-you / redirect step

Two options after booking, configured per event type:

- **Internal confirmation screen** (default).
- **Redirect** to `thankYouRedirectUrl`.

If `passParamsToThankYou` is on, append the form data + booking details to the redirect URL. **Always forward tracking params** (UTMs and other query params captured at the start) to the redirect even when full param-passing is off — marketing attribution depends on it.

When the page is **embedded in an iframe**, you can't navigate the parent directly; post a message to the parent instead (see the embeds guide).

## Round-robin selection

```ts
function pickHost(hosts, availabilityByHost) {
  const eligible = hosts.filter((h) => availabilityByHost[h.userId]?.length);
  if (eligible.length === 0) return null;
  // weighted least-recently-assigned
  return eligible.sort((a, b) => assignedCount(a) / a.weight - assignedCount(b) / b.weight)[0];
}
```

If the chosen host turns out to have no free slot at booking time (race), retry with the next eligible host rather than erroring.

## Working with Claude Code

Build the read path first (page loads, slots render) and confirm slots look right in multiple timezones. Then build `POST /api/bookings` and verify each step persists correctly. Add side effects last, one at a time, each wrapped so it can't break the core booking. Test the full reschedule and cancel paths — they're easy to forget and easy to break.
