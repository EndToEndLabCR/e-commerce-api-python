# Improvement Plan

A phased roadmap from the current state to a production-ready, scalable e-commerce API.

---

## Phase 1 – Quick Wins
**Goal:** Improve code quality, fix low-risk correctness issues, and lay the groundwork for testing.
**Estimated effort:** 1–2 days

---

### 1.1 Uncomment Route Docstrings

**File:** `src/app/features/user/presentation/web/routes/user_routes.py`

Remove the `#` prefixes from the docstrings on `get_user_by_id` and `create_user`. FastAPI uses these to populate OpenAPI descriptions.

**Expected benefits:** Swagger UI becomes fully self-documenting for all endpoints.

**Tools/patterns:** Text editor; no libraries required.

---

### 1.2 Fix `get_config_value` Docstring

**File:** `src/shared/utils/config_util.py`

Swap the `key` and `config` parameter descriptions and fix the "Tey" typo.

**Expected benefits:** Accurate documentation for contributors.

**Tools/patterns:** None.

---

### 1.3 Fix `BaseModel` UUID Generation

**File:** `src/shared/infrastructure/postgres/models/base_model.py`

Replace:
```python
default=uuid.uuid1
```
with:
```python
default=uuid.uuid4
```

**Expected benefits:** UUID primary keys no longer leak MAC address; consistent with `EntityId.generate()`.

**Tools/patterns:** Alembic migration is not required — this only affects new row defaults, not existing data.

---

### 1.4 Add `nullable=False, default=True` to `is_active`

**File:** `src/app/features/user/infrastructure/postgres/models/user_model.py`

```python
is_active = Column(Boolean, nullable=False, default=True)
```

**Expected benefits:** Eliminates three-valued boolean logic; enforces data integrity at the database level.

**Tools/patterns:** Alembic `op.alter_column` migration.

---

### 1.5 Complete Return Type Annotations

**File:** `src/app/features/user/application/services/user_service.py`

Add `-> UserResponse` to `get_user_by_id` and confirm all use case `execute()` methods have full signatures.

**Expected benefits:** Better IDE support; type checkers (mypy / pyright) can validate the call chain.

**Tools/patterns:** `mypy` or `pyright` to verify.

---

### 1.6 Raise Maximum Password Length

**File:** `src/app/features/user/domain/value_objects/hashed_password.py`

Change `_MAX_PASSWORD_LENGTH = 16` to `_MAX_PASSWORD_LENGTH = 128`.

**Expected benefits:** Aligns with NIST SP 800-63B; users can use passphrases and password manager-generated passwords.

**Tools/patterns:** None; adjust any existing test expectations.

---

### 1.7 Write Unit Tests for Value Objects

**Files to create:** `tests/unit/shared/domain/`, `tests/unit/features/user/domain/`

Test classes: `Email`, `HashedPassword`, `Role`, `EntityId`.

Scenarios to cover:
- Valid inputs accepted
- Invalid inputs raise `ValueError` / `TypeError`
- `HashedPassword.from_plain_text` produces a verifiable hash
- `HashedPassword.verify` returns `True` for matching password, `False` otherwise
- `Role.from_str` normalises case
- `EntityId.from_string` rejects malformed UUIDs

**Expected benefits:** Catches regressions in the most critical domain invariants; fast (no I/O).

**Tools/patterns:** `pytest`, `pytest-asyncio` for any async tests.

---

## Phase 2 – Maintainability
**Goal:** Remove structural pain points, improve testability, and clean up cross-cutting concerns.
**Estimated effort:** 3–5 days

---

### 2.1 Fix Database Engine Lifecycle (Critical)

**File:** `src/app/features/user/presentation/web/dependencies.py`, `src/app/app.py`

Move `PostgresDbConnection` instantiation to the FastAPI `lifespan` context:

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.db = PostgresDbConnection(postgres_config)
    yield
    await app.state.db.close_engine()

fastApiApp = FastAPI(lifespan=lifespan)
```

Update `get_database_session` to retrieve the session factory from `app.state`:

```python
async def get_database_session(request: Request) -> AsyncGenerator[AsyncSession, None]:
    async with request.app.state.db.get_session() as session:
        yield session
```

**Expected benefits:** Connection pool is created once and shared across all requests; Postgres connections are stable and bounded; engine is properly disposed on shutdown.

**Tools/patterns:** FastAPI `lifespan`, `app.state`.

---

### 2.2 Add Global Exception Handlers

**File:** `src/app/app.py`

Register handlers for all expected domain and infrastructure exceptions:

```python
@fastApiApp.exception_handler(UserDoesNotExistException)
async def user_not_found_handler(request, exc):
    return JSONResponse(status_code=404, content={"detail": str(exc)})

@fastApiApp.exception_handler(ValueError)
async def value_error_handler(request, exc):
    return JSONResponse(status_code=422, content={"detail": str(exc)})

@fastApiApp.exception_handler(IntegrityError)
async def integrity_error_handler(request, exc):
    return JSONResponse(status_code=409, content={"detail": "A record with this data already exists."})
```

**Expected benefits:** Clients receive correct HTTP semantics; stack traces are not leaked; domain exceptions map cleanly to HTTP status codes.

**Tools/patterns:** FastAPI `exception_handler`; consider a centralised `exception_handlers.py` module.

---

### 2.3 Add `update()` Method to `UserEntity`

**File:** `src/app/features/user/domain/entities/user_entity.py`

Replace direct field mutation in use cases with a controlled entity method:

```python
def update(self, email: Optional[Email] = None, first_name: Optional[str] = None, last_name: Optional[str] = None) -> None:
    if email is not None:
        self.email = email
    if first_name is not None:
        self.first_name = first_name
    if last_name is not None:
        self.last_name = last_name
    self.updated_at = get_current_datetime()
```

Update `UpdateUserUseCase` to call `existing_user.update(...)`.

**Expected benefits:** Invariant enforcement is centralised; the entity controls its own transitions; `updated_at` is reliably refreshed.

**Tools/patterns:** DDD entity behaviour pattern.

---

### 2.4 Write Use Case Unit Tests

**Files to create:** `tests/unit/features/user/application/use_cases/`

For each use case (`SaveUser`, `GetUserByIdUseCase`, `UpdateUserUseCase`, `DeleteUserUseCase`):
- Create a mock `UserRepository` using `unittest.mock.AsyncMock`.
- Test the happy path.
- Test the `UserDoesNotExistException` path.
- Test invalid UUID input.

**Expected benefits:** Confidence in business logic without requiring a real database.

**Tools/patterns:** `pytest`, `pytest-asyncio`, `unittest.mock.AsyncMock`.

---

### 2.5 Resolve or Remove `find_by_name`

**File:** `src/app/features/user/infrastructure/postgres/repository/user_repository_impl.py`

Either add `find_by_name` to the `UserRepository` interface with a corresponding use case and route, or remove the dead code.

**Expected benefits:** Eliminates confusion about whether this method is intentionally public or abandoned.

---

### 2.6 Tighten CORS Policy

**File:** `src/app/app.py`

Replace wildcard method and header lists with explicit sets matching API usage:

```python
allow_methods=["GET", "POST", "PATCH", "DELETE"],
allow_headers=["Content-Type", "Authorization"],
```

Load allowed origins from configuration rather than hardcoding `http://localhost`.

**Expected benefits:** Reduced CSRF attack surface; correct CORS headers in production.

---

## Phase 3 – Structural Improvements
**Goal:** Add missing endpoints, improve the API surface, and enforce timezone correctness.
**Estimated effort:** 3–5 days

---

### 3.1 Fix Timezone-Aware Datetimes

**Files:** `src/shared/utils/date_util.py`, `src/shared/infrastructure/postgres/models/base_model.py`

```python
from datetime import datetime, timezone

def get_current_datetime() -> datetime:
    return datetime.now(timezone.utc)
```

Update SQLAlchemy column defaults to use `func.now()` with a timezone-aware server default, or rely on `TIMESTAMP WITH TIME ZONE` column type.

**Expected benefits:** Correct timestamp comparisons across timezones; no silent datetime arithmetic bugs.

---

### 3.2 Add `GET /v1/user` Pagination Endpoint

**Files:** `user_routes.py`, `user_service.py`, new `GetAllUsersUseCase`

```python
@router.get("/", response_model=list[UserResponse], status_code=200)
async def get_users(limit: int = Query(20, ge=1, le=100), offset: int = Query(0, ge=0), ...) -> list[UserResponse]:
    ...
```

**Expected benefits:** Admin and management use cases become possible; `find_all` in the repository is finally used.

**Tools/patterns:** FastAPI `Query` parameters.

---

### 3.3 Add Structured Logging with Request Correlation

**Files:** New `src/shared/utils/request_context.py`, middleware in `app.py`

Use `contextvars.ContextVar` to store a per-request ID:

```python
request_id_var: ContextVar[str] = ContextVar("request_id", default="")
```

Inject into all log lines via a custom `logging.Filter`. Middleware sets the context variable at request entry.

**Expected benefits:** All log lines for a single request share a correlation ID; production debugging becomes tractable.

**Tools/patterns:** `contextvars`, `logging.Filter`, `starlette.middleware.base.BaseHTTPMiddleware`.

---

### 3.4 Add Integration Tests for Routes

**Files to create:** `tests/integration/features/user/`

Use `httpx.AsyncClient` with FastAPI's test app, backed by a test Postgres database (or `SQLite` with `aiosqlite` for unit-level integration tests).

Cover: create user, get user, update user, delete user, duplicate email conflict, non-existent user.

**Tools/patterns:** `pytest-asyncio`, `httpx`, `anyio`, `config_test.yml`.

---

## Phase 4 – Architectural Alignment
**Goal:** Implement authentication, complete core e-commerce features, and enforce architectural boundaries.
**Estimated effort:** 2–4 weeks

---

### 4.1 Implement Authentication

**New feature:** `src/app/features/auth/`

- `POST /v1/auth/login` – validates credentials, returns JWT access token + refresh token.
- `POST /v1/auth/refresh` – issues a new access token from a valid refresh token.
- JWT validation `Depends` injectable into any route that requires authentication.
- Role-based access: `ADMIN` role required for delete; ownership check required for update.

**Tools/patterns:** `python-jose`, `passlib`, `pyjwt` (already in requirements), FastAPI security utilities.

---

### 4.2 Implement Core E-Commerce Features

Following the established template scaffold:

| Feature       | Key Entities                        | Key Use Cases                              |
|---------------|-------------------------------------|--------------------------------------------|
| `product`     | `ProductEntity`, `Price`, `SKU`    | CreateProduct, GetProductById, ListProducts, UpdateProduct, DeleteProduct |
| `category`    | `CategoryEntity`                   | CreateCategory, ListCategories             |
| `cart`        | `CartEntity`, `CartItem`           | AddItemToCart, RemoveItemFromCart, GetCart |
| `order`       | `OrderEntity`, `OrderItem`, `OrderStatus` | PlaceOrder, GetOrderById, ListOrders, CancelOrder |
| `payment`     | `PaymentEntity`, `PaymentStatus`   | ProcessPayment, GetPaymentStatus           |

**Tools/patterns:** Feature scaffold template; Alembic migrations per feature.

---

### 4.3 Reconsider `UserService` Role

Evaluate whether `UserService` should be eliminated (routes depend on use cases directly) or whether it should take on a genuine orchestration responsibility (e.g., coordinating `SaveUser` + sending a welcome email).

If eliminated: remove the class, wire use cases directly through `Depends`.
If retained: add meaningful cross-cutting logic (transaction scope, event publishing, etc.).

**Expected benefits:** Cleaner dependency graph; avoids the "service as dispatcher" anti-pattern.

---

### 4.4 Introduce Domain Events (Optional but Recommended)

For cross-feature coordination (e.g., order placed → inventory updated → payment initiated), introduce a simple domain event mechanism:

- `DomainEvent` base class
- Entities accumulate events (e.g., `OrderEntity.place()` appends `OrderPlacedEvent`)
- Application layer collects and dispatches events after persistence

**Tools/patterns:** In-process event bus (simple list + dispatcher); upgrade to a message broker (RabbitMQ, Redis Streams) in Phase 5.

---

## Phase 5 – Scalability and Production Readiness
**Goal:** Harden the system for real traffic, observability, and horizontal scaling.
**Estimated effort:** 2–3 weeks

---

### 5.1 Add Rate Limiting

**Tools:** `slowapi` (Starlette-compatible, rate limiting via `limits` library).

Apply per-IP and per-user rate limits to sensitive endpoints (login, create user) to mitigate brute-force and abuse.

---

### 5.2 Add Caching Layer

**Tools:** `redis`, `fastapi-cache2`.

Cache frequently read, rarely mutated resources (product catalog, category list) with configurable TTL. Invalidate on write.

**Expected benefits:** Reduced Postgres load for read-heavy endpoints; lower p99 latency.

---

### 5.3 Add Request Validation Middleware

Add input size limits and content-type enforcement at the middleware level to prevent oversized payloads.

---

### 5.4 Introduce Observability Stack

| Concern    | Tool                        |
|------------|-----------------------------|
| Metrics    | `prometheus-fastapi-instrumentator` |
| Tracing    | `opentelemetry-sdk` + `opentelemetry-instrumentation-fastapi` |
| Log format | JSON formatter (`python-json-logger`) |
| Health     | Extend `/health` to verify DB connectivity |

**Expected benefits:** P50/P99 latency tracking; request tracing across services; structured logs compatible with Datadog, Loki, or ELK.

---

### 5.5 Async Background Tasks

For operations that don't need to block the response (e.g., welcome emails, audit logging, event publishing), use FastAPI's `BackgroundTasks` or integrate a task queue:

**Tools:** FastAPI `BackgroundTasks` (simple cases), `arq` or `Celery` with Redis (heavy or scheduled tasks).

---

### 5.6 Database Performance

- Add composite indexes for common query patterns (e.g., `(is_active, role)` for admin user listings).
- Add `EXPLAIN ANALYZE` testing for all `find_all` queries with large datasets.
- Consider read replicas for reporting queries once load warrants it.

---

### 5.7 Security Hardening

- Enable HTTPS-only in production (TLS termination at load balancer).
- Set `Secure`, `HttpOnly`, and `SameSite=Strict` on cookies (if used).
- Add `Content-Security-Policy` and other security headers via middleware (`secure` library).
- Rotate JWT signing keys; implement key versioning.
- Add audit logging for sensitive mutations (user deletion, role changes).

---

## Summary Roadmap

| Phase | Focus                              | Duration   | Risk if Skipped |
|-------|------------------------------------|------------|-----------------|
| 1     | Quick wins, correctness fixes      | 1–2 days   | Low             |
| 2     | DB lifecycle, tests, error handling| 3–5 days   | 🔴 Critical      |
| 3     | API completeness, timezone, logging| 3–5 days   | Medium          |
| 4     | Auth, e-commerce features, DDD     | 2–4 weeks  | 🔴 Critical      |
| 5     | Observability, scaling, hardening  | 2–3 weeks  | Medium          |
