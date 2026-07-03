# OrthoNow — Interview Deliverables

This package contains the three completed tasks:

1. **Task01_GTM_GA4_Tracking_Plan.md**
   Complete GTM event schema, booking-funnel tracking design, real dataLayer.push()
   JSON for each step, Google Ads conversion recommendation, and full technical flow.

2. **Task02_Landing_Page/orthonow-landing.html**
   Self-contained HTML/CSS/vanilla-JS landing page for the "Book a Consultation" campaign.
   Open the file directly in a browser — no server required. Fires
   `window.dataLayer.push({event: 'consultation_form_submitted', ...})` only on a
   successful form submit (visible via DevTools Console → `window.dataLayer`), never
   on page load.

   Note: this package does not include a PageSpeed Insights screenshot. Generating a
   genuine score requires hosting the file at a reachable URL (or running Lighthouse
   locally in Chrome DevTools) — that step should be run in your own environment
   before submission, rather than a claimed/fabricated score.

3. **Task03_Integration_Design_Answer.md**
   HubSpot + Karix WhatsApp + Google Ads integration architecture, including the
   corrected phone-vs-name deduplication logic for the "same phone number, different
   patient" edge case.

Generated: July 3, 2026
