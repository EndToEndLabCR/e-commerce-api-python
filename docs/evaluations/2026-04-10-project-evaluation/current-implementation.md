# Current Implementation

A constructive analysis of what the project already does well.

---

## Architecture Overview

### Layering

The project follows a strict **Clean Architecture** layering enforced through directory convention:

```
src/
├── shared/                     # Cross-cutting kernel
│   ├── domain/                 # Base contracts: BaseEntity, BaseRepository, EntityId
│   ├── application/            # Reserved for shared app-level concerns
│   ├── infrastructure/         # Postgres engine, base ORM model
│   ├── presentation/           # Reserved for shared web concerns
│   └── utils/                  # Logging, retry, date, config utilities
└── app/
    ├── config/                 # AppConfig singleton, YAML config files, Paths
    ├── features/
    │   ├── template/           # Empty scaffold for new features
    │   └── user/               # Fully implemented feature
    │       ├── domain/         # Entities, value objects, repository interface
    │       ├── application/    # Use cases, DTOs, exceptions, service
    │       ├── infrastructure/ # SQLAlchemy models, repository implementation, mapper
    │       └── presentation/   # FastAPI routes, dependency injection
    └── app.py                  # FastAPI app assembly
```

Each layer depends only inward: the domain has no framework dependencies; application depends only on the domain; infrastructure implements domain interfaces; presentation depends on application DTOs and services.

### Module / Package Structure

Every feature is self-contained. `shared/` provides the interfaces and cross-cutting utilities that features import but never extend in a way that creates circular dependencies. This means a new feature like `product/` can be added by copying the `template/` scaffold and filling it in without touching any existing code.

### Dependency Flow

```
HTTP Request
    → Presentation (FastAPI router + Depends)
        → Application (UserService → Use Cases)
            → Domain (Entities, Value Objects, Repository interface)
            ← Infrastructure (UserRepositoryImpl implements UserRepository)
                ← Shared (AsyncSession from PostgresDbConnection)
```

The `UserRepository` abstract class lives in the domain layer; the concrete `UserRepositoryImpl` lives in infrastructure. FastAPI's dependency injection wires them together at runtime via `dependencies.py`, keeping the application layer free of framework concerns.

---

## DDD Implementation

### Entities

`BaseEntity` (shared) establishes the identity contract — an `EntityId` (a UUID value object), `created_at`, and `updated_at`. `UserEntity` extends it, adding the fields specific to the user aggregate.

```python
class UserEntity(BaseEntity):
    email: Email
    first_name: str
    last_name: str
    hashed_password: HashedPassword
    role: Role
    is_active: bool
```

### Value Objects

All three user-specific value objects are properly immutable (`@dataclass(frozen=True)`):

| Value Object    | Validation                                              |
|-----------------|---------------------------------------------------------|
| `Email`         | Regex pattern validation in `__post_init__`            |
| `HashedPassword`| Validates bcrypt hash prefix and length; `from_plain_text` factory hashes safely; `verify()` for comparison |
| `Role`          | Enum-based; `from_str` normalizes case; `is_admin` / `is_user` properties |
| `EntityId`      | Wraps `UUID`; enforces `isinstance` check; `from_string` and `generate` factory methods |

`HashedPassword.__str__` returns `"********"` and `__repr__` returns `"HashedPassword(********)"`, preventing accidental logging of hashed values.

### Repositories

`BaseRepository[T, ID]` (abstract, generic) defines the standard CRUD contract: `save`, `find_by_id`, `find_all`, `exists`, `update`, `delete`. `UserRepository` extends this with a domain-specific `find_by_email(email: Email)` method. `UserRepositoryImpl` provides the SQLAlchemy async implementation. The domain never imports SQLAlchemy — infrastructure is a plugin.

### Ubiquitous Language

Naming is consistent and domain-aligned:
- `UserEntity`, `UserRepository`, `UserRepositoryImpl`, `UserService`, `SaveUser`, `GetUserByIdUseCase`, `DeleteUserUseCase`, `UpdateUserUseCase`
- Value object names (`Email`, `Role`, `HashedPassword`) read naturally in context
- DTO names (`UserCreate`, `UserUpdate`, `UserResponse`) clearly communicate intent

---

## Design Patterns Used

| Pattern              | Where                                                                 |
|----------------------|-----------------------------------------------------------------------|
| **Repository**       | `BaseRepository` + `UserRepository` (domain) / `UserRepositoryImpl` (infra) |
| **Factory Method**   | `EntityId.generate()`, `HashedPassword.from_plain_text()`, `EntityId.from_string()` |
| **Singleton**        | `AppConfig.instance()` with thread-safe double-checked locking        |
| **Mapper / Assembler** | `map_model_to_entity()`, `map_entity_to_dto_user()` explicit mapping functions |
| **Use Case / Command** | One class per operation: `SaveUser`, `GetUserByIdUseCase`, `DeleteUserUseCase`, `UpdateUserUseCase` |
| **Dependency Injection** | FastAPI `Depends()` wires repositories and services per request    |
| **Template Method**  | `BaseRepository` defines the interface; subclasses fill in the steps  |
| **Decorator**        | `@retry_on_exception`, `@retry_read_operation`, etc. via `backoff`    |

---

## Readability

### Naming Clarity

Class, method, and variable names are specific and self-documenting. `SaveUser.execute(user_create: UserCreate)` reads like natural language. The module names mirror the directory structure, so tracing a call from `user_routes.py → user_service.py → save_user.py → user_repository.py → user_repository_impl.py` is linear and intuitive.

### Code Organization

Files are small and single-purpose. `save_user.py` contains exactly one class with one public method. Route handlers delegate immediately to the service layer — no business logic leaks into routes. The mapper functions (`user_model_mapper.py`, `user_dto_mapper.py`) are isolated in dedicated files rather than scattered across layers.

### Simplicity vs Complexity Balance

The complexity budget is appropriate for the current scope. The DDD constructs (value objects, repositories) add meaningful conceptual clarity without over-engineering. The retry decorators abstract repetitive backoff logic cleanly. Configuration loading via `pyaml_env` supports env-var interpolation in YAML without custom parsing code.

---

## Maintainability

### Modularity

The feature-per-module structure means changes to `user` do not affect future features like `product` or `order`. The shared kernel is limited to pure contracts and utilities, which are unlikely to change frequently.

### Separation of Concerns

- Route handlers handle HTTP serialization and status codes; nothing else.
- Use cases handle one operation each.
- Value objects enforce their own invariants.
- Infrastructure handles persistence details.
- `AppConfig` handles environment-specific configuration loading.

### Ease of Extending Features

Adding a new feature requires:
1. Copy `template/` to `features/new_feature/`
2. Define domain entities, value objects, and repository interface
3. Implement use cases
4. Implement the SQLAlchemy model and repository
5. Register the router in `app.py`

The pattern is explicit enough that a new team member can follow it without guessing.

---

## Scalability (Existing Strengths)

### Decoupling

The domain layer is completely decoupled from frameworks. Swapping FastAPI for another web framework, or SQLAlchemy for another ORM, would require changes only in the presentation and infrastructure layers.

### Statelessness

Route handlers do not maintain state between requests. All state is in the database. This is a prerequisite for horizontal scaling.

### Async Handling

The entire stack — from FastAPI route handlers through SQLAlchemy sessions to database driver (asyncpg) — is non-blocking. Under I/O-bound load, this allows a single Gunicorn worker to serve many concurrent requests without thread-based bottlenecks.

---

## Python Best Practices

- **Type hints** are used consistently in domain, application, and infrastructure layers.
- **Dataclasses** (`@dataclass(frozen=True)`) are used correctly for value objects, providing `__eq__`, `__hash__`, and immutability.
- **Abstract base classes** (`ABC`, `@abstractmethod`) correctly define the repository contract.
- **Generics** (`TypeVar`, `Generic[T, ID]`) enable type-safe repository operations.
- **`__str__`/`__repr__` overrides** protect sensitive data from accidental logging.
- **Enums** (`Role`) are preferred over string constants.
- **`dotenv`** separates secrets from source code.
- **`pyaml_env`** allows YAML config to reference environment variables without custom parsing.

---

## Framework Usage

FastAPI is used appropriately:
- **`APIRouter`** organises routes per feature.
- **`Depends()`** wires services and sessions without manual construction.
- **`response_model`** declarations enforce output serialization.
- **Pydantic v2** DTOs with `alias_generator=to_camel` provide camelCase JSON automatically.
- **`status` constants** (`HTTP_200_OK`, `HTTP_201_CREATED`, `HTTP_204_NO_CONTENT`) are used explicitly.

---

## Testing Strategy

`pytest==7.4.2` is listed as a dependency, signalling intent to test. No test files currently exist, but the architecture is highly testable:
- Use cases accept repository interfaces — trivially mockable.
- Value objects are pure Python with no side effects — simple unit tests.
- Route handlers can be tested with FastAPI's `TestClient` using an in-memory or test database.

The `config_test.yml` file exists, indicating the team has thought about a test configuration profile.

---

## Observability and Operations

- **Logging** – `log_util.py` provides a named logger with configurable level via `LOG_LEVEL` env var and a consistent format (`%(levelname)s %(asctime)s - %(message)s`).
- **Configuration** – Three YAML profiles (`local`, `docker`, `test`) keep environment differences explicit.
- **Retry** – `retry_on_exception`, `retry_read_operation`, `retry_write_operation`, `retry_critical_operation` decorators provide tunable resilience for external calls.
- **Health check** – A `/health` endpoint is present for load balancer probes.
- **Docs gating** – Swagger and ReDoc are disabled in non-local environments.
- **Migrations** – Alembic is configured, providing a controlled database change management process.
- **Container** – `Dockerfile` + `compose.yml` enable consistent local development and production parity.

---

## What Should Be Preserved and Reinforced

1. **The directory structure** – it is clean, discoverable, and scalable. Do not collapse layers to save files.
2. **Value object design** – frozen dataclasses with validation in `__post_init__` are the correct approach.
3. **`HashedPassword` encapsulation** – password logic belongs in the value object; keep it there.
4. **Use case per operation** – single-responsibility use cases are easy to test and reason about.
5. **Mapper pattern** – explicit mapping functions over ORM relationship magic or serializer annotations.
6. **`AppConfig` singleton** – the YAML + env var approach is flexible; preserve the pattern when extending configuration.
7. **Retry decorators** – the separation of read, write, and critical retry strategies is a good operational concern.
8. **Async-first** – maintain async throughout; do not introduce synchronous database calls.
