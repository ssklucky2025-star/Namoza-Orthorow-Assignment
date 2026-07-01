# Task 01 — GTM Event Schema: OrthoNow

## 1. Full Event Schema

| Event Name | Trigger Type (GTM) | Key Parameters | GA4 Report / Audience it Feeds |
|---|---|---|---|
| `booking_step_complete` (step 1) | Custom Event trigger, listens for `dataLayer.push({event:'booking_step_complete', step_number:1})` fired by front-end on step transition | `step_number`, `step_name`, `clinic_location`, `specialty` | Funnel Exploration (Booking Funnel) · Custom Audience: "Started booking, did not finish" |
| `booking_step_complete` (step 2) | Same Custom Event trigger, filtered on `step_number == 2` | `step_number`, `step_name`, `clinic_location`, `preferred_date` | Funnel Exploration (Booking Funnel) · Remarketing audience for step-2 drop-offs |
| `booking_step_complete` (step 3) — treat as **`booking_confirmed`** | Custom Event trigger on final dataLayer push after backend confirms slot | `step_number`, `step_name`, `clinic_location`, `specialty`, `booking_id` | Conversions report · Google Ads import source |
| `call_now_click` | Trigger: Click - All Elements, filtered by CSS class `.call-now-btn`, fires on Click | `page_location`, `clinic_location` (or "sitewide" if on homepage), `phone_number_masked` | Engagement report · Audience: "High-intent, did not book" |
| `whatsapp_chat_click` | Trigger: Click - Just Links, filtered on URL contains `wa.me` | `page_location`, `entry_point` (floating widget vs inline), `clinic_location` | Engagement report · Remarketing audience |
| `patient_guide_download` | Trigger: Custom Event, fires on dataLayer push after gated form validates and PDF link is served | `file_name`, `lead_source`, `form_location` | Lead generation report · Nurture audience for email/WhatsApp follow-up |
| `clinic_page_view` | Trigger: automatically covered by GA4's enhanced measurement `page_view`, but we add a Custom Event for a clean dimension since URLs may not cleanly encode clinic name | `clinic_name`, `clinic_city`, `page_path` | Pages and Screens report, segmented by clinic · Audience: "Viewed clinic page, did not convert" |
| `blog_scroll_depth` | Trigger: Scroll Depth trigger (native GTM trigger type), vertical thresholds 25/50/75/90 | `scroll_percentage`, `article_title`, `article_category` | Engagement report · Content audience for retargeting relevant blog readers |

Notes on parameter choice: every event carries at least one dimension that lets us segment by **clinic** or **location**, since OrthoNow's core business question is which of the 9 clinics is underperforming on digital lead volume, not just aggregate site performance.

## 2. Booking Form Funnel — Step-Level Drop-off Tracking

**The core fact this section is built around: GTM cannot natively detect a JS-driven, no-page-reload transition between steps of a form.** GTM's built-in triggers (Click, Form Submit, DOM visibility) don't fire on an internal state change inside a single-page component — there's no URL change, no native form submit, no new DOM element GTM is guaranteed to see reliably. The only robust way to track this is for the **front-end developer to fire an explicit `dataLayer.push()`** at the moment each step is completed, and for GTM to listen for that push via a **Custom Event trigger**.

### Who writes the push?
The front-end developer writes the `dataLayer.push()` calls inside the application code (React/vanilla JS, whatever the booking form is built in), at the exact point each step's "Next" or "Confirm" logic runs. I write the GTM Custom Event trigger and the GA4 event tag that listens for it. This is a collaborative contract — I hand the dev a spec (the exact event names, parameter names, and data types below), and they implement the pushes in the codebase.

### Step 1 — Clinic + Specialty selected
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```
Fires: on click of the "Next" button on step 1, only after client-side validation confirms both fields are populated (fire on valid completion, not on click attempt — otherwise the funnel over-counts).

### Step 2 — Contact details entered
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "{{clinic name}}",
  "preferred_date": "{{preferred date}}"
}
```
Fires: on click of "Next" on step 2, after validation of name/phone/date fields.

**Briefing note to front-end dev for step 2 specifically:** "When the user clicks Next on the contact-details step, and only after your existing validation passes, add `window.dataLayer.push({event: 'booking_step_complete', step_number: 2, step_name: 'contact_details_entered', clinic_location: <value from step 1 state>, preferred_date: <value from this step's date field>})` before you transition the UI to step 3. Don't fire it on blur or on every keystroke — only once, on successful step completion. Keep `clinic_location` in whatever shared state/context you're already passing between steps."

### Step 3 — Booking confirmed
```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "booking_id": "{{booking id from backend response}}"
}
```
Fires: only after the backend returns a successful booking confirmation (not just on button click) — this is the true conversion event and it must reflect a real, saved appointment, not just user intent.

### GTM Configuration
1. **Variables**: create Data Layer Variables for `step_number`, `step_name`, `clinic_location`, `specialty`, `preferred_date`, `booking_id`.
2. **Trigger**: one Custom Event trigger, Event name = `booking_step_complete`, fires on All Custom Events (we don't need three separate triggers — the step number differentiates the data, not the trigger).
3. **Tag**: one GA4 Event tag, Event Name = `booking_step_complete`, passing through all the Data Layer Variables as event parameters, plus `step_number` explicitly (GA4 needs this to build the funnel).
4. **GA4 side**: register `step_number` and `clinic_location` as custom dimensions (Admin → Custom definitions) so they're queryable outside of the raw event params window.

### Surfacing drop-off in GA4 Funnel Exploration
- Go to Explore → Funnel Exploration.
- Set as an **open funnel** (not closed) so we can see people entering at step 2 or 3 without having triggered step 1 in the same session (e.g. return visitors).
- Steps: `booking_step_complete` where `step_number = 1` → `step_number = 2` → `step_number = 3`.
- Enable "Show elapsed time" to catch whether drop-off correlates with time-on-step (a UX/friction signal) not just abandonment.
- Add a breakdown dimension of `clinic_location` on the funnel visualization — this is the actionable layer, since a funnel that's fine in aggregate but broken for 2 of 9 clinics (e.g. a clinic with no doctor calendar synced) is a very different fix than a UX problem.

## 3. Google Ads Conversion Import

**Import `booking_step_complete` where `step_number = 3` (i.e. `booking_confirmed`) as the Google Ads conversion action.**

Why this one over the others:
- **Not `call_now_click`**: a call click is a strong intent signal but not a confirmed appointment — importing it as the optimization target would train Smart Bidding toward people who click to call and hang up, which inflates cost without guaranteeing a filled appointment slot.
- **Not `patient_guide_download`**: this is a top-of-funnel lead magnet. Optimizing Google Ads bidding toward PDF downloads would fill the funnel with low-intent research-stage users, not people ready to book — historically this tanks down-funnel conversion rate even as the "conversion" count looks healthy.
- **`booking_confirmed`** is the only event in this schema that reflects a backend-confirmed, real appointment. It's the closest proxy to actual revenue (a filled clinic slot), which is what Smart Bidding needs to optimize toward reliably. Everything else in the schema is useful for GA4 analysis and remarketing audiences, but only this one event should carry conversion value into Google Ads.