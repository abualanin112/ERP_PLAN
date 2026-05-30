# Actors And Permissions

## Purpose

This file defines the business actors and their responsibility boundaries. It is a business-level document, not a final authorization schema.

## Core Actors

### System Admin

Responsibilities:

- Manage users.
- Manage roles and permissions.
- Configure system settings.
- Review audit and operational issues.

Can access:

- All modules.
- User management.
- System configuration.
- Sensitive operational logs, with redaction rules.

### Business Owner / General Manager

Responsibilities:

- Monitor company performance.
- Review sales, reservations, revenue, and property KPIs.
- Approve high-impact decisions.

Can access:

- Dashboards.
- Customers and leads.
- Reservations.
- Properties.
- Reports.
- High-level financial summaries.

### Branch Manager

Responsibilities:

- Manage branch-level staff.
- Review leads and reservations assigned to the branch.
- Monitor branch performance.
- Approve exceptions within branch policy.

Can access:

- Branch customers.
- Branch leads.
- Branch reservations.
- Branch properties.
- Branch reports.

### Sales Agent

Responsibilities:

- Create and follow up leads.
- Convert leads to reservations.
- Add notes and activities.
- Communicate with customers.

Can access:

- Assigned leads.
- Assigned customers.
- Relevant properties.
- Reservation creation flow.

### Broker / Field Agent

Responsibilities:

- Manage property visits.
- Capture field notes.
- Upload customer or property documents when allowed.
- Confirm visit/check-in details.

Can access:

- Assigned appointments.
- Assigned properties.
- Limited customer contact data.
- Upload flows.

### Operations Employee

Responsibilities:

- Manage check-in/check-out tasks.
- Upload meter images and documents.
- Verify OCR outputs.
- Track operational status.

Can access:

- Reservations.
- Documents.
- Check-in/check-out records.
- Meter reading records.

### Accountant

Responsibilities:

- Review payments and invoices.
- Prepare revenue summaries.
- Handle future financial workflows.

Can access:

- Payment records.
- Invoice records.
- Financial reports.
- Reservation financial details.

### Customer / Guest

Responsibilities:

- Provide contact information.
- Submit documents if self-service is enabled.
- Confirm booking terms.

Can access:

- Own booking information.
- Own document upload links.
- Own contract/signature flow.

## Permission Principles

- Users should only see data relevant to their role, branch, or assignment.
- Sensitive documents require explicit access rules.
- Financial data should be more restricted than operational data.
- Audit logs should track critical actions.
- Role configuration should be preferred over hardcoded per-user exceptions.

## Future UX Notes

Every actor should later map to:

- Primary dashboard.
- Main navigation.
- Allowed actions.
- Hidden or disabled actions.
- Empty states.
- Approval states.
