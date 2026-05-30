# Domain-Specific Business Requirements

## Purpose

This file captures the real business requirements for the rental/property operation model. These requirements should drive later UX flows, database modeling, service logic, and reporting.

## 1. Pricing

- Pricing is dynamic and may change almost daily.
- The system should support price increase/decrease indicators.
- Prices differ by building and by unit.
- Each unit may have its own pricing profile.
- Pricing history should be preserved for later analysis.
- The price used in a reservation/contract must be snapshotted so later price changes do not alter historical bookings.

## 2. Ownership Model

- There are multiple owners.
- The company office can also be an owner.
- Some units are owned by external owners.
- Some units are owned by the company.
- Each unit/apartment must have a clear owner.
- Some units may have prepaid amounts or money paid in advance related to the unit.

## 3. Buildings And Units

- Units belong to buildings.
- Each building contains multiple units.
- Rental is done at the unit level, not only at the building level.
- Each building should have:
  - Image.
  - Location.
  - General specifications.
- Each unit should have:
  - Images/specifications.
  - Owner.
  - Pricing.
  - Availability status.
  - Furniture/assets assigned to it.

## 4. Reservations, Rentals, And Contracts

- The business has customers, reservations, reservation history, rentals, and contracts.
- A reservation becomes a rental/contract when the customer arrives or the booking is activated.
- A reservation cannot be confirmed unless at least a deposit is paid.
- Minimum deposit is at least the price of one day.
- Every reservation must have a contract record.
- Customer pays part of the amount at reservation time and pays the remaining balance on handover/check-in.
- Contract terms and accepted contract snapshot must be preserved.

## 5. Booking Sources And Future Integrations

- Current bookings are manual.
- There are no current external booking sources.
- Bookings are created by distributed company offices in Ras El Bar.
- The design must remain scalable for future integrations with:
  - Direct booking platforms.
  - Online channel managers.
  - Platforms that provide reservations.
  - iCal feeds.
- Future external integrations must not be allowed to break the internal manual booking source-of-truth.

## 6. Daily Accounting Sheet

- The business needs a daily accounting sheet.
- The daily sheet contains:
  - Contracts.
  - Apartments/units.
  - Dates.
  - Income.
  - Expenses.
  - Collected cash/money.
- The accountant receives or reviews this daily collection.
- Commissions may exist and sometimes should not be registered as expenses on every contract.
- Expense and commission handling needs explicit business rules.

## 7. Utilities And Bills

- Water bills must be recorded in the system through image capture/upload.
- Electricity bills or cards must be recorded in the system through image capture/upload.
- OCR should extract relevant values when possible.
- Human verification is required before values affect accounting or settlement.

## 8. Maintenance And Cleaning

- Maintenance services may be external.
- Cleaning services are usually internal and performed by company workers.
- Worker calendars should support cleaning and operational tasks.
- Maintenance/cleaning tasks should be linked to units, reservations, workers, dates, and costs when applicable.

## 9. Workers And Users

- The system must manage workers and users.
- Workers may include cleaning staff, operations staff, and possibly maintenance coordinators.
- Users may include office staff, managers, accountants, agents, and admins.
- Worker schedules should support operational planning.

## 10. Calendar Requirements

The calendar should eventually include multiple operational views:

- Unit availability for reservations/rentals.
- Worker tasks for cleaning and operations.
- Scheduled rentals.
- Due payments and expenses.
- Annual rent reminders.

## 11. Furniture And Internal Assets

- The company owns furniture/assets.
- Furniture/assets may be distributed across units/apartments/huts.
- The system should track which assets belong to which unit.
- Asset movement history should be preserved when practical.

## 12. Annual Rent

- Some units or obligations have annual rent.
- Annual rent dates must be tracked.
- The system should remind users before payment due dates.
- Annual rent should appear in due payments/expenses calendar.

## 13. Customer Identity Capture

- Rental flow should support capturing national ID by image upload or camera capture.
- OCR should extract customer data from the ID.
- Extracted identity data must be reviewed before being trusted.
- The original ID image must be stored securely in R2 with metadata in PostgreSQL.

## 14. Payment Gateway

- Paymob will be used as the payment gateway.
- Paymob will be connected to the manager's account.
- Payments should be unified through supported channels such as:
  - Vodafone Cash.
  - InstaPay.
  - Fawry.
  - Bank account.
- Gateway webhooks must be verified and idempotent when implemented.

## 15. Analytics For Management

Managers need essential analytics to make business decisions.

Initial analytics should include:

- Occupancy rate.
- Daily/weekly/monthly revenue.
- Reservation count.
- Outstanding balances.
- Unit performance.
- Building performance.
- Lead-to-reservation conversion.
- Expenses by category.
- Maintenance/cleaning costs.
- Owner/company-owned unit performance.
- Price movement indicators.

## 16. Shareable Unit Details

- Unit details should be easy to share.
- A unit share view/message should include:
  - Unit name/code.
  - Building.
  - Photos.
  - Location.
  - Specifications.
  - Current price or price range when appropriate.
  - Availability note when appropriate.
- Shared details must not expose internal owner, commission, cost, or private financial data.
