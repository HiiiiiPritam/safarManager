# System Architecture & Infrastructure Report

## 1. Executive Summary
This report details the architectural foundation of the "Safar Manager" backend system. The application is a RESTful API built using Node.js and Express.js, designed to manage taxi fleet operations. It employs a layered architecture separating concerns into Routes, Controllers, Managers (Business Logic), and Models (Data Access). The system utilizes MongoDB as its primary data store, leveraged via the Mongoose ODM, and integrates with Cloudinary for media management and Node-Cron for scheduled tasks (likely financial reporting or cleanup).

## 2. Project Structure & Organization
The codebase follows a standard, scalable Node.js project structure, promoting modularity and maintainability.

### 2.1 Root Entry Points
- **`index.js` (The Bootstrapper)**: 
  - **Role**: Entry point for the Node process.
  - **Responsibilities**:
    - Loads environment variables (`dotenv`).
    - Connects to Database (`connectDatabase`).
    - Initializes Cron Jobs (`initCronJobs`).
    - Configures Cloudinary (Asset Management).
    - Starts the Express server.
    - **Safety Nets**: Implements global handlers for `uncaughtException` and `unhandledRejection`. This ensures that if the process enters an unstable state (e.g. memory leak, broken promise chain), it crashes fast rather than hanging in a zombie state (which process managers like PM2 would then restart).

- **`app.js` (The Application Configuration)**: 
  - **Role**: Configures the Express app instance.
  - **Middleware Pipeline**:
    1.  `morgan`: Request logging.
    2.  `bodyParser`: JSON and URL-encoded parsing (10mb limit for large uploads).
    3.  `cookieParser`: Auth token extraction.
    4.  `express-fileupload`: handling multipart/form-data.
    5.  `cors`: Cross-Origin Resource Sharing.
  - **Route Mounting**: Mounts API routes at `/api/v1`.
  - **Static Serving**: Serves the React frontend build from `../safar_manager_frontend/build`, enabling a monolithic deployment strategy (Frontend + Backend on one port).

### 2.2 Directory Breakdown Analysis
- **`routes/`**: Defines the HTTP endpoints. Each file maps a resource URL (e.g., `/bookings`) to a Controller method. It is the only layer aware of HTTP verbs (GET, POST).
- **`controllers/`**: The "Traffic Cop". It:
  - Extracts data from `req.body` / `req.params`.
  - Calls the relevant `Manager`.
  - Handles the response (`res.status().json()`).
  - Catches errors via `catchAsyncErrors`.
- **`managers/`**: The "Brain". Contains pure business logic.
  - It does not know about `req` or `res`.
  - It works with Data and Models.
  - This separation makes unit testing logic easier (you don't need to mock Express objects).
- **`models/`**: The "Schema". Defines Mongoose schemas, validation rules, and middleware hooks.
- **`middleware/`**: Cross-cutting concerns.
  - `auth.js`: Authenticates users via JWT.
  - `error.js`: Centralized error formatter.
- **`utils/`**: Shared tools.
  - `ApiFeatures.js`: A specialized class for handling "Search", "Filter", and "Pagination" logic on Mongoose queries.

## 3. Technology Stack & Dependencies

### 3.1 Core Framework
- **Express.js**: The de facto standard server framework.
- **Validation**:
  - `validator`: Used for string validation (email, etc).
  - Custom Mongoose validators (Regex for phone, time).

### 3.2 Database Layer
- **MongoDB & Mongoose**:
  - **Why NoSQL?** Taxi data often dictates flexible schemas (different vehicle attributes, varying contract terms).
  - **Plugins Used**:
    - **`mongoose-delete`**: Implements "Soft Delete". This is crucial for audit trails. A deleted booking is hidden, not destroyed.
    - **`mongoose-sequence`**: Provides auto-incrementing IDs (Booking #101, Invoice #500). Business users prefer these over UUIDs/ObjectIDs.

### 3.3 Authentication & Security
- **JWT (Stateless Auth)**:
  - Users exchange credentials for a signed Token.
  - Token contains `id`.
  - **Verification**: Middleware decodes token, fetches User, and *critically*, fetches the User's `Location`.
- **Multi-Tenancy Setup**:
  - The `auth.js` middleware attaches `req.user.location` to the request object.
  - This implies the system handles multiple branches/franchises.
  - Data isolation is enforced at the Application level (every query usually includes `location: req.user.location`).

## 4. Middleware & Error Handling Architecture

### 4.1 The Error Handling Pipeline
The application uses a robust, centralized error handling strategy (`middleware/error.js`).
- **`ErrorHandler` Class**: Extends the native JS `Error` object to include `statusCode`.
- **Usage**: `throw new ErrorHandler("Booking not found", 404)`.
- **Catch Async Wrapper**:
  ```javascript
  module.exports = (func) => (req, res, next) =>
    Promise.resolve(func(req, res, next)).catch(next);
  ```
  - This high-order function wraps every controller action.
  - If a Promise rejects (DB error), it passes it to `next(err)`.
  - This eliminates generic `try/catch` boilerplate in Controllers.

### 4.2 Application Middleware
- **CORS Config**: Essential for separating Frontend/Backend development.
- **File Upload**: Using `express-fileupload` allows handling driver documents (Licenses, ID Proofs) directly in memory/temp storage before uploading to Cloudinary.

## 5. Architectural Patterns Observations

### 5.1 The Manager Pattern Over Services
The project uses `Managers` (e.g., `BillManager`, `BookingManager`).
- **Characteristics**:
  - They often hold state (`this.bookings`, `this.location`).
  - They encapsulate complex transactions.
  - `BookingManager.save()` handles the complexity of Sequence Counters + DB Insert + Transaction management.
- **Benefit**: Keeps Controllers "Skinny". The Controller just says "Create Booking", the Manager knows *how*.

### 5.2 API Features Utility (`ApiFeatures`)
This class standardizes listing APIs.
- **Search**: `keyword=john` -> matches fields via Regex.
- **Filter**: `price[gt]=1000` -> Mongoose `$gt` query.
- **Pagination**: `page=2&limit=10` -> `skip(10).limit(10)`.
- **Reuse**: Used in `getAllDrivers`, `getAllBookings`, keeping API behavior consistent.

## 6. Infrastructure & Deployment
- **Environment config**: `.env` driven.
- **Statelessness**: The API is stateless (JWT). Scaling can be done horizontally (adding more Node processes), provided the MongoDB cluster and Cloudinary (Media) are external and scalable.
- **Cron Jobs**: `utils/cronJobs.js` suggests background processing. Likely used for:
  - Monthly Invoice generation?
  - Reminder notifications?
  - Cleaning up stale data?

## 7. Conclusion
The `backendTaxi` project follows a mature, structured approach to Node.js development. It matches industry standards for REST APIs. The architecture prioritizes:
1.  **Safety**: Transactions, Error Wrappers.
2.  **Scalability**: Stateless Auth, Modular Code.
3.  **Maintainability**: Clear separation of `Route` -> `Controller` -> `Manager` -> `Model`.
It is well-suited to serve as the backend for a production-grade Taxi Management SaaS.
