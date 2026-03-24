# Database Schema & Data Integrity: Deep Dive

The GDGoC Benha System utilizes PostgreSQL as its primary source of truth, emphasizing strict relational integrity, performance indexing, and auditability.

## 1. Advanced ERD (Full System)

This diagram covers the intricate relationships required for a multi-departmental GDG system.

```mermaid
erDiagram
    %% IAM: Identity & Access
    USERS {
        uuid id PK "B-Tree Indexed"
        string email UK "Case-insensitive Index"
        string password_hash
        string first_name
        string last_name
        string university_id UK "Indexed"
        string phone
        uuid role_id FK
        boolean is_active "Default: true"
        datetime created_at
        datetime updated_at
        datetime deleted_at "Soft Delete Support"
    }

    ROLES {
        uuid id PK
        string slug UK "e.g., 'ocp', 'head-tech'"
        string display_name
        int level "10-1000 for RBAC"
    }

    AUDIT_LOGS {
        uuid id PK
        uuid actor_id FK "References USERS"
        string action "e.g., 'USER_CREATED', 'GRADE_UPDATED'"
        string target_table
        uuid target_id
        jsonb old_values
        jsonb new_values
        datetime created_at
    }

    %% ORGANIZATION
    BOOTCAMPS {
        uuid id PK
        string title
        text description
        date start_date
        date end_date
        string status "Enum: UPCOMING, ACTIVE, COMPLETED"
    }

    TRACKS {
        uuid id PK
        string name "e.g., 'Web Development', 'Flutter'"
        uuid bootcamp_id FK
        uuid lead_id FK "References USERS"
        uuid co_lead_id FK "References USERS"
    }

    TRACK_ENROLLMENTS {
        uuid id PK
        uuid user_id FK
        uuid track_id FK
        string role "Enum: STUDENT, FACILITATOR"
        datetime joined_at
    }

    %% OPERATIONS: Sessions & Attendance
    SESSON_RECORDS {
        uuid id PK
        string title
        uuid track_id FK
        datetime scheduled_at
        int duration_minutes
        string location
    }

    ATTENDANCE {
        uuid id PK
        uuid session_id FK
        uuid user_id FK
        datetime check_in_time
        string status "Enum: PRESENT, ABSENT, EXCUSED"
        int bonus_points
        text note
    }

    %% ACADEMICS: Grading
    GRADES {
        uuid id PK
        uuid user_id FK
        uuid track_id FK
        int score
        text feedback
        uuid graded_by FK "References USERS"
        datetime graded_at
    }

    %% ENGAGEMENT: Forms
    FORMS {
        uuid id PK
        string title
        text description
        jsonb schema "JSON Schema for dynamic validation"
        boolean is_public "Allow non-registered users"
        datetime closes_at
    }

    FORM_RESPONSES {
        uuid id PK
        uuid form_id FK
        uuid user_id FK "Nullable for public forms"
        jsonb data "Answer Payload"
        string status "Enum: PENDING, APPROVED, REJECTED"
        datetime submitted_at
    }

    %% RELATIONS
    USERS }|--|| ROLES : "has one"
    USERS ||--o{ AUDIT_LOGS : "performs"
    BOOTCAMPS ||--o{ TRACKS : "hosts"
    TRACKS ||--o{ TRACK_ENROLLMENTS : "contains"
    USERS ||--o{ TRACK_ENROLLMENTS : "enrolled in"
    TRACKS ||--o{ SESSON_RECORDS : "schedules"
    SESSON_RECORDS ||--o{ ATTENDANCE : "tracks"
    USERS ||--o{ ATTENDANCE : "marked for"
    USERS ||--o{ GRADES : "receives"
    FORMS ||--o{ FORM_RESPONSES : "collects"
```

## 2. SQL Integrity & Performance

### Indexing Strategy
- **Partial Indexes**: For frequently queried subsets (e.g., `WHERE is_active = true`).
- **GIN Indexes**: On `JSONB` columns in the `FORMS` and `FORM_RESPONSES` tables to allow efficient querying of dynamic form fields.
- **Foreign Key Constraints**: All relationships are strictly enforced with `ON DELETE RESTRICT` or `ON DELETE CASCADE` where appropriate (e.g., deleting a Track deletes its Enrollments).

### JSONB Schema Validation
- Form schemas will follow the **JSON Schema** standard.
- Before insertion into `FORM_RESPONSES`, the Go backend will validate the `data` payload against the `schema` defined in the `FORMS` table.

## 3. Data Auditing Policy
- The `AUDIT_LOGS` table records all state-changing operations by high-HL users (Heads/Board).
- It captures `old_values` and `new_values` for debugging and accountability, essential for a system with 10+ core team roles.

## 4. Soft Deletes
- Most tables include a `deleted_at` field. 
- **Purpose**: Prevent accidental data loss and maintain historical records (e.g., keeping attendance records of a student who dropped a bootcamp).
