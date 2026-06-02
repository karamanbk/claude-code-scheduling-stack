# 18 — Testing Strategy

You don't need 100% coverage. You need tests where bugs are **expensive** or **subtle** — and this app has a few well-defined danger zones. This guide says what to test, how, and what to skip.

## The testing pyramid for this app

```
        ┌─────────────────────────┐
        │  A few end-to-end flows  │   book → calendar → email
        ├─────────────────────────┤
        │  Some integration tests  │   API routes against a test DB
        ├─────────────────────────┤
        │   Many unit tests on     │   slot math, routing rules,
        │   pure logic             │   template rendering, validation
        └─────────────────────────┘
```

Most of your value is at the bottom: pure functions that are easy to test and historically bug-prone.

## Unit-test the pure logic (highest ROI)

Keep core logic in pure functions separate from I/O so they're trivially testable.

### Slot generation
The single most bug-prone function in the app (timezones, DST, buffers). Test:

- A normal working day → expected slots.
- **DST transition** days (spring forward / fall back) → no missing/duplicate slots.
- Visitor timezone ≠ host timezone → correct conversion.
- Fully booked day → no slots.
- A slot just inside vs. just outside `minimumNotice`.
- `bufferBefore`/`bufferAfter` correctly subtract adjacent time.

```ts
import { computeSlots } from "@/lib/availability";

test("excludes slots inside minimum notice", () => {
  const slots = computeSlots({ /* ...fixed inputs, fixed 'now' */ });
  expect(slots).not.toContainEqual(expect.objectContaining({ start: tooSoon }));
});
```

> Inject "now" as a parameter (don't call `new Date()` inside) so tests are deterministic.

### Manual routing
Rule evaluation is pure and decision-critical. Test operator behavior (`equals`, `contains`, `greater_than`, `in_list`), order/first-match-wins, and the default-action fallback.

### Template rendering
Variable substitution (`{{first_name}}`, `{{meeting_link}}`, name resolution from first/last vs. full name). Test missing-variable behavior (renders empty, not `undefined`).

### Validation schemas
If you use a schema validator, test that bad payloads are rejected and good ones pass.

## Integration-test the critical API routes

Run these against a **disposable test database** (a separate schema or an ephemeral Postgres). Wrap each test in a transaction or reset between tests.

Worth covering:

- `POST /api/bookings` — creates booking + contact + invitee; re-checks availability; rejects a taken slot.
- Reschedule/cancel — updates status, validates token, cancels pending workflow jobs.
- The workflow cron — given due jobs, sends and marks them; skips cancelled bookings.
- CRM sync's **drop-bad-property retry** — feed a payload with one invalid property and assert the rest still syncs.

Mock the *external* calls (calendar, CRM, LLM, email) — you're testing *your* logic, not the provider. A thin interface around each external service makes mocking clean.

## End-to-end (a few, high-value)

Pick the handful of flows that would be catastrophic if broken, and automate them with a browser tool (Playwright):

- Open a booking page → pick a slot → submit → see confirmation.
- The same flow **embedded in an iframe** on a test page, asserting UTMs reach the confirmation/redirect.

Keep E2E few — they're slow and flaky-prone. They guard the money path, not every branch.

## What NOT to test

- Generated code (Prisma client), framework internals, trivial getters.
- Exact LLM outputs — they're non-deterministic. Instead, test that you **handle** malformed/empty LLM responses and always fall back.
- UI styling. Snapshot tests on markup tend to be noise.

## Tooling

- **Vitest** or **Jest** for unit/integration.
- **Playwright** for E2E.
- A `tsc --noEmit` type check in CI catches a huge class of errors for free — treat it as your cheapest test.
- Run tests + type check on every push (GitHub Actions or your CI). Block merges on red.

## A pragmatic CI pipeline

```yaml
# conceptual
on: [push, pull_request]
jobs:
  check:
    steps:
      - install deps
      - prisma generate
      - tsc --noEmit          # cheapest, catches the most
      - run unit tests
      - run integration tests (against test DB)
      # E2E on a schedule or pre-deploy, not every push
```

## Working with Claude Code

Ask Claude Code to extract any I/O-tangled logic into pure functions first (it makes everything testable), then write the slot-math tests including DST cases. Have it add the CRM retry test and the booking-route integration test next. Wire `tsc --noEmit` + unit tests into CI early — it's the highest-leverage safety net and Claude Code can scaffold the workflow file in minutes.
