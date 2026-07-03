# Task 01 — GTM Event Schema (OrthoNow)

## 1. Full Event Schema

| Event Name | Trigger Type | Key Parameters | GA4 Report / Audience it feeds |
|---|---|---|---|
| `clinic_page_view` | GTM Trigger: History Change / Page View, fires on `/clinics/*` URL pattern | `clinic_name`, `clinic_city`, `page_path` | Engagement → Pages and screens; Audience: "Visited a clinic page" (used for remarketing) |
| `phone_call_click` | Click Trigger on `tel:` links (Click ID/Class = `call-now-btn`) | `click_location` (homepage/clinic page/landing page), `clinic_name`, `phone_number` | Engagement → Events; feeds "Call Click" conversion event; Audience: "High intent — called" |
| `whatsapp_click` | Click Trigger on elements linking to `wa.me/*` | `click_location`, `page_path`, `widget_state` (open/closed) | Engagement → Events; Audience: "Engaged via WhatsApp" |
| `patient_guide_form_submit` | Form Submission trigger on gated PDF form | `form_name` = `patient_guide`, `lead_source_page`, `has_phone_number` (bool) | Engagement → Events; feeds a secondary (micro) conversion; Audience: "Downloaded guide — nurture" |
| `patient_guide_download` | Custom Event fired after successful form submit + PDF link resolves | `file_name`, `clinic_interest` (if asked on form), `form_name` | Engagement → File downloads report; Audience: "Guide downloaders" |
| `blog_scroll_depth` | Scroll Depth trigger at 25/50/75/90% thresholds | `scroll_percentage`, `article_title`, `article_category` | Engagement → Events, layered with GA4's built-in `scroll` event; Audience: "Engaged readers (75%+)" for content remarketing |
| `booking_step_view` | Custom Event, dataLayer push when each form step renders | `step_number`, `step_name`, `clinic_location` (once selected) | Funnel Exploration (step entry); Audience: "Started booking" |
| `booking_step_complete` | Custom Event, dataLayer push on "Next"/"Continue" click per step | `step_number`, `step_name`, `clinic_location`, `specialty` | Funnel Exploration (step completion); feeds drop-off calculation |
| `booking_confirmed` | Custom Event, dataLayer push on final confirmation screen render (not just button click — must confirm server-side success) | `booking_id`, `clinic_location`, `specialty`, `appointment_date` | Primary conversion event → imported into Google Ads |
| `booking_abandoned` (optional, inferred) | Timer/History trigger — session ends or exits without `booking_confirmed` after a `booking_step_view` fired | `last_step_reached`, `clinic_location`, `time_on_form` | Funnel Exploration drop-off segment; Audience: "Abandoned booking" for remarketing |

**Note on WhatsApp and Call tracking:** these are native `href` clicks, so GTM's built-in Click trigger works without any custom dataLayer code from the front-end team. This is different from the booking form (see below).

---

## 2. Booking Funnel — Step Drop-off Tracking

**The key point:** GTM cannot auto-detect "step 2 of a 3-step form" the way it can auto-detect a click or a URL change, because all three steps usually live on one page/component with no URL change and no native browser event marking "step advanced." **The front-end developer has to manually push a dataLayer event at each step transition.** GTM only listens — it does not know the form's internal state.

### What fires at each step

- **Step 1 (location + specialty selected):** front-end dev pushes `booking_step_complete` (step 1) the moment the user clicks "Continue" *and* validation passes.
- **Step 2 (name/phone/date entered):** same pattern — pushed on "Continue" from step 2, only after client-side validation passes (so we don't count invalid attempts as progress).
- **Step 3 (confirm booking):** push `booking_confirmed` only after the booking API returns success — not on button click alone, otherwise failed submissions get counted as conversions.

In GTM, each of these is a **Custom Event trigger** listening for the event name (`booking_step_complete`, `booking_confirmed`) — GTM does not need a separate trigger per step; the `step_number` parameter differentiates them inside GA4.

### Example dataLayer pushes (actual JSON, not pseudocode)

**Step 1 complete:**
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Koramangala",
  "specialty": "Knee & Sports Injury"
}
```

**Step 2 complete:**
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "Koramangala",
  "specialty": "Knee & Sports Injury",
  "preferred_date": "2026-07-10"
}
```

**Step 3 / final confirmation (fires only after API success):**
```json
{
  "event": "booking_confirmed",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "booking_id": "ON-48213",
  "clinic_location": "Koramangala",
  "specialty": "Knee & Sports Injury",
  "appointment_date": "2026-07-10"
}
```

### Surfacing drop-off in GA4 Funnel Exploration

1. Create a new **Funnel Exploration**.
2. Steps, in order:
   - Step 1 = `booking_step_view` (form opened / step 1 shown)
   - Step 2 = `booking_step_complete` where `step_number = 1`
   - Step 3 = `booking_step_complete` where `step_number = 2`
   - Step 4 = `booking_confirmed`
3. Turn on **"Show elapsed time"** and set the funnel to **"Open funnel"** (so we still see users who entered mid-way, e.g. from a saved link), but keep it in strict order since steps are sequential.
4. GA4 will render the drop-off % between every consecutive step automatically — that % between step 2 and step 3, for example, is exactly where most healthcare booking forms leak (usually the phone-number field, due to trust/spam concerns).
5. Break down the funnel by `clinic_location` as a segment to see if drop-off is worse for specific clinics (signals a local ops issue, not a UX issue).

**Briefing the front-end dev, in short:** "GTM can see clicks and URL changes for free. It cannot see internal React/Vue state changes inside a single-page form. Every time the form moves from one step to the next, you need to add one line — `window.dataLayer.push({...})` — with the exact keys above. I'll build the GTM triggers and GA4 events around whatever you push; I just need the event name and step number to be exact and consistent every time."

---

## 3. Which Conversion to Import into Google Ads

**Import `booking_confirmed` — not `phone_call_click`, `whatsapp_click`, or `patient_guide_form_submit`.**

**Why:**
- It is the only event that represents a **completed, revenue-relevant action** (an actual scheduled appointment), not just an expression of interest. Google Ads' Smart Bidding optimises toward whatever conversion you feed it — feed it a soft signal like a call click, and Ads will spend budget chasing tyre-kickers who click "call" out of curiosity.
- Call clicks and WhatsApp clicks are useful as **secondary/observed conversions** for reporting, but they're noisy: a call click doesn't confirm a call happened or led anywhere, and WhatsApp clicks include people who just wanted directions.
- `patient_guide_form_submit` sits too far up the funnel — it's a lead magnet, not a purchase-intent action; optimising Ads spend toward it would fill the funnel with low-intent downloaders rather than people who actually want an appointment.
- `booking_confirmed` also has a clean, fireable-once-per-user trigger (fired only after backend success), which avoids the double-counting risk you'd get bidding on a button click that might fire twice.
