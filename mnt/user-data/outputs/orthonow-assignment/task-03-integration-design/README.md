# Task 03 — Integration Design: Landing Page → HubSpot → WhatsApp → Google Ads

## Architecture

I'd route the form through a **direct API call from our own backend**, not the native HubSpot
embed and not Zapier/Make. Reasoning: the form fires two side effects HubSpot doesn't natively
handle (a WhatsApp send via Karix, and a Google Ads conversion), so no matter what, something has
to sit outside HubSpot orchestrating them. Zapier/Make can do this, but they add 15–90 seconds of
polling-based latency per step by default, which eats directly into our 2-minute WhatsApp SLA, and
they're a second system to monitor and debug. A native HubSpot embed is out entirely — it can't
run our custom validation, can't trigger the WhatsApp send, and ties us to HubSpot's own hosted
form URL and error handling instead of ours.

So: the landing page form submits via `fetch()` to a lightweight endpoint we control (e.g. a
Node/Express or Cloud Function). That endpoint does three things in sequence:

1. **HubSpot**: call the HubSpot Contacts API (`PATCH /crm/v3/objects/contacts` with an `idProperty`
   set to a custom **Phone (unique)** property — see the dedup note below) to create-or-update the
   contact with Name, Phone, Clinic Preference, Source, and Lead Status.
2. **WhatsApp**: call Karix's WhatsApp Business API to send a templated confirmation message,
   using the phone number just captured.
3. **Google Ads**: fire the `consultation_form_submitted` conversion — either via the existing
   GTM dataLayer push (Ads conversion tag already listening for that event, as set up in Task 01)
   or, more reliably for a backend-confirmed lead, via the Google Ads Enhanced Conversions API
   called server-side once HubSpot confirms the contact was actually created.

Steps 1–3 don't need to be strictly sequential — HubSpot and the WhatsApp send should fire in
parallel (`Promise.all`) since neither depends on the other's result, which also helps the SLA.
The Ads conversion can fire immediately client-side (fast, keeps Smart Bidding data fresh) with a
server-side confirmation as a backup, rather than waiting on the full chain.

## Biggest Failure Point + Fallback

The **HubSpot phone-number deduplication** is the biggest risk, and it's not really about uptime —
it's about data integrity. HubSpot's default dedup key is **email**, not phone. Since this form
only collects Name and Phone, two different patients submitting under the same name, or the same
patient submitting twice with a slightly different name, won't get deduplicated by HubSpot's
defaults at all — we'd get duplicate contact records, which then means duplicate WhatsApp sends
and a confusing pipeline for the clinic coordinators calling leads back.

The fix: create a **custom unique property in HubSpot (`phone_normalized`)**, normalize every
incoming number server-side (strip spaces/dashes, enforce a consistent `+91XXXXXXXXXX` format)
before the API call, and use that property as the `idProperty` on the upsert request so HubSpot
matches on phone, not email. If two patients genuinely share a phone number (a common real-world
case — shared family phones, or someone booking for a parent), I'd not silently overwrite the
existing contact's name; instead, log the name mismatch and append it as a note/timeline entry on
the contact, and route it to a coordinator to confirm manually rather than guessing which patient
is which.

The secondary fallback: if the HubSpot API call itself fails (rate limit, timeout, auth expiry),
the backend queues the payload (a simple database row or a message queue) and retries with
backoff, so a transient HubSpot outage doesn't silently drop a lead — the WhatsApp confirmation
and Ads conversion should still fire even if the CRM write is queued for retry, since the patient
experience shouldn't depend on HubSpot's uptime.

## Protecting the 2-Minute WhatsApp SLA

What could break it: Karix API rate limits or downtime, WhatsApp template approval issues
(Meta occasionally pauses templates for policy review), our own backend being slow under load, or
the parallel HubSpot call blocking the WhatsApp send if they're wired sequentially instead of in
parallel.

Monitoring: log a timestamp at form submission and at WhatsApp delivery confirmation (Karix
returns delivery webhooks), then alert if the delta exceeds 90 seconds — giving a 30-second buffer
before the SLA is actually breached. I'd put this on a simple dashboard (or a Slack alert) so
someone notices before a patient does, plus a daily rollup of P50/P95 send times to catch slow
creep before it becomes a full SLA breach.
