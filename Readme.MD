Problem 1: Tatkal Booking Crashes at 10:00 AM [Given]
What is broken:

The website becomes extremely slow or unresponsive during Tatkal booking hours due to a massive surge in users.

Affected users:
Daily commuters
Emergency travelers
Students
Business travelers
Frequency:

Occurs almost every day during Tatkal booking windows (10 AM and 11 AM).

Current flow — step by step:
User logs into IRCTC.
Searches for trains.
Selects Tatkal quota.
Enters passenger details.
Moves to payment page.
Website slows down or crashes.
Where exactly it breaks:

Step 5–6: During payment processing or final booking confirmation.

Problem 2: Payment Success but Ticket Not Generated [Given]
What is broken:

The payment gets deducted from the user's account, but the ticket is not generated immediately.

Affected users:
All online ticket-booking users
Frequency:

Occasional but highly frustrating, especially during peak demand periods.

Current flow — step by step:
User selects train and enters passenger details.
Proceeds to payment.
Payment gateway processes transaction.
Amount is deducted.
Confirmation page fails to load.
Where exactly it breaks:

Step 5: Ticket booking status remains pending or unknown after successful payment.

Problem 3: Long Passenger Form Entry Process [Given]
What is broken:

Entering passenger details is repetitive and time-consuming, especially for family bookings.

Affected users:
Families
Senior citizens
Frequent travelers
Frequency:

Every booking involving multiple passengers.

Current flow — step by step:
User selects train.
Opens passenger details page.
Enters each passenger's:
Name
Age
Gender
Berth preference
Reviews information.
Continues to payment.
Where exactly it breaks:

Step 3: Manual data entry takes significant time and increases booking failures during high-demand periods.

Problem 4: Difficulty Finding Alternative Trains After Waitlist [Self-Discovered]
What is broken:

When preferred trains are fully booked, IRCTC does not proactively recommend the best alternative trains.

Affected users:
Travelers booking close to travel dates
Festival-season passengers
Frequency:

Common during holidays, weekends, and peak travel seasons.

Current flow — step by step:
User searches route.
Opens preferred train.
Sees waitlist status.
Returns to search results.
Manually checks other trains one by one.
Where exactly it breaks:

Step 4–5: Users must perform repetitive searches to find viable alternatives.

Problem 5: Poor Visibility of Refund Status [Self-Discovered]
What is broken:

Refund tracking is difficult and users often do not know when they will receive their money.

Affected users:
Users canceling tickets
Users affected by failed bookings
Frequency:

Whenever tickets are canceled or transactions fail.

Current flow — step by step:
User cancels ticket.
Receives cancellation confirmation.
Searches refund information.
Checks booking history repeatedly.
Waits for refund.
Where exactly it breaks:

Step 3–4: Refund status lacks clear progress updates and estimated timelines.

Problem 6: Train Delay Information Is Hard to Notice During Booking [Self-Discovered]
What is broken:

Users often book trains without easily seeing recent delay trends or reliability information.

Affected users:
Long-distance travelers
Business travelers
Time-sensitive passengers
Frequency:

Daily across many routes.

Current flow — step by step:
User searches trains.
Compares departure and arrival times.
Selects train.
Books ticket.
Where exactly it breaks:

Step 2–3: Historical punctuality and delay insights are not prominently displayed during decision-making.

Key Insights
Problem	Severity	User Impact
Tatkal booking crashes	High	Users unable to secure tickets
Payment success but ticket not generated	High	Financial anxiety and booking uncertainty
Long passenger form entry	Medium	Slow booking process
Alternative train discovery	Medium	Poor travel planning
Refund tracking visibility	Medium	User frustration
Train delay visibility	Medium	Poor booking decisions

These six problems cover performance, payments, usability, discoverability, transparency, and decision support, giving a strong foundation for the next phase of IRCTC product analysis.