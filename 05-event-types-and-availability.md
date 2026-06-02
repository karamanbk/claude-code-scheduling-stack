# 05 — Event Types & Availability

This guide builds the two configuration primitives behind every booking: **event types** (what can be booked) and **availability** (when).

## Event types

An event type is a reusable meeting template. Key fields (see the data-model guide):

- `title`, `slug`, `description`, `duration`, `color`
- `bookingMode` — `DIRECT` (lead picks a time immediately) or `ROUTER` (lead fills a form / gets routed first)
- `schedulingType` — `SINGLE` (one host) or `ROUND_ROBIN` (rotate across hosts)
- `locationType` — Meet / Zoom / Teams / Phone / In-person / Custom
- buffers, `minimumNotice`, `schedulingWindow`, `dailyLimit`
- `formFields` — the booking form definition (JSON)
- `internalNote` — a team-only note (never shown to invitees)

### The form-field editor

`formFields` is an ordered array of field definitions:

```ts
type FormField = {
  id: string;        // canonical id; "email", "name", "phone", "company" have special meaning
  type: "text" | "full_name" | "first_name" | "last_name" | "email" | "phone" | "number" | "select" | "company" | "textarea" | "url";
  label: string;
  required: boolean;
  options?: string[]; // for "select"
  order: number;
};
```

Design notes that save you pain later:

- **Don't over-constrain the editor.** Let the admin add any combination of fields. Enforce only that **at least one field exists**. Auto-suffix ids when a semantic type (e.g. `email`) is chosen twice so ids stay unique.
- Map a few **semantic types to canonical ids** (`email`→`email`, `full_name`→`name`, `first_name`→`first_name`, …). Downstream code (CRM mapping, enrichment, workflow variables) keys off these ids.
- Support drag-and-drop reordering; persist `order`.

### Round-robin

For `ROUND_ROBIN`, store eligible hosts in `EventTypeHost` with a `weight`. At booking time, pick the host with availability and the fewest recent assignments (weighted). Always fall back gracefully: if the "next" host has no free slot, try the others rather than failing the booking.

## Availability

Store a weekly schedule plus date overrides as JSON on `AvailabilitySchedule`:

```ts
type AvailabilityRules = {
  // 0 = Sunday … 6 = Saturday
  weekly: { day: number; start: string; end: string }[]; // "09:00"
  overrides?: { date: string; start?: string; end?: string; unavailable?: boolean }[];
};
```

Each user has at least one schedule, in their own timezone. Event types reference a schedule (or default to the host's).

## Computing available slots

This is the heart of the system. Given an event type, a host, and a date range:

1. **Expand availability** into concrete time windows for each day in range, in the host's timezone.
2. **Slice** windows into candidate slots of `duration`, stepping by a slot interval (often the duration or a fixed 15/30 min grid).
3. **Subtract existing bookings** for that host (plus `bufferBefore`/`bufferAfter`).
4. **Subtract calendar busy times** fetched from the host's connected calendar (next guide).
5. **Apply constraints**: `minimumNotice` (drop slots too soon), `schedulingWindow` (drop slots too far out), `dailyLimit` (cap bookings/day).
6. **Convert to the visitor's timezone** for display.

Expose this as `GET /api/availability?eventTypeId=...&startDate=...&endDate=...&timezone=...`. Keep the pure slot math in a testable function separate from the HTTP handler.

```ts
// src/lib/availability.ts
export function computeSlots(input: {
  rules: AvailabilityRules;
  duration: number;
  bufferBefore: number;
  bufferAfter: number;
  minimumNotice: number;
  existingBookings: { start: Date; end: Date }[];
  busyTimes: { start: Date; end: Date }[];
  rangeStart: Date;
  rangeEnd: Date;
  hostTimezone: string;
}): { start: string; end: string }[] {
  // ...pure function, no I/O — easy to unit test
}
```

> **Timezones are where bugs live.** Always store times in UTC, convert at the edges, and test slot generation across DST boundaries and across the visitor/host timezone gap. A date library with timezone support (e.g. `date-fns` with `date-fns-tz`) is worth it.

## Working with Claude Code

Implement `computeSlots` as a pure function first and write a few unit tests (normal day, DST change, fully booked, slot just inside/outside minimum notice). Then wire the availability endpoint, then the event-type CRUD + form editor UI. Having the pure function tested makes the rest far less error-prone.
