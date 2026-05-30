# Business Workflows

## Purpose

This file captures end-to-end workflows. These workflows can later be converted into UX journeys, screen maps, API workflows, and test scenarios.

## 1. Lead To Reservation

Goal:

- Convert a potential customer into a confirmed reservation.

Flow:

1. Sales agent creates a lead.
2. Agent assigns source, interest, budget, dates, and preferred property type.
3. Agent follows up with the customer.
4. Lead moves through pipeline stages.
5. Agent selects a property/unit.
6. System checks availability.
7. Agent creates reservation draft.
8. Agent records required deposit; minimum deposit is at least one day price.
9. Customer confirms.
10. Reservation becomes confirmed.
11. Contract record is created for the reservation.
12. Follow-up tasks and documents are attached.

Business outputs:

- Lead status updated.
- Reservation created.
- Deposit recorded.
- Remaining balance calculated.
- Contract linked to reservation.
- Customer linked to reservation.
- Sales activity history preserved.

## 2. Customer Document Collection

Goal:

- Collect and organize customer documents safely.

Flow:

1. Agent requests required document.
2. Customer or agent uploads file.
3. File is stored in Cloudflare R2.
4. Metadata is stored in PostgreSQL.
5. OCR job is queued if needed.
6. Extracted fields are reviewed.
7. Document becomes verified or rejected.

Business outputs:

- Document linked to customer or reservation.
- OCR output stored as metadata.
- Verification status visible.

## 3. Reservation Check-In

Goal:

- Start a customer stay or reservation usage with proper records.

Flow:

1. Operations employee opens confirmed reservation.
2. Employee checks required documents.
3. Employee uploads check-in images if needed.
4. Meter/card image is uploaded if applicable.
5. OCR extracts starting balance.
6. Employee confirms or corrects extracted data.
7. Reservation status becomes checked in.

Business outputs:

- Check-in timestamp.
- Starting meter balance.
- Check-in documents.
- Operational audit trail.

## 4. Reservation Check-Out

Goal:

- Close the stay and capture final operational readings.

Flow:

1. Operations employee opens active reservation.
2. Employee uploads check-out meter/card image.
3. OCR extracts ending balance.
4. Employee verifies extracted value.
5. System calculates difference if needed.
6. Reservation status becomes checked out.
7. Any remaining payment or document tasks are flagged.
8. Reservation becomes closed after all checks pass.

Business outputs:

- Ending meter balance.
- Check-out timestamp.
- Closure status.
- Operational notes.

## 5. Property Performance Review

Goal:

- Compare property performance and support operational decisions.

Flow:

1. Manager selects a time period.
2. Manager selects properties or branch.
3. System displays KPIs.
4. Manager compares occupancy, revenue, ADR, RevPAR, lead conversion, cancellation rate, and ratings.
5. Manager identifies best/worst performing properties.

Business outputs:

- Property comparison insight.
- Potential pricing or marketing actions.
- Maintenance or operational follow-up.

## 6. Payment Tracking

Goal:

- Track reservation payment state without full accounting at MVP stage.

Flow:

1. Reservation has expected amount.
2. User records payment manually or through future gateway.
3. Payment status updates.
4. Reservation financial summary updates.
5. Manager/accountant reviews outstanding balances.

Business outputs:

- Payment status.
- Paid amount.
- Remaining amount.
- Payment audit notes.

## 7. Daily Accounting Sheet

Goal:

- Prepare daily income/expense/collection summary for accountant review.

Flow:

1. System lists today's contracts/reservations.
2. User reviews unit, dates, customer, income, paid amount, and remaining amount.
3. User records expenses and commissions when applicable.
4. Accountant reviews collected money.
5. Daily sheet is marked reviewed/closed.

Business outputs:

- Daily sheet.
- Income total.
- Expense total.
- Commission notes.
- Collected amount.
- Accountant review status.

## 8. Dynamic Pricing Review

Goal:

- Track and adjust prices that may change daily.

Flow:

1. User opens building/unit pricing.
2. User reviews current price and previous price movement.
3. User updates daily or seasonal price.
4. System stores price history.
5. Future reservations use current price, while old reservations keep their snapshot.

Business outputs:

- Current unit price.
- Price history.
- Price increase/decrease indicator.
- Reservation price snapshot.

## 9. Utility Bill Capture

Goal:

- Record water/electricity bills or card readings using image capture.

Flow:

1. User opens reservation or unit.
2. User uploads/captures water or electricity image.
3. OCR extracts text and suggested value.
4. User verifies or corrects value.
5. System stores image, OCR text, parsed value, and verification status.

Business outputs:

- Utility record.
- Verified reading/value.
- Linked image.
- Audit trail.

## 10. Worker Calendar And Cleaning/Maintenance

Goal:

- Coordinate worker tasks around reservations and unit status.

Flow:

1. User creates cleaning or maintenance task.
2. Task is linked to unit/reservation.
3. Task is assigned to internal worker or external vendor.
4. Task appears in worker/operations calendar.
5. Worker/operations employee updates task status.
6. Cost is recorded when applicable.

Business outputs:

- Worker task.
- Unit operational status.
- Cleaning/maintenance calendar.
- Cost record when applicable.

## 11. Unit Share Flow

Goal:

- Share unit details easily with customers.

Flow:

1. User opens unit profile.
2. User selects share action.
3. System prepares safe public/shareable details.
4. User sends link/message externally.

Business outputs:

- Shareable unit summary.
- No private owner/commission/internal financial data exposed.

## 12. Exception Handling

Examples:

- Availability conflict.
- Missing documents.
- OCR cannot extract value.
- Customer cancels.
- Payment incomplete.
- Sales agent reassigns lead.

Business rule:

- Every exception should have a visible status, a responsible actor, and a next action.
