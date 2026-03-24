# API Documentation: The Community Hub (v1)

The GDGoC Benha System API is built with a focus on security, performance, and developer experience. This document provides a production-grade reference for all features, including scheduling, attendance codes, and bootcamp management.

## 1. Authentication & Session Management
All authenticated requests must include the `Authorization: Bearer <JWT>` header.

### POST `/v1/auth/login`
- **Backend logic**: 
  - Verify password hash.
  - Store JTI (JWT ID) in **Redis** for revocation.
  - Cache user's HL (Hierarchy Level) in Redis.

---

## 2. Bootcamp & Track Management
### POST `/v1/bootcamps`
**Min Role: Board (HL 950)**
- **Description**: Creates a new bootcamp in `DRAFT` status.
- **Request Body**: `{ "title": "Summer 2026", "description": "...", "status": "DRAFT" }`

### PATCH `/v1/bootcamps/{id}/status`
**Min Role: Head (HL 800)**
- **Description**: Moves bootcamp to the next phase (e.g., `APPLICATION`, `ACTIVE`).

---

## 3. Dynamic Form Engine
### POST `/v1/forms`
**Min Role: Head (HL 800)**
- **Description**: Creates a form with a **JSON Schema** validation block.
- **Request Body**:
```json
{
  "title": "Backend Registration",
  "schema": {
    "type": "object",
    "required": ["github_url"],
    "properties": {
      "github_url": { "type": "string", "pattern": "^https://github.com/.*" }
    }
  }
}
```

---

## 4. Sessions & Attendance (Edge Case Support)
### POST `/v1/sessions`
**Min Role: Lead (HL 600)**
- **Description**: Creates a session (Online/Offline).
- **Online Specifics**: Generates an `attendance_code` (4-digit) stored in Redis with a TTL.
- **Offline Specifics**: Generates a signed QR JWT.

### POST `/v1/sessions/{id}/attendance`
**Description**: Marks a student as `PRESENT`.
- **Request Body (Online)**: `{ "user_id": "uuid", "code": "8271" }`
- **Request Body (Offline)**: `{ "user_id": "uuid", "qr_token": "signed.jwt.token" }`
- **Verification**: 
  - For codes: Match against Redis key `session:{id}:code`.
  - For QR: Decode and verify signature and expiry.

---

## 5. Public Events & Ticketing
### POST `/v1/events/{id}/register`
- **Description**: Registers a student/guest for an event.
- **Concurrency Protection**: Uses **PostgreSQL FOR UPDATE** lock on the `max_seats` count.
- **Success (201 Created)**: Returns a `ticket_code` (UUID) for the QR ticket.

### POST `/v1/events/check-in`
**Min Role: Facilitator (HL 300)**
- **Description**: Scans a ticket at the venue.
- **Request Body**: `{ "ticket_code": "uuid" }`
- **Status Change**: Updates `EVENT_TICKETS.status` to `USED`.

---

## 6. Core Team Performance & Evaluation
### GET `/v1/core-team/leaderboard`
**Min Role: Board (HL 950)**
- **Description**: Returns the real-time core team points rankings from **Redis ZSET**.

### POST `/v1/evaluations`
**Min Role: Head (HL 800)**
- **Description**: Submits a performance evaluation for a lead or member.

---

## Standard Global Error Codes

| Code | Status | Meaning |
| :--- | :---: | :--- |
| `ERR_UNAUTHORIZED` | 401 | JWT is missing, invalid, or expired. |
| `ERR_FORBIDDEN` | 403 | User's HL (Hierarchy Level) is insufficient. |
| `ERR_VALIDATION` | 422 | Request body failed JSON Schema validation. |
| `ERR_NOT_FOUND` | 404 | Resource does not exist in the database. |
| `ERR_RATE_LIMIT` | 429 | Redis rate limit (Managed by Token Bucket). |
| `ERR_INTERNAL` | 500 | Unexpected error on the server side. |
