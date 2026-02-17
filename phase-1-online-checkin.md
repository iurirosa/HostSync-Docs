Phase 1 — Online Check-In + SIBA/SEF Submission

This document defines the complete functional and technical scope for Phase 1 of the HostSync platform.

Phase 1 delivers a legally compliant online guest check-in flow with automated SIBA/SEF submission for Portuguese accommodation providers.

HostSync acts as Data Processor.
Clients act as Data Controllers and are responsible for long-term legal archiving.

1. Scope of Phase 1

Phase 1 includes:

Booking ingestion via channel manager webhook (create/update/cancel)

Guest online check-in via secure tokenized URL

Secure storage of booking and guest data

Automatic SIBA/SEF submission (Portugal)

Export batching every 15 days

Delivery of exports via cloud storage or encrypted email fallback

GDPR-aligned data retention and deletion

Customer admin UI (operational visibility)

Guest-facing check-in UI

2. Out of Scope (Non-Goals)

The following are explicitly not included in Phase 1:

Payment processing

Guest messaging

Portuguese invoicing integrations

HostSync billing system

INE reporting

Booking portal

Analytics dashboards

These belong to future phases.

3. Tenant & Authentication Model

HostSync is a multi-tenant platform.

Each customer is a tenant.

All data objects are strictly isolated by tenant_id.

Each tenant has:

tenant_id

Channel manager API credentials

SIBA/SEF credentials

Rules

Every webhook request must identify the tenant.

No cross-tenant data access is permitted.

Admin UI access is authenticated and tenant-scoped.

Tenant isolation must be enforced at database level.

4. Core Entities (Minimum)

Phase 1 must implement at least the following entities:

tenants

bookings

guests

checkin_tokens

sef_submissions

export_batches

export_batch_records

audit_logs (no PII)

5. Booking Ingestion (Channel Manager Webhooks)

HostSync receives booking updates via webhook from the channel manager.

Webhook Characteristics

Payload is an array

May contain one or multiple bookings

Delivery is at-least-once

Processing must be fully idempotent

Required Behavior

Each webhook must:

Create booking if new

Update booking if existing (upsert logic)

If booking.status = cancelled:

Mark booking as cancelled internally

Invalidate existing check-in token

Prevent guest check-in emails

Cancel any scheduled SIBA/SEF jobs

The full webhook payload (including invoiceItems) must be stored as raw JSON for operational and future automation use.

HostSync is not an accounting system of record.

Guest identity fields may initially be empty and are completed later via check-in.

Example Booking Webhook Payload (Simplified Real Structure)
[
  {
    "timeStamp": "2026-02-15T10:07:22Z",
    "booking": {
      "id": 82279106,
      "propertyId": 125517,
      "roomId": 281392,
      "status": "confirmed",
      "arrival": "2026-02-16",
      "departure": "2026-02-17",
      "numAdult": 2,
      "numChild": 0,
      "firstName": "",
      "lastName": "",
      "email": "",
      "phone": "",
      "country": "",
      "bookingTime": "2026-02-12T19:49:04Z",
      "modifiedTime": "2026-02-15T10:07:22Z",
      "price": 713,
      "deposit": 0,
      "tax": 13,
      "commission": 100
    },
    "invoiceItems": [
      {
        "id": 148343768,
        "bookingId": 82279106,
        "type": "charge",
        "description": "Room",
        "qty": 1,
        "amount": 500,
        "vatRate": 20
      }
    ],
    "messages": [],
    "retries": 0
  }
]

Payload Handling Rules

Payload may contain multiple bookings.

Unknown fields must be tolerated.

Duplicate deliveries must not create duplicate records.

invoiceItems must be stored even if unused in Phase 1.

6. Check-In Link Generation

When booking is received:

Generate secure random token (high entropy, unguessable)

Store token hashed server-side

Token validity tied to booking status and dates

HostSync must call channel manager API:

GuestCheckIn_URL=https://hostsync/checkin/{token}


If booking becomes cancelled:

Token must become invalid immediately

7. Guest Online Check-In

Access is token-based only.

No booking ID or name lookup.

Guest must complete legally required SIBA/SEF fields:

Identity document details

Nationality

Address

Arrival/departure confirmation

Other required legal fields

On Successful Submission

HostSync must:

Validate fields

Store guest data

Mark check-in completed

Call channel manager API:

GuestCheckIn=OK

8. SIBA/SEF Submission

Submission is scheduled:

1 day after check-in completion

Submission only if:

Booking not cancelled

Guest data complete

Not already successfully submitted

Submission States

pending

retrying

submitted

failed

cancelled

HostSync is always the source of truth.

9. Result Handling
Success

Mark as submitted

Store timestamp and reference ID

Call:

SIBA_SEF=OK

Permanent Failure (Validation)

Examples:

Missing required fields

Invalid document number

API validation rejection

Actions:

Mark as failed

Store sanitized error message

Call:

SIBA_SEF=FAIL

Temporary Failure (Retryable)

Examples:

Timeouts

5xx responses

Rate limits

Actions:

Mark as retrying

Retry with exponential backoff

Do NOT set FAIL unless retries exhausted

10. Background Jobs & Queues

All external integrations must be asynchronous.

Queue workers handle:

SIBA/SEF submissions

Export generation

Email delivery

Cloud uploads

Rules:

No blocking HTTP requests

Exponential backoff

Default max attempts: 5 (configurable)

11. Export Batches (Every 15 Days)

Every 15 days HostSync creates immutable export batches.

Inclusion Rules

Include only records where:

Check-in completed

Submission status = submitted

Not included in previous batch

Batch Creation

On creation:

Fix period_start and period_end

Link records to batch

Generate CSV

Calculate SHA256

Calculate byte size

Record count

Manual “Generate Batch Now” must be available.

12. Export Delivery
Cloud Storage (Preferred)

If connected:

Upload CSV

Store:

Provider file ID

Timestamp

SHA256

Status

Encrypted Email Fallback

If no cloud provider:

Encrypt CSV (ZIP AES or PGP)

Email to customer

Allow UI download for 30 days

13. Retention Rules
Guest Personal Data

Deleted 30 days after checkout

Not reconstructable

Backups & Binlogs

Max 30–35 days retention

Encrypted

Stored externally

Replication ≠ backup

Export Files

Deleted 30 days after batch creation

Long-Term Logs (No PII)

Retained up to 10 years:

export_batch_id

timestamps

success/failure

provider IDs

SHA256

byte size

schema version

No personal data stored in logs.

14. Customer Admin UI

Authenticated tenants must access:

Booking list + status

Guest check-in status

SIBA/SEF status + last error

Export batches:

Download (30 days)

Resend

Connect cloud storage

Configure export email

15. Guest UI

Token-based page must support:

Check-in form

Confirmation screen

Error states:

Booking cancelled

Link expired

Already submitted

16. Internal API Endpoints (Minimum)
POST /webhooks/channel-manager

GET  /checkin/{token}
POST /checkin/{token}/submit

POST /sef/submit

POST /exports/generate

GET /admin/bookings
GET /admin/exports

17. Security Requirements

HTTPS enforced

Tokens hashed at rest

Signed webhooks

Rate limiting

Audit logs

Strict tenant isolation

End of Phase 1 Specification
