# OrthoNow — GTM / GA4 Tracking Plan
**Prepared for:** Paid Google Ads Launch
**Scope:** Appointment Booking Funnel, Call Tracking, WhatsApp, Patient Guide Download, Clinic Pages, Blog Engagement
**Status:** Production-Ready — Pending Frontend Implementation

---

## 1. Complete GTM Event Schema

| Event Name | Business Purpose | GTM Trigger | dataLayer Event | Event Parameters (min. 3) | GA4 Event Name | Recommended Custom Dimensions | Notes |
|---|---|---|---|---|---|---|---|
| Booking Step 1 Viewed | Measure entry into booking funnel (location + specialty selection) | Custom Event trigger: `event equals booking_step1` | `booking_step1` | `clinic_location`, `specialty`, `step_number` | `booking_step1` | `clinic_location`, `specialty` | First funnel touchpoint; used as funnel step 1 in Exploration |
| Booking Step 2 Viewed | Measure lead-info entry (name, phone, date) | Custom Event trigger: `event equals booking_step2` | `booking_step2` | `clinic_location`, `specialty`, `preferred_date`, `step_number` | `booking_step2` | `clinic_location`, `specialty` | Do **not** capture raw name/phone as GA4 params (PII risk) |
| Booking Complete | Primary conversion — confirmed appointment | Custom Event trigger: `event equals booking_complete` | `booking_complete` | `clinic_location`, `specialty`, `booking_id`, `appointment_date` | `booking_complete` | `clinic_location`, `specialty`, `booking_id` | This is the event imported into Google Ads as Conversion |
| Call Now Click | Track call-intent clicks by page/context | Click Trigger — Click ID/Class `tel-link`, matches `href contains tel:` | `call_click` | `page_location`, `call_source` (homepage/landing/clinic), `clinic_location` | `call_click` | `call_source`, `clinic_location` | Micro-conversion; NOT the primary Ads conversion |
| WhatsApp Click | Track WhatsApp intent from floating button | Click Trigger — matches `href contains wa.me` | `whatsapp_click` | `page_location`, `button_position` (floating), `clinic_location` | `whatsapp_click` | `clinic_location` | Micro-conversion; low buyer-intent signal |
| Patient Guide Download | Capture lead + track PDF download | Custom Event trigger: `event equals guide_download` (fired after form submit success) | `guide_download` | `form_location`, `guide_name`, `lead_source` | `guide_download` | `guide_name`, `lead_source` | Requires name+phone submit before file serves; treat as secondary conversion |
| Clinic Page View | Measure interest per clinic (9 pages) | Trigger: Page View — Page Path matches RegEx `/clinics/.*` | `clinic_page_view` (or native `page_view` + `clinic_id` param) | `clinic_id`, `clinic_name`, `clinic_location` | `page_view` (enhanced) | `clinic_id`, `clinic_name` | Push `clinic_id`/`clinic_name` to dataLayer before GA4 config tag fires |
| Blog Scroll Depth | Measure content engagement on blog | Built-in Scroll Depth Trigger (25/50/75/90%) | `scroll` (GTM built-in, auto-fires) | `percent_scrolled`, `page_path`, `page_title` | `scroll` (GA4 auto-captures as `percent_scrolled`) | none required (GA4 default) | Use GTM's native Scroll Depth trigger — no custom dev work needed |

---

## 2. Booking Funnel — Detailed Tracking Design

### How each step is tracked

**Step 1 — Location & Specialty**
- Fires the moment the user completes both selections and clicks "Next" (not on page load — page load would overcount users who bounce immediately).
- Frontend developer adds a `dataLayer.push()` call inside the "Next" button's `onClick` handler, after validating both fields are filled.

**Step 2 — Name, Phone, Date**
- Fires when the user clicks "Next" from Step 2, after client-side validation passes.
- PII (name, phone) is **never** pushed into the dataLayer as GA4 parameters. Only non-PII context (`preferred_date`, `clinic_location`, `specialty`) is pushed. Name/phone go directly to the CRM/backend via API call, separate from the dataLayer.

**Step 3 — Booking Complete**
- Fires only after the backend confirms the booking was successfully created (i.e., on the "Thank You" / confirmation screen render, or on a successful API response callback) — not simply on button click. This ensures GA4 only counts *successful* bookings, not failed/abandoned submissions.

### When each dataLayer event fires (sequence)

| Step | Trigger Point | Timing |
|---|---|---|
| `booking_step1` | User clicks "Next" after selecting clinic + specialty | Client-side, on button click |
| `booking_step2` | User clicks "Next" after entering name/phone/date | Client-side, after validation passes |
| `booking_complete` | Backend confirms booking (API 200 response) | On confirmation screen render |

### How GTM listens for each event

1. GTM's Data Layer listener is always active once the GTM container snippet loads (`window.dataLayer = window.dataLayer || []`).
2. For each step, a **Custom Event Trigger** is created in GTM:
   - Trigger type: *Custom Event*
   - Event name: `booking_step1` / `booking_step2` / `booking_complete` (must match dataLayer key exactly, case-sensitive)
3. Each trigger fires an associated **GA4 Event Tag**.
4. **Data Layer Variables** (`DLV - clinic_location`, `DLV - specialty`, `DLV - booking_id`, etc.) are created in GTM to read parameter values pushed alongside each event, and are mapped into the GA4 event tag's "Event Parameters" fields.

### How GA4 receives it

- Each GTM tag is a **GA4 Event Tag** referencing the GA4 Configuration Tag (for consistent `client_id`/session context).
- Event Parameters in the GA4 tag are mapped from the Data Layer Variables (e.g., GA4 param `clinic_location` = `{{DLV - clinic_location}}`).
- GA4 receives these as standard events via the Measurement Protocol (handled internally by gtag.js/GTM), visible in GA4 under **Admin → Events** and in **DebugView** during QA.
- Because `booking_step1`, `booking_step2`, and `booking_complete` are non-standard names, they must be manually marked as **Key Events (Conversions)** in GA4 Admin (at minimum, `booking_complete`).

### How Funnel Exploration is configured

In GA4 → Explore → Funnel Exploration:

1. **Step 1:** `booking_step1` (condition: event name = `booking_step1`)
2. **Step 2:** `booking_step2`
3. **Step 3:** `booking_complete`
4. Set **"Open funnel"** = OFF (closed funnel — user must complete Step 1 before Step 2 counts), since this is a linear, required-sequence flow.
5. Enable **"Show elapsed time"** to see time-to-convert between steps.
6. Breakdown dimension: add `clinic_location` and `specialty` as a secondary breakdown to see which clinics/specialties convert best.
7. Set the **trailing/lookback window** appropriately (e.g., session-scoped) so drop-off reflects same-session behavior.

### How drop-offs are measured

- GA4's Funnel Exploration automatically calculates **step-to-step completion % and abandonment %** between each defined step.
- Example read-out: "1,000 users hit Step 1 → 640 reached Step 2 (64% completion, 36% drop-off) → 410 completed booking (64% completion, 36% drop-off)."
- Drop-off between Step 1→2 typically signals friction in specialty/location selection (UX issue); drop-off between Step 2→3 typically signals form-friction or trust issue (phone number required, etc.) — this segmentation is exactly why each step must be a distinct dataLayer event rather than one combined "form_submit" event.

---

## 3. Real dataLayer.push() Implementations

### Step 1 — Location & Specialty Selected

```javascript
dataLayer.push({
  event: "booking_step1",
  step_number: 1,
  clinic_location: "OrthoNow Andheri West",
  specialty: "Sports Medicine",
  form_name: "appointment_booking"
});
```

### Step 2 — Contact Details & Preferred Date

```javascript
dataLayer.push({
  event: "booking_step2",
  step_number: 2,
  clinic_location: "OrthoNow Andheri West",
  specialty: "Sports Medicine",
  preferred_date: "2026-07-10",
  form_name: "appointment_booking"
  // NOTE: name and phone are intentionally NOT included here.
  // They are sent directly to the backend/CRM via API call,
  // never pushed into the dataLayer, to avoid PII in GA4/Ads.
});
```

### Step 3 — Booking Complete (fires only after backend confirmation)

```javascript
dataLayer.push({
  event: "booking_complete",
  step_number: 3,
  clinic_location: "OrthoNow Andheri West",
  specialty: "Sports Medicine",
  appointment_date: "2026-07-10",
  booking_id: "ORTH-2026-0071542",
  form_name: "appointment_booking",
  value: 1,
  currency: "INR"
});
```

*(`value` and `currency` are included on the completion event so this event can be imported into Google Ads as a valued conversion action.)*

---

## 4. Recommended Google Ads Conversion Action

### Import: **`booking_complete`** (GA4 Key Event → Google Ads Conversion)

**Why `booking_complete` is the correct conversion to import — and why Call Clicks / WhatsApp Clicks are not:**

| Criteria | Booking Complete | Call Click | WhatsApp Click |
|---|---|---|---|
| Confirms real business outcome | ✅ Actual confirmed appointment | ❌ Click only — call may not connect or may be spam/misdial | ❌ Click only — opens chat, doesn't confirm inquiry or booking |
| Immune to accidental/junk clicks | ✅ Requires full 3-step form completion + backend confirmation | ❌ Easily clicked by accident, especially on mobile | ❌ Same risk — floating button is one-tap |
| Usable for Smart Bidding (tCPA/tROAS) | ✅ High-intent signal, low noise → reliable optimization | ⚠️ Noisy signal, will mislead bidding algorithm toward junk clicks | ⚠️ Same — algorithm optimizes for clicks, not patients |
| Attributable to revenue/value | ✅ Can carry `value`/`currency` for ROAS tracking | ❌ No inherent value signal | ❌ No inherent value signal |
| Duplicate-proof | ✅ One booking_id per confirmed booking | ❌ Same user can click call multiple times | ❌ Same user can click multiple times |

**Bottom line:** Call clicks and WhatsApp clicks are *engagement* signals, not *conversion* signals — a user can tap "Call Now" out of curiosity without ever speaking to staff or booking. Importing them as the primary Google Ads conversion would train Smart Bidding to chase clicks, not patients, inflating cost-per-lead and degrading lead quality. `booking_complete` is a backend-verified, funnel-gated, high-intent event — the closest proxy to an actual paying patient that client-side tracking can provide. Call Click and WhatsApp Click should still be tracked (as shown in the schema) and can be added as **secondary/observation conversions** in Google Ads for visibility, but should not be the action optimized against.

---

## 5. Complete Technical Flow (End-to-End)

```
USER ACTION
   (User completes Step 3 and backend confirms booking)
        │
        ▼
FRONTEND JAVASCRIPT
   (Developer's confirmation-handler function executes on API success callback)
        │
        ▼
dataLayer.push({ event: "booking_complete", clinic_location: "...", booking_id: "...", value: 1, currency: "INR" })
        │
        ▼
GTM DATA LAYER LISTENER
   (GTM container snippet, already loaded on page, detects the push in real time)
        │
        ▼
GTM CUSTOM EVENT TRIGGER
   ("event equals booking_complete" trigger fires)
        │
        ▼
GTM TAG FIRES
   (GA4 Event Tag "GA4 - Booking Complete" fires, reading DLVs for clinic_location, booking_id, value, currency)
        │
        ▼
GA4 EVENT RECEIVED
   (Event logged in GA4 in real time → visible in DebugView → aggregated into Reports/Explorations)
        │
        ▼
GA4 KEY EVENT (CONVERSION) MARKED
   (booking_complete flagged as a Key Event in GA4 Admin)
        │
        ▼
GOOGLE ADS CONVERSION IMPORT
   (Google Ads linked to GA4 property imports booking_complete as a Conversion Action)
        │
        ▼
SMART BIDDING OPTIMIZATION
   (Google Ads uses this high-intent signal to optimize tCPA/tROAS bidding toward users likely to book)
```

---

## 6. Critical Implementation Notes (Explicit)

> ⚠️ **GTM does NOT generate booking events automatically.** GTM is a tag/trigger management layer — it has no awareness of form state, backend confirmations, or business logic.

> ⚠️ **Frontend developers must implement every `dataLayer.push()` call** shown in Section 3, at the exact moment specified (button click for Steps 1–2, backend confirmation callback for Step 3). Without this code shipped by engineering, **no events will exist for GTM to listen to** — the tracking plan is inert until implemented.

> ⚠️ **GTM's role is strictly to listen and forward:** it watches the dataLayer for matching event names, and forwards the associated parameters into GA4 (and eventually Google Ads) via tags. GTM cannot detect a "successful booking" on its own — that determination must be made in application code (typically, only after a successful API/backend response) and communicated to GTM exclusively through the dataLayer.

**Recommended pre-launch QA checklist:**
- [ ] Confirm each `dataLayer.push()` fires at the correct step using GTM Preview mode
- [ ] Confirm no PII (name, phone, email) appears in any dataLayer object or GA4 DebugView
- [ ] Confirm `booking_complete` only fires on true backend confirmation, not on button click alone (test a forced API failure to verify it does *not* fire)
- [ ] Verify GA4 DebugView shows all events with correct parameters
- [ ] Mark `booking_complete` (and optionally `guide_download`) as Key Events in GA4
- [ ] Confirm Google Ads ↔ GA4 link is active and `booking_complete` appears as an importable conversion
- [ ] Validate Funnel Exploration step order and drop-off numbers against a manual test booking
