# Task 03: Integration Design

## End-to-end architecture

When the patient submits the form, the landing page fires `window.dataLayer.push({ event: 'consultation_form_submitted', ... })`. GTM picks this up and does two things in parallel: fires the Google Ads conversion tag client-side via gtag, and sends a POST to our backend endpoint with name, phone, and clinic preference.

Keeping Google Ads client-side is deliberate. It fires on submit, independent of any server response, so no backend delay affects conversion tracking.

The backend is a small Node.js/Express function on Railway. It handles two steps in sequence:

**Step 1: HubSpot via direct API call**, not Zapier, not Make, not native embed. Native embed breaks our custom thank-you state. Zapier and Make add per-task costs that compound at scale and introduce a third-party failure point. A direct call keeps the logic in our codebase and is faster to debug.

The critical detail: HubSpot deduplicates contacts on email by default, not phone. This form collects no email. If two patients submit with the same phone number, HubSpot creates two separate contacts. The fix is to call the HubSpot Search API first, filtering on the phone property, then update if a match exists or create if not. This upsert has to be built manually — there is no native phone-based dedup in HubSpot.

Fields written: `firstname`, `phone`, `hs_lead_status` = New Enquiry, `lead_source` = Google Ads - Consultation Landing Page, and a custom property `clinic_preference`.

**Step 2: Karix WhatsApp API call**, made immediately after the HubSpot write succeeds. We POST to Karix with the patient's phone and a pre-approved template confirming their enquiry and expected callback time.

## Biggest failure point

The HubSpot API call failing silently and the lead being lost. Fallback: on any non-200 response, the backend writes the raw payload to a dead-letter collection in MongoDB. A cron retries those records every 5 minutes. Ops gets a Slack alert if anything sits there longer than 15 minutes. The patient sees the thank-you state immediately regardless, because the form UX never waits on backend latency.

## WhatsApp 2-minute SLA

Three things can break it:

**Karix downtime or latency.** We set a 10-second timeout. On failure, the payload goes to the dead-letter queue and retries within 60 seconds, still inside the 2-minute window.

**Backend cold starts.** Railway containers idle down. A keep-alive cron pings `/health` every 60 seconds so the container stays warm.

**Phone number format.** Karix requires E.164 format (`+91XXXXXXXXXX`). We normalise before the API call: strip non-digits, prepend `+91` to any bare 10-digit number.

Monitoring: every Karix call logs a dispatch timestamp. We expect a delivery webhook from Karix within 30 seconds. If nothing arrives in 90 seconds, we log it as a failed delivery and fire a Slack alert — giving near real-time visibility on SLA breaches.
