# API Documentation: v1 Deep Dive

The GDGoC Benha System API is built with a focus on security, performance, and developer experience. This document provides a production-grade reference for all core endpoints.

## 1. Authentication & Session Management
All authenticated requests must include the `Authorization: Bearer <JWT>` header.

### POST `/v1/auth/login`
**Description**: Authenticates a user and starts a session.
- **Request Body**:
```json
{
  "email": "user@example.com",
  "password": "secure_password"
}
```
- **Response (200 OK)**:
```json
{
  "access_token": "ey...",
  "refresh_token": "ey...",
  "user": {
    "id": "uuid",
    "role": "head-tech",
    "level": 800
  }
}
```
- **Backend logic**:
  1. Verify password hash from PostgreSQL.
  2. Fetch Role and Level.
  3. Store `jti` in Redis for session management.
  4. Cache user's HL (Hierarchy Level) in Redis for $O(1)$ authorization.

---

## 2. Dynamic Form Engine
### POST `/v1/forms`
**Min Role: Head (HL 800)**
**Description**: Creates a new form with a JSONB validation schema.
- **Request Body**:
```json
{
  "title": "Flutter Bootcamp Registration",
  "description": "Apply now!",
  "schema": {
    "type": "object",
    "required": ["github_url"],
    "properties": {
      "github_url": { "type": "string", "pattern": "^https://github.com/.*" }
    }
  },
  "is_public": true,
  "closes_at": "2026-12-31T23:59:59Z"
}
```

### POST `/v1/forms/{id}/submit`
**Description**: Submits data to a form.
- **Request Body**:
```json
{
  "data": {
    "github_url": "https://github.com/ottafa"
  }
}
```
- **Error (422 Unprocessable Entity)**:
```json
{
  "error": "VALIDATION_FAILED",
  "message": "Field 'github_url' does not match the required pattern.",
  "errors": [
    { "field": "github_url", "message": "invalid github link" }
  ]
}
```

---

## 3. Events & High-Concurrency Attendance
### POST `/v1/sessions/{id}/attendance`
**Min Role: Facilitator (HL 300)**
**Description**: Logs attendance for a specific session.
- **Request Body**:
```json
{
  "user_id": "uuid",
  "status": "PRESENT",
  "bonus_points": 5,
  "note": "Excellent participation"
}
```
- **Backend Logic (Atomic)**:
  1. Use Redis **Distributed Lock** (`lock:session:{id}:user:{uid}`) to prevent double-logging.
  2. Insert into PostgreSQL `ATTENDANCE` table.
  3. Trigger async task to update student's track performance score.
  4. Update Redis Leaderboard for the track.

---

## 4. Grading & Evaluation
### POST `/v1/grades`
**Min Role: Facilitator (HL 300)**
**Description**: Input a grade for a student.
- **Request Body**:
```json
{
  "user_id": "uuid",
  "track_id": "uuid",
  "score": 85,
  "feedback": "Great project architecture."
}
```

### PATCH `/v1/grades/{id}`
**Min Role: Lead (HL 600)**
**Description**: Approve or override a grade.
- **Request Body**:
```json
{
  "status": "APPROVED",
  "override_score": 90
}
```

---

## 5. Global & Track-Specific Feed
### GET `/v1/news`
**Description**: Fetch the filtered news feed for the current user.
- **Query Params**: `?type=global`, `?track_id=uuid`
- **Response**:
```json
{
  "data": [
    {
      "id": "uuid",
      "title": "Welcome to GDGoC Benha!",
      "content": "...",
      "author": "OCP Name",
      "published_at": "2026-03-24T10:00:00Z"
    }
  ]
}
```

---

## Standard Global Error Codes

| Code | Status | Meaning |
| :--- | :---: | :--- |
| `ERR_UNAUTHORIZED` | 401 | Invalid or expired JWT. |
| `ERR_FORBIDDEN` | 403 | User's HL is below the required level. |
| `ERR_NOT_FOUND` | 404 | Resource (User/Form/Session) doesn't exist. |
| `ERR_RATE_LIMIT` | 429 | Too many requests (Redis bucket full). |
| `ERR_INTERNAL` | 500 | An unexpected server error occurred. |
