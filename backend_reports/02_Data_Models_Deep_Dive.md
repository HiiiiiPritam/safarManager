# Core Data Models & Schema Design Report

## 1. Executive Summary
This report provides an in-depth analysis of the Data Layer of the Safar Manager application. The system uses MongoDB with Mongoose schemas to define rigorous data structures for Users, Clients, Drivers, Vehicles, and their interactions (Bookings, Duties, Bills). A key feature of the schema design is the heavy use of references (`ObjectId`) to strict schemas, ensuring relational-like data integrity within a NoSQL environment.

## 2. Core Entity Models

### 2.1 User Model (`userModel.js`)
The `User` model represents the system administrators and staff. It is the root of the authentication and authorization chain.

#### 2.1.1 Schema Definition Detail
| Field | Type | Required | Validation/Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `firstName` | `String` | **Yes** | - | User's given name. |
| `lastName` | `String` | No | - | User's surname. |
| `email` | `String` | **Yes** | `validator.isEmail`, Unique | Primary identifier for login. Uniqueness is enforced at the DB level. |
| `contactNumber` | `Number` | **Yes** | Unique, Custom Regex | Must match `^[1-9][0-9]{9}$`. Ensures valid 10-digit Indian mobile numbers. |
| `password` | `String` | - | `minLength: 6`, `select: false` | Hashed via `bcrypt`. Hidden by default to prevent leakage. |
| `avatar` | `Object` | No | `{ public_id, url }` | Cloudinary asset reference for profile picture. |
| `role` | `Enum` | No | Default: `admin` | Used by `authorizeRoles` middleware. |
| `status` | `Enum` | No | `active`, `inactive` | Allows banning/disabling users without deletion. |

#### 2.1.2 Methods & Hooks
- **`pre('save')`**: Automatically hashes the password if it has been modified. This creates a secure-by-default architecture where developers don't need to manually hash passwords in controllers.
- **`getJWTToken()`**: Instance method that encapsulates the `jwt.sign` logic, ensuring consistency in token expiration (`process.env.JWT_EXPIRE`) and payload structure.
- **`comparePassword()`**: Instance method for secure login verification.

### 2.2 Client Model (`clientModel.js`)
Represents the customers (Corporate or Individual) being served. This is a "heavy" model containing significant configuration data.

#### 2.2.1 Configuration Sub-Documents
**Night Settings (`nightSettings`)**:
Controls automated billing logic for night drives.
- `applyNight` (String): Default 'yes'.
- `nightStartTime` (String): Default '22:00'. Regex validated `HH:MM`.
- `nightEndTime` (String): Default '05:00'. Regex validated `HH:MM`.
*Implication*: The system can automatically flag a duty as a "Night Duty" if timestamps overlap this window.

**GST Settings (`GSTSettings`)**:
Critical for B2B invoicing.
- `GSTNumber`, `PANNumber`: Tax identifiers.
- `GSTState`: Reference to the `State` model.
- `cgst`, `sgst`, `igst`: Numeric fields (default 0).
*Implication*: Different clients can have different tax rules configured, allowing the system to handle exemptions or interstate supply logic.

#### 2.2.2 Addressing & Contact
- **Structure**: Nested `address` object keeps the root schema clean.
- **Fields**: `tempAddress`, `permAddress`, `pincode`, `contactPersonName`.
- **Validation**: Reuses the strict phone number and email validators seen in the User model.

### 2.3 Driver Model (`driverModel.js`)
Represents the fleet operators. The schema focuses on Compliance and HR data.

#### 2.3.1 Critical Fields
- **`idProof`**: Enum supporting `pan card`, `aadhar card`, `voter id`, `passport`, `driving license`. Ensures standardization of KYC documents.
- **`bankDetails`**: Embedded object (`accountNumber`, `bankName`, `IFSCCode`). Used for salary payouts.
- **`user`**: Reference to `User`. This links the driver to a specific admin or fleet manager.
- **`driverCode`**: Auto-incrementing field (using `mongoose-sequence`). Useful for physical file management or ID cards (e.g., Driver #105).

## 3. Operational Models Deep Dive

### 3.1 Booking Model (`bookingModel.js`)
The central entity of the system. It acts as the "Order" in an e-commerce context.

#### 3.1.1 Relationship Mapping
The model is a nexus of references:
- `location` -> `LocationModel` (Partitioning key)
- `client` -> `ClientModel` (Customer)
- `city` -> `CityModel` (Service Area)
- `quotation` -> `QuotationModel` (Pricing Context)
- `quotationVehicle` -> `QuotationVehicleModel` (Specific price tier)

#### 3.1.2 Status & Flags
- **`status`**: Enum [`confirm`, `dispatch`, `cancel`]. Default `confirm`.
  - *Workflow*: Created as `confirm`. Moved to `dispatch` when car is assigned? moved to `cancel` if client cancels.
- **`invoiceGenerated`**: Boolean flag. Critical for the billing "Inbox". The "Create Invoice" page likely queries `invoiceGenerated: false` bookings.
- **`invoiceNumber`**: String. Links back to the generated Bill.

#### 3.1.3 Guest vs Booker
The schema explicitly handles the "Executive Assistant" scenario:
- **Booker**: The admin/EA making the call (e.g., Jane Doe).
- **Guest**: The actual passenger (e.g., The CEO).
- Each has dedicated `name`, `email`, `contactNumber` fields. Indexes are placed on `guest.name` for quick lookup ("Where is Mr. CEO's cab?").

### 3.2 Duty Model (`dutyModel.js`)
Represents the "Execution" or "Fulfillment" of a Booking.

#### 3.2.1 Why separate Booking and Duty?
- **Booking** captures the *Intent* (Expected Start: 10:00).
- **Duty** captures the *Actuals* (Actual Start: 10:15, Actual End: 18:00).
- **Billing** often depends on Actuals (Extra KMs, Extra Hours).

#### 3.2.2 Validation Logic
- **Meter Logic**: `startMeter` and `endMeter` are mandatory numbers.
- **Pre-Save Hook**:
  ```javascript
  dutyModel.pre("save", function (next) {
    if (this.endMeter < this.startMeter) {
      return next(new Error("End meter must be greater than start meter"));
    }
    next();
  });
  ```
  This is a critical integrity check. It prevents "negative distance" bugs that would ruin billing calculations.

### 3.3 Bill/Invoice Model (`billModel.js`)
The financial aggregator.

#### 3.3.1 Structure
- **`bookings`**: Array of `ObjectId`s.
- **`amount`**: The finalized total value.
- **`location`**: Scopes the bill.

#### 3.3.2 Indexing Strategy
`billSchema.index({ location: 1, invoiceNumber: 1 }, { unique: true });`
This compound index is a best practice for multi-tenant apps.
- It allows "Invoice #001" to exist in Location A.
- And "Invoice #001" to exist in Location B.
- But prevents duplicate "Invoice #001" in Location A.

## 4. Auxiliary Models & Master Data

### 4.1 Master Data Models
- **`stateModel.js`, `cityModel.js`**: Standard master data.
- **`vehicleModel.js`**: Represents a *physical* asset (Car).
  - Fields: `registrationNumber`, `model`, `type` (owned/attached).
- **`locationModel.js`**: Represents the Branch.

### 4.2 Pricing Engine Models
- **`quotationModel.js`**: Represents a Contract or Rate Card.
- **`quotationVehicleModel.js`**: The granular pricing rows.
  - Linked to a Quotation.
  - Defines `dutyType` (Local/Outstation).
  - Defines `vehicleType` (Sedan/SUV).
  - Defines `baseRate`, `extraKmRate`, `extraHourRate`, `driverAllowance`.
  - *Use Case*: When a Booking is created, the system likely looks up the `QuotationVehicle` to estimate the cost.

## 5. Schema Design Patterns & Best Practices

### 5.1 Soft Deletion Strategy
Using `mongoose-delete` plugin globally.
- **Configuration**:
  ```javascript
  plugin(mongoose_delete, {
    deletedAt: true,
    overrideMethods: "all",
    indexFields: "all",
  });
  ```
- **Benefit**: "Deleting" a client doesn't break their historical Bookings or Bills. The references remain valid, but the user vanishes from lists. `overrideMethods: 'all'` ensures standard Mongoose queries respect this filter automatically.

### 5.2 Auto-Incrementing IDs
Using `mongoose-sequence`.
- **Usage**: `clientCode`, `bookingNumber`, `invoiceNumber`, `driverCode`.
- **Benefit**: User Friendly. "Booking #B-1025" is easier to discuss over the phone than "Booking #64b5f..."
- **Implementation**: The plugin manages a separate `counters` collection to maintain atomicity and prevent duplicates even under load.

### 5.3 Data Normalization vs Embedding
- **Heavy Normalization**: The schema prefers references (`ref`) over embedding.
  - *Example*: `Booking` refers to `Client`, `Vehicle`, `Quotation` by ID.
  - *Pros*: Single source of truth. Updating a Client's phone number reflects everywhere.
  - *Cons*: Read operations require `populate()`. As seen in `BookingManager`, fetching a booking listing requires 4-5 levels of lookup.

## 6. Conclusion
The Safar Manager database schema is robust and rigorous. It successfully translates complex real-world logistics (Duties, Night Allowances, GST, Contracts) into structured MongoDB documents. The use of pre-save hooks for validation and plugins for soft-deletes and sequences indicates a high level of engineered maturity, prioritizing data safety and auditing capabilities essential for a financial/ERP system.
