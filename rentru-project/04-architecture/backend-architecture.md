# Backend Architecture Guidelines

## Purpose

This document defines the backend design guidelines for the real-estate and daily-rentals ERP. The system targets small and medium businesses (`SMB`) operating in a practical environment with limited training, unstable internet, and a small engineering team.

The main priorities are:

- Simplicity.
- Reliability.
- Fast delivery.
- Low operating cost.
- Clear auditability.
- Gradual scalability.

The backend must support real operations before advanced engineering patterns. The critical business areas are reservations, availability, payments, occupancy, maintenance, treasury, and owner settlements.

## 1. Start With a Modular Monolith

Use a modular monolith as the default architecture. Do not introduce microservices early.

A modular monolith is preferable because it is:

- Faster to develop.
- Easier to debug.
- Cheaper to host.
- Easier to deploy and back up.
- Easier for a small team to understand.

Split services only when operational evidence justifies it, such as an independent scaling requirement, a distinct deployment lifecycle, or a clearly isolated workload.

## 2. Organize Code by Business Domain

Structure the backend around business capabilities, not generic technical folders.

Recommended modules:

```text
src/
  reservations/
  properties/
  pricing/
  contracts/
  finance/
  owners/
  maintenance/
  crm/
  iam/
  notifications/
  reporting/
  shared/
```

Each module owns its application services, domain rules, persistence layer, and API handlers. Shared code should remain small and intentional to avoid hidden coupling.

## 3. Treat the Database as the System Core

Database design is more important than framework choice. Model the real business workflow before implementation.

Prioritize:

- Correct relationships.
- Explicit ownership.
- Constraints that protect business rules.
- Indexes for operational queries.
- Traceability between records.
- Schema migrations with review and rollback planning.

Avoid storing critical derived values as the only source of truth. For example, do not store only:

```text
balance = 5000
```

Store the movements:

```text
transactions
ledger_entries
payment_records
```

Then calculate or cache balances from those records.

## 4. Preserve Financial Integrity

Financial records must be auditable and immutable whenever possible.

Rules:

- Do not delete posted financial transactions.
- Do not silently edit historical transactions.
- Correct mistakes using reversal entries and new corrective entries.
- Link every payment to its source, actor, branch, treasury, and related business record.
- Keep owner settlements separate from customer payments while preserving traceability.

Start with the financial scope needed for operations:

- Treasury.
- Income and expenses.
- Payments.
- Basic ledger entries.
- Owner settlements.

Expand toward more advanced accounting only when the business requires it.

## 5. Enforce Audit Trails

Important records must answer: who changed what, when, and why?

Store fields such as:

```text
created_at
created_by
updated_at
updated_by
deleted_at
deleted_by
```

Use activity logs for important actions:

- Booking confirmation or cancellation.
- Manual price overrides.
- Check-in and check-out.
- Payment creation, reversal, or reconciliation.
- Treasury handover.
- Owner settlement approval.
- Permission changes.

## 6. Prefer Soft Delete

Use soft deletion for operational records unless legal or security requirements demand physical removal.

Example fields:

```text
is_deleted
deleted_at
deleted_by
```

Financial entries should not be deleted. Use reversal flows instead.

## 7. Protect Business Invariants in the Backend

Frontend validation improves usability, but backend validation is authoritative.

The backend must prevent:

- Double booking.
- Overlapping active reservations.
- Invalid state transitions.
- Duplicate payment processing.
- Negative balances where not explicitly allowed.
- Check-in without the required payment or approved exception.
- Overlapping owner contracts when prohibited.
- Unauthorized access across branches or tenants.

Use database constraints and transactions where possible, not only application-level checks.

## 8. Use Database Transactions for Multi-Step Operations

A single business action may affect multiple records:

- Reservation.
- Contract.
- Payment.
- Treasury balance.
- Ledger entries.
- Unit status.
- Audit log.

Changes that must remain consistent should succeed or fail together inside a database transaction.

Keep external side effects, such as WhatsApp messages and PDF generation, outside the database transaction. Publish a background job or an outbox record after the transaction commits.

## 9. Require Idempotency for Retry-Prone Actions

Internet interruptions and repeated button taps are normal operating conditions.

Use idempotency keys for:

- Payment requests.
- Payment webhooks.
- Booking confirmation.
- Check-in and check-out.
- Contract generation.
- Treasury handovers.

Repeated requests with the same key must return the existing result instead of creating duplicate records.

## 10. Model Reservation Lifecycles With State Machines

Reservations are not free-form status text. Define valid states and transitions explicitly.

Example:

```text
Inquiry
  -> Pending
  -> Confirmed
  -> CheckedIn
  -> CheckedOut
  -> Completed
```

Additional terminal or exceptional states may include:

```text
Expired
Cancelled
NoShow
```

Examples of invalid transitions:

- `CheckedOut` before `CheckedIn`.
- `Confirmed` without a unit.
- `CheckedIn` without payment or an approved exception.

## 11. Treat the Calendar Engine as High-Risk Core

Availability calculation is one of the hardest and most important parts of the system.

Design and test carefully:

- Date overlap detection.
- Temporary holds and expiration.
- Reservation states that block availability.
- Maintenance blocks.
- Seasonal pricing.
- Orphan-day detection.
- Multi-branch concurrency.
- Calendar query performance.

The booking calendar must remain fast because operational trust depends on it.

## 12. Support Messy Real-World Data

Do not assume ideal data entry. The backend must tolerate incomplete information while protecting critical rules.

Expect:

- Missing optional customer details.
- Duplicate names.
- Partially completed drafts.
- Incorrectly formatted phone numbers.
- Late expense records.

Normalize data where practical, identify required fields by workflow stage, and allow drafts when the operation has not yet reached a legally or financially binding state.

## 13. Apply RBAC and Data Scoping

Permissions are required even for small businesses.

Use role-based access control (`RBAC`) with branch, tenant, and ownership scoping.

Examples:

- Booking staff should not see profit data.
- Maintenance workers should not see customer identity documents.
- Owners should see only their own units and settlements.
- Branch staff should access only permitted branches.

Enforce authorization in the backend. Hiding buttons in the frontend is not a security boundary.

## 14. Plan for SaaS Isolation Without Overengineering

If the product will serve multiple businesses, include `tenant_id` in the data model early and enforce tenant isolation consistently.

Consider:

- Tenant-scoped queries.
- Tenant ownership checks.
- Branch-level access.
- Storage path isolation.
- Cache key isolation.
- Background job isolation.

Avoid complex infrastructure until usage justifies it.

## 15. Design for Unstable Internet

The frontend and backend should cooperate to handle unreliable connections.

Support:

- Retry-safe APIs.
- Idempotency.
- Draft auto-save.
- Background synchronization.
- Queued operations where appropriate.
- Clear conflict responses.

Do not hide conflicts silently. Return actionable errors when a booking or payment changed while the user was offline.

## 16. Use Background Jobs for Slow Side Effects

Do not keep API requests waiting for work that can run asynchronously.

Use background jobs for:

- WhatsApp messages.
- Reminders.
- Contract PDF generation.
- Invoice delivery.
- Payment reconciliation.
- Reports.
- Aggregation refreshes.
- Expired hold cleanup.

Workers must be retry-safe and observable.

## 17. Make WhatsApp Part of the Workflow

WhatsApp is an operational channel, not an optional marketing integration.

The backend should support:

- Message templates.
- Booking details.
- Payment links.
- Invoices.
- Contract PDFs.
- Payment reminders.
- Check-in and check-out reminders.
- Delivery and failure tracking.

External provider failures must not roll back successful core transactions. Queue and retry messages separately.

## 18. Keep Reporting Away From Critical OLTP Paths

Heavy reporting queries must not slow down reservation and payment operations.

Start with:

- Cached summaries.
- Background aggregation.
- Materialized views or summary tables.
- Indexed operational reports.

Add read replicas only when actual load requires them.

## 19. Return Business-Aware Errors

Errors should help the user resolve the issue.

Prefer:

```text
This unit is already booked for the selected dates.
```

Instead of:

```text
500 Internal Server Error
```

Log technical details internally while returning safe, clear messages to the user.

## 20. Make Backups and Restore Testing Mandatory

Data loss can stop the business.

Require:

- Automated database backups.
- Storage backups for contracts and identity documents.
- Restore testing.
- Export capabilities.
- Backup retention rules.
- Audit history preservation.

## 21. Add Observability From the Start

Even a small system needs enough visibility to diagnose operational failures.

Include:

- Structured logging.
- Error tracking.
- Health checks.
- Basic metrics.
- Background job monitoring.
- Activity logs.
- Alerts for critical failures.

## High-Risk Core

The following areas deserve the strongest review and test coverage:

- Reservation overlap engine.
- Availability calculation.
- Financial ledger.
- Owner settlements.
- Seasonal pricing.
- Calendar performance.
- Check-in and check-out flows.
- Multi-branch operations.
- Payment webhook idempotency.

## Recommended Initial Stack

| Area | Technology |
| --- | --- |
| Frontend | `Next.js` |
| Backend | `NestJS` modular monolith |
| Database | `PostgreSQL` |
| Cache | `Redis` |
| Queue | `BullMQ` |
| Object storage | S3-compatible storage |
| Authentication | JWT, refresh tokens, and RBAC |
| Infrastructure | Docker on a VPS initially |

## Design Review Checklist

Before approving a backend feature, verify:

- Is the workflow based on a real operational need?
- Does the module belong to a clear business domain?
- Are critical database constraints defined?
- Is the operation transaction-safe?
- Is the endpoint idempotent where retries are likely?
- Is the action auditable?
- Is authorization enforced in the backend?
- Can slow side effects run in a background job?
- Does the error message help the user recover?
- Is the operational query indexed and fast?
- Can the data be backed up and restored?
- Is the implementation simple enough for a small team to maintain?

## Guiding Principle

Build a system that survives messy reality. A successful `SMB ERP` is reliable, understandable, fast to evolve, and inexpensive to operate. Scalable simplicity is more valuable than premature architectural complexity.
