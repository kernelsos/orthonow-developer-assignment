# Task 02 — Landing Page Build

**File:** [`index.html`](./index.html) — single self-contained file (HTML + CSS + vanilla JS, no frameworks, no external requests). Open it directly in any browser, no server required.

## PageSpeed Insights — Mobile

![PageSpeed Mobile Score](./pagespeed-score.png)

**Performance: 99** &nbsp;|&nbsp; Accessibility: 91 &nbsp;|&nbsp; Best Practices: 100 &nbsp;|&nbsp; SEO: 100

Captured via PageSpeed Insights, Mobile (Moto G Power emulation, Slow 4G throttling).

## What to check when reviewing

1. Open `index.html` in a browser.
2. Open DevTools → Console (so `window.dataLayer` pushes are visible).
3. Try submitting the form empty — no `dataLayer` push should fire, and validation errors appear inline.
4. Fill in a name and a valid 10-digit Indian mobile number, then submit — a single `consultation_form_submitted` event should log to the console, and the page swaps to a thank-you state with no reload.
5. Resize to a mobile width (or open DevTools device toolbar) to see the mobile-first layout and the sticky "Book Free Consultation" bar appear once you scroll past the hero form.
