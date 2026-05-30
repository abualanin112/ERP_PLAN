# UX Translation Brief

## Purpose

This file prepares the business requirements for later UX work. It does not define layouts or visual design.

## UX Conversion Inputs

Before designing screens, each workflow should provide:

- Actor.
- Goal.
- Starting trigger.
- Required data.
- Business rules.
- Success state.
- Failure/exception states.
- Next action.
- Permissions.
- Audit requirements.

## Primary UX Journeys To Design Later

### 1. Sales Agent Daily Work

Questions to answer:

- What leads require follow-up today?
- What is the next best action?
- Which leads are close to booking?
- Which customers need documents?

### 2. Lead Creation And Follow-Up

Questions to answer:

- How fast can an agent add a lead?
- What fields are required vs optional?
- How does the agent schedule follow-up?
- How are lost reasons captured?

### 3. Reservation Creation

Questions to answer:

- How does the agent select customer and property?
- Where is availability verified?
- What happens if there is a conflict?
- How does the system move from draft to confirmed?

### 4. Check-In / Check-Out

Questions to answer:

- What tasks must be completed before status changes?
- How does the user upload documents/images?
- How is OCR output reviewed?
- What if OCR fails?

### 5. Manager Dashboard

Questions to answer:

- Which KPIs matter daily?
- Which KPIs matter weekly/monthly?
- What needs alerts vs passive visibility?
- How are property comparisons presented?

### 6. Document Review

Questions to answer:

- Which documents are pending?
- Which documents failed OCR?
- Who can verify or reject?
- What reason is required for rejection?

## UX Rules

- Prioritize task completion over decorative screens.
- Show next action clearly.
- Avoid hiding critical business statuses.
- Make exceptions visible and assignable.
- Keep sensitive data exposure role-aware.
- Design for mobile-friendly operational tasks where field agents are involved.

## Output Expected Later

These business files should later produce:

- User journeys.
- Screen list.
- Navigation map.
- Form requirements.
- Empty states.
- Error states.
- Permission-aware UI behavior.
- API workflow candidates.
