# System Workflows & Complex Logic

This document details the complex logic within the GDGoC Benha System using sequence diagrams.

## 1. Automated Attendance & Performance Scoring

When a Facilitator logs attendance for a student, the system automatically calculates the student's performance score based on the status (Present/Absent/Excused) and any bonus points.

```mermaid
sequenceDiagram
    participant Fac as Facilitator App
    participant API as Backend (Go)
    participant DB as Postgres (sqlc)
    participant Cache as Redis (User Stats)

    Fac->>API: POST /sessions/{id}/attendance {userID, status, bonus}
    API->>DB: Verify Facilitator has permission (HL 300+)
    API->>DB: Record attendance entry
    
    Note over API: Business Logic Calculation
    alt status == PRESENT
        API->>API: Base Points = 10 + bonus
    else status == ABSENT
        API->>API: Base Points = 0
    else status == EXCUSED
        API->>API: Base Points = 5
    end
    
    API->>DB: Update track-specific student performance score
    API->>Cache: Invalidate student's performance stats cache
    API-->>Fac: 200 OK (Attendance Recorded)
```

## 2. Dynamic Form Life Cycle

This workflow describes how a Board/Head creates a registration form with a dynamic schema and how the system handles submissions.

```mermaid
sequenceDiagram
    participant Head as Department Head
    participant API as Backend (Go)
    participant DB as Postgres (JSONB)
    participant Student as Student User

    Head->>API: POST /forms {title, description, schema_json}
    Note right of Head: Schema defines fields like "GitHub URL (Required)"
    API->>DB: Store in FORMS table as JSONB
    API-->>Head: 201 Created

    Student->>API: GET /forms/{id}
    API->>DB: Retrieve Form Schema
    API-->>Student: Return JSON Schema for frontend to render

    Student->>API: POST /forms/{id}/submit {data_json}
    API->>API: Validate data_json against the stored schema
    
    alt Validation Failed
        API-->>Student: 400 Bad Request (Validation Errors)
    else Validation Success
        API->>DB: Insert response into FORM_RESPONSES (JSONB)
        API-->>Student: 200 OK (Submission Received)
    end
```

## 3. Global News & Announcement Broadcast

The Board (OCP/Vice) can broadcast announcements to all users, while Track Leads can target specific track members.

```mermaid
sequenceDiagram
    participant Board as Board Member
    participant API as Backend (Go)
    participant DB as Postgres
    participant Redis as Redis (Pub/Sub)
    participant Clients as All User Devices

    Board->>API: POST /news {title, content, target: "PUBLIC"}
    API->>DB: Persist news entry
    API->>Redis: Publish "new_announcement" event
    
    Note over Redis: All connected websocket clients receive push
    Redis-->>Clients: Real-time notification "GDGoC Benha just posted news!"
    API-->>Board: 204 No Content (Broadcast Sent)
```

## 4. Academic Evaluation & Grading Flow

Tracking grades for student assignments and projects across different tracks.

```mermaid
sequenceDiagram
    participant Fac as Facilitator
    participant Lead as Track Lead
    participant API as Backend
    participant DB as Postgres

    Fac->>API: POST /grades {userID, trackID, score, feedback}
    API->>DB: Store Grade as "PENDING"
    API-->>Fac: 200 OK (Grade Submitted for Review)

    Lead->>API: GET /grades/pending (Filter by track)
    API->>DB: List all pending grades in Lead's track
    API-->>Lead: List of evaluations

    Lead->>API: PATCH /grades/{id} {status: "APPROVED"}
    API->>DB: Update grade status to "APPROVED"
    API->>DB: Recalculate student's track average
    API-->>Lead: 200 OK
```
