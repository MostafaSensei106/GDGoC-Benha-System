# Database Schema & Data Integrity: The Ultimate Community Engine

The GDGoC Benha System utilizes PostgreSQL as its primary source of truth, emphasizing strict relational integrity, performance indexing, and auditability for a large-scale community.

## 1. Unified Entity-Relationship Diagram (ERD)

This diagram represents the final, production-ready schema including all edge-case support (QR, Tickets, Scheduling, Performance).

```mermaid
erDiagram
    %% IAM: Identity & Access Management
    USERS {
        uuid id PK
        string email UK "Indexed"
        string password_hash
        string first_name
        string last_name
        string university_id UK "Indexed"
        string phone
        uuid role_id FK
        boolean is_active "Default: true"
        datetime created_at
    }

    ROLES {
        uuid id PK
        string slug UK "e.g., 'ocp', 'head-tech', 'core-team-member', 'student'"
        string display_name
        int level "Hierarchy Level (HL)"
    }

    %% INTERNAL: Core Team Performance
    CORE_TEAM_STATS {
        uuid id PK
        uuid user_id FK UK
        int total_points
        int tasks_completed
        int sessions_attended
        float performance_rating "0.0 - 5.0"
        datetime last_evaluated_at
    }

    EVALUATIONS {
        uuid id PK
        uuid target_user_id FK "Member being evaluated"
        uuid evaluator_id FK "Head/Lead giving the evaluation"
        int score "1-100"
        text feedback
        datetime created_at
    }

    %% ORGANIZATION: Bootcamps & Tracks
    BOOTCAMPS {
        uuid id PK
        string title
        text description
        string status "Enum: DRAFT, APPLICATION, SCREENING, ACTIVE, FINALIZING, ARCHIVED"
        date start_date
        date end_date
    }

    TRACKS {
        uuid id PK
        string name "e.g., 'Backend Development', 'Flutter', 'Cyber Security'"
        uuid bootcamp_id FK
        uuid head_id FK "References USERS (HL 800)"
        uuid vice_head_id FK "References USERS (HL 750)"
        uuid lead_id FK "References USERS (HL 600)"
    }

    TRACK_ENROLLMENTS {
        uuid id PK
        uuid user_id FK
        uuid track_id FK
        string enrollment_role "Enum: STUDENT, CORE_TEAM_MEMBER, FACILITATOR"
        datetime joined_at
    }

    %% OPERATIONS: Rich Sessions & Media
    SESSON_RECORDS {
        uuid id PK
        string title
        text description
        string session_type "Enum: ONLINE, OFFLINE"
        uuid track_id FK
        datetime scheduled_at
        int duration_minutes
        string location "Physical address or meeting link"
        string thumbnail_url "Main session image"
        string drive_pdf_link "Google Drive PDF link"
        string attendance_code "4-digit code for Online sessions"
    }

    SESSION_GALLERY {
        uuid id PK
        uuid session_id FK
        string image_url
        string caption
    }

    ATTENDANCE {
        uuid id PK
        uuid session_id FK
        uuid user_id FK
        datetime check_in_time
        string status "Enum: PRESENT, ABSENT, EXCUSED, LATE"
        int points_awarded
    }

    %% ENGAGEMENT: Public Events & Ticketing
    EVENTS {
        uuid id PK
        string title
        text description
        datetime start_at
        datetime end_at
        string venue
        int max_seats
        boolean is_published
        jsonb form_schema "JSON Schema for registration"
    }

    EVENT_TICKETS {
        uuid id PK
        uuid event_id FK
        uuid user_id FK "Nullable for public guest registration"
        string ticket_code UK "Unique UUID4 for QR"
        string holder_email
        string status "Enum: VALID, USED, VOID"
        datetime issued_at
    }

    %% SCHEDULING & FEED
    SCHEDULED_TASKS {
        uuid id PK
        string task_type "Enum: NEWS_PUBLISH, EVENT_START"
        uuid target_id "Reference to News/Event ID"
        datetime run_at
        string status "Enum: PENDING, COMPLETED, FAILED"
    }

    NEWS_FEED {
        uuid id PK
        string title
        text content
        string image_url
        string target_audience "Enum: PUBLIC, CORE_TEAM_ONLY, TRACK_ONLY"
        uuid track_id FK "Nullable"
        uuid author_id FK
        boolean is_published "Managed by Scheduler"
        datetime published_at
    }

    %% AUDIT & LOGS
    AUDIT_LOGS {
        uuid id PK
        uuid actor_id FK
        string action "e.g., 'USER_CREATED', 'GRADE_UPDATED', 'TICKET_VOIDED'"
        string target_table
        uuid target_id
        jsonb old_values
        jsonb new_values
        datetime created_at
    }

    %% RELATIONS
    USERS }|--|| ROLES : "has"
    USERS ||--o| CORE_TEAM_STATS : "has performance"
    USERS ||--o{ EVALUATIONS : "receives/gives"
    BOOTCAMPS ||--o{ TRACKS : "hosts"
    TRACKS ||--o{ TRACK_ENROLLMENTS : "has members"
    TRACKS ||--o{ SESSON_RECORDS : "schedules"
    SESSON_RECORDS ||--o{ SESSION_GALLERY : "contains photos"
    SESSON_RECORDS ||--o{ ATTENDANCE : "tracks"
    EVENTS ||--o{ EVENT_TICKETS : "issues"
    USERS ||--o{ EVENT_TICKETS : "holds"
    NEWS_FEED ||--o{ SCHEDULED_TASKS : "triggers via"
    USERS ||--o{ AUDIT_LOGS : "performs"
```

## 2. Advanced Data Management Policies

### A. High-Concurrency Ticketing (Postgres Lock)
To prevent overselling of public events, the registration logic uses **Pessimistic Locking**:
```sql
-- During registration
BEGIN;
SELECT count(*) FROM event_tickets WHERE event_id = $1 FOR UPDATE;
-- If count < max_seats, then INSERT
INSERT INTO event_tickets (event_id, ...) VALUES (...);
COMMIT;
```

### B. Anti-Cheating Attendance Logic
- **Online Sessions**: The `attendance_code` in `SESSON_RECORDS` is rotated or generated dynamically. The student's submission must match this code *and* be within the session's active window (determined by `scheduled_at` + `duration_minutes`).
- **Offline Sessions**: The QR code scanned by the student contains a signed JWT with the `session_id` and an expiration timestamp to prevent reusing old QR codes.

### C. GIN Indexing for JSONB
The `EVENTS.form_schema` and `AUDIT_LOGS.old_values/new_values` columns use `JSONB`. We apply GIN indexes to allow fast searching:
```sql
CREATE INDEX idx_audit_new_values ON audit_logs USING GIN (new_values jsonb_path_ops);
```

### D. Soft Deletes & Auditability
No data is ever truly deleted from the `BOOTCAMPS`, `TRACKS`, or `USERS` tables. Instead:
1. `deleted_at` timestamp is set.
2. An entry in `AUDIT_LOGS` captures the state of the row before deletion.
3. Foreign key constraints use `ON DELETE RESTRICT` to prevent orphaned data (e.g., you cannot delete a Bootcamp while it still has active Tracks).
