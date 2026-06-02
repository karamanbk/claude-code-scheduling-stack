# 06 — Calendar & Video Integrations

This guide connects hosts' calendars (Google, Microsoft) for free/busy lookups and event creation, and conferencing providers (Google Meet, Zoom, Teams) for meeting links.

## The pattern

Every integration follows the same shape:

1. **OAuth connect** — the user clicks "Connect," authorizes scopes, you store encrypted tokens.
2. **Token refresh** — before each API call, refresh if the access token is near expiry.
3. **Use the API** — read free/busy, create/update/delete events.

Store tokens in `CalendarConnection` / `VideoConnection` (per user, encrypted).

## Encrypting tokens at rest

Never store OAuth tokens in plaintext. A small helper:

```ts
// src/lib/encryption.ts
import crypto from "node:crypto";
const KEY = crypto.createHash("sha256").update(process.env.NEXTAUTH_SECRET!).digest();

export function encrypt(text: string): string {
  const iv = crypto.randomBytes(12);
  const cipher = crypto.createCipheriv("aes-256-gcm", KEY, iv);
  const enc = Buffer.concat([cipher.update(text, "utf8"), cipher.final()]);
  const tag = cipher.getAuthTag();
  return [iv, tag, enc].map((b) => b.toString("base64")).join(".");
}

export function decrypt(payload: string): string {
  const [iv, tag, enc] = payload.split(".").map((s) => Buffer.from(s, "base64"));
  const decipher = crypto.createDecipheriv("aes-256-gcm", KEY, iv);
  decipher.setAuthTag(tag);
  return Buffer.concat([decipher.update(enc), decipher.final()]).toString("utf8");
}
```

## Token refresh helper

Centralize "give me a valid token" so every call site is simple:

```ts
async function getValidToken(connection): Promise<string> {
  let accessToken = decrypt(connection.accessToken);
  if (isBefore(connection.tokenExpiry, addMinutes(new Date(), 5))) {
    const refreshed = await refreshProviderToken(decrypt(connection.refreshToken));
    await prisma.calendarConnection.update({
      where: { id: connection.id },
      data: {
        accessToken: encrypt(refreshed.accessToken),
        refreshToken: encrypt(refreshed.refreshToken ?? decrypt(connection.refreshToken)),
        tokenExpiry: refreshed.expiresAt,
      },
    });
    accessToken = refreshed.accessToken;
  }
  return accessToken;
}
```

> A common bug: an endpoint reads the stored token directly **without** refreshing, so calls start failing silently once the token expires. Route every external call through `getValidToken`.

## Google Calendar

- **Scopes:** `https://www.googleapis.com/auth/calendar.events` and `calendar.readonly` (free/busy).
- **Free/busy:** `POST /freeBusy` with the host's calendar id and time window.
- **Create event:** `POST /calendars/{id}/events` with `conferenceData` to auto-generate a **Google Meet** link (set `conferenceDataVersion=1`).
- Extract the Meet URL from `conferenceData.entryPoints`.

## Microsoft Outlook / Teams

- **Scopes:** `Calendars.ReadWrite`, `OnlineMeetings.ReadWrite`, plus `offline_access`.
- **Free/busy:** Graph `POST /me/calendar/getSchedule`.
- **Create event:** Graph `POST /me/events` with `isOnlineMeeting: true` and `onlineMeetingProvider: "teamsForBusiness"` to get a **Teams** link.

## Zoom

- OAuth connect, store tokens in `VideoConnection`.
- **Create meeting:** `POST /users/me/meetings`; use the returned `join_url`.

## Choosing the meeting link

Decide the conferencing provider from the event type's `locationType`:

| `locationType` | Link source |
|----------------|-------------|
| `GOOGLE_MEET` | Google Calendar conferenceData |
| `TEAMS` | Microsoft online meeting |
| `ZOOM` | Zoom create-meeting |
| `PHONE` / `IN_PERSON` / `CUSTOM` | No URL; show `customLocation` text |

After creating the calendar event, store `meetingUrl`, `meetingProvider`, and `hostCalendarEventId` on the `Booking` so reschedule/cancel can update or delete the right event later.

## Reschedule & cancel

- Generate a `rescheduleToken` per booking; links are `/<reschedule|cancel>/<bookingId>?token=...`.
- On reschedule: update the calendar event in place (don't create a duplicate).
- On cancel: delete the calendar event using the stored provider + event id.

## Renewing webhooks / watches (optional)

If you subscribe to calendar change notifications, those subscriptions expire. Add a periodic job to renew them (see the workflows/cron guide for the cron pattern).

## Working with Claude Code

Wire one provider end-to-end (connect → refresh → free/busy → create event → delete event) and book a real test meeting before adding the next. Verify the event actually appears on the host's calendar with the right link. Then repeat for the others — the structure is identical, only the API shapes differ.
