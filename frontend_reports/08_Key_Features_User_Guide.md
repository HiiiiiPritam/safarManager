# Key Features & Domain Guide

## 1. Executive Summary
This document acts as a "User Manual" for the Safar Manager application, translating the code modules into business features. It explains the domain terminology (Masters, Duties, Cover Notes) essential for explaining the project in a business context.

## 2. Module 1: Master Data Management (The Foundation)
Before any operation can happen, the "Masters" must be configured. This is the setup phase.
- **Location Master**: Defines the branches (e.g., "Main Office", "Airport Branch"). All operations are scoped to a location.
- **Client Master**: The customers.
  - *Key Feature*: **Night Settings**. You can define custom "Night Hours" (e.g., 10 PM to 6 AM) per client. The system uses this to automatically add night allowances to bills.
  - *Key Feature*: **GST Config**. Tax settings are granular per client (Inter-state vs Intra-state).
- **Vehicle Master**: The fleet inventory.
- **Driver Master**: The workforce. Includes bank details for salary processing.
- **Quotation Master (Rate Cards)**: The most complex master. It defines the "Price List" for a client.
  - *Structure*: Client -> Quotation -> QuotationVehicle (Sedan @ 1000rs/8hr, SUV @ 1500rs/8hr).

## 3. Module 2: Booking Management (The Promise)
This module handles upcoming demand.
- **Contract-Based Booking**: When you select a client, the system only shows Vehicles/Packages that exist in their Rate Card (Quotation). This prevents mis-selling.
- **Guest vs Booker**: Distinct fields allow tracking "Who called" (The Secretary) vs "Who rode" (The Boss).
- **Dispatch**: Bookings start as "Confirmed". When a driver acts on it, it moves to "Dispatch".

## 4. Module 3: Duty Management (The Execution)
This is the operational heart. A "Duty Slip" is the digital version of the paper logbook driver's carry.
- **Meter Tracking**: Captures Opening and Closing Odometer readings.
- **Validation**: System prevents entering a Closing Meter lower than the Opening Meter.
- **Calculation**: Automatically calculates `Total Km` and `Total Hours`.
- **Slips**:
  - **Unbilled Slips**: Duties completed but not yet invoiced.
  - **Billed Slips**: Closed loops.

## 5. Module 4: Billing & Invoicing (The Revenue)
The system supports two major billing workflows:
1.  **Regular Billing**: Select specific duty slips and invoice them.
2.  **Monthly Billing**: A bulk process for corporate clients who pay net-30 or net-60.
- **Features**:
  - **Bill Cover**: A summary page often attached to a bundle of invoices sent to a big client.
  - **Manual vs Auto Invoice Numbers**: The system allows overriding the invoice number sequence for historical data entry, but defaults to auto-increment for compliance.
  - **PDF Generation**: Immediate generation of professional invoices with tax breakdowns.

## 6. Module 5: Financial Transactions (The Ledger)
- **Client Accounts**: A ledger view showing "Total Billed" vs "Total Received".
- **Fuel Expenses**: Tracks fuel consumption. Allows calculating "Mileage" (Km run / Litres consumed) per vehicle to detect theft or inefficiency.
- **Driver Salaries**: Records payouts.
- **Vehicle Maintenance**: Tracks repair costs.
- **Profitability**: By comparing Invoices (Revenue) vs (Fuel + Salary + Maintenance), the operator can see the business health.

## 7. Module 6: Administration
- **User Management**: Creating sub-admins.
- **Role Based Access**:
  - **Super Admin**: Can see everything, manage subscriptions.
  - **Admin**: Restricted to their Location.
- **Location Switching**: A User can be mapped to a Location. This strictly partitions the data.

## 8. Summary for Interview
"Safar Manager is not just a booking app; it's a specialized ERP. It handles the specific complexity of Indian Taxi Operations, such as Night Allowances, GST states, specific Corporate Rate Cards, and split invoicing. It closes the loop from 'Booking Request' to 'Cash in Bank'."
