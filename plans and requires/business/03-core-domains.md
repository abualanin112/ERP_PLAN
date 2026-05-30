# Core Business Domains

## Purpose

This file defines the major business domains and what each domain is responsible for.

## 1. Customer Domain

Purpose:

- Store customer identities, contact data, preferences, tags, and relationship history.

Main records:

- Customer.
- Customer contact methods.
- Customer notes.
- Customer tags.
- Customer documents.

Key business questions:

- Is this a new or returning customer?
- What properties did this customer ask about?
- What reservations or visits are linked to this customer?
- Are documents complete?

## 2. Leads Domain

Purpose:

- Track prospects before they become confirmed reservations or customers.

Main records:

- Lead.
- Lead source.
- Lead status.
- Lead owner / assigned agent.
- Follow-up activity.

Common statuses:

- New.
- Contacted.
- Interested.
- Negotiation.
- Booked.
- Lost.
- Closed.

## 3. Sales Pipeline Domain

Purpose:

- Convert leads into reservations or closed deals through visible stages.

Main records:

- Pipeline stage.
- Lead movement history.
- Follow-up tasks.
- Sales notes.

Key business questions:

- Which leads are stuck?
- Which agent owns the next action?
- What is the next follow-up date?
- Why was a lead lost?

## 4. Property & Unit Domain

Purpose:

- Manage buildings, units, availability, ownership, pricing references, furniture/assets, and operational details.

Main records:

- Building / property.
- Unit.
- Owner.
- Amenities.
- Availability periods.
- Dynamic pricing records.
- Unit price history.
- Furniture/assets assigned to units.
- Property documents.
- Property media.

Key business questions:

- Is the unit available?
- Who owns the property?
- Is the unit owned by an external owner or the company?
- What is the current price and how did it move recently?
- What furniture/assets are assigned to the unit?
- What documents are attached?
- What is the current operational status?

## 5. Reservations Domain

Purpose:

- Manage reservation lifecycle from inquiry to booking, check-in, check-out, and closure.

Main records:

- Reservation / booking.
- Guest/customer.
- Property/unit.
- Date range.
- Status.
- Price summary.
- Deposit amount.
- Remaining balance.
- Contract snapshot.
- Documents.
- Check-in/check-out records.

Common statuses:

- Draft.
- Pending confirmation.
- Confirmed.
- Checked in.
- Checked out.
- Cancelled.
- No-show.
- Closed.

## 6. Documents Domain

Purpose:

- Manage attachments such as contracts, national IDs, invoices, utility bills, and meter images.

Main records:

- Document metadata.
- File key in R2.
- Owner entity.
- Access scope.
- OCR result.
- Verification status.

Key business questions:

- Which business entity owns the document?
- Is the document sensitive?
- Is OCR required?
- Has a human verified the extracted data?

## 7. OCR & Verification Domain

Purpose:

- Extract structured information from uploaded documents and images.

Use cases:

- National ID extraction.
- Utility bill extraction.
- Meter balance extraction.
- Contract or invoice text extraction.

Important rule:

- OCR output should be treated as suggested data until verified when the field is sensitive or financially relevant.

## 8. Payments Domain

Purpose:

- Track reservation payments, deposits, remaining balances, and prepare for Paymob integration.

Initial scope:

- Manual payment records.
- Payment status.
- Payment notes.
- Reservation payment summary.
- Deposit tracking.
- Remaining balance tracking.

Future scope:

- Paymob payment gateway integration.
- Vodafone Cash, InstaPay, Fawry, and bank-account payment channels through Paymob/account setup.
- Refunds.
- Webhook verification.
- Invoices.

## 9. Contracts Domain

Purpose:

- Ensure every confirmed reservation has a contract record and preserved terms.

Main records:

- Contract.
- Contract version/snapshot.
- Linked reservation.
- Customer identity data.
- Accepted terms.

Key business questions:

- Which contract belongs to this reservation?
- What terms did the customer accept?
- What deposit and remaining balance were agreed?

## 10. Daily Accounting Domain

Purpose:

- Prepare a daily accounting sheet for accountant review.

Main records:

- Daily sheet.
- Contract entries.
- Unit/apartment references.
- Income.
- Expenses.
- Commissions.
- Collected amounts.

Key business questions:

- What was collected today?
- Which contracts generated income?
- What expenses were recorded?
- Which commissions are included or excluded from expenses?

## 11. Utilities Domain

Purpose:

- Record water/electricity bills and meter/card readings through images and OCR.

Main records:

- Utility bill image.
- Utility type.
- OCR text.
- Parsed value.
- Verification status.
- Linked reservation/unit.

## 12. Maintenance & Cleaning Domain

Purpose:

- Manage internal cleaning work and external maintenance services.

Main records:

- Task.
- Worker/vendor.
- Unit.
- Reservation link when applicable.
- Task status.
- Cost when applicable.

## 13. Calendar & Scheduling Domain

Purpose:

- Provide operational scheduling for reservations, rentals, cleaning, workers, expenses, and annual rent reminders.

Views needed later:

- Unit availability.
- Worker tasks.
- Scheduled rentals.
- Due payments.
- Expenses.
- Annual rent due dates.

## 14. Reporting & Dashboard Domain

Purpose:

- Provide operational visibility without building heavy accounting early.

Initial KPIs:

- Leads count.
- Conversion rate.
- Reservations count.
- Occupancy rate.
- Revenue summary.
- Daily collections.
- Outstanding balances.
- Dynamic price movement indicators.
- Owner/company-owned unit performance.
- Follow-up delays.
- Property comparison metrics.
- Maintenance and cleaning costs.

Implementation note:

- Start with optimized SQL/read models/materialized views. Add read cache later only when monitoring proves the need.
