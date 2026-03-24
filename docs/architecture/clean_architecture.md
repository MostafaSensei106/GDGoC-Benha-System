# System Architecture: Lean Clean Architecture in Go

The GDGoC Benha System utilizes a high-performance, strictly typed architecture designed for extreme scalability and maintainability. It follows a "Lean" version of Clean Architecture, optimized for the Go ecosystem by reducing boilerplate while maintaining strict separation between business logic and side effects (DB, Cache, Network).

## 1. Directory Structure (Implementation Standard)

The codebase is structured to enforce the dependency rule (dependencies point inward).

```text
/
├── cmd/                # Entry points (API server, Background Workers)
├── internal/
│   ├── domain/         # CORE: Business entities (structs) and Repository Interfaces
│   ├── usecase/        # LOGIC: Pure business rules (Services)
│   ├── delivery/       # I/O: HTTP Handlers (Fiber/Echo), Middleware
│   └── infrastructure/ # IMPL: sqlc generated code, Redis client, Third-party SDKs
├── migrations/         # goose SQL migration files
├── pkg/                # UTILS: Shared utilities (Logger, Validator)
└── sql/                # RAW SQL: Used by sqlc to generate type-safe code
```

## 2. The Dependency Flow (Sequence of Implementation)

1. **Domain Layer**: Define the `User` struct and `UserRepository` interface.
2. **Infrastructure Layer**: Write the `sqlc` query. Implement the `UserRepository` using the generated code.
3. **Usecase Layer**: Inject the `UserRepository` interface into the `UserUsecase` service.
4. **Delivery Layer**: Call the `UserUsecase` from an HTTP handler.

## 3. Tech Stack Deep-Dive

### A. PostgreSQL & sqlc (Persistence)
We use `sqlc` to eliminate the "Magic String" problem of raw SQL and the performance overhead of ORMs.
- **Workflow**: Write raw SQL in `sql/queries/*.sql` -> Run `sqlc generate` -> Get type-safe Go methods.
- **Benefits**: Zero reflection, compile-time SQL validation, maximum performance.

### B. Redis (The Accelerator)
Redis is not just a cache; it's a critical infrastructure component for state management.
- **State Storage**: Storing active JWT JTI (JWT ID) for immediate session revocation.
- **Authorization Cache**: Storing a user's `RoleID` and `Level` to avoid DB joins on every authorized request.
- **Distributed Locking**: Preventing race conditions during high-concurrency event registrations.

### C. goose (Schema Evolution)
- All schema changes are versioned SQL files. 
- **Rule**: Never use `sqlc` to modify schema; use `goose` to ensure consistency across environments.

## 4. Cross-Cutting Concerns

### Error Handling Strategy
We use custom error types in the `domain` layer (e.g., `domain.ErrUserNotFound`) and map them to HTTP status codes in the `delivery` layer.

### Observability (Logging & Tracing)
- **Structured Logging**: Using `zap` or `slog` for JSON logs.
- **Trace IDs**: Injected into every request context via middleware and propagated through the Use Case and Repository layers for deep debugging.

## 5. System Scalability Diagram

```mermaid
graph TD
    LB[Load Balancer] --> WebCluster[Go API Cluster (Delivery Layer)]
    
    subgraph AppServer [API Server Node]
        WebCluster --> Usecase[Usecase Layer (Business Logic)]
        Usecase --> RepoIntf[Repository Interfaces]
        RepoIntf --> PostgresImpl[sqlc / Postgres Impl]
        RepoIntf --> RedisImpl[Redis Client Impl]
    end

    PostgresImpl --> RDS[(Primary PostgreSQL)]
    RDS -. "Replication" .-> RDSR[(Read Replica)]
    RedisImpl --> RedisCluster[(Redis Cluster - Sessions & Cache)]

    subgraph Workers [Background Workers]
        Worker[Go Background Consumer] --> RDS
        Worker --> RedisCluster
    end
```
