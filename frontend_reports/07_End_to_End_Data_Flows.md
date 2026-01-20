# End-to-End Data Flows & Lifecycle Analysis

## 1. Executive Summary
This report traces the path of data through the Safar Manager application for its most critical business workflows. Understanding these flows is essential for debugging issues and explaining the system's logic in technical interviews. We will dissect the "Create Booking" and "Generate Bill" lifecycles.

## 2. Workflow A: The Booking Lifecycle
"From Call to Confirmation"

### Phase 1: User Interaction (Frontend)
1.  **Initialization**: User navigates to `/dashboard/booking/booking/new`.
    - `useEffect` triggers: Fetch Locations, Clients, Drivers, and Cities.
    - **UI**: The form renders. `SelectBox` components are populated with the fetched data.
2.  **Selection & calculation**:
    - User selects "Client A".
    - System runs `useEffect` to fetch **Quotations** specific to "Client A".
    - User selects a "Sedan".
    - System auto-populates `Duty Type` based on the contract (e.g., "Airport Drop").
    - **Logic**: Frontend proactively prevents errors by only showing valid options found in the contract (Quotation).
3.  **Submission**:
    - User clicks "Save".
    - `validateBooking()` runs (Checks dates, phone numbers).
    - `dispatch(createBooking(bookingData))` is called.

### Phase 2: Action & Transport (Network)
4.  **Redux Thunk**: `createBooking` action sets `loading: true`.
5.  **API Request**: `POST /api/v1/booking/new` is fired with the JSON payload.
    - Payload includes: `clientId`, `vehicleId`, `dates`, `guestDetails`.
    - Header: `Authorization: Bearer <JWT_TOKEN>`.

### Phase 3: Processing (Backend)
6.  **Routing**: `bookingRoute.js` directs request to `BookingController.newBooking`.
7.  **Manager Layer**: Controller instantiates `BookingManager`.
8.  **Transaction**: `BookingManager.save()` starts a MongoDB session.
    - **Step A**: Call `SequenceCounterManager` -> Increment `booking_seq` for Mumbai -> Returns "mum-1005".
    - **Step B**: `BookingModel.create({... data, bookingNumber: "mum-1005" })`.
    - **Step C**: Commit Transaction.
9.  **Response**: Server returns `201 Created` with the new Booking Object.

### Phase 4: Feedback (Frontend)
10. **State Update**: Redux dispatch `CREATE_BOOKING_SUCCESS`.
11. **UI Feedback**: `useEffect` listening to `isBookingCreated` triggers `showToast("Booking Created Successfully")`.
12. **Navigation**: User is redirected to `/dashboard/booking/booking` (The list view).

## 3. Workflow B: The Billing & Invoicing Cycle
"From Operations to Revenue"

### Phase 1: Selection (Frontend)
1.  **View**: User goes to "Process Bill".
2.  **Filter**: User selects "Client: Google", "Date: Last Month".
3.  **Query**: Frontend fetches **Unbilled** bookings (`invoiceGenerated: false`).
    - Note: This uses the `getAllBookings` API with specific filters.
4.  **Selection**: Admin ticks checkboxes for 5 specific duties to invoice.
5.  **Calculation**: Frontend runs `getBillAmount()`.
    - Sums up base rates.
    - Applies `Client.GSTSettings` (CGST/SGST) to calculate tax locally for preview.

### Phase 2: Action (Network)
6.  **Trigger**: User clicks "Generate Invoice".
7.  **Payload**: The API receives an array of IDs: `{ bookingIds: [...], dutyIds: [...] }`.

### Phase 3: Financial Integrity (Backend)
8.  **Validation**: `BillManager` checks if any of these IDs have *already* been invoiced by another admin in the last few seconds. (Concurrency check).
9.  **Transaction**:
    - **Create Bill**: Generate `Invoice #INV-2024-001`. Total Amount is calculated/verified on server side.
    - **Lock Records**: Update the 5 Booking documents: set `invoiceGenerated = true`, `invoiceNumber = "INV-2024-001"`.
    - **Commit**.

### Phase 4: Artifact Generation
10. **Success**: Frontend receives the new Invoice Object.
11. **PDF Generation**:
    - The browser does *not* download a file from the server.
    - Instead, the Frontend uses `jspdf`. It takes the JSON response (Line items, Tax amounts) and draws a PDF in the browser memory using the `generateInvoicePDF` helper.
    - The user sees a "Download" prompt.

## 4. Workflow C: Audit & Logging (Implicit)
Throughout these flows, implicit data tracking happens:
- **Soft Deletes**: If a user creates a booking and then deletes it, the backend only sets `deleted: true`. The `BookingManager` filters these out from standard lists, but a "Restore" or "Audit" flow can still access them.
- **CreatedBy**: Every record (Bill, Booking) stores the ID of the admin who created it (`req.user.id`). This creates an accountability trail.

## 5. Conclusion
The data flows in Safar Manager are designed for **Transactional Safety** and **User Convenience**.
- **Safety**: Heavy use of transactions and server-side validation ensures money and bookings aren't lost.
- **Convenience**: The frontend does heavy lifting (auto-calculations, PDF generation) to make the experience snappy, while the backend focuses on being the "Source of Truth".
