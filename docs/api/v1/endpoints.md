# API Documentation: The Community Hub API (v1)

The GDGoC Benha System API is designed for building a rich community experience. This document provides the final production-grade reference for all features, including scheduling and media.

## 1. Advanced Community Feed (News & Events)
### POST `/v1/news`
**Min Role: Lead (HL 600)**
**Description**: Creates a news post. Supports immediate or scheduled publishing.
- **Request Body**:
```json
{
  "title": "Flutter Workshop Tomorrow",
  "content": "Don't forget to bring your laptops!",
  "image_url": "https://s3.gdgoc.com/news/123.jpg",
  "published_at": "2026-03-25T10:00:00Z",
  "target_audience": "TRACK_ONLY",
  "track_id": "uuid"
}
```
- **Backend Behavior**: If `published_at` is in the future, the backend creates a `SCHEDULED_TASK` for the Go worker.

---

## 2. Rich Session & Media Management
### GET `/v1/sessions/{id}`
**Description**: Returns full session details including media and PDF links.
- **Response (200 OK)**:
```json
{
  "id": "uuid",
  "title": "Backend Best Practices",
  "description": "Deep dive into Go and Postgres.",
  "session_type": "OFFLINE",
  "drive_pdf_link": "https://drive.google.com/...",
  "thumbnail_url": "...",
  "gallery": [
    { "url": "...", "caption": "Coding session in action" },
    { "url": "...", "caption": "Group photo" }
  ]
}
```

### POST `/v1/sessions/{id}/gallery`
**Min Role: Lead (HL 600)**
**Description**: Adds an image to the session gallery.
- **Request Body**: `{ "image_url": "...", "caption": "..." }`

---

## 3. Core Team & Performance Tracking
### GET `/v1/core-team/metrics`
**Min Role: Head (HL 800)**
**Description**: Fetches performance rankings for the core team.
- **Query Params**: `?track_id=uuid`, `?role=lead`
- **Response**:
```json
{
  "data": [
    {
      "user_id": "uuid",
      "name": "Mostafa Sensei",
      "points": 1250,
      "rating": 4.8,
      "attendance": 98
    }
  ]
}
```

### POST `/v1/evaluations`
**Min Role: Head (HL 800)**
**Description**: Submits a performance evaluation for a core team member.
- **Request Body**:
```json
{
  "target_user_id": "uuid",
  "score": 95,
  "feedback": "Outstanding leadership in the Backend track."
}
```

---

## 4. Track Leadership (Custom Sub-Roles)
### GET `/v1/tracks/{id}/leadership`
**Description**: Returns the specific leadership structure of a track.
- **Response**:
```json
{
  "track_name": "Backend Development",
  "head": { "id": "...", "name": "..." },
  "vice_head": { "id": "...", "name": "..." },
  "leads": [ { "id": "...", "name": "..." } ],
  "core_team": [ { "id": "...", "name": "..." } ]
}
```

---

## Standard Error Response (Global)

| Code | Status | Meaning |
| :--- | :---: | :--- |
| `ERR_UNAUTHORIZED` | 401 | JWT is missing, invalid, or expired. |
| `ERR_FORBIDDEN` | 403 | User's HL (Hierarchy Level) is insufficient. |
| `ERR_VALIDATION` | 422 | Request body failed schema validation. |
| `ERR_NOT_FOUND` | 404 | Resource does not exist in DB. |
| `ERR_RATE_LIMIT` | 429 | Redis rate limit (Managed by Token Bucket). |
| `ERR_INTERNAL` | 500 | Unexpected error on the server side. |
