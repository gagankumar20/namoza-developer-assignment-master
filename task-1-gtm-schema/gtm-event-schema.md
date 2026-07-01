# Task 01 — GTM Event Schema (OrthoNow)

OrthoNow's site currently has no event tracking beyond default pageviews. Before any paid campaign goes live, the following event schema needs to be implemented in GTM and verified in GA4 DebugView.

## Full Event Table

| Event Name | Trigger Type | Key Parameters | Feeds Into (GA4) |
|---|---|---|---|
| `page_view` (default) | All Pages – History Change / Page View | `page_path`, `page_title`, `clinic_location` (if on a location page) | Engagement report, baseline traffic by clinic |
| `clinic_location_view` | Page View – URL matches `/clinics/*` | `clinic_name`, `clinic_city`, `specialty_list` | Clinic-level traffic comparison; feeds "Top Clinics" exploration |
| `call_now_click` | Click – Just Links, click URL contains `tel:` | `click_url`, `click_location` (homepage/clinic-page/landing-page), `clinic_name` | "Call Now" conversion audience; Google Ads call-intent conversion |
| `whatsapp_chat_open` | Click – Click ID = `whatsapp-widget` | `click_url` (wa.me link), `page_path`, `device_category` | Engagement event; remarketing audience "WhatsApp Intent" |
| `guide_download_form_submit` | Custom Event – fired via `dataLayer.push` on form success | `form_id`, `guide_name`, `lead_phone_provided` (boolean) | Lead-magnet conversion; nurture audience for retargeting |
| `guide_download_complete` | Click – Just Links, file extension `.pdf` | `file_name`, `file_extension`, `page_path` | Content engagement report |
| `blog_scroll_75` | Scroll Depth – 75% vertical | `page_path`, `page_title`, `blog_category` | Content engagement audience; informs which articles to expand |
| `booking_step_view` | Custom Event – `dataLayer.push` on each step render | `step_number`, `step_name`, `clinic_location` (if pre-selected) | Funnel Exploration step entry |
| `booking_step_complete` | Custom Event – `dataLayer.push` on "Next" click per step | `step_number`, `step_name`, `clinic_location`, `specialty` | Funnel Exploration step completion / drop-off |
| `booking_confirmed` | Custom Event – `dataLayer.push` on final confirmation screen | `clinic_location`, `specialty`, `preferred_date`, `lead_source` | Primary conversion; imported into Google Ads as "Appointment Booked" |

## Funnel Tracking for the 3-Step Booking Form

GTM cannot natively listen to multi-step form interactions on a single-page widget (no URL/page change happens between steps) — it has no way of knowing "step 2 of a form" unless the front-end explicitly tells it via a `dataLayer.push()`. The front-end developer owns writing these pushes; I own the GTM trigger and the GA4 tag configuration that listens for them.

**Step 1 — clinic + specialty selected** (fires on "Next" click, not page load):
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Koramangala",
  "specialty": "Orthopaedics - Knee"
}
```

**Step 2 — name/phone/date entered** (fires after client-side validation passes):
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "Koramangala",
  "preferred_date": "2026-07-04"
}
```

**Step 3 — final confirmation** (fires `booking_confirmed`, not `booking_step_complete`, so it can be imported cleanly as the primary conversion):
```json
{
  "event": "booking_confirmed",
  "clinic_location": "Koramangala",
  "specialty": "Orthopaedics - Knee",
  "preferred_date": "2026-07-04",
  "lead_source": "organic"
}
```

**GTM/GA4 setup:** A single Custom Event trigger per event name (`booking_step_complete`, `booking_confirmed`) feeds a GA4 Event tag, with `step_number`, `step_name`, `clinic_location`, and `specialty` mapped as event parameters. In GA4, mark `booking_step_complete` and `booking_confirmed` as key events, then build a Funnel Exploration: `step_view(1)` → `step_complete(1)` → `step_complete(2)` → `booking_confirmed`. The step-level breakdown is what surfaces exactly where users abandon — e.g. if 70% complete Step 1 but only 40% complete Step 2, the contact-details step is the leak, not the form generally.

**Who writes the dataLayer push — me or the front-end dev?** The front-end dev. I write the GTM trigger and GA4 tag; they write the push, because GTM has no native concept of a JS-driven step change. My brief to them for Step 2: "When the user clicks Next on the contact-details step, after your existing validation passes and before the UI advances, run `window.dataLayer.push()` with this exact object" — handing them the JSON verbatim, asking them to fire it once per step (not per keystroke), and to confirm via console before I build the GTM trigger against it.

## Conversion Action Imported into Google Ads

**`booking_confirmed`** — not `booking_step_complete`, and not `guide_download_form_submit`. Google Ads' automated bidding (Target CPA / Maximize Conversions) optimises toward whatever signal it's fed. Step-level events are too noisy and too easy to trigger — optimising toward them would teach the algorithm to find people who fill in a name and phone number, not people who actually book. `booking_confirmed` is the closest digital signal to a real commercial outcome (a scheduled clinic visit), so it's the only one worth letting the bidding algorithm chase.
