# Task 01: GTM event schema for OrthoNow

**Client:** OrthoNow (9 orthopaedic clinics across Bengaluru, Hyderabad, Chennai)
**Goal:** Set up event tracking in GTM and GA4 before any paid campaigns start, so the marketing team can see what users actually do on the site instead of only counting pageviews.

---

## 1. Complete event schema

> **PII:** Name, phone, and email never go into the dataLayer or GA4. That's GA4's own policy, and it's standard practice for Indian healthcare lead gen. Those values stay in the CRM and backend. GA4 only receives non-identifying context like clinic and specialty.

| # | Event Name | Trigger Type | Key Parameters (min 3) | Feeds Into (GA4 report / audience) |
|---|-----------|--------------|------------------------|-------------------------------------|
| 1 | `booking_step_complete` | Custom Event (dataLayer push from the front-end dev, fired at each of the 3 form steps) | `step_number` (1/2/3), `step_name`, `clinic_location`, `specialty` | Funnel Exploration (booking drop-off) plus a "Started Booking" remarketing audience |
| 2 | `call_now_click` | Click (Click URL contains `tel:`, or click element matches a Call button) | `click_location` (homepage / clinic_page / landing_page), `clinic_location`, `page_path` | Events report plus a "High-Intent Callers" audience; secondary Google Ads conversion |
| 3 | `whatsapp_click` | Click (Click URL contains `wa.me`) | `click_location`, `clinic_location`, `page_path` | Engagement report plus a "WhatsApp Engagers" audience |
| 4 | `patient_guide_download` | Custom Event (pushed on successful gated-form submit; fires on submit, not on page load) | `guide_name`, `click_location`, `page_path` | Lead-gen report plus a "Guide Downloaders" audience |
| 5 | `clinic_page_view` | Page View or History Change (Page Path matches `/clinics/*`) | `clinic_name`, `clinic_city`, `page_path` | Location performance report plus city audiences (Bengaluru / Hyderabad / Chennai) |
| 6 | `blog_article_read` | Scroll Depth, GTM built-in (fires at 25 / 50 / 75 / 90%) | `scroll_percentage`, `article_title`, `article_category` | Content engagement report plus a "Blog Readers" audience |

Custom dimensions to register in GA4 (Admin > Custom definitions) so these parameters work in reports and audiences: `step_number`, `step_name`, `clinic_location`, `specialty`, `click_location`, `clinic_name`, `clinic_city`, `scroll_percentage`, `article_category`.

---

## 2. Booking form funnel: step-level drop-off tracking

The booking form is a single-page, 3-step flow with no page reload between steps.

> GTM cannot see the steps inside a multi-step form on its own. Steps 1 and 2 don't submit or reload the page, so there's no built-in event for GTM to catch. The front-end developer has to push a `dataLayer` event at the end of each step. I write the schema; the dev wires each push into the form's step-advance logic. (See "Implementation ownership" below.)

### GTM triggers

One Custom Event trigger per step. All three listen for the same event name and filter on `step_number`:

| Step | GTM Trigger | Trigger Condition |
|------|-------------|-------------------|
| Step 1: location + specialty | Custom Event | Event equals `booking_step_complete` and `step_number` equals `1` |
| Step 2: name / phone / date | Custom Event | Event equals `booking_step_complete` and `step_number` equals `2` |
| Step 3: confirm booking | Custom Event | Event equals `booking_step_complete` and `step_number` equals `3` |

Each trigger fires a GA4 Event tag that forwards `booking_step_complete` with all its parameters.

### dataLayer pushes (actual JSON)

Step 1, after clinic and specialty are selected:
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Indiranagar, Bengaluru",
  "specialty": "Knee Care"
}
```

Step 2, after name / phone / preferred date are entered (no PII in the push):
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "Indiranagar, Bengaluru",
  "specialty": "Knee Care",
  "preferred_date_selected": true
}
```

Step 3, after the booking is confirmed (this step is the conversion):
```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Indiranagar, Bengaluru",
  "specialty": "Knee Care",
  "booking_id": "ON-2026-04821"
}
```

### Surfacing drop-off in GA4 Funnel Exploration

1. Go to Explore > Funnel exploration and set it as a closed funnel, so users have to pass the steps in order.
2. Define 3 steps. Each one is the event `booking_step_complete` with a condition on the `step_number` custom dimension: step 1 = `1`, step 2 = `2`, step 3 = `3`.
3. GA4 then shows the completion and abandonment percentage between each step. If 1,000 users hit step 1, 620 reach step 2, and 290 confirm, you can see exactly where they drop.
4. Add `clinic_location` or `specialty` as a breakdown to see which clinic or specialty drops off worst. That's what the marketing team acts on.

---

## 3. Google Ads conversion to import

**Import `booking_step_complete` where `step_number = 3` (booking confirmed).**

Why this one over the others:

- It's the actual business outcome: a confirmed booking, which is the closest online action to real revenue. Smart Bidding optimises toward whatever conversion you feed it, so you want it chasing booked patients rather than cheaper, weaker signals.
- `call_now_click` and `whatsapp_click` show intent but aren't qualified. A click isn't a booking, and bidding toward them buys volume that doesn't convert.
- Steps 1 and 2 are mid-funnel. Importing those as the primary conversion would train bidding toward form-starters who never finish.

Practical caveat: at around 2.1%, confirmed bookings may be low volume early on, and Smart Bidding needs enough conversions to learn. So I'd set `booking_step_complete (step 3)` as the primary conversion and add `call_now_click` as a secondary one for volume, while keeping only the booking as the optimisation target.

---

## Implementation ownership (who writes what)

| Piece | Owner |
|-------|-------|
| Event schema, naming, parameters, triggers, GA4 config, custom dimensions | Me (martech / this task) |
| The `dataLayer.push()` calls inside the 3-step form's JS | Front-end developer. I hand them the JSON above and the exact spot in the step-advance handler for each push. |
| Google Ads conversion setup and linking | Me, with the paid team |

Dev brief for Step 2 (example): in the form's `goToStep3()` handler, right after step-2 validation passes and before step 3 renders, call `window.dataLayer.push({ event: 'booking_step_complete', step_number: 2, step_name: 'contact_details_entered', clinic_location: <selected clinic>, specialty: <selected specialty>, preferred_date_selected: true })`. Leave name and phone out. Read `clinic_location` and `specialty` from the values captured in step 1.
