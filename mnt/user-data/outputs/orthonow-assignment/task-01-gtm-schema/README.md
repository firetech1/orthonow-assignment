# Task 01 — GTM Event Schema: OrthoNow

## 1. Full Event Schema

| Event Name | Trigger Type | Key Parameters | Feeds Into (GA4) |
|---|---|---|---|
| `booking_step_complete` | Custom Event — fired via **dataLayer push from front-end code**, not a native GTM trigger (see note below) | `step_number`, `step_name`, `clinic_location`, `specialty` (step 1) / `preferred_date` (step 2) | Funnel Exploration report; "Booking Funnel" audience for remarketing to drop-offs |
| `booking_confirmed` | Custom Event (dataLayer push on step 3 success) | `clinic_location`, `specialty`, `booking_id`, `value` (optional, e.g. consult fee) | Conversion event → Ecommerce/Key Events report; **imported into Google Ads** |
| `call_now_click` | GTM Click Trigger — Just Links, filtered on `Click URL` containing `tel:` | `page_location`, `clinic_location` (from page or data attribute), `click_text` | Engagement report; "High Intent — Call" audience |
| `whatsapp_chat_open` | GTM Click Trigger — Just Links, filtered on `Click URL` containing `wa.me` | `page_location`, `widget_position` (e.g. "floating_button"), `click_text` | Engagement report; retargeting audience for WhatsApp-intent users |
| `patient_guide_download_start` | Form Submission Trigger on the gated form (name + phone) | `form_id`, `page_location`, `lead_source` | Lead-gen funnel step 1 |
| `patient_guide_download_complete` | Custom Event — fired on successful PDF trigger/redirect (dataLayer push, since native Form Submit fires on click, not on confirmed gate-pass) | `file_name`, `file_url`, `page_location` | File Download report; **conversion candidate** |
| `clinic_page_view` | GTM Trigger — Page View, filtered on URL path pattern `/clinics/*` (or a `clinic_page` dataLayer variable set per template) | `clinic_name`, `clinic_city`, `page_location` | Pages and Screens report, segmented by clinic; feeds "Location Interest" audience for geo-targeted remarketing |
| `blog_scroll_depth` | GTM Trigger — Scroll Depth (Vertical, thresholds 25/50/75/90%), filtered to blog templates | `percent_scrolled`, `page_location`, `article_title` | Engagement report; content-affinity audience for nurture campaigns |
| `blog_read_complete` | Scroll Depth trigger at 90% **AND** min. time-on-page (25s+) via a custom JS trigger condition | `article_title`, `article_category`, `time_on_page` | Engagement / content performance report |

> Every event above carries at least 3 parameters as required. `page_location` and `page_referrer` are also auto-collected by GA4's base config tag on every hit, so they're not double-counted as "custom" params where GA4 already supplies them.

---

## 2. Booking Funnel — Step-Level Drop-off Tracking

**The core thing to get right here: GTM cannot natively "see" a multi-step form.** A 3-step form is almost always one page (or one SPA route) where steps are shown/hidden via JavaScript — there's no page load, no URL change, and no native GTM trigger (Click, Form Submit, Page View) fires *specifically* when a user moves from step 1 to step 2. GTM only reacts to events that already exist in the browser (clicks, page views, timers, native form submits) or to signals it's explicitly told about via `dataLayer.push()`.

So step-level tracking requires the **front-end developer to push a dataLayer event at each step transition** — GTM's job is only to listen for that event and route it to GA4 via a Custom Event trigger. This is a collaboration point, not something I can wire up unilaterally from GTM alone.

### What fires at each step

| Step | What triggers the push (front-end logic) | GTM Trigger | Tag fired |
|---|---|---|---|
| Step 1 → 2 | User selects clinic + specialty and clicks "Next" — front-end validates fields client-side, then pushes to dataLayer | Custom Event trigger, Event name = `booking_step_complete`, condition `step_number = 1` | GA4 Event tag → `booking_step_complete` |
| Step 2 → 3 | User enters name/phone/date and clicks "Next" | Custom Event trigger, `step_number = 2` | GA4 Event tag → `booking_step_complete` |
| Step 3 (confirm) | User clicks final "Confirm Booking" and the backend returns a success response (not just on click — on confirmed success) | Custom Event trigger, Event name = `booking_confirmed` | GA4 Event tag → `booking_confirmed` (marked as key event) |

### Actual dataLayer JSON (not pseudocode)

**Step 1 complete — location & specialty selected:**
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Indiranagar - Bengaluru",
  "specialty": "Knee & Sports Injury"
}
```

**Step 2 complete — contact details entered:**
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "Indiranagar - Bengaluru",
  "preferred_date": "2026-07-05"
}
```

**Step 3 — booking confirmed:**
```json
{
  "event": "booking_confirmed",
  "clinic_location": "Indiranagar - Bengaluru",
  "specialty": "Knee & Sports Injury",
  "booking_id": "ORTH-2026-004821"
}
```

### Briefing the front-end dev (what I'd actually say)

"For each step's 'Next'/'Confirm' button, after client-side validation passes and before advancing the UI, fire `window.dataLayer.push({...})` with the exact key names above — `event`, `step_number`, `step_name`, plus the relevant fields already sitting in the form's state object. Don't push on every keystroke or on validation failure, only on a successful step transition. I'll handle everything downstream in GTM (triggers, GA4 tags, funnel report) — I just need those pushes to fire reliably and exactly once per step."

### Surfacing drop-off in GA4 Funnel Exploration

1. Create a new **Funnel Exploration** in GA4 Explore.
2. Steps: `booking_step_complete` (step_number=1) → `booking_step_complete` (step_number=2) → `booking_confirmed`.
3. Set the funnel to **"open funnel"** (not closed) initially, so we can see users entering mid-funnel too — then switch to closed for cleaner completion-rate reads.
4. Turn on **"Show elapsed time"** to catch not just *where* people drop but *how long* they sat on a step before abandoning (a proxy for friction/confusion vs. a hard blocker).
5. Break down each step by `device_category` — mobile drop-off on step 2 (the data-entry step) is the single most common healthcare-form failure pattern, worth isolating early.

---

## 3. Conversion Action to Import into Google Ads

**Import `booking_confirmed`, not `booking_step_complete` or `patient_guide_download_complete`.**

Why this one over the others:
- It's the closest event to actual business value (a confirmed appointment), so Google's bidding algorithms (Target CPA/ROAS) optimize toward people who look like *patients*, not just form-starters.
- `booking_step_complete` (step 1) is tempting because it's a bigger, "easier" volume signal — but importing an early-funnel micro-conversion trains Smart Bidding to chase browsers who select a clinic and bounce, which historically drags down lead quality within 2–3 weeks of optimization.
- `patient_guide_download_complete` is a real lead but a colder one (content download intent ≠ booking intent) — it's better kept as a *secondary* conversion for remarketing list building, not as the primary optimization signal.
- Given OrthoNow's current 2.1% conversion rate, volume on `booking_confirmed` alone may be too thin for Smart Bidding to learn from in month one — worth flagging that we may need to run on Maximize Clicks or manual CPC initially and switch to Target CPA once we've banked ~30 conversions/month, the rough threshold Google recommends for exiting the learning phase.
