# Namoza Developer Assignment — OrthoNow

Submission for: Developer – Position 1 (Client Web + Martech)

## Structure

```
task1-gtm-schema.md       → Task 01: full GTM event schema, booking funnel dataLayer JSON, Google Ads conversion pick
task2-landing-page.html   → Task 02: single-file HTML/CSS/JS "Book a Consultation" landing page
task2-pagespeed.png       → PageSpeed Insights Mobile screenshot (add after you deploy + test)
task3-integration.md      → Task 03: HubSpot + WhatsApp + Google Ads integration architecture (written answer)
```

## How to view Task 2 live

1. Enable GitHub Pages on this repo (Settings → Pages → Deploy from branch → `main` → `/root`).
2. Open the published URL, e.g. `https://<username>.github.io/<repo>/task2-landing-page.html`.
3. Open browser DevTools → Console, fill the form, submit, and confirm `window.dataLayer` shows the `consultation_form_submitted` push.
4. Run that same published URL through PageSpeed Insights (Mobile) and save the screenshot as `task2-pagespeed.png`.

## Notes

- Task 2 has no external requests (no font/image/script CDN calls) by design — this keeps Core Web Vitals high without needing image optimisation tooling.
- Task 1's booking-funnel dataLayer JSON assumes the front-end team pushes events at step transitions; GTM cannot detect in-page step changes on its own — this is called out explicitly in the schema doc.
**https://srishtiverma12.github.io/Namoza-Developer-Assignment/task2-landing-page.html**
  
