# The Dictionary of Safar (Safar Manager Glossary)

## 1. Executive Summary
This document serves as a "Terminology Engine" for the Safar Manager application. It defines every specialized term found in the codebase, explains its business meaning, and provides the "Tech Translation" (what it maps to in the DB/Code). Read this to speak the language of the Domain Experts.

## 2. Core Entities (The Big Four)

### 2.1 Booking (`BookingModel`)
**Business Definition**: A promise or reservation. It represents a request from a client for a vehicle at a future date/time.
**In Code**: `models/bookingModel.js`
**Lifecycle**: `Confirm` -> `Dispatch` -> `Completed` -> `Billed`.
**Key Nuance**: A booking is "Planned work". It might change. A booking for a "Sedan" might end up being fulfilled by an "SUV" if the sedan breaks down.

### 2.2 Duty (`DutyModel`)
**Business Definition**: The actual *execution* of the work. This is the digital equivalent of the "Duty Slip" or "Logbook" the driver carries.
**In Code**: `models/dutyModel.js`
**Difference from Booking**:
- Booking = "I need a car on Monday" (Plan).
- Duty = "Driver Ramesh drove Car MH-01-AB-1234 from 9 AM to 9 PM" (Actual).
**Critical Fields**:
- `Start Meter` / `End Meter`: Odometer readings.
- `Time`: The actual hours the driver was on the road.

### 2.3 Quotation (`QuotationModel`)
**Business Definition**: The "Rate Card" or "Contract". It defines *how much* we charge a specific client.
**In Code**: `models/quotationModel.js`
**Structure**:
- A Client has *many* Quotations (e.g., "Airport Transfer Rates", "Monthly Rental Rates").
- A Quotation has *many* `QuotationVehicles` (Prices for Sedan, SUV, Bus).
**Why it matters**: You cannot create a Booking without selecting a valid Quotation. This prevents billing errors.

### 2.4 Bill / Invoice (`BillModel`)
**Business Definition**: The legal demand for payment.
**In Code**: `models/billModel.js`
**Aggregation**: One Bill can contain *multiple* Duties (e.g., "Monthly Invoice for April" contains 30 duties).
**Key Flags**: `invoiceGenerated` (Boolean). When true, the linked Duties are locked and cannot be edited.

## 3. Operational Terminology Service

### 3.1 "Masters"
**Definition**: Static data that rarely changes but drives the transaction logic.
- **Location Master**: A branch office (e.g., "Mumbai HQ", "Pune Branch"). Data is isolated by Location.
- **Vehicle Master**: The fleet inventory (Reg Number, Chassis No, Insurance Expiry).
- **Driver Master**: The workforce (License No, Bank Details for salary).

### 3.2 "TF" (Transfer) vs "Local" vs "Out Station"
These are **Duty Types** which determine the billing formula in `billAmounts.js`:
- **Local**: Billed by Hours & Kilometers (e.g., "8 Hours / 80 Km package"). Extra hours/km are charged at a rate.
- **Out Station**: Long-distance travel. Billed by *Days* (Minimum 300km/day).
- **TF (Transfer)**: Fixed point-to-point drop (e.g., "Airport Drop"). Fixed price, regardless of time/traffic.

### 3.3 "Night Count" or "Night Battha"
**Definition**: An extra allowance paid to drivers (and charged to clients) if driving occurs between specific hours (e.g., 10 PM - 6 AM).
**Code Logic**: `calculateNightCount()` in `helpers/DateTime.js`.
- It checks if the duty interval overlaps with the client-specific "Night Window".
- If `Start: 9 PM`, `End: 11 PM`, and `Night Start: 10 PM`, then Night Count = 1.

### 3.4 "Dry Run" / "Garage to Garage"
**Definition**: The distance the taxi travels *before* picking up the guest (Garage -> Guest) and *after* dropping them (Guest -> Garage).
**Business Impact**: Some clients agree to pay for this ("Garage to Garage" billing), others only pay from pickup ("Point to Point" billing).

## 4. Financial Terminology

### 4.1 "Unbilled Slips"
**Definition**: Duties that have been completed (Driver has returned) but haven't been added to an Invoice yet.
**UI**: `views/duty/billed/Unbilled.js`. This is a "To-Do List" for the Accounts team.

### 4.2 "Bill Cover" or "Cover Note"
**Definition**: A summary sheet attached to a physical bundle of invoices.
**Usage**: Large corporates (like Tata/Google) might receive 50 invoices in a month. The "Cover Note" lists them all: "Enclosed 50 invoices totaling ₹5,00,000".

### 4.3 "GST Settings" (Inter/Intra State)
**Definition**: Tax rules based on geography.
- **CGST + SGST**: Charged if the Taxi Operator and Client are in the same state (e.g., MH to MH).
- **IGST**: Charged if they are in different states.
**Code**: `calculateGSTAmounts` in `billAmounts.js` automatically applies this logic based on `client.state`.

### 4.4 "Driver Battha"
**Definition**: Daily allowance for food/expenses given to the driver, distinct from Salary.

## 5. System Fields & Flags

### 5.1 `forceHalf` / `forceFull`
**Definition**: Manual overrides for "Local" duties.
- Even if a car was used for 2 hours, the operator might `forceFull` to charge the "8hr/80km" full day package rate.

### 5.2 `parkingCharge`
**Definition**: Reimbursable expense. Driver pays for parking at the airport -> adds it to the Duty Slip -> Client is billed for it. It is usually **Non-Taxable** (GST is not applied on top of the parking ticket reimbursement).

### 5.3 `taxiRefNo`
**Definition**: An external reference. Usually the Client's "PO Number" or the Guest's "Flight Number". Essential for the invoice to be approved by the client's finance team.

## 6. Summary
Safar Manager is a "Compliance First" system. It tracks not just *where* the car went, but *how* it should be billed (TF vs Local), *what* taxes apply (IGST vs CGST), and *who* did the work (Driver Battha).
