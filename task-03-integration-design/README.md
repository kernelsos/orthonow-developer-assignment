# Task 03 — Integration Design

## End-to-end architecture

The form posts directly to a **backend endpoint I control** — a small serverless function (AWS Lambda or a Vercel/Netlify function) that receives `name`, `phone`, and `clinic_preference`. This function is the single source of truth for the submission and does two things in parallel, plus one independent client-side step:

1. **HubSpot contact upsert** — call the HubSpot Contacts API directly (`POST /crm/v3/objects/contacts`, upserted via a custom unique property), rather than HubSpot's native embed or Zapier/Make. A direct call gives full control over the dedup key (below), lets me set custom properties (`clinic_preference`, `source`, `lead_status`) in one request, and avoids the extra latency of a middleware layer on a flow with a hard 2-minute SLA.
2. **WhatsApp send via Karix** — fired in parallel with the HubSpot call (not after it), using a pre-approved template message with the patient's name and clinic.
3. **Google Ads conversion** — the browser-side `consultation_form_submitted` dataLayer push (from Task 02) fires independently via GTM on submit, so it doesn't wait on either backend call.

## Biggest failure point: HubSpot's default deduplication

**HubSpot deduplicates contacts by email by default — not phone.** Since this form collects no email, two patients submitting with the same phone number but different names would silently create two separate contact records, splitting their history and confusing lead status tracking. The fix: create a custom unique-value property (`patient_phone`) on the Contacts object, and use HubSpot's upsert-by-custom-property behavior (via the `idProperty` parameter in the API call) instead of relying on the default email match. This makes phone number the actual dedup key, matching how the business actually identifies a patient.

## WhatsApp 2-minute SLA

What could break it: Karix API rate limits or downtime, WAN latency during peak traffic, or the HubSpot call blocking the function before Karix is even attempted. Mitigation: fire the Karix call in parallel with (not after) the HubSpot write, with a 30-second timeout and one automatic retry; log every send attempt with a timestamp; and set up a monitoring alert (e.g. a scheduled check or Karix delivery webhook) that flags any message not confirmed sent within 90 seconds, so there's buffer before the SLA is breached.

*(~362 words)*