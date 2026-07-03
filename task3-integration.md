# Task 03 — Integration Design (OrthoNow: Landing Page → HubSpot → WhatsApp → Google Ads)

## Architecture, end-to-end

I'd use a **direct server-side API call from a lightweight middleware function** (a small Node/Express endpoint, or a serverless function on Vercel/AWS Lambda) rather than the native HubSpot embed or a no-code tool like Zapier/Make.

Flow: **Form submit → our own backend endpoint → HubSpot Contacts API (upsert) → Karix WhatsApp API → Google Ads Enhanced Conversions (server-side, via Conversion API / gtag measurement protocol).**

Why not the native HubSpot embed: it posts straight to HubSpot and gives us no hook to also trigger WhatsApp and Ads in the same flow — we'd need a workflow or webhook anyway, adding a hop and latency.

Why not Zapier/Make: they work, but add 2–5 seconds of polling/trigger latency per step, which is risky against a 2-minute WhatsApp SLA once you chain 3 steps, and they're a weaker place to write custom validation/dedup logic.

Why a direct API call: our own endpoint controls order and can run all three actions in parallel once the contact upsert succeeds, with full logging and retry logic that we own.

**Order of operations:** validate form → upsert contact in HubSpot (by phone, see below) → on success, fire WhatsApp send and Google Ads conversion ping in parallel (they don't depend on each other) → log all three outcomes to a database row for monitoring.

## The dedup trap

HubSpot's default deduplication key is **email**, not phone — and this form doesn't collect email. Without intervention, every submission creates a **new** contact, even from a returning patient, corrupting Lead Status and reporting. My middleware handles this itself: **search HubSpot by phone number custom property before creating** (via the Contacts Search API), and update that record if found, rather than relying on HubSpot's native create-or-update. If two different people submit the same phone number (e.g., a shared family phone), I'd append the new name to a "Also known as" note property and keep Lead Status on the most recent enquiry, rather than silently overwriting the original name.

## Biggest failure point + fallback

The single biggest risk is the **HubSpot upsert call failing or timing out**, since WhatsApp and the Ads conversion both currently depend on it succeeding first. Fallback: write the raw form payload to a queue (e.g., SQS) immediately on submit, before calling HubSpot — so if HubSpot fails, a retry worker reprocesses it, and the patient's WhatsApp confirmation can still fire independently off the queued payload rather than waiting on HubSpot.

## Protecting the 2-minute WhatsApp SLA

Risks: Karix API rate limits/downtime, WhatsApp template approval delays, or our own endpoint queuing behind the HubSpot call. Monitoring: log a timestamp at form-submit and at WhatsApp-send-confirmed, alert (Slack/PagerDuty) if the delta exceeds 90 seconds, and dashboard the p95 send time daily.
