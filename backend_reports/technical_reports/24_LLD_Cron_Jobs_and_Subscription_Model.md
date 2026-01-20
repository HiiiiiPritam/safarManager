# Low Level Design: Cron Jobs & Subscription Model

## 1. Overview
The **Subscription Model** forces users (Admins) to pay a monthly fee to usage the platform.
It uses a **Hybrid Manual/Automated Implementation** involving backend schedulers and frontend locks.

## 2. Core Components

### 2.1 The Scheduler (`utils/cronJobs.js`)
**Technology**: `node-cron`.
**Schedule**: `1 0 1 * *` (1st of Month, 00:01 AM).
**Logic**:
1.  Wakes up.
2.  Queries `User.find({ role: 'admin' })`.
3.  Loops through every admin.
4.  Creates a record in `UserPayment` collection:
    *   `Month`: Previous Month (e.g., "January").
    *   `Status`: `PENDING`.
    *   `Amount`: Fixed Constant (`USER_PAYMENT_AMOUNT`).
5.  **Output**: Quietly populates the DB with "Bills" for every user.

### 2.2 The Frontend "Guard" (`routes/protectedRoutes.js`)
**Logic**:
1.  **Global Check**: This component wraps *all* dashboard routes.
2.  **The Fetch**: On load, Redux fetches `getUserPayments`.
3.  **The Logic**: `checkPastDuePayments(payments)`:
    *   Is there a `PENDING` payment?
    *   Is today > 15th of the month? (Grace Period).
4.  **The Action**:
    *   If **YES** (Overdue): Redirect to `/dashboard-home`. Hide Sidebar. Show Red Banner.
    *   If **NO**: Allow normal navigation.

### 2.3 The Manual Unlock Workflow (`UserPayments.js`)
**Problem**: We don't have a payment gateway (Stripe/Razorpay) integrated yet.
**Solution**: "Trust-based" workflow.
1.  User clicks "Pay".
2.  Modal shows a Static QR Code image (`QR_image`).
3.  User pays offline via UPI.
4.  User emails Admin.
5.  **Super Admin** manually updates the Database record to `COMPLETED`.
6.  User refreshes page -> Frontend Guard sees `COMPLETED` -> Unlocks.

## 3. Why this Design?
*   **Low Complexity**: Integrating a real Payment Gateway requires KYC, webhooks, and complex failure handling. This "Manual" approach allows the business to launch immediately with zero code integration.
*   **Grace Period**: Hard-coded logic (15 days) prevents users from being locked out immediately on the 1st, giving them time to process manual payments.

## 4. Potential Interview Questions
*   "How does the system lock non-paying users?" -> **Frontend Route Guards checking backend Payment Status.**
*   "Is the cron job reliable?" -> **Currently, it runs only if the server is UP at 00:01. Ideally, it should be replaced by a Job Queue (BullMQ) for reliability.**
