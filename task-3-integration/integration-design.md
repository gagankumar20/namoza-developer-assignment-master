# Task 03 — Integration Design (Landing Page → HubSpot → WhatsApp → Google Ads)

Written task — no code required, per the brief. Architecture below.

## End-to-End Architecture

A **direct API call from a serverless function** — not a native HubSpot embedded form, and not Zapier/Make. Reasoning:

1. A native HubSpot embed hands control of validation, styling, and the dataLayer push to HubSpot's iframe, which breaks the custom GTM tracking this build already requires.
2. Zapier/Make adds a third-party hop with trigger-polling latency (often 1–2 minutes), which directly threatens the 2-minute WhatsApp SLA.
3. A direct call gives full control over error handling and retries, which a no-code connector does not.

### The Flow

1. User submits the 2-field form on the landing page → client-side validation passes → form data POSTs to a lightweight serverless function (e.g. a Vercel/Cloud Function endpoint — **not** the browser calling HubSpot directly, since that would expose the HubSpot private app token client-side).
2. The serverless function fires three things **in parallel**, not sequentially, so one slow/failed step doesn't block the others:
   - (a) HubSpot Contacts API (POST/PATCH) to create or update the contact
   - (b) Karix WhatsApp Business API to send the confirmation template message
   - (c) Nothing extra needed here — the browser-side `dataLayer.push({event:'consultation_form_submitted', ...})` has already fired GTM's Google Ads conversion tag independently on form submit, since the conversion only needs to know the form was submitted, not that the CRM write succeeded.
3. The serverless function logs the outcome of each of the three calls (HubSpot, Karix, conversion already fired client-side) to a simple table/log store, so failures are visible without digging through separate dashboards.
4. A native HubSpot workflow (inside HubSpot, not the serverless function) handles any secondary business rule, like assigning the contact to a clinic-specific owner — this stays in HubSpot since it's a CRM-side rule, not an integration step.

## Biggest Failure Point and the Fallback

**Phone-number deduplication in HubSpot.** HubSpot's default dedup key is **email**, not phone — and this form deliberately doesn't collect email. Without an explicit fix, every submission creates a new contact even from a repeat patient, fragmenting lead history and risking the clinic team calling the same patient twice off two conflicting records.

**Fallback:** before creating a contact, the serverless function first searches HubSpot via the Contacts Search API filtered on the `phone` property. If a match exists, it PATCHes the existing contact (clinic preference, source, lead status) instead of creating a new one.

**Trap question:** *If two patients submit with the same phone number but different names, what happens?* — A real scenario in this market (shared family phones are common). The integration does **not** silently overwrite the existing name, and does **not** blindly create a duplicate either. It appends the new name as a note/activity on the matched contact and flags the record for manual review. Safer than guessing which name is correct, and it surfaces the edge case to a human instead of hiding it.

## What Could Break the 2-Minute WhatsApp SLA, and Monitoring

Three realistic failure modes:
1. Karix API rate limiting or downtime during a traffic spike (e.g. right after a campaign goes live).
2. The WhatsApp template message getting rejected because the patient hasn't messaged the business number in the last 24 hours and the template wasn't pre-approved for this use case — a Meta/WhatsApp policy issue, not a Karix issue.
3. The serverless function itself cold-starting slowly or timing out under load, delaying the call to Karix.

**Monitoring:** every WhatsApp send attempt and its HTTP response code is logged with a timestamp delta from form submission. A scheduled check (every 5 minutes) flags any submission older than 2 minutes with no successful Karix delivery confirmation, and pages the on-call developer via Slack — deliberately low-tech, since current volume (a handful of consultation requests/day, not thousands) doesn't justify a full observability stack. If Karix delivery fails outright, the fallback is a templated SMS via a backup provider, since SMS doesn't carry WhatsApp's 24-hour template-window restriction.
