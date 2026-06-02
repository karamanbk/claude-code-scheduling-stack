# 09 — Workflows & Notifications

This guide builds scheduled, event-driven messaging: confirmation emails, reminders, follow-ups — by email or SMS.

## The model

A `Workflow` has a **trigger**, a **time offset**, and an **action**:

- `trigger`: `BOOKING_CREATED`, `BEFORE_EVENT`, `AFTER_EVENT`, `FORM_NO_BOOKING`
- `offsetMinutes`: negative = before the event, positive = after (e.g. `-1440` = 24h before)
- `action`: `EMAIL` or `SMS`, with the message body/subject

Workflows attach to one or more event types via `WorkflowEventType`.

## Two execution styles

1. **Immediate** — send right when the trigger fires (e.g. a confirmation on `BOOKING_CREATED`). Do this inline in the booking side-effects.

2. **Scheduled** — anything time-relative (reminders, follow-ups). When a booking is created, compute each applicable workflow's `sendAt` and insert `ScheduledWorkflowJob` rows. A cron job sends due jobs.

```ts
// when a booking is created
for (const wf of workflowsForEventType) {
  if (wf.trigger === "BEFORE_EVENT" || wf.trigger === "AFTER_EVENT") {
    const sendAt = addMinutes(booking.startTime, wf.offsetMinutes);
    if (isFuture(sendAt)) {
      await prisma.scheduledWorkflowJob.create({ data: { workflowId: wf.id, bookingId: booking.id, sendAt } });
    }
  }
}
```

## The cron job

A single endpoint processes due jobs. Protect it with a shared secret.

```ts
// src/app/api/cron/run-workflows/route.ts
export async function GET(req: Request) {
  const auth = req.headers.get("authorization");
  if (process.env.CRON_SECRET && auth !== `Bearer ${process.env.CRON_SECRET}`) {
    return new Response("Unauthorized", { status: 401 });
  }

  const jobs = await prisma.scheduledWorkflowJob.findMany({
    where: { status: "PENDING", sendAt: { lte: new Date() }, attempts: { lt: 3 } },
    take: 50,
    orderBy: { sendAt: "asc" },
  });

  for (const job of jobs) {
    // Skip if the booking was cancelled, or the workflow deactivated
    // Render variables, send email/SMS, mark SENT (or increment attempts on failure)
  }
  return Response.json({ processed: jobs.length });
}
```

Schedule it with your platform's cron (e.g. a `vercel.json` cron entry, or any external scheduler hitting the URL on an interval). Run frequently (every minute) for timely reminders.

> **Cancellation:** when a booking is cancelled or rescheduled, mark its pending jobs `CANCELLED` (or recompute `sendAt`). Otherwise you'll send "reminder" emails for meetings that no longer exist.

## Message templating

Support variables in subjects/bodies so messages are personal:

```
Hi {{first_name}}, your {{event_type}} with {{host_name}} is on {{meeting_time}}.
Join: {{meeting_link}}
Reschedule: {{reschedule_link}}  ·  Cancel: {{cancel_link}}
```

Resolve variables from the booking + contact + event type. Resolve a name robustly: prefer `first_name`/`last_name` fields, fall back to a full-name field, then email.

## Sending email

Use SMTP (nodemailer) with credentials from settings/env. Wrap it in one `sendEmail({ to, subject, html })` helper used everywhere (confirmations, reminders, verification, password reset). Keep a shared HTML shell for consistent branding.

## Sending SMS (optional)

Integrate an SMS provider behind a `sendSms({ to, body })` helper. Gate it so SMS workflows are skipped cleanly if SMS isn't configured.

## "No booking" follow-ups

`FORM_NO_BOOKING` is special: it targets people who started a routing form but didn't book. Track form starts, and have the cron compare against bookings to send a nudge after a delay.

## Working with Claude Code

Build immediate confirmation emails first (simplest, instantly testable). Then the `ScheduledWorkflowJob` + cron loop with a reminder. Test the cancellation path explicitly — create a booking, confirm a job is scheduled, cancel, confirm the job is `CANCELLED`. Add SMS and `FORM_NO_BOOKING` last.
