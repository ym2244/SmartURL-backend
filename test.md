This document outlines the database schema for the SmartUrl service, including the new table for password reset tokens.

## Database Tables

### Table: users

| Column             | Type      | Constraints      | Description                                                        |
| ------------------ | --------- | ---------------- | ------------------------------------------------------------------ |
| id                 | SERIAL    | PRIMARY KEY      | Unique identifier for each user                                    |
| username           | TEXT      | UNIQUE, NOT NULL | User's chosen username                                             |
| email              | TEXT      | UNIQUE, NOT NULL | User's email address                                               |
| password\_hash     | TEXT      |                  | Hashed password (NULL for OAuth users)                             |
| auth\_provider     | TEXT      |                  | Authentication provider (NULL for local auth, "google" for Google) |
| auth\_provider\_id | TEXT      |                  | Provider-specific user ID                                          |
| created\_at        | TIMESTAMP | NOT NULL         | When the user was created                                          |
| updated\_at        | TIMESTAMP | NOT NULL         | When the user was last updated                                     |

**Indexes:**

* PRIMARY KEY on `id`
* UNIQUE INDEX on `username`
* UNIQUE INDEX on `email`
* INDEX on `auth_provider` and `auth_provider_id`

**SQL for creating the table:**

```sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username TEXT UNIQUE NOT NULL,
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,
    auth_provider TEXT,
    auth_provider_id TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_auth_provider ON users (auth_provider, auth_provider_id);
```

---

### Table: password\_reset\_tokens

| Column      | Type      | Constraints                                     | Description                                   |
| ----------- | --------- | ----------------------------------------------- | --------------------------------------------- |
| id          | SERIAL    | PRIMARY KEY                                     | Unique identifier for each reset token record |
| user\_id    | INTEGER   | NOT NULL REFERENCES users(id) ON DELETE CASCADE | User requesting the reset                     |
| token       | TEXT      | NOT NULL UNIQUE                                 | Secure random token                           |
| expires\_at | TIMESTAMP | NOT NULL                                        | Expiration timestamp for this token           |
| used        | BOOLEAN   | NOT NULL DEFAULT FALSE                          | Whether the token has been used               |
| created\_at | TIMESTAMP | NOT NULL DEFAULT CURRENT\_TIMESTAMP             | When the token record was created             |

**Indexes:**

* INDEX on `user_id`
* INDEX on `expires_at`

**SQL for creating the table:**

```sql
CREATE TABLE IF NOT EXISTS password_reset_tokens (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token TEXT NOT NULL UNIQUE,
    expires_at TIMESTAMP NOT NULL,
    used BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX IF NOT EXISTS idx_prt_user ON password_reset_tokens (user_id);
CREATE INDEX IF NOT EXISTS idx_prt_expires ON password_reset_tokens (expires_at);
```

---

### Table: urls

| Column        | Type      | Constraints                         | Description                                            |
| ------------- | --------- | ----------------------------------- | ------------------------------------------------------ |
| id            | SERIAL    | PRIMARY KEY                         | Unique identifier for each URL entry                   |
| user\_id      | INTEGER   | FOREIGN KEY NULL                    | ID of user who created this URL (NULL if user deleted) |
| original\_url | TEXT      | NOT NULL                            | The original long URL                                  |
| short\_code   | TEXT      | UNIQUE, NOT NULL                    | The randomly generated code                            |
| alias         | TEXT      | UNIQUE                              | Optional custom alias (unique across service)          |
| title         | TEXT      |                                     | Title of the website (extracted from HTML)             |
| clicks        | INTEGER   | DEFAULT 0                           | Number of times the short URL has been accessed        |
| created\_at   | TIMESTAMP | NOT NULL DEFAULT CURRENT\_TIMESTAMP | When the short URL was created                         |

**Indexes:**

* UNIQUE INDEX on `short_code`
* UNIQUE INDEX on `alias`
* INDEX on `user_id`
* INDEX on `short_code`

---

### Table: qrcodes

| Column        | Type      | Constraints                         | Description                                                |
| ------------- | --------- | ----------------------------------- | ---------------------------------------------------------- |
| id            | SERIAL    | PRIMARY KEY                         | Unique identifier for each QR code entry                   |
| user\_id      | INTEGER   | FOREIGN KEY NULL                    | ID of user who created this QR code (NULL if user deleted) |
| original\_url | TEXT      | NOT NULL                            | The original URL encoded in the QR code                    |
| qr\_code\_id  | TEXT      | UNIQUE, NOT NULL                    | The unique identifier for the QR code                      |
| title         | TEXT      |                                     | Title of the website (extracted from HTML)                 |
| scans         | INTEGER   | DEFAULT 0                           | Number of times the QR code has been scanned               |
| created\_at   | TIMESTAMP | NOT NULL DEFAULT CURRENT\_TIMESTAMP | When the QR code was created                               |

**Indexes:**

* UNIQUE INDEX on `qr_code_id`
* INDEX on `user_id`
* INDEX on `qr_code_id`

---

### Table: barcodes

| Column        | Type      | Constraints                         | Description                                                |
| ------------- | --------- | ----------------------------------- | ---------------------------------------------------------- |
| id            | SERIAL    | PRIMARY KEY                         | Unique identifier for each barcode entry                   |
| user\_id      | INTEGER   | FOREIGN KEY NULL                    | ID of user who created this barcode (NULL if user deleted) |
| original\_url | TEXT      | NOT NULL                            | The original URL encoded in the barcode                    |
| barcode\_id   | TEXT      | UNIQUE, NOT NULL                    | The unique identifier for the barcode                      |
| title         | TEXT      |                                     | Title of the website (extracted from HTML)                 |
| scans         | INTEGER   | DEFAULT 0                           | Number of times the barcode has been scanned               |
| created\_at   | TIMESTAMP | NOT NULL DEFAULT CURRENT\_TIMESTAMP | When the barcode was created                               |

**Indexes:**

* UNIQUE INDEX on `barcode_id`
* INDEX on `user_id`
* INDEX on `barcode_id`

---

### Table: analytics

| Column         | Type      | Constraints                         | Description                                                 |
| -------------- | --------- | ----------------------------------- | ----------------------------------------------------------- |
| id             | SERIAL    | PRIMARY KEY                         | Unique identifier for each analytics entry                  |
| resource\_type | TEXT      | NOT NULL                            | Type of resource ("url", "qrcode", "barcode")               |
| resource\_id   | TEXT      | NOT NULL                            | ID of the resource (short\_code, qr\_code\_id, barcode\_id) |
| event\_type    | TEXT      | NOT NULL                            | Type of event ("click" or "scan")                           |
| event\_date    | TIMESTAMP | NOT NULL DEFAULT CURRENT\_TIMESTAMP | Date and time of the event                                  |
| referrer       | TEXT      |                                     | Source of the visit (for URLs)                              |
| browser        | TEXT      |                                     | Browser used (Chrome, Firefox, etc.)                        |
| os             | TEXT      |                                     | Operating system                                            |
| device\_type   | TEXT      |                                     | Device type (desktop, mobile, tablet)                       |
| country        | TEXT      |                                     | Country of the visitor                                      |
| region         | TEXT      |                                     | Region or state                                             |
| city           | TEXT      |                                     | City of the visitor                                         |
| language       | TEXT      |                                     | Language setting of the visitor's browser                   |

**Indexes:**

* INDEX on `(resource_type, resource_id)`
* INDEX on `event_date`
* INDEX on `device_type`
* INDEX on `country`

## Key Relationships

**Relationship 1: User-to-Resource**

* Type: One-to-Many relationship
* Description: A single user can create multiple resources (URLs, QR codes, barcodes)
* Implementation: Through `user_id` foreign key in resource tables
* Creation requirement: All resources must be created by authenticated users (enforced at application level)
* Deletion behavior: When a user is deleted, resources remain but user association is removed (SET NULL)
* Resource persistence: All URLs, QR codes, and barcodes continue to function normally even after the creating user is deleted

**Relationship 2: Resource-to-Analytics**

* Type: One-to-Many relationship
* Description: Each resource (URL, QR code, barcode) can have multiple analytics entries
* Implementation: Through `resource_type` and `resource_id` fields in the analytics table
* Event tracking: Each row in the analytics table represents a single interaction event (one click or scan)
* Data captured: The analytics table stores detailed information about each interaction with a resource

**Relationship 3: User-to-PasswordResetTokens**

* Type: One-to-Many relationship
* Description: A single user may request multiple password reset tokens
* Implementation: Through `user_id` foreign key in `password_reset_tokens` table
* Token lifecycle: Each token record tracks creation time (`created_at`), expiry (`expires_at`), and usage status (`used`)
* Deletion behavior: When a user is deleted, all their related password reset tokens are also removed (ON DELETE CASCADE)

## Notes on Implementation

1. **QR Codes and Barcodes Generation**

   * The actual QR code and barcode images are generated on-demand
   * No image data is stored in the database, only the metadata
   * Images are created when requested through the image endpoints

2. **Authentication**

   * Supports both local authentication (username/password) and OAuth (Google)
   * Passwords are stored as secure bcrypt hashes, never as plaintext
   * JWT authentication is used instead of sessions

3. **Resource Persistence**

   * All resources require user authentication to create
   * Resources persist even if the creating user's account is deleted
   * This ensures that shared links and codes remain functional

4. **Website Title Extraction**

   * When a resource is created, the system attempts to extract the title from the target website's HTML
   * If extraction fails (due to network issues, invalid URL, or missing title tag), the title field may be NULL
   * Titles provide context about the destination without requiring manual entry

5. **Database Indexing**

   * Indexes are used to improve query performance, especially for lookups and analytics
   * Compound indexes like `idx_resource` optimize common query patterns
   * Each analytics event (click or scan) is stored as a separate row for detailed analysis

6. **Password Management Flows**

   * **Forgot Password (Not Logged In):**

     * Generate a secure `token` in `password_reset_tokens`, email a reset link containing this token
     * On confirmation, verify `token`, `expires_at > now()`, and `used = false`, then update password and set `used = true`
   * **Change Password (Logged In):**

     * Users provide `current_password` and `new_password` via `/users/me/password`, then password is updated immediately without email verification
