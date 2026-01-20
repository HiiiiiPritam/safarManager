# Low Level Design: Auth, Security & Infrastructure

## 1. Overview
The system uses a **Stateless** authentication mechanism (JWT) combined with a **Stateful** authorization context (DB check on every request).
It supports **Multi-Tenancy** (Shared Database, Isolated Data) via `Location` scoping.

## 2. Core Components

### 2.1 The Auth Middleware (`middleware/auth.js`)
**The Gatekeeper**:
1.  **Token Extraction**: Reads `Authorization: Bearer <token>` header.
2.  **Verification**: decoding via `jwt.verify(token, secret)`.
3.  **Context Loading (The "Secret Sauce")**:
    *   Once the `User ID` is decoded, the middleware fetches the User from DB.
    *   **Crucial Step**: It *also* finds the `Location` linked to this User.
    *   It attaches `req.user.location` to the request object.
    *   **Result**: Every controller downstream (Booking, Billing) doesn't need to ask "Who is logged in?". It just uses `req.user.location` to filter data.

### 2.2 Role-Based Access Control (RBAC)
**Implementation**:
*   `authorizeRoles('super_admin')` wrapper.
*   **Logic**:
    ```javascript
    if (!roles.includes(req.user.role)) {
       return next(new ErrorHandler("Role not allowed", 403));
    }
    ```
*   **Roles**:
    *   `Super Admin`: System Owner. Can delete Users.
    *   `Admin`: Business Owner. Manage their own Location.
    *   *Future*: `Driver`, `Employee` (Partial implementations).

### 2.3 Cloudinary Integration (`helpers/cloudinary.js`)
**Problem**: Storing user uploads (Logos, Signatures).
**Design**:
*   We do *not* store files on the server disk (Ephemeral file system).
*   **Flow**:
    1.  Frontend sends Base64 / Multipart.
    2.  Backend streams to Cloudinary API.
    3.  Cloudinary returns a URL (`https://res.cloudinary...`).
    4.  Backend stores *only* the URL string in MongoDB.

## 3. Security Decisions
*   **Passwords**: Hashed with `bcryptjs`. Never stored in plaintext.
*   **JWT Expiry**: Tokens are short-lived.
*   **Data Isolation**:
    *   Every `find()` query in the system MUST include `{ location: req.user.location }`.
    *   This is not enforced at the DB level (Sharding) but at the **Application Logic** level (Controller code). If a developer forgets this filter, a data leak could occur (Bug risk).

## 4. Why this Design?
*   **Scalability**: Stateless JWTs allow us to scale the Node process horizontally without sticky sessions.
*   **Simplicity**: Logical separation of tenants (by ID) is easier to manage than Physical separation (separate DBs) for a small-to-medium scale ERP.

## 5. Potential Interview Questions
*   "How do you ensure Tenant A doesn't see Tenant B's data?" -> **Middleware injects Location ID, and Controllers apply it as a mandatory filter.**
*   "Where do you store images?" -> **Cloudinary (CDN), to keep the server stateless.**
