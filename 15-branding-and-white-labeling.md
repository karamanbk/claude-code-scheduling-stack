# 15 — Branding & White-Labeling

This guide makes the public-facing surfaces (booking pages, routing forms, embeds, emails) look like *your* company rather than a generic tool. In single-tenant mode there's one brand, so this is straightforward configuration rather than per-tenant theming.

## What's brandable

- **Logo** — shown on booking pages, the routing form, and emails.
- **Brand color** — primary/accent color for buttons, highlights, focus rings.
- **Background & text colors** — for embeds that sit inside a customer's page.
- **Font** — a web font that matches your site.
- **Thank-you content** — custom title/message after a booking.
- **"Hide platform branding"** — remove any "powered by" mark.

## Where it lives

Store branding on the single `Settings` row (see the data-model guide), or a dedicated `BrandingConfig` table if you want to keep it separate:

```prisma
model BrandingConfig {
  id              String  @id @default(uuid())
  logo            String?
  favicon         String?
  primaryColor    String  @default("#5046e5")
  backgroundColor String?
  fontFamily      String?
  customCss       String? @db.Text
  thankYouTitle   String?
  thankYouMessage String? @db.Text
  hideBranding    Boolean @default(false)
  updatedAt       DateTime @updatedAt
}
```

Expose a `GET/PUT /api/branding` (admin-only) and a small branding page in the dashboard.

## Applying colors with CSS variables

The cleanest approach: drive everything off CSS custom properties, then override them per page from the stored config.

```tsx
// On a booking/embed page (server component)
const overrides = [
  branding.primaryColor && `--color-primary: ${branding.primaryColor}`,
  branding.backgroundColor && `--color-background: ${branding.backgroundColor}`,
  branding.backgroundColor && `--color-card: ${branding.backgroundColor}`,
].filter(Boolean).join("; ");

return (
  <>
    {overrides && <style>{`:root { ${overrides} }`}</style>}
    {/* page content uses var(--color-primary), etc. */}
  </>
);
```

Because the design system already references `--color-primary`, `--color-background`, etc., a single injected `<style>` re-themes the whole page with no component changes.

## Embed color overrides

Embeds need to blend into the host site, so let the embed snippet pass colors via `data-*` attributes that become query params (`_bgColor`, `_textColor`, `_buttonColor`). The embed page reads them and injects the same `:root` override. This was wired in the [Embeddable Widgets](./11-embeddable-widgets.md) guide — branding just supplies the defaults when the embed doesn't specify.

> Precedence: **embed `data-*` override → stored branding → design-system default.** Keep that order consistent.

## Logo & favicon upload

1. Admin uploads an image; store it in your file bucket (see [Deployment](./14-deployment.md) → Supabase Storage).
2. Save the public URL on `BrandingConfig.logo` / `.favicon`.
3. Validate type and size (e.g. PNG/SVG/WebP, ≤ 2 MB) server-side.
4. Render the logo on booking/routing pages and in the email shell.

## Fonts

If you allow a custom font, load it from a web-font provider in the page `<head>` and set `--font-sans`. Provide a sensible default so pages never render unstyled while a custom font loads.

## Branding emails

The shared email shell (from [Workflows](./09-workflows-and-notifications.md)) should pull the logo, brand color, and "from" name from settings so confirmations and reminders match the booking pages. Keep one HTML shell function; don't re-template per message.

## Custom domain for booking pages (optional)

Two common approaches:

- **Subdomain** — host booking pages at `book.yourdomain.com` (a DNS record + your host's domain settings). Simplest.
- **Path on your site** — keep everything under `yourdomain.com/book/...`.

Whichever you pick, keep it consistent with the canonical-host rule from [Deployment](./14-deployment.md) so cookies and OAuth redirects don't break.

## "Hide branding"

When `hideBranding` is true, suppress any "powered by" mark on public pages. Keep the toggle honest — it should remove the mark everywhere it appears (booking page, routing form, embed, emails).

## Working with Claude Code

Implement the CSS-variable override first and confirm one booking page re-themes from the stored config. Then add logo upload, then the email shell wiring, then embed `data-*` precedence. Test an embed on a dark-background page to confirm the color overrides actually win.
