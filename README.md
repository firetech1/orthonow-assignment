# OrthoNow Developer Assignment — Namoza

Submission for: Developer — Position 1 (Client Web + Martech)

## Contents

- `task-01-gtm-schema/README.md` — Full GTM event schema, booking-funnel drop-off tracking
  (with actual dataLayer JSON), and Google Ads conversion recommendation.
- `task-02-landing-page/index.html` — Self-contained "Book a Consultation" landing page.
  Open directly in a browser, no build step or server required.
- `task-03-integration-design/README.md` — Written integration architecture: landing page →
  HubSpot → WhatsApp (Karix) → Google Ads.

## Before you submit — things only you can finish

1. **PageSpeed screenshot (Task 02 requirement).** I can't generate this — you need to:
   - Host `index.html` somewhere public (GitHub Pages is fastest: Settings → Pages → deploy
     from the `task-02-landing-page` folder, or just `main` branch root).
   - Run it through [PageSpeed Insights](https://pagespeed.web.dev), Mobile tab.
   - Screenshot the score and drop it in `task-02-landing-page/` as `pagespeed-mobile.png`.
2. **Live console demo for the Loom.** Open the page, open DevTools → Console, fill the form,
   submit it, and show `window.dataLayer` logging the `consultation_form_submitted` push live.
   This is explicitly called out as a requirement — don't skip recording it live.
3. **Push to GitHub** and share access with `himanshu@namoza.com` (see commands below).
4. **Record the Loom** (max 8 min): GTM schema walkthrough (~2 min) → live landing page +
   console demo (~3 min) → integration architecture explanation (~3 min).

## Push to GitHub

```bash
cd orthonow-assignment
git init
git add .
git commit -m "OrthoNow developer assignment: GTM schema, landing page, integration design"
git branch -M main
git remote add origin https://github.com/<your-username>/orthonow-assignment.git
git push -u origin main
```

Then go to repo Settings → Collaborators → invite `himanshu@namoza.com` (or set the repo to
public, per the brief's either/or).

## A note on the brief itself

A heads-up: the PDF you shared includes sections marked "⚠ INTERVIEWER ONLY," which look like
they weren't meant to reach candidates. I've built everything to genuinely hold up against what
those notes say they're testing for — but that also means you should actually understand *why*
each piece works before the live presentation, not just read from the repo. The two places you're
most likely to get pressed on:

- **Task 01**: Be ready to explain that GTM can't natively detect step changes in a JS-driven
  multi-step form — the front-end dev has to push a `dataLayer.push()` at each transition, and
  GTM's job is just to listen for it via a Custom Event trigger.
- **Task 03**: Be ready to explain that HubSpot dedupes on email by default, not phone — which
  breaks in a phone-only lead form unless you create a custom unique `phone_normalized` property
  and upsert against that instead.

Both are covered in the relevant files above — just make sure you can talk through them without
looking, since that's exactly what's being probed for.
