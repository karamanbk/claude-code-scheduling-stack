# 08 — Lead Routing & Smart Forms

This guide covers the two "intelligent" features: **routing** (send the right lead to the right place) and **smart forms** (validate and enrich submissions).

## Routing

A routing rule shows a qualification form, then decides what to do with each submission: show a specific event type, redirect to a URL, or show a thank-you message.

### The routing config

Stored as JSON on `RoutingRule.routingConfig`:

```ts
type RoutingConfig = {
  mode: "manual" | "ai" | "both";
  formFields: FormField[];          // the qualification form
  manualRules: ManualRule[];        // evaluated in order, first match wins
  aiPrompt?: string;                // natural-language routing logic
  aiGeneratedPrompt?: string;       // an optimized/system version
  aiConditionActions?: { key: string; action: ActionConfig }[];
  defaultAction: ActionConfig;      // fallback when nothing matches
};

type ManualRule = {
  field: string;
  operator: "equals" | "contains" | "greater_than" | "less_than" | "in_list";
  value: string;
  action: ActionConfig;
};

type ActionConfig = {
  type: "show_event_type" | "redirect" | "thank_you";
  eventTypeId?: string;
  redirectUrl?: string;
  passParams?: boolean;
};
```

### Manual routing

Evaluate `manualRules` top to bottom; first match wins; otherwise `defaultAction`. Simple, predictable, no LLM cost. Good for firmographic rules ("company size > 500 → enterprise rep").

### AI routing

Instead of brittle if/then chains, let an LLM decide. The flow:

1. Admin writes intent in plain English ("Send enterprise leads to the senior team; SMB to self-serve; route by country where possible").
2. Optionally generate an optimized system prompt from that intent.
3. At submission time, send the form data (plus any IP-derived geo) to the LLM and ask for a structured decision mapping to an action.

```ts
// POST /api/routing/evaluate
const decision = await llmRoute({
  prompt: config.aiGeneratedPrompt ?? config.aiPrompt,
  formData,        // enriched with geo, etc.
  actions: config.aiConditionActions,
});
// decision -> { action: "show_event_type" | "redirect" | "thank_you", eventTypeId?, redirectUrl? }
```

Keep `defaultAction` as a safety net for when the model is unsure or the call fails. `mode: "both"` can run manual rules first and fall back to AI.

### The routing page

Public page at `/route/[ruleId]`:

1. Render the qualification form.
2. On submit, call `/api/routing/evaluate`.
3. Apply the decision: redirect, show thank-you, or hand off to the event-type booking page (passing form data + tracking params along).

When embedded, redirects use the parent-postMessage trick (see embeds guide). Always forward tracking params through routing → booking → redirect.

## Smart forms

Smart forms add an LLM validation/enrichment step to the booking form before a slot is held.

### Validation

Configure checks per field; the LLM returns a structured verdict:

```ts
type SmartFormResult = {
  valid: boolean;
  flags: { field: string; type: "error" | "warning"; message: string }[];
};
```

Typical checks: reject obviously fake emails/domains, nonsense phone numbers, gibberish company names — while being lenient on unusual-but-plausible entries. Block submission on `error` flags; surface `warning` flags without blocking.

Generate the system prompt from the admin's plain-English rules so they don't hand-write prompt engineering.

### Enrichment

After (or alongside) a booking, enrich the contact:

- Define enrichment fields (company website, industry, size, role, etc.).
- Send the contact's email/company/name to the LLM (optionally with web data) and store results in the `Contact`'s dedicated columns + `enrichmentData` JSON.
- These feed CRM sync and routing decisions.

Keep a sensible default set of enrichment fields and let admins customize (cap the count to keep prompts bounded).

### Social proof (optional)

Smart forms can show social proof on the booking page ("X companies booked recently") computed from recent contacts. A small touch that lifts conversion.

## Working with Claude Code

Implement manual routing first (no LLM, fully deterministic, easy to test). Then add the AI evaluate endpoint with a hard fallback to `defaultAction`. Build smart-form **validation** before **enrichment**. For every LLM call: validate the model's JSON output, handle malformed responses, and never let an LLM failure block a booking.
