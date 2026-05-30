# Business Context

## Product Vision

Build a practical PropTech CRM/ERP system that starts with CRM Lite and reservations, then grows gradually into a broader ERP for property operations.

The system should help the business:

- Manage customers and leads.
- Track sales and reservation workflows.
- Manage buildings, units, owners, and company-owned units.
- Support dynamic unit pricing and price movement indicators.
- Track reservations, rentals, contracts, deposits, and remaining balances.
- Record daily income, expenses, commissions, and collected money for accountant review.
- Reduce manual follow-up.
- Keep documents and booking evidence organized.
- Capture national IDs, water bills, and electricity bills through image upload/OCR.
- Coordinate cleaning, maintenance, workers, annual rent reminders, and unit assets.
- Provide operational dashboards.
- Prepare for Paymob payments, future accounting expansion, integrations, and advanced reporting.

## MVP Business Strategy

Start with CRM Lite rather than full ERP.

MVP focus:

- Authentication and basic roles.
- Customers.
- Leads.
- Sales pipeline.
- Properties.
- Reservations Lite.
- Contracts per reservation.
- Deposits and remaining balances.
- Daily accounting sheet.
- Documents and attachments.
- Utility bill/meter image capture.
- Basic worker/cleaning/maintenance task tracking.
- Basic dashboard.

Deferred ERP scope:

- Full accounting.
- General ledger.
- Payroll.
- Advanced inventory.
- Advanced tax engines.
- Complex financial closing.
- External booking/channel-manager integrations.
- Read caching layer, unless performance metrics justify it later.

## Target Users

- Business owner / company manager.
- Branch manager.
- Sales agent.
- Broker / field agent.
- Operations employee.
- Accountant, later phase.
- Cleaning worker / maintenance coordinator.
- Admin / system operator.
- Customer / guest, if self-service flows are added later.

## Business Goals

- Increase lead follow-up quality.
- Improve reservation accuracy.
- Reduce double booking risk.
- Keep customer and booking documents searchable.
- Track property performance.
- Track building/unit performance and price movement.
- Track owner/company-owned unit performance.
- Track daily collections for accountant review.
- Improve sales team visibility.
- Enable gradual ERP expansion without rewriting the core.

## Success Metrics

- Number of active users.
- Leads created and followed up.
- Reservations created.
- Lead-to-booking conversion rate.
- Average follow-up delay.
- Time saved in document collection.
- Booking conflicts prevented.
- Properties with complete operational data.
- Units with complete pricing/owner/document data.
- Deposits collected before confirmed reservations.
- Daily sheets reviewed by accountant.
- Utility bills captured and verified.

## Product Principle

Build a useful operating system first, not a large theoretical ERP. Every module should solve a real daily workflow before adding advanced automation.
