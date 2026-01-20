# Low Level Design: Master Data & Relations

## 1. Overview
The system relies on a strict hierarchy of Master Data to enforce business rules.
**Hierarchy**: `Company (Location)` -> `Client` -> `Quotation` -> `QuotationVehicle`.

This design solves a specific problem: **Contract Enforcement**. When an admin makes a booking, they shouldn't "guess" the price. The price should be dictated by the signed contract.

## 2. Core Components

### 2.1 Quotation Manager (`quotationManager.js`)
**The Structure**:
*   A `Quotation` is a container. It links a `Client` to a specific validity period (implied/versions).
*   It serves as the parent for `QuotationVehicles`.

### 2.2 Quotation Vehicle Manager (`quotationVehicleManager.js`)
**The "Menu" Logic**:
*   This model stores the **Atomic Price Unit**.
*   **Fields**: `VehicleType`, `DutyType` ("Local", "Outstation"), `PackagePrice`, `ExtraKmRate`.
*   **Data Integrity**:
    *   The `dutyType` is an Enum. This prevents typos like "Locl" or "Outstn".
    *   The database refuses to save a record if required financial fields are missing.

### 2.3 The Dropdown Cascade (Frontend Implementation)
One of the most complex parts of the system is the **Dependency Chain** in the "New Booking" form.
**Logic Flow**:
1.  **Select Client**: API calls `/quotation/client/:id`.
    *   *Backend*: Returns only the Active Quotation for this client.
2.  **Select Quotation**: (Auto-selected usually).
3.  **Select Vehicle**: API calls `/quotation-vehicle/:quotationId`.
    *   *Backend*: Returns *only* the vehicles defined in that contract.
    *   *Result*: If the contract with Google only includes "Sedans", the Admin *cannot* select "Bus". The dropdown will be empty.
4.  **Select Duty Type**: The available duty types are filtered based on the selected Vehicle's config in the Quotation.

## 3. Reference Handling (`models/*.js`)
*   **ObjectIds**: We strictly use Mongoose `ObjectId` refs (`ref: 'Client'`, `ref: 'Vehicle'`).
*   **Population**:
    *   Fetching a Duty requires deeply nested population: `Duty` -> `QuotationVehicle` -> `Vehicle`.
    *   To keep performance high, we index these foreign keys (`index: true` in schema).

## 4. Why this Design?
*   **Defensive Design**: By forcing data entry to flow through the Quotation, we eliminate "Human Price Entry Errors". An operator cannot charge ₹1100 if the contract says ₹1000.
*   **Flexibility**: A Client can have multiple active quotations (e.g., "Mumbai Contract" and "Delhi Contract"), handled by the Location scoping.

## 5. Potential Interview Questions
*   "Why is the data structure so deep (Duty -> Quotation -> Client)?" -> **To enforce strict contract pricing.**
*   "How do you handle price changes?" -> **We create a NEW Quotation version. Old prices remain linked to old duties.**
