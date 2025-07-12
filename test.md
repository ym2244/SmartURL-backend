### 1. User Management Endpoints

#### 1.8. Request Password Reset

**Purpose:** Start a password-reset flow by sending a reset link to the user’s email.

**Endpoint:** `/auth/password-reset/request`

**Method:** `POST`

**Request Body:**

```json
{
  "email": "user@example.com",
  "username": "alice"      // optional, for extra verification
}
```

**Response:** (200 OK)

```json
{
  "message": "Password reset email sent."
}
```

**Example:**

* If the email corresponds to a registered user, you will receive:

  ```json
  { "message": "Password reset email sent." }
  ```
* If not registered, you may still receive the same response (to avoid user enumeration).

---

#### 1.9. Confirm Password Reset

**Purpose:** Complete password reset using the token sent by email.

**Endpoint:** `/auth/password-reset/confirm`

**Method:** `POST`

**Request Body:**

```json
{
  "token": "abcdef1234567890",
  "new_password": "NewP@ssw0rd!"
}
```

**Response:** (200 OK)

```json
{
  "message": "Password has been reset successfully."
}
```

**Example:**

* Valid token & password: returns success message.
* Invalid or expired token: `400 Bad Request` with:

  ```json
  { "error": { "code": "INVALID_TOKEN", "message": "Token is invalid or expired." } }
  ```

---

#### 1.10. Change Password (Logged-in User)

**Purpose:** Allow a signed-in user to update their password.

**Endpoint:** `/users/me/password`

**Method:** `PATCH`

**Headers:**

* `Authorization: Bearer {access_token}`

**Request Body:**

```json
{
  "current_password": "OldP@ss1",
  "new_password": "NewP@ssw0rd!"
}
```

**Response:** (200 OK)

```json
{
  "message": "Password changed successfully."
}
```

**Example:**

* Correct current password: success.
* Wrong current password: `403 Forbidden` with:

  ```json
  { "error": { "code": "INVALID_CREDENTIALS", "message": "Current password is incorrect." } }
  ```

---

### 2. Short URL Endpoints

#### 2.1. Create Short URL

**Purpose:** Convert a long URL into a short URL with a randomly generated code—or with a user-supplied alias. Automatically extracts website title.

**Endpoint:** `/shorten`

**Method:** `POST`

**Headers:**

* `Authorization: Bearer {access_token}` (optional, for authenticated users)

**Request Body:**

```json
{
  "target_url": "https://example.com/your/long/url",
  "alias": "your-custom-alias"    // optional: custom alias (3–30 chars, letters/numbers/_/-)
}
```

**Response:** (201 Created)

```json
{
  "original_url": "https://example.com/your/long/url",
  "short_code": "abc123",
  "alias": "your-custom-alias",   // present if alias provided
  "short_url": "https://your-domain.com/your-custom-alias",
  "title": "Example Website Homepage",
  "clicks": 0,
  "user_id": 42,
  "created_at": "2025-07-12T00:00:00Z"
}
```

**Examples:**

* **Without alias:**

  ```bash
  POST /shorten
  { "target_url": "https://example.com" }
  ```

  returns `201 Created` with random `short_code` and no `alias` field.

* **With alias:**

  ```bash
  POST /shorten
  { "target_url": "https://example.com", "alias": "promo" }
  ```

  returns `201 Created` with `alias: "promo"` and `short_url: "https://your-domain.com/promo"`.

**Error Responses:**

* 400 Bad Request if `alias` or `target_url` is invalid.
* 409 Conflict if `alias` is already taken.

---

#### 2.3. Set or Update Alias

**Purpose:** Assign or change a custom alias for a short code. Aliases can be set either at creation time (see Create Short URL) or updated later using this endpoint.

**Endpoint:** `/urls/{shortCode}`

**Method:** `PATCH`

**Headers:**

* `Authorization: Bearer {access_token}`

**Request Body:**

```json
{
  "alias": "your-custom-alias"
}
```

**Response:** (200 OK)

```json
{
  "original_url": "https://example.com/your/long/url",
  "short_code": "abc123",
  "alias": "your-custom-alias",
  "short_url": "https://your-domain.com/your-custom-alias",
  "clicks": 0,
  "user_id": 42,
  "created_at": "2025-07-12T00:00:00Z"
}
```

**Examples:**

* **Successful update:**

  ```bash
  PATCH /urls/abc123
  { "alias": "promo2025" }
  ```

  returns `200 OK` with new alias in response body.

* **Alias format invalid:**

  ```bash
  PATCH /urls/abc123
  { "alias": "!*invalid*" }
  ```

  returns `400 Bad Request`:

  ```json
  { "error": { "code": "INVALID_ALIAS_FORMAT", "message": "Alias must be 3–30 characters, letters/numbers/_/- only." } }
  ```

* **Alias already taken:**

  ```bash
  PATCH /urls/abc123
  { "alias": "existingAlias" }
  ```

  returns `409 Conflict`:

  ```json
  { "error": { "code": "ALIAS_TAKEN", "message": "Alias 'existingAlias' is already in use." } }
  ```

---

#### 2.4. Get Alias

**Purpose:** Retrieve the custom alias for a given short code, if one exists.

**Endpoint:** `/urls/{shortCode}/alias`

**Method:** `GET`

**Headers:**

* `Authorization: Bearer {access_token}`

**Response:** (200 OK)

```json
{
  "short_code": "abc123",
  "alias": "your-custom-alias"
}
```

**Example:**

* If no alias set, returns `alias: null`.

---

#### 2.5. Remove Alias

**Purpose:** Delete the custom alias and revert to the default randomly generated code.

**Endpoint:** `/urls/{shortCode}/alias`

**Method:** `DELETE`

**Headers:**

* `Authorization: Bearer {access_token}`

**Response:** (204 No Content)

**Example:**

* Upon successful deletion, accessing `/urls/abc123/alias` returns:

  ```json
  { "short_code": "abc123", "alias": null }
  ```

This document outlines the database schema for the SmartUrl service, including the new table for password reset tokens.

## Database Tables

### Table: users

| Column             | Type      | Constraints                         | Description                                |
| ------------------ | --------- | ----------------------------------- | ------------------------------------------ |
| id                 | SERIAL    | PRIMARY KEY                         | Unique identifier for each user            |
| username           | TEXT      | UNIQUE, NOT NULL                    | User's chosen username                     |
| email              | TEXT      | UNIQUE, NOT NULL                    | User's email address                       |
| password\_hash     | TEXT      |                                     | Hashed password (NULL for OAuth users)     |
| auth\_provider     | TEXT      |                                     | Authentication provider (NULL or "google") |
| auth\_provider\_id | TEXT      |                                     | Provider-specific user ID                  |
| created\_at        | TIMESTAMP | NOT NULL DEFAULT CURRENT\_TIMESTAMP | When the user was created                  |
| updated\_at        | TIMESTAMP | NOT NULL DEFAULT CURRENT\_TIMESTAMP | When the user was last updated             |

**Indexes:**

* PRIMARY KEY on `id`
* UNIQUE INDEX on `username`
* UNIQUE INDEX on `email`
* INDEX on `(auth_provider, auth_provider_id)`

---

### Table: password\_reset\_tokens

| Column      | Type      | Constraints                                     | Description                                   |
| ----------- | --------- | ----------------------------------------------- | --------------------------------------------- |
| id          | SERIAL    | PRIMARY KEY                                     | Unique identifier for each reset token record |
| user\_id    | INTEGER   | NOT NULL REFERENCES users(id) ON DELETE CASCADE | The user requesting the reset                 |
| token       | TEXT      | NOT NULL UNIQUE                                 | Secure random token                           |
| expires\_at | TIMESTAMP | NOT NULL                                        | Expiration timestamp for this token           |
| used        | BOOLEAN   | NOT NULL DEFAULT FALSE                          | Whether the token has been used               |
| created\_at | TIMESTAMP | NOT NULL DEFAULT CURRENT\_TIMESTAMP             | When the token record was created             |

**Indexes:**

* INDEX on `user_id`
* INDEX on `expires_at`

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
