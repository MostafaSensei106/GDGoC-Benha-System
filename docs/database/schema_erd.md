# Database Schema & Data Integrity: The Community Hub Edition

The GDGoC Benha System utilizes PostgreSQL as its primary source of truth, emphasizing strict relational integrity, performance indexing, and auditability for a large-scale community.

## 1. Ultimate Entity-Relationship Diagram (ERD)

```mermaid
erDiagram
    %% IAM: Identity & Access Management
    USERS {
        uuid id PK
        string email UK
        string password_hash
        string first_name
        string last_name
        string university_id UK
        string phone
        uuid role_id FK
        datetime created_at
    }

    ROLES {
        uuid id PK
        string slug UK "e.g., 'ocp', 'head-of-track', 'core-team-member', 'student'"
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

    %% ORGANIZATION: Tracks & Leadership
    BOOTCAMPS {
        uuid id PK
        string title
        text description
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
        string status "Enum: PRESENT, ABSENT, EXCUSED"
        int points_awarded
    }

    %% SCHEDULING & FEED: Delayed Publishing
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

    %% RELATIONS
    USERS }|--|| ROLES : "has"
    USERS ||--o| CORE_TEAM_STATS : "has performance"
    USERS ||--o{ EVALUATIONS : "receives/gives"
    BOOTCAMPS ||--o{ TRACKS : "hosts"
    TRACKS ||--o{ TRACK_ENROLLMENTS : "has members"
    TRACKS ||--o{ SESSON_RECORDS : "schedules"
    SESSON_RECORDS ||--o{ SESSION_GALLERY : "contains photos"
    SESSON_RECORDS ||--o{ ATTENDANCE : "tracks"
    NEWS_FEED ||--o{ SCHEDULED_TASKS : "triggers via"
```

## 2. Advanced Features Deep-Dive

### A. Core Team Performance Logic
- **`CORE_TEAM_STATS`**: Each member of the core team (including Heads and Leads) has a performance row. Points are accumulated from:
  - **Attendance**: 10 pts per internal session.
  - **Tasks**: Defined in the future Task Management module.
  - **Evaluations**: Quarterly feedback from higher-HL roles (e.g., a Board member evaluates a Head).

### B. Rich Sessions & Media Management
- **Drive Integration**: The `drive_pdf_link` allows facilitators to attach slides or handouts directly to a session.
- **Gallery**: A dedicated `SESSION_GALLERY` table allows multiple images per session to showcase community activity.
- **Hybrid Support**: The `session_type` (Online/Offline) and `location` allow for both physical meetups and Google Meet/Zoom links.

### C. The Scheduling System
- **`SCHEDULED_TASKS`**: This table acts as a queue for a Go background worker (using a `time.Ticker` or `Redis Streams`).
- **Logic**: When a news item is created with a `published_at` in the future, `is_published` is set to `false`, and a task is added to this table. The worker flips the bit to `true` at the exact timestamp.

## 3. Indexing for High Performance
- **B-Tree Index**: On `SCHEDULED_TASKS.run_at` to allow the worker to quickly poll for pending tasks.
- **GIN Index**: On `NEWS_FEED.content` for full-text search across the community portal.
- **Compound Index**: On `ATTENDANCE(session_id, user_id)` to ensure $O(1)$ lookup for checking if a student is already marked.
