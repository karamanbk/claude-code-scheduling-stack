# 16 — Security & Data Handling

This platform holds OAuth tokens for your team's calendars and personal data for leads (names, emails, phone numbers, enrichment). That makes security a feature, not an afterthought. This guide consolidates the practices referenced elsewhere and adds the ones that don't have a home.

## Secrets

- **Never commit secrets.** Keep them in environment variables (and the encrypted `Settings`/connection rows). Add `.env*` to `.gitignore`.
- **Rotate** anything that leaks immediately — and assume anything committed to git history *has* leaked.
- Use a strong, random `NEXTAUTH_SECRET` and a separate `CRON_SECRET`. Don't reuse one secret for multiple purposes.
- Server-only secrets (DB URL, service keys, provider secrets) must never be referenced from client components or `NEXT_PUBLIC_*` vars.

## Encrypting tokens at rest

OAuth access/refresh tokens and SMTP/LLM keys stored in the database must be **encrypted**, not plaintext. Use the AES-GCM helper from the [Calendar guide](./06-calendar-and-video-integrations.md):

- Encrypt before writing, decrypt on read.
- Derive the key from a dedicated secret (or `NEXTAUTH_SECRET`) — and remember that **rotating that secret invalidates stored ciphertext**, so plan a re-encryption path if you rotate.
- Handle the "stored value isn't valid ciphertext" case gracefully (e.g. legacy plaintext) instead of crashing.

## Protecting endpoints

Classify every route and protect it accordingly:

| Endpoint type | Protection |
|---------------|-----------|
| Dashboard pages & admin APIs | Require a session; check `role === "ADMIN"` for settings/integrations |
| Cron endpoints | `Authorization: Bearer ${CRON_SECRET}` check; reject everything else |
| Public booking/routing/embed | No auth, but **rate-limited** and input-validated |
| Reschedule/cancel | Validate the per-booking `rescheduleToken` — it's the only auth the public has |
| Webhooks (CRM, calendar) | Verify the provider's signature/secret |

Don't rely on "nobody knows the URL." Token-guard or signature-verify anything that mutates data.

## Input validation

- Validate and type every request body (a schema validator like Zod is ideal). Reject unexpected shapes.
- Treat all public form input as hostile: enforce field types, lengths, and required-ness server-side — never trust the client.
- Re-check availability server-side at booking time; never trust a slot the client claims is free.

## Rate limiting & abuse

Public endpoints (availability, booking, routing evaluate, embed script) are scrapeable and spammable:

- Rate-limit by IP (and/or email) on booking and routing-evaluate. A simple fixed-window counter in your DB or an edge KV works.
- Smart-form validation (LLM) doubles as spam defense — but gate the LLM call behind a cheap pre-check so you don't pay for obvious junk.
- Cap expensive operations (enrichment, AI routing) per IP/time window.

## PII & data handling

You're a data processor for lead information. Minimum bar:

- **Collect only what you need.** Don't request fields you won't use.
- **Deletion:** provide a way to delete a contact and their bookings on request. Cascade deletes in the schema (`onDelete: Cascade`) make this clean.
- **Export:** be able to produce a contact's stored data on request.
- **Retention:** consider auto-pruning stale contacts/bookings after a period.
- **Third parties:** when you sync to a CRM or call an LLM, you're sharing PII — make sure that's covered by your privacy policy and the provider's terms.
- **Logs:** don't log full PII or secrets. Log ids and statuses, not email bodies or tokens. (The debugging guide says log the *payload you sent to an external API* — scrub secrets from those logs.)

## Cross-origin & embedding

- The embed script sets permissive CORS **only** on the script + embed endpoints — not on your data APIs.
- Validate `event.origin` on `postMessage` handlers (both sides) so a malicious parent/iframe can't drive your page.
- Don't expose admin data on any page that can be embedded.

## Auth hardening

- Hash passwords with bcrypt (or argon2); never store or log plaintext.
- Require email verification before credential login (covered in [Auth](./04-authentication.md)).
- Use HTTPS everywhere; OAuth providers reject non-HTTPS redirect URIs in production anyway.
- Keep the `www`→apex redirect so cookies stay on one canonical host (it's also a subtle security win — fewer cookie domains to reason about).

## Dependency & supply-chain hygiene

- Keep dependencies patched; watch for advisories on auth, crypto, and the ORM.
- Pin versions and review lockfile changes.
- Be cautious with the embed script — it runs on *other people's sites*. Keep it tiny, dependency-free, and serve it from your own origin.

## Working with Claude Code

Ask Claude Code to audit, route by route: *"list every API route and tell me its auth model — session, admin-only, cron-secret, token, or public."* Then close the gaps. Have it add a validation schema to each public endpoint and a rate-limit guard to booking/routing. Finish with a deletion endpoint for contacts and confirm the cascade actually removes bookings and invitees.
