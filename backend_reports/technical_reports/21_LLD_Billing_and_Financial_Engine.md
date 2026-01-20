# Low Level Design: Billing & Financial Engine

## 1. Overview
The **Billing Engine** is the "Cash Register". It takes operational data (Duties) and converts it into financial documents (Invoices).
**Critical Requirement**: Accuracy and Immutability. Once a bill is sent to a client, the underlying data *cannot* change.

## 2. Core Components

### 2.1 Bill Manager (`billManager.js`)
**Key Logic: The Transactional Generate**
Generating a bill is not a simple "Create" operation. It is a multi-step transaction:
1.  **Fetch Next Number**: Like bookings, Invoices use per-location sequence counters.
2.  **Create Invoice Record**: Create the `Bill` document containing the `totalAmount` and the list of `duty_ids`.
3.  **Atomic Lock (The Critical Step)**:
    *   The system executes `Duty.updateMany({ _id: { $in: duty_ids } }, { invoiceGenerated: true, invoiceNumber: 123 })`.
    *   **Why?** This prevents the *same* duty from being billed twice. Before creating a bill, the `validateInvoiceGeneration` function checks if *any* directly selected duty already has `invoiceGenerated: true`. If yes, it throws an error.

### 2.2 Financial Logic: The "Tax Brain" (`frontend/helpers/billValuesHelper.js`)
Although the backend stores the final values, the **Calculation Logic** currently resides heavily in the frontend helpers to allow real-time UI updates (Client-side Calculation).

**The Logic Flow**:
1.  **Input**: List of Duties.
2.  **Grouping**: Group duties by Client (You can't bill Google and Infosys on the same invoice).
3.  **Tax Decision Engine**:
    *   Get `Client.GSTStateCode` (e.g., 27 for MH).
    *   Get `Location.GSTStateCode` (e.g., 27 for MH).
    *   **Logic**:
        ```javascript
        if (Client.State == Location.State) {
            Apply CGST (2.5%) + SGST (2.5%);
        } else {
            Apply IGST (5.0%);
        }
        ```
    *   *Implementation Detail*: This check happens dynamically. If you change the Client's state in Master, future bills change logic. Past bills remain static because their values are saved.

### 2.3 The "Reverse Calculation" (Package Logic)
For **Packages** (e.g., "8hr/80km for ₹2000, Extra km ₹15"):
*   The system calculates: `Used Km - Package Km`.
*   If positive, `Extra Cost = Difference * Extra Rate`.
*   If negative (used less than package), `Extra Cost = 0`.
*   **Result**: The system ensures the "Minimum Guarantee" (Package Price) is always charged.

## 3. Data Integrity & Safety
*   **Double-Bill Protection**: The `invoiceGenerated` boolean flag on the Duty model is the primary guard.
*   **Soft Deletes**: We use `mongoose-delete`. If a Bill is "Deleted", the system performs a **Rollback**:
    *   The `Bill` is marked deleted.
    *   The linked `Duties` set `invoiceGenerated: false`.
    *   This releases the duties back into the "Unbilled" pool.

## 4. Why this Design?
*   **Locking vs Copying**: We chose to **Lock** the original Duty records rather than copying their data into the Invoice.
    *   *Pros*: Single source of truth. If you fix a spelling mistake in the Duty remark, the Bill reflects it.
    *   *Cons*: Must be very careful not to let users edit financial fields (Price, Km) of a locked Duty.
*   **Location-Based Sequencing**: Essential for compliance. Each branch needs its own customized Invoice Series (Start from 001) for GST filing.

## 5. Potential Interview Questions
*   "How do you prevent billing the same duty twice?" -> **By flagging the Duty record updates atomically.**
*   "What happens if I delete an invoice?" -> **It triggers a rollback that unlocks the duties.**
