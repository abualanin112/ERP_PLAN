# Business Rules

## Purpose

This file defines business rules independently from UI and code implementation.

## General Rules

- Every critical action should have an actor, timestamp, and reason when applicable.
- Business statuses should be explicit and easy to audit.
- Deleting critical records should be avoided; prefer archive, cancel, close, or deactivate.
- Sensitive documents should require controlled access.
- OCR output is not automatically trusted for sensitive or financial fields.

## Customer Rules

- A customer may have multiple contact methods.
- A customer may have multiple leads and reservations.
- Duplicate customers should be detected by phone/email/national ID where legally appropriate.
- Customer notes should preserve author and timestamp.

## Lead Rules

- Every lead should have a source.
- Every active lead should have an owner or assignment queue.
- Lost leads should require a lost reason.
- A lead can become a reservation.
- A lead should not be marked closed without a final outcome.

## Reservation Rules

- A reservation must have a customer, property/unit, date range, and status.
- Confirmed reservations should not overlap on the same unit.
- A reservation cannot be confirmed without a deposit.
- Minimum deposit should be at least the price of one day unless an authorized override exists.
- Every confirmed reservation must have a contract record.
- The price agreed in the reservation/contract must be snapshotted.
- Remaining balance should be visible and due at handover/check-in unless another policy is configured.
- Draft reservations do not block availability unless explicitly configured.
- Cancelled reservations should preserve cancellation reason and timestamp.
- Check-in cannot happen before reservation confirmation unless an authorized override exists.
- Check-out cannot happen before check-in.
- Closed reservations should be read-only except for controlled administrative corrections.

## Availability Rules

- Availability should be verified from source-of-truth data before confirming a reservation.
- Future cache, if added, must not be used as the final source of truth for booking creation.
- Calendar/iCal data should be refreshed before high-risk actions such as checkout/payment confirmation.

## Document Rules

- Documents must be linked to an owner entity such as customer, reservation, property, or payment.
- Sensitive documents should be private by default.
- Store files in R2 and metadata in PostgreSQL.
- Document verification status should be explicit: pending, verified, rejected, expired.
- Rejected documents should have a reason.

## OCR Rules

- OCR jobs should store raw text and parsed fields.
- Parsed OCR fields should keep confidence/source metadata when available.
- Human verification is required for national ID, payment, and meter-reading values if they affect operations or finance.
- Failed OCR should not block the workflow permanently; it should create a manual review task.

## Payment Rules

- MVP payment records can be manual, while Paymob integration is planned.
- Every payment should be linked to reservation/customer when applicable.
- Payment status should be explicit: pending, partial, paid, failed, refunded, cancelled.
- Deposit and remaining balance must be tracked separately for reservations.
- Supported future payment channels through Paymob/account setup include Vodafone Cash, InstaPay, Fawry, and bank account.
- Payment changes should be auditable.
- Gateway webhooks later must be idempotent and verified.

## Property Rules

- A building/property may contain one or more units.
- A unit can have independent availability and pricing.
- Rental is done at the unit level.
- Each unit must have an owner: external owner or company-owned.
- Each building should have image, location, and general specifications.
- Each unit should have specifications, price, availability, and optional furniture/assets.
- Property status should be explicit: active, inactive, maintenance, unavailable.
- Owner data should be protected based on user role.

## Pricing Rules

- Pricing is dynamic and may change daily.
- Price changes should be stored as history.
- Price movement should indicate increase, decrease, or no change.
- Different units in the same building can have different prices.
- Historical reservations/contracts must keep their original agreed price.

## Contract Rules

- Every confirmed reservation must generate or link to a contract.
- Contract data should include reservation, customer, unit, date range, price, deposit, remaining balance, and accepted terms.
- Contract records should not be silently overwritten.
- Contract changes should create version/history when legally relevant.

## Daily Accounting Rules

- A daily sheet should list contracts/reservations, units, dates, income, expenses, and collected amounts.
- Accountant should be able to review or close the daily sheet.
- Commissions may exist but should not always be registered as expenses on every contract.
- Commission handling must be explicit per contract or per business policy.

## Utility Rules

- Water and electricity records should support image capture/upload.
- OCR output must be verified before affecting financial/settlement records.
- Utility records should be linked to unit and reservation when applicable.

## Maintenance And Cleaning Rules

- Cleaning is usually internal and assigned to company workers.
- Maintenance can be external and may include vendor/cost data.
- Tasks should be linked to unit, reservation, worker/vendor, date, status, and cost when applicable.

## Asset/Furniture Rules

- Company-owned furniture/assets should be tracked.
- Assets should be assignable to units/apartments/huts.
- Asset movement should preserve history when practical.

## Calendar Rules

- Calendar should support reservations/rentals, worker tasks, cleaning, maintenance, due payments, expenses, and annual rent reminders.

## Reporting Rules

- KPIs must define their calculation period.
- Property comparisons must use the same date range.
- Comparisons should normalize values where property sizes differ.
- Reports should show when data was last refreshed.
- Read cache is future-only and should be added only after monitoring proves a need.

## Configuration vs Customization Rules

- Prefer configuration for common business differences.
- Use customization only for client-specific workflows that cannot be represented safely as settings.
- If the same custom request appears repeatedly, promote it into configurable product behavior.
