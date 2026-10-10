# Salon Booking Agent

A simple n8n automation for South African salons that keeps online booking **and** protects against no-shows, without making customers feel their money "vanished."

Built on self-hosted n8n (v2.35.5, Railway) with Paystack, Twilio, and Google Calendar.

---

## The problem it solves

Salons were abandoning online booking and falling back to "first come, first served" because of three linked problems:

1. **Cancellations not showing up.** Bookings lived in several places (phone, walk-in, online, notebook), so a cancellation in one place never reached the others.
2. **No-shows cost real money.** Salons took upfront deposits, and customers who missed an appointment saw their money simply disappear, which made them angry and killed trust in online booking.
3. **Walk-ins colliding with online bookings.**

## The idea in one sentence

**One shared calendar (a single Data Table) that every booking, cancellation, and walk-in reads from and writes to, with deposits that are only *held* (never charged) unless the customer genuinely doesn't show up.**

---

## How it works

### Booking lifecycle

```
awaiting_hold  ->  hold_requested  ->  confirmed  ->  (cancelled | no_show_credit_issued)
```

| Status | Meaning |
|---|---|
| `awaiting_hold` | Customer booked online. No hold requested yet (appointment is more than 3 days away). |
| `hold_requested` | A Paystack hold link was texted to the customer. Waiting for them to authorize it. |
| `confirmed` | Hold authorized (online) or logged by staff (walk-in). Slot is taken. |
| `cancelled` | Customer replied CANCEL. Hold released, slot freed. |
| `no_show_credit_issued` | Appointment passed with no cancel. Hold captured; customer told it is now salon credit. |

### The six branches

| # | Branch | Trigger | What it does |
|---|---|---|---|
| 1 | **Online Booking Request** | Public form | Saves the booking as `awaiting_hold` and texts a confirmation. No payment yet. |
| 2 | **Create Hold Sweep** | Every hour | Finds `awaiting_hold` bookings within 3 days, creates a Paystack preauthorization, saves the reference, and texts the customer the authorization link. |
| 3 | **Confirm Hold via Paystack** | Paystack webhook | On `preauthorization.reserve.success`, marks the booking `confirmed` and creates a Google Calendar event. |
| 4 | **Reminder Sweep** | Every 15 min | Finds `confirmed`, un-reminded bookings starting within 4 hours and sends an SMS and a WhatsApp reminder. |
| 5 | **Cancel via SMS Reply** | Twilio inbound SMS | If the reply contains CANCEL, finds the customer's most recent open booking, releases the hold (if one exists), removes the calendar event, marks it `cancelled`, and confirms by SMS. |
| 6 | **No-Show Credit Sweep** | Every 15 min | Finds `confirmed` bookings more than 20 min past their slot, captures the Paystack hold, marks `no_show_credit_issued`, and texts the customer. |
| + | **Walk-In Entry** | Staff-only form | Logs a walk-in into the same table (only if the slot has no confirmed booking) and mirrors it to Google Calendar. |

### Why the hold is requested 3 days out

Paystack preauthorization holds last **5 days by default (max 10)**. Requesting the hold at booking time would let it expire before appointments made weeks ahead. So bookings simply wait, and the hourly sweep requests the hold once the appointment is within 3 days.

### Source of truth

The `salon_bookings` Data Table is the single source of truth. Google Calendar is a **visual mirror only**; nothing reads from it.

---

## Data model: `salon_bookings`

| Column | Type | Notes |
|---|---|---|
| `customerName` | string | |
| `phone` | string | International format, e.g. `+27821234567` |
| `customerEmail` | string | Required by Paystack |
| `service` | string | Haircut, Color, Blowout, Treatment |
| `slotDateTime` | string | **Must be `YYYY-MM-DD HH:mm`**, interpreted in South Africa time |
| `status` | string | See lifecycle above. Always lowercase. |
| `source` | string | `online` or `walkin` |
| `reminded` | boolean | `true` once the reminder has been sent. Walk-ins are created as `true`. |
| `paystackReference` | string | Set when the hold is requested |
| `holdAuthUrl` | string | The authorization link sent to the customer |
| `calendarEventId` | string | Google Calendar event ID, used for deletion on cancel |
| `paymentIntentId`, `checkoutUrl` | string | Unused leftovers from the original Stripe design. Safe to ignore or delete. |

---

## Integrations

| Service | Used for | Credential |
|---|---|---|
| **Twilio** | SMS (and WhatsApp) messages | Predefined `Twilio account` credential |
| **Paystack** | Preauthorization: initialize, capture, release | **No credential.** Authorization header set directly on the node (see below) |
| **Google Calendar** | Visual calendar mirror | OAuth2 `Google Calendar account` credential |

### Why Paystack, not Stripe
Stripe is not directly available to South African-registered businesses. Paystack is, and its **Card Preauthorization API** is built for exactly this "hold now, capture only if needed" pattern.

### Paystack authentication (important)
On this n8n instance, generic header-auth credentials (`httpHeaderAuth`, `httpTemplatedCustomAuth`) fail n8n's internal permission check when attached via the API, even when correct. The reliable workaround:

- The three Paystack HTTP Request nodes use **Authentication: None** with **Send Headers** on.
- A header `Authorization: Bearer sk_...` is typed directly into each node.
- **All three nodes must use the same key.** If you rotate the key, update:
  - `Initialize Paystack Preauthorization`
  - `Release Paystack Hold`
  - `Capture No-Show Hold`

Trade-off: the key lives in the workflow JSON rather than n8n's encrypted credential vault. Do not share exported copies of this workflow.

OAuth credentials (Google Calendar) and Twilio did **not** show this problem.

---

## Setup

1. **n8n**: import or open the workflow; confirm workflow timezone is `Africa/Johannesburg` (Workflow settings).
2. **Data Table**: `salon_bookings` with the columns above.
3. **Twilio**: connect credential; account must be **upgraded from trial** (trial accounts can only send predefined templates). Set your Twilio number as the `from` value on every Twilio node.
4. **WhatsApp (optional)**: register your Twilio number as a WhatsApp sender (Meta business verification). Until then the WhatsApp reminder node will fail while SMS keeps working.
5. **Paystack**:
   - Complete Compliance/FICA verification.
   - Ask Paystack Support to enable **Card Preauthorization** (new merchants see "Merchant is not eligible for Preauthorization" until approved).
   - Paste the same `Bearer sk_...` header into the three HTTP nodes.
6. **Webhooks** (workflow must be published/active for production URLs):
   - Twilio inbound SMS: `https://<your-n8n-host>/webhook/twilio-sms-inbound` (set as the incoming-message webhook on your Twilio number)
   - Paystack events: `https://<your-n8n-host>/webhook/paystack-events` (set as the webhook URL in Paystack Settings, API Keys & Webhooks)
7. **Google Calendar**: in Google Cloud, enable the Calendar API, configure the OAuth consent screen (add yourself as a test user), create a **Web application** OAuth client using the redirect URI shown in n8n, then paste the Client ID and Secret into the n8n credential.
8. **Forms**: share the *Online Booking Request* form URL with customers; keep the *Walk-In Entry* form URL internal.

---

## Configuration quick reference

| What | Where | Current value |
|---|---|---|
| Deposit amount | `Initialize Paystack Preauthorization` and `Capture No-Show Hold` (`amount`, in cents) | `10000` = R100 |
| Currency | same nodes | `ZAR` |
| Hold request window | `Due For Hold` filter | within 3 days |
| Hold expiry backstop | `Initialize Paystack Preauthorization` (`expire_after_days`, `expire_action`) | 5 days, release |
| Reminder window | `Due For Reminder` filter | within 4 hours |
| No-show grace period | `Overdue By Grace Period` filter | 20 minutes |
| Appointment length on calendar | `Add to Calendar` / `Add Walk-In to Calendar` | 1 hour |
| Services list | Both forms (`service` dropdown) | Haircut, Color, Blowout, Treatment |

If you change the deposit, change it in **both** the initialize and capture nodes.

---

## Known limitations

- **Date format is free text.** `slotDateTime` is typed into a form field. A wrong format (e.g. `2/9/2026 3pm`) will not parse and the booking will be silently skipped by the sweeps. A date/time picker or a stricter form would be a worthwhile upgrade.
- **"Salon credit" is a policy, not a ledger.** The no-show step genuinely captures the hold via Paystack and tells the customer it is credit. The system does not track credit balances; honoring the credit at the next visit is manual.
- **Last-minute bookings.** A booking made inside the 3-day window gets its hold request on the next hourly sweep (up to about an hour later). One made less than an hour before the slot may never receive a hold.
- **Customers who never authorize the hold** stay in `hold_requested`. They are not reminded and not treated as no-shows, because both sweeps only act on `confirmed` bookings.
- **Walk-in duplicate protection is silent.** If a walk-in is entered for a slot that already has a confirmed booking, nothing is saved and no message is shown to staff. Staff should still check the table first.
- **Calendar is one-way.** Editing or deleting an event in Google Calendar does not change the Data Table.
- **Paystack key is stored in the node** (see above).

---

## Testing checklist

Everything except the actual hold can be tested before Paystack approves Preauthorization.

1. Submit the online booking form with your own number and email; confirm the booking text arrives.
2. In the Data Table, set the row to `confirmed` (lowercase) with a `slotDateTime` within the next 4 hours and `reminded` = false. Run the Reminder Sweep and check for SMS/WhatsApp.
3. Reply `CANCEL` and confirm: status becomes `cancelled`, you get a confirmation text, and (if present) the calendar event is removed.
4. Add a walk-in via the staff form; confirm the row and the calendar event appear.
5. Once Preauthorization is approved: run the hold sweep on an `awaiting_hold` booking within 3 days, authorize the link with a Paystack test card, and confirm the webhook flips the booking to `confirmed` and creates the calendar event.
6. Test the no-show path with a `confirmed` row whose slot is more than 20 minutes in the past.

## Go-live checklist

- [ ] Twilio upgraded from trial
- [ ] Paystack Preauthorization approved; **live** secret key pasted into all three HTTP nodes (switch from `sk_test_` to `sk_live_`)
- [ ] Paystack **live** webhook URL set
- [ ] Twilio inbound webhook pointed at the production URL
- [ ] WhatsApp sender registered (or remove the WhatsApp reminder node)
- [ ] Google Calendar credential connected and OAuth consent screen set up for the right account
- [ ] Test data deleted from `salon_bookings`
- [ ] Workflow **published** (schedule triggers and production webhooks only run when active)
- [ ] Customers told the deposit policy in plain language before they pay (see the one-page explainer)

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| "Invalid template name. Trial accounts can only use predefined SMS templates" | Twilio account is still on trial. Upgrade it. |
| "Merchant is not eligible for Preauthorization" | Paystack has not enabled Preauthorization on your account yet. Contact Paystack Support. |
| Paystack 401 or auth errors on only one of the three nodes | The Bearer keys are out of sync across the three HTTP nodes. |
| "uses invalid credential" after you deleted and recreated a credential | A recreated credential gets a new ID and nodes do not follow it automatically. Re-select it in each node (or re-link it). |
| Reminder never sends | Status is not exactly `confirmed` (lowercase), `reminded` is already true, or `slotDateTime` is outside the 4-hour window or in the wrong format. |
| Timings are off by several hours | Workflow timezone is not `Africa/Johannesburg`. |
| Customer texts CANCEL and is told no booking exists | Phone number in the table is not in the same international format as the number they text from. |
| Google Calendar node errors | Credential not linked on that node, or OAuth client redirect URI does not match n8n's. |

---

## Project reference

- n8n workflow ID: `NZRTOv6XHoHXo1j2`
- Data Table: `salon_bookings` (ID `ttaJt4MevMeNZn7z`)
- n8n project: Personal (`CKCVbYVF8eDSXz19`)
