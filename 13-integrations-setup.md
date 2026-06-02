# 13 — Integrations Setup (Credentials)

This guide is the practical "where do I get the keys" companion to the feature guides. It covers obtaining and configuring credentials for **Google**, **Microsoft**, **Zoom**, **Twilio (SMS)**, and **SMTP**. Provider dashboards change over time — treat exact menu names as approximate and follow the current official docs.

> **Golden rules for all integrations**
> - Store secrets in environment variables (and the encrypted `Settings`/connection rows for per-user tokens). Never commit them.
> - Register **both** production and `http://localhost:3000` redirect URIs so local dev works.
> - Keep login OAuth (identity) separate from calendar OAuth (calendar scopes) — clearer consent and easier debugging.

---

## Google (OAuth login + Calendar + Meet)

1. In **Google Cloud Console**, create a project.
2. **APIs & Services → Enable APIs:** enable **Google Calendar API**.
3. **OAuth consent screen:** configure app name, support email, and add the scopes you'll request:
   - `openid`, `email`, `profile` (login)
   - `https://www.googleapis.com/auth/calendar.events` (create/update/delete events)
   - `https://www.googleapis.com/auth/calendar.readonly` (free/busy)
   - Add test users while the app is in "testing," or publish it.
4. **Credentials → Create OAuth client ID → Web application.** Add redirect URIs:
   - `https://yourdomain.com/api/auth/callback/google`
   - `http://localhost:3000/api/auth/callback/google`
   - If your "Connect Calendar" flow uses a separate callback, add that too.
5. Copy the client id/secret into env:

```bash
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
```

**Meet links** come automatically when you create a Calendar event with `conferenceData` and `conferenceDataVersion=1`. No separate Meet credentials needed.

To get refresh tokens, request `access_type=offline` and `prompt=consent`.

---

## Microsoft (OAuth login + Outlook Calendar + Teams)

1. In **Microsoft Entra admin center → App registrations → New registration.**
2. **Supported account types:** choose multi-tenant + personal if you want any Microsoft account, or single-tenant for your org only. This decides your `MICROSOFT_TENANT_ID` (`common` for multi-tenant + personal, or your tenant id).
3. **Redirect URI (Web):**
   - `https://yourdomain.com/api/auth/callback/microsoft-entra-id`
   - `http://localhost:3000/api/auth/callback/microsoft-entra-id`
4. **Certificates & secrets → New client secret.** Copy the **value** immediately.
5. **API permissions → Microsoft Graph → Delegated:**
   - `openid`, `email`, `profile`, `User.Read` (login)
   - `Calendars.ReadWrite` (events + free/busy)
   - `OnlineMeetings.ReadWrite` (Teams links)
   - `offline_access` (refresh tokens)
   - Grant admin consent if required.
6. Env:

```bash
MICROSOFT_CLIENT_ID=...
MICROSOFT_CLIENT_SECRET=...
MICROSOFT_TENANT_ID=common   # or your tenant id
```

**Teams links:** create the Graph event with `isOnlineMeeting: true` and `onlineMeetingProvider: "teamsForBusiness"`.

---

## Zoom (video links) — optional

1. In the **Zoom App Marketplace**, build an **OAuth** app (user-managed).
2. Set the redirect URL to your "Connect Zoom" callback (e.g. `https://yourdomain.com/api/integrations/zoom/callback`).
3. **Scopes:** `meeting:write` (create meetings) and the user-read scope.
4. Env:

```bash
ZOOM_CLIENT_ID=...
ZOOM_CLIENT_SECRET=...
```

Store the per-user tokens (encrypted) in `VideoConnection`. Create meetings via `POST /users/me/meetings` and use the returned `join_url`.

---

## Twilio (SMS / WhatsApp) — optional

For `SMS` workflow actions.

1. Create a **Twilio** account; get your **Account SID** and **Auth Token** from the console.
2. **SMS:** buy/verify a sending phone number (or a Messaging Service SID).
3. **WhatsApp (optional):** use the WhatsApp sandbox to start, then a registered sender + approved templates for production. WhatsApp business messages outside the 24-hour window require **pre-approved templates** (you send a template SID, not free text).
4. Env:

```bash
TWILIO_ACCOUNT_SID=...
TWILIO_AUTH_TOKEN=...
TWILIO_FROM_NUMBER=+1XXXXXXXXXX        # or:
TWILIO_MESSAGING_SERVICE_SID=...
```

Wrap it in one helper so workflows don't care about the provider:

```ts
// src/lib/sms.ts
export async function sendSms({ to, body }: { to: string; body: string }) {
  const sid = process.env.TWILIO_ACCOUNT_SID;
  const token = process.env.TWILIO_AUTH_TOKEN;
  if (!sid || !token) { console.warn("SMS not configured; skipping"); return; }

  const res = await fetch(`https://api.twilio.com/2010-04-01/Accounts/${sid}/Messages.json`, {
    method: "POST",
    headers: {
      Authorization: "Basic " + Buffer.from(`${sid}:${token}`).toString("base64"),
      "Content-Type": "application/x-www-form-urlencoded",
    },
    body: new URLSearchParams({ To: to, From: process.env.TWILIO_FROM_NUMBER!, Body: body }),
  });
  if (!res.ok) console.error("SMS send failed:", res.status, await res.text());
}
```

> Gate SMS so a missing config **skips cleanly** rather than throwing — SMS is optional and shouldn't break a booking.

---

## SMTP (transactional email)

Used for confirmations, reminders, email verification, and password reset. Any transactional SMTP provider works.

1. Create an account with an email/SMTP provider and verify your sending **domain** (SPF/DKIM) so mail doesn't land in spam.
2. Create SMTP credentials (host, port, username, password).
3. Env:

```bash
SMTP_HOST=smtp.your-provider.com
SMTP_PORT=587
SMTP_USER=...
SMTP_PASS=...
SMTP_FROM_EMAIL=notifications@yourdomain.com
SMTP_FROM_NAME=Scheduling Platform
```

One shared helper (nodemailer) used everywhere:

```ts
// src/lib/email.ts
import nodemailer from "nodemailer";

let transporter: nodemailer.Transporter | null = null;
function getTransporter() {
  if (transporter) return transporter;
  const port = parseInt(process.env.SMTP_PORT || "587");
  transporter = nodemailer.createTransport({
    host: process.env.SMTP_HOST,
    port,
    secure: port === 465,
    auth: { user: process.env.SMTP_USER, pass: process.env.SMTP_PASS },
  });
  return transporter;
}

export async function sendEmail({ to, subject, html }: { to: string; subject: string; html: string }) {
  await getTransporter().sendMail({
    from: `"${process.env.SMTP_FROM_NAME}" <${process.env.SMTP_FROM_EMAIL}>`,
    to, subject, html,
  });
}
```

Test deliverability early (verification + a real reminder), and check that links use your production URL, not `localhost`.

---

## LLM API key (AI features)

AI routing, smart-form validation, and enrichment need an LLM. Store the key in env (or the encrypted `Settings` row if you let admins set it in-app):

```bash
LLM_API_KEY=...
```

Validate the model's JSON output and always have a deterministic fallback (see the routing guide) so an LLM hiccup never blocks a booking.

---

## Putting it together

You don't need everything at once. A reasonable order:

1. **SMTP** (verification emails make auth usable).
2. **Google or Microsoft** OAuth (login) — then add calendar scopes for booking.
3. **LLM** when you build routing/smart forms.
4. **Zoom / Twilio** only if you need those channels.

Next: deploy it all → [14 — Deployment](./14-deployment.md).
