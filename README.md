# Namoza Developer Assignment — OrthoNow

**Submitted by:** Shubham Nishad
**Role:** Developer — Position 1 (Client Web + Martech)
**Client:** OrthoNow — 9-clinic orthopaedic chain across Bengaluru, Hyderabad & Chennai

## What's in this repo

| Folder | Contains |
|---|---|
| `task-1-gtm-schema/` | Full GTM event schema table + booking-funnel dataLayer JSON + conversion-action reasoning |
| `task-2-landing-page/` | `index.html` — single-file, framework-free consultation landing page, plus PageSpeed mobile score screenshot |
| `task-3-integration/` | Written integration architecture: landing page → HubSpot → Karix WhatsApp → Google Ads conversion |

## Task 02 — Live Demo

`task-2-landing-page/index.html` is a self-contained file (HTML/CSS/vanilla JS, zero external requests — no frameworks, no web fonts, no images) that runs directly in a browser with no server.

- **Form submit** fires `window.dataLayer.push({event: 'consultation_form_submitted', ...})` inside the submit handler, after validation, not on page load. Open browser console before submitting to see it fire live.
- **Thank-you state** swaps in via JS, no page reload.
- **Mobile Performance score:** 96/100 (Lighthouse — same engine as PageSpeed Insights). Screenshot included in this folder.

> Note: the included screenshot was generated via a local Lighthouse run during development. For final verification, host this file (Netlify Drop or GitHub Pages) and re-run against the live public URL on [pagespeed.web.dev](https://pagespeed.web.dev).

## How to run locally

No build step. Just open `task-2-landing-page/index.html` directly in a browser, or serve it:

```bash
cd task-2-landing-page
python3 -m http.server 8000
# visit http://localhost:8000
```

## Loom Walkthrough

Link: *[add Loom link here before sending]*

Covers (≤8 min): GTM schema decisions (2 min) → live landing page demo with dataLayer push shown in console (3 min) → integration architecture walkthrough, including the HubSpot phone-dedup edge case (3 min).
