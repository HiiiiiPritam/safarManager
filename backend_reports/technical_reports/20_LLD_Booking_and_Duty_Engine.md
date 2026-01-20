# Low Level Design: Booking & Duty Engine

## 1. Overview
The **Booking** and **Duty** modules represent the operational core of the system.
*   **Booking**: A "Promise" or "Plan" (Future).
*   **Duty**: The "Execution" or "Reality" (Past/Present).

This distinction is critical because **Plans change**. A user might book a "Small Car" for "8 Hours", but actually use a "Big Car" for "12 Hours". The system must preserve the original request (Booking) while capturing the actual usage (Duty).

## 2. Core Components

### 2.1 Sequence Counter Manager (`sequenceCounterManager.js`)
**Problem**: MongoDB uses long, ugly IDs (`64b1f...`). Human operators need short, readable numbers (`BK-1001`, `DT-505`).
**Implementation**:
*   We use a separate collection `sequencecounters`.
*   **Multi-Tenancy**: The counter is scoped by **Location**.
    *   Delhi Branch might have `BK-1` to `BK-100`.
    *   Mumbai Branch might have `BK-1` to `BK-50`.
    *   *The system ensures no collision because they belong to different `location_id`s.*
*   **Atomic Increment**: We use `findOneAndUpdate` with `{ $inc: { bookingNumber: 1 } }` to ensure concurrency safety. Two admins clicking "Save" at the exact same millisecond will get unique numbers (101 and 102).

### 2.2 Booking Manager (`bookingManager.js`)
**Key Logic**:
1.  **Data Graph**: It doesn't just save a booking. It resolves links.
    *   It takes a `client_id` -> fetches the stored `Client` object.
    *   It takes a `quotation_id` -> ensures the specific `Quotation` is valid.
2.  **Date Validation**:
    *   The frontend handles basic checks, but the backend stores dates in **UTC**.
    *   *Critical Detail*: The system creates `start_date` (for filtering) and `start_time` (string for display).

### 2.3 Duty Manager (`dutyManager.js`)
**Key Logic**:
1.  **The "Conversion" Link**:
    *   When a Booking is "Converted" to a Duty, the Frontend sends the **SAME** `bookingNumber`.
    *   The `Duty` model stores this `bookingNumber`.
    *   *Queries*: To find the duty for a booking, we query `Duty.findOne({ bookingNumber: booking.bookingNumber })`. We do *not* use a hard foreign key reference ID for this link to allow flexibility (e.g., if a Booking is deleted but the Duty must remain).
2.  **Live State Tracking**:
    *   A Duty has a complex lifecycle: `Scheduled` -> `In Progress` -> `Completed` -> `Billed`.
    *   **In Progress**: Driver has started the trip.
    *   **Completed**: Driver returned, meters entered.
    *   **Billed**: Invoice generated. The system **LOCKS** the duty here. `DutyManager` will reject updates if `invoiceGenerated: true`.

## 3. Data Flow: The "Night Count" Loop

One of the most complex logic pieces is calculating **Night Allowances**.

**The Problem**: A driver drives from **10 PM to 6 AM**. Does he get 1 Night Allowance or 2?
**The Implementation (`billAmounts.js` + Backend Validation)**:
1.  **Config**: The Client Master has `nightStartTime` (e.g., 23:00) and `nightEndTime` (e.g., 05:00).
2.  **Algorithm**:
    *   The system iterates through every hour of the Duty execution.
    *   It checks overlap with the Client's specific Night Window.
    *   If overlap exists -> Increment `nightCount`.
    *   *Edge Case*: A trip approaches 12:00 AM. The system splits the check into "Before Midnight" and "After Midnight" segments to ensure accuracy.

## 4. Why this Design?
*   **Decoupling**: We separated Booking and Duty so operations can be messy (changing cars, dates) without breaking the original contract record.
*   **Scoped Counters**: We used Location-based counters so branches perceive themselves as independent companies.
*   **Soft Linking**: Matching by `bookingNumber` string instead of `ObjectID` allows legacy data integration and robustness against accidental deletion of parents.

## 5. Potential Interview Questions
*   "How do you handle two admins creating a booking at the same time?" -> **Atomic Sequence Counters.**
*   "Why separate Booking and Duty models?" -> **To separate Plan vs Execution.**
