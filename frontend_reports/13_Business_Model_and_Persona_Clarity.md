# Business Model & Actor Clarity Report

## 1. The Core Concept: "Who is this for?"

**This is NOT Uber.**
Uber is a B2C (Business to Consumer) app where *you* (the passenger) open an app to book a ride for yourself.

**This IS a "Fleet Management ERP" (B2B)**.
Imagine a company called **"Star Travels"**. They own 500 cars.
They don't pick up random people from the street. They have **Corporate Contracts** with big companies like **Google, Infosys, and TCS**.

-   **The End User of this Software**: The **Employees of Star Travels** (Receptionists, Dispatchers, Accountants). *Only they log in.*
-   **The "Client"**: **Google / Infosys**. They are the *Customer* paying the bills. They do *not* use this software. They just send emails saying "We need a car".
-   **The "Guest"**: Mr. Smith (Google Employee). He sits in the car. He *never* sees this software.

## 2. Who are the Actors? (The Roles)

| Role | Who are they? | Do they log in? | What do they do? |
| :--- | :--- | :--- | :--- |
| **Super Admin** | The Owner of "Star Travels" | **YES** | Sets up the company, sees total revenue, creates other admins. |
| **Admin** | The Manager at the "Mumbai Branch" | **YES** | Creates bookings, assigns drivers, generates invoices. |
| **Client** | "Google India Pvt Ltd" | **NO** | They are just a name in the database. They pay the bills. |
| **Guest** | Mr. Sharma (Google VP) | **NO** | The passenger. His name is on the Booking so the driver knows who to pick up. |
| **Booker** | Mr. Sharma's Secretary | **NO** | The person who called "Star Travels" to request the car. |
| **Driver** | Ramesh (Employee of Star Travels) | **NO** | He drives the car. He doesn't use the app (in this specific version); the Admin enters his data. |

## 3. The "Quotation" Logic (Fixed Access)

**Why do we need Quotations?**
Because "Star Travels" has different deals for different companies.
-   **Deal with Google**: "We will charge you **₹1000** for an Airport Drop."
-   **Deal with a Startup**: "We will charge you **₹1200** for an Airport Drop."

**How it works in the software**:
1.  Admin selects "Client: Google".
2.  System looks at **Google's Quotation**.
3.  System says: "Okay, for Google, the price is fixed at ₹1000. I will not let you change it."
4.  **Result**: The Admin *cannot* overcharge or undercharge. The price is **Fixed by Contract**.

## 4. The Workflow Confusion: Booking vs. Duty

You asked: *"Do we convert booking to duty or does a booking itself go to unbilled slips?"*

**The Answer: Booking -> CONVERTED TO -> Duty -> Unbilled Slip.**

Think of it like a Restaurant:
1.  **Booking (The Reservation)**: "I reserve a table for Friday."
    -   *Logic*: This is a Plan. No money is involved yet. You can cancel it.
    -   *System*: ID `BK-101`. Status: `Confirmed`.

2.  **Duty ( The Meal)**: You arrive, you eat, you leave.
    -   *Logic*: This is the Execution. The waiter notes down exactly what you did (Start Time, End Time, Km Run).
    -   *System*: The Admin clicks "Create Duty" on `BK-101`. A **NEW** record `DT-505` is created. `BK-101` is marked "Completed".

3.  **Unbilled Slip (The Bill on the Counter)**:
    -   *Logic*: You have finished eating, but you haven't paid yet. The waiter takes the note to the cashier.
    -   *System*: `DT-505` appears in the "Unbilled List". The Accountant sees it.

4.  **Invoice (The Final Receipt)**:
    -   *Logic*: The Cashier prints the final paper with tax added.
    -   *System*: The Accountant selects `DT-505` and clicks "Generate Bill". A PDF `INV-2024` is created. Now `DT-505` disappears from "Unbilled".

## 5. Why separate Booking and Duty?
Because Plans change!
-   **Booking**: "I need a Sedan for 8 Hours."
-   **Actual Duty**: "The traffic was bad, so I used the car for 12 Hours."
-   The **Duty** record stores the *actual* 12 hours. The **Booking** record stays as the original request. The **Invoice** charges for 12 hours.

## 6. Summary for You
This software is a **Management Tool** for the Back-Office of a Taxi Company. It automates their contracts, tracks their cars, and ensures they bill their Corporate Clients correctly based on pre-agreed Rates (Quotations).
