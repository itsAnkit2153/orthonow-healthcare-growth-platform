# Task 03 — Integration Design: HubSpot + WhatsApp (Karix) + Google Ads

## Architecture: Consultation Form → HubSpot → WhatsApp → Google Ads

**Chosen stack: HubSpot CRM API (not Forms API) + Native HubSpot Workflows + Direct REST call to Karix.** Forms API is for anonymous form-capture UX, not lead lifecycle logic — it can't dedupe by phone or branch on clinic preference. CRM API gives full control over search-before-create. I rejected Zapier/Make for the core path: they add 3rd-party latency and opaque retry queues that jeopardize the 2-minute WhatsApp SLA, and they're an unnecessary cost/failure layer when HubSpot's native Workflow engine + a direct API call already do the job reliably. Zapier/Make are fine for secondary, non-time-critical automations (e.g., Slack alerts), not the critical path.

**Flow:**
```
Landing Page
     │
     ▼
Frontend Validation
     │
     ▼
Backend → HubSpot CRM API: Search Contact by phone (custom unique property)
     │
     ├── Match found → Update (clinic_preference, source, lead_status)
     │
     └── No match → Create Contact
              │
              ▼
     HubSpot Workflow (enrollment trigger: lead_status = New Enquiry)
              │
              ▼
     Webhook → Karix WhatsApp API
              │
              ▼
     Patient receives confirmation (<2 min)

Frontend dataLayer → GTM → GA4 Key Event → Google Ads Conversion Import
```

**Biggest failure point:** Karix API timeout/rate-limit during peak ad hours. Impact: broken SLA, patient thinks booking failed, may re-submit or churn. Mitigation: push WhatsApp jobs to a queue (SQS/RabbitMQ) with exponential-backoff retries (3 attempts), dead-letter queue for exhausted retries, structured logging (request ID, contact ID, timestamp) into Datadog/CloudWatch, alerting on queue depth and failure rate, and an SMS fallback (via the same or backup gateway) if WhatsApp fails twice.

**SLA risks:** HubSpot workflow processing delay, Karix throttling, webhook cold-starts. Monitor: end-to-end latency (form-submit → WhatsApp-delivered), Karix delivery-status webhooks, p95/p99 latency, error rate — alert if p95 exceeds 90 seconds.

**Deduplication (critical):** HubSpot dedupes by email by default — unusable here since most Indian healthcare leads submit phone-only. Solution: create a unique custom property `phone_number`, search by it via CRM API before every write, and branch update-vs-create explicitly rather than relying on HubSpot's native matching, which will otherwise silently create duplicate contacts per submission.

---

## The edge case: same phone number, different name

This is the scenario the interviewer note flags directly, and it deserves a precise answer rather than a generic "search-before-create" description.

Shared phones are common in Indian households — a spouse books for a partner, an adult child books for a parent, one number covers a family. **Blindly overwriting the `name` field on a phone match is the wrong move**: it silently loses the previous patient's identity and risks mislabeling medical records under the wrong person — a clinical/compliance issue, not just a CRM hygiene one.

**Correct logic — not "one contact per phone number," but phone-plus-name matching:**

1. **Search by phone** first, using a custom unique property `phone_number` (indexed for fast lookup).
2. **If a match is found, compare the submitted name against the stored name:**
   - **Name matches (same or close variant)** → same patient, returning enquiry → **Update** the existing contact (`clinic_preference`, `source`, `lead_status`, `last_enquiry_date`).
   - **Name differs meaningfully** → different person sharing that phone → **do not overwrite**. **Create a new Contact**, keyed on a composite identifier (`phone_number + name`, or a system-generated `patient_id`), and associate it to the same phone number via a custom lookup/"Household" relationship so the call-center still dials the right number but sees every distinct patient logged against it.
3. **Log ambiguous matches** (same phone, name is a near-variant like "Rahul" vs. "Rahul Sharma") to a review flag (`needs_dedup_review = true`) for manual merge by the front-desk team, rather than an automated guess in either direction.

**Update vs. create, summarized:**
- Phone found + name matches → **Update**.
- Phone found + name doesn't match → **Create** new contact, link to shared phone.
- Phone not found → **Create** new contact.

This preserves accurate per-patient records without generating a duplicate contact on every genuine resubmission — the actual failure mode of relying on HubSpot's default email-based dedup in a phone-first market.
