# 11 — Embeddable Widgets

This guide makes the booking and routing flows embeddable on any website via a lightweight script + iframe.

## The approach

You serve a tiny JavaScript snippet that the customer drops on their site. The snippet finds a container element, reads config from `data-*` attributes, and injects an iframe pointing at your hosted booking/routing page.

```html
<!-- What the customer pastes -->
<div id="booking-embed" data-event-type="discovery-call"></div>
<script src="https://yourdomain.com/api/embed/script.js"></script>
```

## Serving the script

Serve the script from an API route as JavaScript with permissive CORS. Keep it tiny and dependency-free.

```ts
// src/app/api/embed/script.js/route.ts
import { NextResponse } from "next/server";

export async function GET() {
  const script = `(function(){
    var c = document.getElementById("booking-embed");
    if (!c) return;
    var t = c.getAttribute("data-event-type");
    if (!t) return;

    // The script tag's own origin = your platform origin
    var s = document.querySelectorAll('script[src*="/api/embed/script"]');
    var base = s.length ? new URL(s[s.length-1].src).origin : "";
    if (!base) return;

    // Capture the parent page URL (for params) with cross-origin fallbacks
    var href = "";
    try { href = window.top.location.href; }
    catch(e){ try { href = window.parent.location.href; } catch(e2){ href = document.referrer || window.location.href; } }

    var u = base + "/embed/" + t + "?parentUrl=" + encodeURIComponent(href);

    // Optional styling via data-* attributes
    var bg = c.getAttribute("data-bg-color"); if (bg) u += "&_bgColor=" + encodeURIComponent(bg);

    // Forward the parent page's query params (UTMs, gclid, custom)
    try {
      var q = new URLSearchParams(new URL(href).search);
      q.forEach(function(v,k){ if (v && k !== "parentUrl") u += "&" + encodeURIComponent(k) + "=" + encodeURIComponent(v); });
    } catch(e){}

    var f = document.createElement("iframe");
    f.src = u;
    f.style.cssText = "width:100%;border:none;min-height:700px;";
    f.setAttribute("loading","lazy");
    c.appendChild(f);

    // Listen for messages from the iframe
    window.addEventListener("message", function(e){
      if (e.origin !== base) return;
      var d = e.data;
      if (d.type === "embed:resize") f.style.height = Math.max(d.height, 500) + "px";
      if (d.type === "embed:redirect" && d.url) window.location.href = d.url;
    });
  })();`;

  return new NextResponse(script, {
    headers: {
      "Content-Type": "application/javascript",
      "Access-Control-Allow-Origin": "*",
      "Cache-Control": "public, max-age=300",
    },
  });
}
```

## The embed page

`/embed/[eventTypeId]` is a stripped-down booking page tuned for iframes:

- Reads `parentUrl` and forwards its query params into the form (cross-origin fallback when the iframe can't read the parent directly).
- Accepts styling overrides (`_bgColor`, `_textColor`, etc.).
- Posts height changes to the parent so the iframe auto-resizes.
- On redirect/booking completion, posts a message to the parent instead of navigating itself.

### The two hard problems

**1. Reading parent params.** Inside the iframe, `window.location.search` is the *iframe's* URL, not the parent's. The script (which runs on the parent page) captures the parent URL and forwards params into the iframe `src`. The embed page then reads them from its own query string. Don't try to read the parent from inside the iframe — cross-origin blocks it.

**2. Redirecting the parent.** An iframe generally can't navigate the top window directly. On booking completion, `postMessage` to the parent and let the parent script do `window.location.href = url`. Keep a `window.top` fallback for same-origin cases.

```ts
// inside the iframe, after booking
if (window.parent !== window) {
  window.parent.postMessage({ type: "embed:redirect", url }, "*");
  try { window.top.location.href = url; } catch {}
} else {
  window.location.href = url;
}
```

## Auto-resize

A small client component in the embed page observes content height and posts it up:

```ts
const ro = new ResizeObserver(() => {
  window.parent.postMessage({ type: "embed:resize", height: document.body.scrollHeight }, "*");
});
ro.observe(document.body);
```

## Always forward tracking params

The whole point of embedding on landing pages is attribution. Forward the parent's `utm_*`, `gclid`, and any custom params: into the iframe, into the booking, and onward to the thank-you/redirect URL. Test this end-to-end with a real landing page that has query params.

## Working with Claude Code

Build the embed page first and test it directly (open `/embed/<id>?utm_source=test`). Then add the script and a local `test.html` that embeds it, and confirm params flow parent → iframe → redirect. Test inside a real third-party-style page (different origin) since cross-origin behavior differs from same-origin.
