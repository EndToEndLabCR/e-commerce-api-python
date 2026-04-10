# Issues and Improvements

A critical analysis of the current weaknesses, ordered by impact.

---

## 1. Database Engine Instantiated Per HTTP Request

**File:** `src/app/features/user/presentation/web/dependencies.py`

**Description:**
`get_database_session()` calls `PostgresDbConnection(postgres_config)` on every request, which internally calls `create_async_engine(...)` and `async_sessionmaker(...)`. SQLAlchemy engines are expensive objects that manage a connection pool. Creating a new one per request bypasses pooling entirely, opens a fresh TCP connection to Postgres for each call, and leaks engine objects that are never disposed.

```python
# Current – wrong: new engine on every request
async def get_database_session() -> AsyncGenerator[Any, Any]:
    postgres_config = get_config_value(app_config, "postgres", {})
    postgres_db_session_manager = PostgresDbConnection(postgres_config)
    async with postgres_db_session_manager.get_session() as session:
        yield session
```

**Why it matters:** Under any non-trivial load, this exhausts Postgres connection limits and introduces significant per-request latency.

**Impact:** 🔴 High

**Suggested direction:** Initialise `PostgresDbConnection` once at application startup using FastAPI's `lifespan` context manager. Store the session factory in application state and yield sessions from the shared factory.

---

## 2. No Test Suite

**Description:**
`pytest` is listed in `requirements.txt` but there are zero test files in the repository. No unit tests, no integration tests, no end-to-end tests exist.

**Why it matters:** Without tests, any refactoring is high-risk, regressions cannot be detected automatically, and confidence in the correctness of value object validations, use case logic, and route behaviour depends entirely on manual testing.

**Impact:** 🔴 High

**Suggested direction:**
- Unit tests for all value objects (`Email`, `HashedPassword`, `Role`, `EntityId`) — these are pure Python with no I/O.
- Unit tests for each use case using mock repositories.
- Integration tests for routes using `httpx.AsyncClient` + `TestClient` against a real test database configured via `config_test.yml`.

---

## 3. No Authentication or Authorization

**Description:**
There are no login or token endpoints, no JWT validation middleware, and no route-level access control. All four `user` CRUD endpoints are fully public.

**Why it matters:** Any user data can be read, modified, or deleted by an unauthenticated caller. This is a critical security gap that makes the API production-unsuitable.

**Impact:** 🔴 High

**Suggested direction:**
- Add a `POST /v1/auth/login` endpoint returning a signed JWT.
- Implement a FastAPI `Depends` dependency that validates the JWT and injects the current user.
- Add role-based access control (e.g., only `ADMIN` can delete users).
- Store refresh tokens or use short-lived access tokens + refresh token rotation.

---

## 4. No Global Exception Handler

**File:** `src/app/app.py`

**Description:**
Domain exceptions (`UserDoesNotExistException`, `ValueError` from value objects) are raised by use cases but not caught at the presentation boundary. FastAPI will return a 500 with a Python traceback in development mode, or an opaque 500 in production.

**Why it matters:** Clients receive unusable error responses. Stack traces may leak internal details. HTTP status semantics are violated (a missing user should be 404, not 500).

**Impact:** 🔴 High

**Suggested direction:**
Register `@app.exception_handler` functions (or use `ExceptionMiddleware`) to map:
- `UserDoesNotExistException` → `HTTP 404`
- `ValueError` (invalid UUID, validation) → `HTTP 422`
- `IntegrityError` (duplicate email) → `HTTP 409`
- Unhandled `Exception` → `HTTP 500` with a safe generic message

---

## 5. Domain Entity Directly Mutated in Use Case

**File:** `src/app/features/user/application/use_cases/update_user.py`

**Description:**
`UpdateUserUseCase.execute()` fetches a `UserEntity` from the repository and then mutates its fields directly:

```python
existing_user.email = Email(user_update.email)
existing_user.first_name = user_update.first_name
```

**Why it matters:** DDD entities should update themselves through behaviour methods that enforce invariants and can emit domain events. Direct attribute assignment from outside the entity bypasses any future invariant checks. It also ties the use case tightly to the internal field names of the entity.

**Impact:** 🟡 Medium

**Suggested direction:** Add an `update(...)` method to `UserEntity` that accepts the changeable fields and applies them internally, allowing the entity to validate the transition.

---

## 6. `UserService` Is a Shallow Pass-Through

**File:** `src/app/features/user/application/services/user_service.py`

**Description:**
`UserService` constructs a use case instance on every call and delegates immediately. It adds no orchestration logic, no transaction management, and no cross-cutting concern.

```python
async def get_user_by_id(self, user_id: str):
    use_case = GetUserByIdUseCase(self.user_repository)
    return await use_case.execute(user_id)
```

**Why it matters:** This is the "service as dispatcher" anti-pattern. It adds an indirection layer without value. Use cases already encapsulate the full operation; routing through a service that only re-instantiates them obscures the code path without benefit.

**Impact:** 🟡 Medium

**Suggested direction:** Either eliminate `UserService` and inject use cases directly into routes, or give `UserService` a real responsibility — such as coordinating multiple use cases within a single transaction, or enforcing cross-cutting concerns like rate limiting.

---

## 7. Incomplete Feature Set for an E-Commerce API

**Description:**
Only the `user` feature is implemented. The project is called `e-commerce-api-python` but contains none of the core e-commerce entities: products, categories, inventory, cart, orders, payments, or shipping.

**Why it matters:** The system cannot fulfil its stated purpose. The `user` feature alone is a user management service, not an e-commerce API.

**Impact:** 🔴 High (for completeness / usefulness)

**Suggested direction:** Implement features following the established template: `product`, `category`, `cart`, `order`, and `payment`, each with their own domain entities, value objects, use cases, and routes.

---

## 8. `BaseModel` Uses `uuid.uuid1` for Primary Keys

**File:** `src/shared/infrastructure/postgres/models/base_model.py`

**Description:**
```python
id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid1, ...)
```
`uuid1` generates UUIDs based on the MAC address and current timestamp. This leaks host identity, creates identifiable patterns, and is inconsistent with `EntityId.generate()` which uses `uuid4`.

**Why it matters:** UUID v1 leaks infrastructure details (MAC address) and generates UUIDs with a predictable time component that can be sequenced by an attacker. The mismatch with `EntityId.generate()` (uuid4) is a latent bug — if the ORM ever generates an ID independently, it will differ from what the domain would generate.

**Impact:** 🟡 Medium

**Suggested direction:** Replace `default=uuid.uuid1` with `default=uuid.uuid4`.

---

## 9. Timezone-Naive Datetimes

**Files:** `src/shared/utils/date_util.py`, `src/shared/infrastructure/postgres/models/base_model.py`

**Description:**
`datetime.now()` returns a naive datetime (no timezone info). Postgres stores timestamps in UTC by default; comparing a naive Python datetime with a UTC-stored timestamp can produce incorrect orderings.

```python
def get_current_datetime() -> datetime:
    return datetime.now()  # naive — no timezone
```

**Why it matters:** Datetime arithmetic, sorting, and comparison across timezones will silently produce wrong results. This is a common source of difficult-to-diagnose bugs.

**Impact:** 🟡 Medium

**Suggested direction:** Replace `datetime.now()` with `datetime.now(timezone.utc)` and update the return type to `datetime` with `tzinfo`. Update `BaseModel` column defaults similarly.

---

## 10. Docstrings Commented Out in Route Handlers

**File:** `src/app/features/user/presentation/web/routes/user_routes.py`

**Description:**
The docstrings for `get_user_by_id` and `create_user` are commented out using `#`:

```python
@router.get("/{user_id}", ...)
async def get_user_by_id(user_id: UUID, ...) -> UserResponse:
    # """
    # Get a user by their ID.
    # ...
    # """
```

**Why it matters:** Commented-out docstrings are invisible to Python's `help()`, IDE introspection tools, and documentation generators. FastAPI uses the function docstring to populate the OpenAPI schema's operation description. With the docstrings commented out, the Swagger UI shows no description for these endpoints.

**Impact:** 🟢 Low

**Suggested direction:** Uncomment the docstrings.

---

## 11. Overly Permissive CORS Policy

**File:** `src/app/app.py`

**Description:**
```python
allow_methods=["*"]  # Allows all methods
allow_headers=["*"]  # Allows all headers
```
All HTTP methods and all headers are allowed from `http://localhost`. In production, this should be restricted to the actual methods and headers used by the frontend.

**Why it matters:** An overly broad CORS policy can enable cross-origin requests that should be blocked, contributing to CSRF exposure surface area.

**Impact:** 🟡 Medium

**Suggested direction:** Restrict `allow_methods` to `["GET", "POST", "PATCH", "DELETE"]` and `allow_headers` to the specific headers your API requires (e.g., `["Content-Type", "Authorization"]`).

---

## 12. `get_config_value` Docstring Has Swapped Parameter Descriptions

**File:** `src/shared/utils/config_util.py`

**Description:**
```python
def get_config_value(config: dict, key: str, ...):
    """
    :param key: The value with validation and casting.       # ← actually describes 'config'
    :param config: Tey to fetch from the configuration.     # ← actually describes 'key'; "Tey" is a typo for "Key"
    """
```
The `key` and `config` parameter descriptions are swapped, and `config`'s description contains a typo: "Tey" should be "Key".

**Why it matters:** Misleads contributors reading the function signature.

**Impact:** 🟢 Low

**Suggested direction:** Fix the parameter order and correct the typo.

---

## 13. Password Maximum Length Is Too Restrictive

**File:** `src/app/features/user/domain/value_objects/hashed_password.py`

**Description:**
```python
_MAX_PASSWORD_LENGTH = 16
```
A 16-character maximum password length is significantly below current NIST SP 800-63B recommendations (minimum 64 characters for user-chosen secrets). It forces users towards weaker passwords and violates modern security guidelines.

**Why it matters:** A 16-character maximum is considered a security anti-pattern. It frustrates users who use a password manager and creates artificial ceiling on password entropy.

**Impact:** 🟡 Medium

**Suggested direction:** Raise `_MAX_PASSWORD_LENGTH` to at least 64 or 128 characters. The bcrypt 72-byte limit applies at the byte level; the application-level character check should allow longer passphrases.

---

## 14. `find_by_name` Is Implemented but Never Called

**File:** `src/app/features/user/infrastructure/postgres/repository/user_repository_impl.py`

**Description:**
`UserRepositoryImpl` implements a `find_by_name` method that is not declared in the `UserRepository` interface and has no corresponding use case or route.

**Why it matters:** Dead code increases cognitive load. It is also architecturally incorrect to add methods to a concrete repository that are not part of the domain interface — this leaks infrastructure concerns upward.

**Impact:** 🟢 Low

**Suggested direction:** Either add `find_by_name` to the `UserRepository` interface (if needed) and create a corresponding use case, or remove the method.

---

## 15. No Pagination Endpoint for Users

**Description:**
`BaseRepository` defines `find_all(limit, offset)` and `UserRepositoryImpl` implements it, but no route exposes this operation. There is no `GET /v1/user` endpoint.

**Why it matters:** Without a list endpoint, the API cannot support any admin or management UI.

**Impact:** 🟡 Medium

**Suggested direction:** Add `GET /v1/user?limit=20&offset=0` returning a paginated list of `UserResponse`. Introduce a `GetAllUsersUseCase`.

---

## 16. No Structured Logging or Request Correlation IDs

**Description:**
The current logger uses a plain text format with no request ID, correlation ID, or trace context. Log lines from concurrent requests are interleaved with no way to associate them.

**Why it matters:** In production, diagnosing multi-step request failures requires log correlation. Without it, debugging requires manual timestamp matching.

**Impact:** 🟡 Medium

**Suggested direction:** Add a FastAPI middleware that generates a UUID request ID per request, stores it in a `contextvars.ContextVar`, and includes it in every log line produced during that request's lifecycle. Use a JSON log formatter for compatibility with log aggregators (Datadog, Loki, ELK).

---

## 17. Return Type Annotations Missing on `UserService` Methods

**File:** `src/app/features/user/application/services/user_service.py`

**Description:**
`get_user_by_id` and `delete_user` lack return type annotations:

```python
async def get_user_by_id(self, user_id: str):   # missing -> UserResponse
async def delete_user(self, user_id: str) -> bool:  # this one is present
```

**Why it matters:** Incomplete type annotations reduce IDE assistance, make the API surface less discoverable, and can allow type errors to go undetected.

**Impact:** 🟢 Low

**Suggested direction:** Annotate all method return types.

---

## 18. `is_active` Column Allows NULL in Database

**File:** `src/app/features/user/infrastructure/postgres/models/user_model.py`

**Description:**
```python
is_active = Column(Boolean)  # nullable=True by default
```
`is_active` has no `nullable=False` constraint, so the column can store `NULL`, which would require three-valued boolean logic (`True`, `False`, `None`) in the application.

**Why it matters:** A boolean flag used to enable/disable accounts should never be `NULL`. A `NULL` `is_active` cannot be reliably evaluated by business logic.

**Impact:** 🟡 Medium

**Suggested direction:** Add `nullable=False, default=True` to the column definition.
