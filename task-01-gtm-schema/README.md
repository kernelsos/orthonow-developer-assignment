# Task 01 — GTM Event Schema (OrthoNow)

This is the full event tracking plan for OrthoNow's site, to be implemented in GTM before any paid campaigns go live. It covers every interaction listed in the brief, the booking funnel drop-off design, and one recommended Google Ads conversion import.

---

## 1. Full Event Schema

| Event Name | Trigger Type (GTM) | Key Parameters | Feeds Into (GA4) |
|---|---|---|---|
| `booking_form_start` | Custom Event — fires on dataLayer push when user lands on / opens Step 1 of the booking form | `form_location` (page URL where form opened), `entry_point` (e.g. "header_cta", "landing_page_hero"), `clinic_preselected` (true/false, if clinic was passed via URL param) | GA4 Funnel Exploration (denominator step 0); "Booking Started" audience |
| `booking_step_complete` (step 1) | Custom Event — dataLayer push from front-end JS when user completes location + specialty selection and clicks "Next" | `step_number: 1`, `step_name`, `clinic_location`, `specialty` | GA4 Funnel Exploration (step 1); remarketing audience "Selected clinic, did not continue" |
| `booking_step_complete` (step 2) | Custom Event — dataLayer push from front-end JS when user submits name/phone/preferred date and clicks "Next" | `step_number: 2`, `step_name`, `clinic_location`, `specialty`, `preferred_date` | GA4 Funnel Exploration (step 2); remarketing audience "Entered details, did not confirm" |
| `booking_confirmed` | Custom Event — dataLayer push from front-end JS on successful confirmation (Step 3), ideally fired after backend returns a booking ID | `step_number: 3`, `clinic_location`, `specialty`, `booking_id`, `preferred_date` | GA4 Key Event / Conversion; Google Ads conversion import; "Converted Patients" audience (used as exclusion list for remarketing) |
| `call_now_click` | Click — Just Links, scoped via Click URL contains `tel:`, fired on homepage, all 9 clinic pages, and the landing page | `page_type` (homepage / clinic_page / landing_page), `clinic_name` (blank on homepage), `phone_number_clicked` (the tracking number variant shown, for call-tracking QA) | GA4 Engagement report; marked as Key Event; optional Google Ads "Calls from ads" or website call conversion |
| `whatsapp_click` | Click — Just Links, scoped via Click URL contains `wa.me` | `page_type`, `page_location`, `clinic_context` (clinic name if clicked from a clinic page, else "general") | GA4 Engagement report; Key Event; "High intent — chat initiated" audience |
| `patient_guide_lead_submit` | Form Submission (built-in GTM trigger) on the gating form, scoped to the guide-download page | `form_location`, `lead_source`, `gated_asset_name` (e.g. "Knee Pain Patient Guide") — **no name/phone values pushed into dataLayer/GA4, only booleans/IDs, to avoid sending PII to Google** | GA4 Lead Generation report; "Guide downloaders" nurture audience |
| `file_download` | Click — Just Links, scoped to Click URL ending in `.pdf`, fires only after the gate form succeeds | `file_name`, `file_extension`, `link_url` | GA4 recommended `file_download` event; content engagement report |
| `clinic_page_view` | Page View — DOM Ready, scoped to URL path matching `/clinics/*` | `clinic_name` (from a Data Layer Variable or URL path variable), `clinic_city`, `page_location` | GA4 Pages and Screens report, segmented by clinic; "Location interest" audience used for geo-targeted remarketing |
| `blog_scroll_depth` | Scroll Depth (built-in GTM trigger), % thresholds 25/50/75/90, scoped to URL path matching `/blog/*` | `percent_scrolled`, `article_title` (DLV or Page Title variable), `article_category` | GA4 recommended `scroll` event; content engagement report; "Engaged readers" audience for blog-to-booking nurture |

**Note on PII:** None of the events above push name, phone number, or other personal identifiers into the dataLayer. GA4's terms prohibit sending PII, and Google Ads conversion imports inherit whatever GA4 collects — so the actual name/phone values only ever go to the backend/CRM via the form's normal POST, never through GTM/dataLayer.

---

## 2. Booking Funnel Drop-Off Tracking (3-Step Form)

### Implementation note — who writes this code

**The front-end developer writes the dataLayer pushes, not GTM.** GTM has no native way to detect "the user is now on step 2 of a form" — it can only listen for things that are explicitly pushed to `window.dataLayer`, or for native DOM events like clicks and built-in form submissions. A 3-step form is usually a single page with JS-driven state changes (no real page load between steps), so there's no URL change or form `submit` event GTM can hook into for steps 1 and 2 — only the final submission is a "real" form submit.

So the brief to the front-end dev is:
1. At the moment the user successfully completes Step 1 and clicks "Next" (after client-side validation passes), fire `window.dataLayer.push({...})` with the exact event name and parameters below — *before* or *as* the UI transitions to Step 2.
2. Same for Step 2 → Step 3.
3. On final confirmation, fire `booking_confirmed` only once the booking is actually accepted (ideally after a success response from the backend, not just on button click, so we don't count failed submissions as conversions).
4. Event names and parameter keys must match exactly what's specified here — GTM triggers match on exact string values.

### GTM Trigger Setup
One **Custom Event** trigger (`CE - booking_step_complete`) listens for `event: 'booking_step_complete'` and fires on all three steps. Step number is *not* split into three separate triggers — that would be unnecessary duplication. Instead, a single GA4 Event tag fires on this trigger and maps `step_number` and `step_name` as event parameters pulled from Data Layer Variables. GA4 then differentiates steps using the parameter value, not separate GTM triggers.

A second trigger, `CE - booking_confirmed`, listens for `event: 'booking_confirmed'` specifically — this is kept as a distinct event (not `step_number: 3`) so it can be cleanly marked as a GA4 Key Event and imported into Google Ads without including the in-progress steps.

### dataLayer JSON — exact pushes

**Step 1 complete (location + specialty selected):**
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

**Step 2 complete (name/phone/date entered):**
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "preferred_date": "{{preferred appointment date}}"
}
```

**Step 3 — booking confirmed (final conversion):**
```json
{
  "event": "booking_confirmed",
  "step_number": 3,
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "preferred_date": "{{preferred appointment date}}",
  "booking_id": "{{booking confirmation ID from backend}}"
}
```

Note: name and phone number are deliberately **not** included in these payloads — they're collected by the form and sent to the CRM directly, not pushed through the dataLayer.

### Surfacing Step-Level Drop-Off in GA4 Funnel Exploration

1. In GA4, go to Explore → Funnel Exploration.
2. Build an **open funnel** (not closed) with 4 steps so partial drop-off before Step 1 is visible too:
   - Step 1: `booking_form_start`
   - Step 2: `booking_step_complete` where `step_number = 1`
   - Step 3: `booking_step_complete` where `step_number = 2`
   - Step 4: `booking_confirmed`
3. Set "Show elapsed time" on to see how long users sit on each step before dropping or converting — useful for spotting if Step 2 (the form-fill step) is where people stall, which usually signals friction in field count or validation errors rather than indecision.
4. Use the funnel's built-in drop-off percentage between each step to identify the single highest-leak step, then break that step down by a secondary dimension (e.g. `clinic_location` or device category) to see if drop-off concentrates in one clinic's flow or on mobile specifically.
5. Save the funnel's "did not complete next step" segment at each stage as a GA4 Audience, which feeds remarketing (e.g. retarget users who selected a clinic but never entered details).

---

## 3. Google Ads Conversion Import Recommendation

**Import `booking_confirmed` as the Google Ads conversion action — not `call_now_click`, `whatsapp_click`, or `patient_guide_lead_submit`.**

Reasoning:
- It's the only event in this schema that represents a completed, backend-confirmed appointment booking — the actual business outcome Namoza is being paid to drive, not a proxy for intent.
- `call_now_click` and `whatsapp_click` measure *intent to contact*, not a completed booking — counting them as the primary conversion would have Google's bidding algorithms optimize toward clicks that don't reliably turn into appointments, inflating apparent performance.
- `patient_guide_lead_submit` is a softer, top-of-funnel lead (someone researching, not necessarily booking) and is better used as a secondary/observation conversion for remarketing list building rather than the primary signal fed to Smart Bidding.
- Because `booking_confirmed` only fires after the backend returns a successful booking ID, it's resistant to inflation from abandoned or failed submissions — which matters a lot once Smart Bidding strategies start optimizing spend against this signal.

Recommendation: set `booking_confirmed` as the **primary** conversion action (counted in "Conversions" column, used for bidding), and import `call_now_click`, `whatsapp_click`, and `patient_guide_lead_submit` as **secondary/observation-only** conversions so the marketing team retains visibility into the full funnel without diluting the bidding signal.