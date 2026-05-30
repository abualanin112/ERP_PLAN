# Open Business Questions

## Purpose

This file captures unanswered business decisions. These should be resolved before final UX and backend implementation.

## Company Structure

- Will the system support one company only at first?
- Are branches required in MVP?
- Are properties assigned to branches?
- Can users belong to multiple branches?

## Customer And Lead Management

- What fields are mandatory for a customer?
- What fields are mandatory for a lead?
- What lead sources should exist initially?
- Should duplicate detection be strict or advisory?
- Who can reassign leads?

## Sales Pipeline

- What are the final pipeline stages for MVP?
- Is every stage required?
- Can stages be customized later?
- What reasons are required for lost leads?

## Properties And Reservations

- Are reservations for short rental only, annual rental, visits, sales, or all of them?
- What are the exact building/unit types: apartment, hut, room, chalet, other?
- How should company-owned units differ from external-owner units in reporting?
- Do draft reservations block availability?
- What is the cancellation policy?
- Are no-show and expired reservations required?
- Who can override the minimum one-day deposit rule?
- When exactly does reservation become rental/contract: arrival, deposit, confirmation, or check-in?
- What contract fields are mandatory?

## Pricing

- What pricing periods are needed: daily, seasonal, weekend, holiday, custom date range?
- Who can change prices?
- Should price changes require approval?
- What price movement indicator is needed for managers?
- Do old reservations keep the agreed price even if unit price changes later?

## Owners And Commissions

- Can one unit have multiple owners?
- How are owner payouts calculated?
- What does "money paid in the unit in advance" mean operationally: owner advance, maintenance advance, prepaid rent, or deposit?
- When is commission recorded?
- When should commission not be registered as an expense?

## Documents And OCR

- Which documents are required for reservation?
- Which documents are optional?
- Who can upload documents?
- Who verifies OCR output?
- Which OCR fields can be auto-filled without review?

## Check-In / Check-Out

- Is meter reading mandatory for all reservations?
- Which utility types are required: electricity, water, gas?
- Can check-in happen with missing documents?
- Can check-out happen with unresolved payment?

## Payments

- Is payment tracking required in MVP?
- Are payments manual in MVP?
- Which payment statuses are needed?
- Is Paymob required in the first production version or later?
- Which Paymob channels are required at launch: Vodafone Cash, InstaPay, Fawry, bank account?
- Is the manager account the only settlement account?
- How should partial payments and remaining balances be reconciled?

## Daily Accounting

- What is the exact daily sheet format the accountant expects?
- Who creates, edits, reviews, and closes the daily sheet?
- What expense categories are required?
- Are maintenance and cleaning costs included in the daily sheet?
- How are collected cash and electronic payments separated?

## Workers, Cleaning, And Maintenance

- What worker roles are required?
- Should workers log into the system or only managers assign tasks?
- What statuses are needed for cleaning tasks?
- What statuses are needed for maintenance tasks?
- How are external maintenance vendors recorded?

## Assets And Annual Rent

- What asset categories should be tracked?
- Is asset condition tracked?
- Who can move assets between units?
- Which annual rent obligations need reminders?
- How many days before due date should reminders appear?

## Reporting

- Which KPIs are mandatory for launch?
- What date ranges should dashboards support?
- Who can see revenue?
- Are property comparisons required in MVP?
- Which management analytics are most important: occupancy, revenue, outstanding balances, owner performance, company-owned unit performance, cleaning/maintenance cost, price movement?

## Permissions

- What are the exact MVP roles?
- Does every user belong to a branch?
- Can sales agents see each other's leads?
- Can operations employees see financial data?

## Future Cache

- What measurable threshold will trigger adding read cache?
- Which endpoints are most likely to need future cache?
- Who reviews monitoring data before approving cache implementation?

## UX Preparation

- What is the first screen for each actor?
- What actions must be one-click or very fast?
- Which workflows happen on mobile?
- Which workflows require manager approval?
