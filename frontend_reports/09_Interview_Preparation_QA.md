# Technical Interview Preparation Q&A

## 1. Executive Summary
This document anticipates technical interview questions based on the Safar Manager stack and architecture. It provides "Model Answers" derived directly from the code analysis.

## 2. Architecture & Design Questions

**Q1: Can you explain the high-level architecture of your application?**
*Answer*: "It is a Monolithic Client-Server architecture. The Frontend is a React SPA built with Vite and uses CoreUI. The Backend is a Node.js/Express REST API using MongoDB. We follow a layered backend architecture comprising Routes, Controllers, Managers (Business Logic), and Models. The frontend and backend are decoupled but deployed as a unit, where the Express server serves the React build static files."

**Q2: Why did you choose the 'Manager' pattern in the backend instead of just Controllers?**
*Answer*: "Controllers should only handle HTTP concerns (Parsing requests, sending responses). By moving logic to Manager classes (like `BookingManager`), we achieved two things:
1. **Reusability**: I can call `BookingManager.create()` from a Controller OR from a Cron Job without duplicating code.
2. **Testability**: I can write unit tests for the Manager logic without mocking `req` and `res` objects."

**Q3: How do you handle Multi-Tenancy (Multiple Branches)?**
*Answer*: "We handle it at the application layer. Every User is linked to a `Location`. In the backend middleware (`auth.js`), we attach `req.user.location`. Then, purely by discipline, every database query includes `{ location: req.user.location }`. This ensures strict data isolation between branches."

## 3. Frontend Specific Questions

**Q: How do you manage State in the application?**
*Answer*: "We use Redux with Thunk middleware. We chose Redux because the application is 'Master Data Heavy'. We need to share lists of Cities, Drivers, and Vehicles across many different forms (Booking form, Duty form, Expense form). Redux allows us to fetch this data once and cache it. We also use `redux-persist` to keep the user's session active across browser refreshes."

**Q: How is the Folder Structure organized?**
*Answer*: "It's feature-based within technical layers. We have `views/` for pages, `components/` for reusable widgets (like our `SelectBox` wrapper), and `actions/` for API logic. This separation allows frontend developers to focus on UI without worrying about API implementation details."

## 4. Backend & Database Questions

**Q: You are using MongoDB. How do you handle relationships (like Booking -> Client)?**
*Answer*: "We use Mongoose `ObjectId` references. However, looking back, we identified some **N+1 Query issues** in our Billing API. Currently, to show a list of Bills with Client names, we loop through bills and fetch client details individually. In a future refactor, I would denormalize data by storing the `ClientName` snapshot directly in the `Bill` document to optimize read performance."

**Q: How do you handle Transactions?**
*Answer*: "Financial integrity is critical. When we generate an Invoice, we use Mongoose Sessions. We start a transaction, create the Bill record, update the associated Booking records to flag them as 'Billed', and then commit. This ensures we never have a 'Ghost Bill' (Bill created but bookings not marked) or 'Double Billing' (Bookings marked but Bill failed)."

**Q: How do you ensure unique Invoice Numbers?**
*Answer*: "We use the `mongoose-sequence` plugin. It manages a separate counter collection. This gives us atomic, sequential numbers (INV-1001, INV-1002) which are legally required for tax invoices, rather than random UUIDs."

## 5. Security Questions

**Q: How is Authentication handled?**
*Answer*: "We use stateless JWT authentication. The user logs in, receives a token, and stores it in LocalStorage. An Axios interceptor attaches this token to the `Authorization` header of every request. On the backend, middleware verifies the signature and checks if the user is still active."

**Q: How do you secure user passwords?**
*Answer*: "We use `bcrypt` to hash passwords before saving. Crucially, in our User Schema, we set `select: false` on the password field. This is a safety mechanism that ensures even if a developer writes `User.find()`, the password hash is never accidentally sent to the frontend API."

## 6. Challenges & Learnings
*Use this for "Tell me about a challenge you faced"*

"One challenge was the complex pricing logic. A contract can be 'Flat Rate', 'Per Km', or 'Per Hour', and can differ by Vehicle Type (Sedan vs SUV). We solved this by creating a granular `Quotation` and `QuotationVehicle` schema modification. The frontend Booking form dynamically fetches these linked quotations to ensure the operator can only book valid, agreed-upon rates."
