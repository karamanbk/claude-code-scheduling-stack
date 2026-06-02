# 10 — CRM Sync

This guide pushes contacts and meetings into a CRM after a booking. The examples use a generic CRM with OAuth and a REST API; the pattern applies to most.

## What syncs

- **Contact** — upsert by email; map standard fields (name, email, phone, company) plus custom and enriched fields.
- **Meeting/engagement** — create an activity linked to the contact, with the meeting time, title, and join URL.
- **Tracking fields** — UTM params and internal identifiers (which routing rule / event type produced the lead).

## Connect & tokens

OAuth-connect the CRM and store encrypted tokens in a `CrmConnection`-style table (single-tenant: one connection). Refresh tokens before each call, exactly like the calendar integrations:

```ts
async function getValidCrmToken(): Promise<string | null> {
  const conn = await prisma.crmConnection.findFirst({ where: { isActive: true } });
  if (!conn) return null;
  // refresh if near expiry, persist new tokens, return access token
}
```

> Use the **refreshing** token helper everywhere. A frequent bug: a "fetch properties" or "sync" endpoint reads the stored token directly and works for an hour, then silently 401s once it expires.

## Field mapping

Let admins map your fields → CRM properties in the UI. Three groups:

1. **Standard** — email, first/last name, phone, company (auto-mapped).
2. **Custom form fields** — map each to a CRM property.
3. **Booking fields** — meeting URL, meeting time, reschedule link, cancel link.

Store the mapping as `{ ourFieldId: "crm_property_name" }`. Prefix special groups so they're easy to handle (`_booking_meeting_url`, `_enrichment_company_size`, etc.).

### Match field types to property types

This prevents a whole class of sync failures:

- A **URL** value must map to a CRM text/URL property. Validate it starts with `http(s)://` before sending.
- A **datetime** value (meeting time) must map to a CRM date/datetime property — send it in the format that property expects (often a Unix-ms timestamp), not an arbitrary string.

Filter the property dropdown in the UI by the field's value kind, and **still show an already-saved invalid mapping** (with a warning) so admins can fix it. A mapping hidden by a filter is a mapping nobody can correct.

## The sync, and why it must be resilient

CRMs typically **reject the entire upsert if any single property is invalid**. If one bad mapping (say, a date sent to a URL field) fails the whole call, you lose *everything* — including the identifiers you care about.

Make the upsert self-healing: parse the offending property name out of the error, drop just that property, and retry (bounded).

```ts
let props = { ...allProperties };
for (let attempt = 0; attempt < 5; attempt++) {
  const res = await upsertContact(props);
  if (res.ok) return res.id;

  const errorText = await res.text();
  const badProp = parseBadPropertyName(errorText); // from error JSON / message
  if (badProp && badProp in props) {
    console.warn(`CRM sync: dropping bad property "${badProp}" and retrying`);
    delete props[badProp];
    continue;
  }
  console.error("CRM sync failed, no recoverable property:", errorText);
  return null;
}
```

This way a misconfigured field never blocks the rest of the sync.

## Order of operations

```
1. Upsert the contact (with retry-on-bad-property)
2. Persist the returned CRM contact id on your Contact
3. If not already created, create the meeting/engagement linked to that contact
4. Persist the CRM meeting id on the Booking (so re-syncs don't duplicate)
```

Guard meeting creation with the stored id so re-running sync is idempotent.

## Auto-creating custom properties

If you write to CRM properties that may not exist (e.g. your own tracking fields), create them idempotently on first sync (ignore "already exists" errors). Group them under a recognizable label.

## A re-sync utility

Add an admin-only endpoint that re-syncs a contact by email (finds their latest booking and runs the sync). Invaluable for testing and for recovering specific records after fixing a mapping.

## Working with Claude Code

Build contact upsert first and confirm a contact appears in the CRM with standard fields. Add the retry-on-bad-property logic early — it saves you during field-mapping experimentation. Then add meeting creation (idempotent), then the field-mapping UI with type filtering, then the re-sync utility. Log the exact CRM error text on failure; CRM validation messages name the offending property and save hours.
