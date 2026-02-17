# Phase 1 — Online Check-In + SIBA/SEF Submission

This document defines the complete functional and technical flow for Phase 1 of the HostSync platform.

Scope of Phase 1:
- Booking ingestion via channel manager webhook (create/update/cancel)
- Guest online check-in via tokenized URL
- Storage of booking and guest data in HostSync
- Automatic submission to SIBA/SEF (Portugal)
- Export batching every 15 days
- Delivery of exports via cloud storage or encrypted email fallback
- GDPR-aligned data retention and deletion

HostSync acts as **Data Processor**.  
Clients act as **Data Controllers** and are responsible for long-term archiving.

---

## 1. Booking Ingestion (Channel Manager Webhooks)

Booking Webhook Payload

HostSync receives booking updates via webhook from the channel manager. The webhook payload is delivered as an array and may contain one or multiple booking objects. Delivery must be treated as at-least-once, therefore webhook processing must be fully idempotent.

Each webhook event must upsert the booking in HostSync (create if new, update if existing).
If booking.status = "cancelled", HostSync must immediately:

Mark the booking as cancelled internally

Invalidate any existing check-in token

Prevent guest check-in emails

Stop any scheduled SIBA/SEF submission jobs

The complete webhook payload (including invoiceItems) must be stored as raw JSON for operational use and future automation modules (e.g. invoicing). This data is retained only according to Phase 1 retention rules (deleted 30 days after checkout). HostSync does not act as an accounting system of record.

Guest identity fields in the webhook may initially be empty and are completed later via the online check-in flow.

Example Booking Webhook Payload (Real Structure – Simplified)

```json
[
  {
    "timeStamp": "2026-02-15T10:07:22Z",
    "booking": {
      "id": 82279106,
      "masterId": null,
      "bookingGroup": null,
      "propertyId": 125517,
      "roomId": 281392,
      "unitId": 1,
      "roomQty": 1,
      "offerId": 1,
      "status": "confirmed",
      "subStatus": "none",
      "arrival": "2026-02-16",
      "departure": "2026-02-17",
      "numAdult": 2,
      "numChild": 0,
      "title": "",
      "firstName": "",
      "lastName": "",
      "email": "",
      "phone": "",
      "mobile": "",
      "fax": "",
      "company": "",
      "address": "",
      "city": "",
      "state": "",
      "postcode": "",
      "country": "",
      "country2": null,
      "arrivalTime": "",
      "voucher": "",
      "comments": "",
      "notes": "",
      "message": "",
      "groupNote": "",
      "custom1": "",
      "custom2": "",
      "custom3": "",
      "custom4": "",
      "custom5": "",
      "custom6": "",
      "custom7": "",
      "custom8": "",
      "custom9": "",
      "custom10": "",
      "flagColor": "",
      "flagText": "",
      "statusCode": 0,
      "lang": "pt",
      "referer": "iuri_rosa",
      "refererEditable": "iuri_rosa",
      "reference": "",
      "channel": "direct",
      "apiSourceId": 0,
      "apiSource": "Direct",
      "apiReference": "",
      "allowChannelUpdate": "all",
      "allowAutoAction": "enable",
      "allowReview": "default",
      "allowCancellation": {
        "type": "daysBeforeArrival",
        "daysBeforeArrivalValue": 2
      },
      "invoiceeId": null,
      "bookingTime": "2026-02-12T19:49:04Z",
      "modifiedTime": "2026-02-15T10:07:22Z",
      "cancelTime": null,
      "price": 713,
      "deposit": 0,
      "tax": 13,
      "commission": 100,
      "rateDescription": "Display name for the Offer 1 Standard - Portuguese, \r\n2026-02-16 100 2 persons,",
      "stripeToken": null,
      "pcibookingToken": null,
      "apiMessage": ""
    },
    "infoItems": [],
    "invoiceItems": [
      {
        "id": 148343768,
        "bookingId": 82279106,
        "type": "charge",
        "subType": 1,
        "description": "Room",
        "status": "7.00",
        "qty": 1,
        "amount": 500,
        "lineTotal": 500,
        "vatRate": 20,
        "invoiceeId": null,
        "createdBy": 11764,
        "createTime": "2026-02-12T19:49:04Z"
      },
      {
        "id": 148343769,
        "bookingId": 82279106,
        "type": "charge",
        "subType": 3,
        "description": "limp 13.4%",
        "status": "",
        "qty": 1,
        "amount": 13,
        "lineTotal": 13,
        "vatRate": 0,
        "invoiceeId": null,
        "createdBy": 11764,
        "createTime": "2026-02-12T19:49:04Z"
      },
      {
        "id": 148343770,
        "bookingId": 82279106,
        "type": "charge",
        "subType": 15,
        "description": "",
        "status": "",
        "qty": 1,
        "amount": 200,
        "lineTotal": 200,
        "vatRate": 0,
        "invoiceeId": null,
        "createdBy": 11764,
        "createTime": "2026-02-12T19:49:04Z"
      }
    ],
    "messages": [],
    "retries": 0
  }
]
```

### Triggers

The channel manager sends webhooks for:
- Booking created
- Booking updated (any change)
- Booking cancelled

### Required Behavior

Each webhook must upsert the booking in HostSync:

- Create if new
- Update if existing

If booking is cancelled:
- Mark booking as cancelled in HostSync
- Invalidate any existing check-in token
- Prevent check-in emails from being sent
- Prevent SIBA/SEF submission jobs from executing

Webhook payload data is stored as the operational source of truth for bookings.

---

## 2. Check-In Link Generation

When a booking is received:

1. HostSync generates a secure random token (unguessable, high entropy).
2. Token is stored server-side (hashed).
3. Token expiry is validated by booking status and dates.

HostSync then calls the channel manager API to set:

GuestCheckIn_URL=https://hostsync/checkin/{token}


The channel manager uses this URL in its own auto-actions to email the guest.

If a booking becomes cancelled at any time:
- The token must become invalid immediately.

---

## 3. Guest Online Check-In Page (Token Access)

Access is token-based only (no booking ID or name entry).

Initial page must show minimal booking context.

Guest completes legally required fields for Portugal SIBA/SEF:
- Identity document details
- Nationality
- Address
- Dates
- Other required fields

### On Successful Submission

HostSync must:

1. Validate all fields
2. Store guest data in HostSync database
3. Mark check-in as completed internally
4. Call channel manager API:

GuestCheckIn=OK


This indicates the guest successfully completed the form.

If booking is cancelled after submission:
- Data remains until retention rules apply
- Submission logic follows business rules

---

## 4. SIBA/SEF Submission (Scheduled)

SIBA/SEF submission is scheduled:

- 1 day after guest check-in

Submission occurs only if:
- Booking is not cancelled
- Guest data is complete
- Submission not already successful

HostSync maintains internal submission states:

- pending
- retrying
- submitted
- failed
- cancelled

HostSync is always the source of truth.

---

## 5. SIBA/SEF Result Handling

### Success

When SIBA/SEF confirms success:

- Mark submission as `submitted`
- Store submission timestamp and reference ID (if provided)
- Call channel manager API:

SIBA_SEF=OK


---

### Permanent Failure (Guest Data / Validation Errors)

Examples:
- Missing required fields
- Invalid document number
- Invalid nationality
- API validation rejection

Actions:

- Mark submission as `failed`
- Store error code and message (sanitized)
- Call channel manager API:

SIBA_SEF=FAIL


---

### Temporary Failure (Technical / Retryable Errors)

Examples:
- Timeouts
- 5xx responses
- Rate limiting
- Connectivity issues

Actions:

- Mark submission as `retrying`
- Retry with exponential backoff
- Do NOT set `SIBA_SEF=FAIL`

Only set FAIL if retries are exhausted or validation error is returned.

Info codes must be idempotent.

---

## 6. Export Batches (Every 15 Days)

Every 15 days HostSync creates an immutable export batch.

### Inclusion Rules

A batch includes only records where:

- Guest check-in completed
- SIBA/SEF submission status = `submitted`
- Record not included in any previous batch

Pending, failed, or cancelled records are excluded.

---

### Batch Creation

When a batch is created:

- period_start and period_end are fixed
- records are linked to the batch
- CSV is generated
- SHA256 hash, byte size, and record count are calculated

Batch contents are immutable.

Manual “Generate batch now” must be available for onboarding/support.

---

## 7. Export Delivery

### Cloud Storage (Preferred)

If customer connected Google Drive / OneDrive / Dropbox:

- Upload CSV to provider
- Store delivery metadata:
  - provider file ID
  - timestamps
  - SHA256
  - status

---

### Encrypted Email Fallback

If no cloud storage is connected:

- Generate CSV
- Encrypt file (ZIP AES or PGP)
- Email encrypted file to customer
- Keep batch downloadable in UI for 30 days

Email delivery is fallback only.

---

## 8. Retention Rules

### Guest Personal Data

- Deleted 30 days after checkout
- Must not be reconstructable after deletion

### Backups and Binlogs

- Retained max 30–35 days
- Encrypted
- Stored externally

Cluster replication is not backup.

### Export Files

- Available for download max 30 days after batch creation
- Deleted after 30 days

### Long-Term Logs (No PII)

Retained up to 10 years:
- export_batch_id
- timestamps
- success/failure
- provider confirmation IDs
- SHA256
- byte size
- schema version

No personal data is stored in logs.

---

## 9. Customer Admin UI (Phase 1)

Authenticated customers must have access to:

- Booking list and status
- Guest check-in status
- SIBA/SEF submission status + last error
- Export batches:
  - download (30-day window)
  - resend
  - connect cloud storage
  - configure export email

---

## 10. Guest UI

Token-based page must support:

- Check-in form
- Confirmation screen
- Error states:
  - booking cancelled
  - link expired
  - already submitted

---

## 11. Phase Roadmap

Phase 1:
- Online Check-In + SIBA/SEF Submission

Future phases (not part of this document):

2. Portuguese invoicing integrations (KeyInvoice, Moloni, Fact)
3. HostSync customer billing system (usage-based)
4. INE reporting and automated submissions  
5. Optional booking portal

---

End of Phase 1 specification.

