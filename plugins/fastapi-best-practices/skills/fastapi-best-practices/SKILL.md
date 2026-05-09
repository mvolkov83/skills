---
name: fastapi-best-practices
description: FastAPI production-grade best practices — covers project structure (domain-organized modules), async correctness (no blocking I/O in async paths, no CPU work in `BackgroundTasks`), dependency injection patterns with per-request caching and chaining, Pydantic v2 model design (split create/update/response, `ConfigDict`, validators), REST API conventions and status codes, Pydantic Settings v2 with `.env` and `lru_cache`, async database integration, and async-first testing with `httpx.AsyncClient`. Use this skill whenever the user is writing, reviewing, debugging, or designing FastAPI code — including any work involving `APIRouter`, `Depends`, `BackgroundTasks`, `BaseSettings`, Pydantic `BaseModel` for request/response schemas, async endpoint handlers, dependency injection chains, FastAPI middleware, OpenAPI/docs configuration, or testing FastAPI apps with `httpx` or `pytest`. Trigger even on small endpoint reviews or when the user just shows code importing from `fastapi` or `pydantic_settings` — the patterns matter on every change. Do NOT use for Flask, Django views, raw Starlette without FastAPI, or generic Python web work that doesn't use FastAPI specifically.
---

# FastAPI Best Practices (Production-Grade)

This skill captures production-tested defaults for FastAPI on Pydantic v2 + an async stack. It assumes async-first design (`async def` handlers, async DB drivers, `httpx.AsyncClient` for outbound calls and tests). When you spot Pydantic v1 syntax (`Config` inner class, `@validator`, `.dict()`, `.json()`), `requests` / sync DB calls inside `async def`, or `TestClient`-only tests in an async-first app, surface the modern equivalent and explain why.

For ORM and async session guidance referenced from rules 25 and 27, defer to the `sqlalchemy-best-practices` skill — this one stays focused on the FastAPI surface (routers, deps, Pydantic, settings, testing).

## Project Structure & Organization

1. Organize by **domain**, not by file type: `users/`, `products/`, `orders/` — not `models/`, `schemas/`, `routes/`.
   - **Why:** when one feature changes, all its code lives in one folder. Cross-cutting refactors stay local. At 50+ endpoints the file-type layout becomes a directory of unrelated files glued by Python's import system.
2. Each domain module contains `router.py`, `schemas.py`, `service.py`, `dependencies.py`. Keep the convention identical across modules so navigation is muscle-memory.
3. Keep `main.py` at the project root for app instantiation, router registration, and lifecycle hooks. Domain modules export their `APIRouter`; `main.py` mounts them.
4. `snake_case.py` for files, `PascalCase` for class names, `snake_case` for functions and module-level vars.

## Async Patterns

5. Use `async def` for any handler that performs I/O (database, HTTP, file, queue). Sync handlers run in a threadpool — fine for CPU-light pure-compute work, but mixed sync/async inside one app makes performance reasoning harder.
6. **Never block the event loop from async code.** Replace `time.sleep()` → `asyncio.sleep()`, `requests.get()` → `httpx.AsyncClient.get()`, sync DB sessions → `AsyncSession`. A single sync I/O call inside an `async def` freezes *every concurrent request* on the same worker until it returns.

   ```python
   # Wrong — blocks the event loop:
   @router.get("/data")
   async def get_data():
       return requests.get("https://api.example.com").json()  # sync HTTP

   # Correct:
   @router.get("/data")
   async def get_data(client: AsyncClient = Depends(get_http_client)):
       resp = await client.get("https://api.example.com")
       return resp.json()
   ```

7. For **CPU-intensive work** (image processing, ML inference, heavy crypto), do **not** use `BackgroundTasks`. Use a worker queue (Celery / RQ / arq) or `loop.run_in_executor()` with a `ProcessPoolExecutor`.
   - **Why:** `BackgroundTasks` run in the same event loop after the response is sent — they're for fire-and-forget I/O (sending email, structured logging). They do not escape the event loop. CPU-bound work there still serializes all incoming requests on the worker.
8. Run independent I/O concurrently with `await asyncio.gather(*tasks)`. Sequential `await`s for unrelated calls leave throughput on the table.

## Dependency Injection

9. Validate domain constraints (record exists, user has permission, foreign key valid) in **dependencies** — not just in Pydantic schemas. Pydantic enforces shape; only the database can answer "does user 42 exist?".
10. **Chain dependencies** instead of repeating the same lookup in every handler. Each handler asks for the most specific dependency it needs:
    ```python
    async def get_current_user(token: str = Depends(oauth2_scheme), db: AsyncSession = Depends(get_db)) -> User: ...
    async def get_active_user(user: User = Depends(get_current_user)) -> User: ...
    async def require_admin(user: User = Depends(get_active_user)) -> User: ...
    ```
11. FastAPI **caches dependencies per request** by default — if `get_current_user` is depended on by 5 sub-deps in one request, it runs once.
    - **Why:** this is the mechanism that makes deep dependency chains cheap. Override only when you genuinely want recomputation (rare): `Depends(my_dep, use_cache=False)`.
12. Prefer dependencies over middleware for request-specific logic. Middleware can't easily access path/query parameters, can't be applied per-route, and can't return typed values for other code to consume.

## Pydantic v2 Patterns

13. Define a project-wide base model with shared `model_config` and inherit from it. Single source of truth for `from_attributes`, `str_strip_whitespace`, JSON encoders.
14. Validation: `@field_validator` for single-field rules, `@model_validator(mode='after')` for cross-field invariants. Both run after type coercion — use `mode='before'` to massage raw input (empty string → `None`, dollars → cents).
15. `model_config = ConfigDict(from_attributes=True, str_strip_whitespace=True)` covers two common needs: `from_attributes` enables `Model.model_validate(orm_object)` (the v1 "ORM mode" replacement); `str_strip_whitespace` silently trims leading/trailing whitespace on all string fields, eliminating a class of form-input bugs.
16. **Split models per operation**: `UserCreate` (POST input, includes plaintext password), `UserUpdate` (PATCH input, all fields optional), `UserResponse` (excludes `password_hash` and other server-only fields).
    - **Why:** one model for everything leaks server-side fields into responses or forces clients to send fields they shouldn't know about. Splitting is FastAPI's serialization safety belt.

   ```python
   class UserBase(AppBaseModel):
       email: EmailStr
       full_name: str

   class UserCreate(UserBase):
       password: str = Field(min_length=8)

   class UserUpdate(AppBaseModel):
       email: EmailStr | None = None
       full_name: str | None = None

   class UserResponse(UserBase):
       id: int
       created_at: datetime
       # password_hash deliberately absent
   ```

## API Design

17. Follow REST conventions: `GET /items/{id}`, `POST /items`, `PUT /items/{id}` (full replace), `PATCH /items/{id}` (partial), `DELETE /items/{id}`.
18. Use specific status codes: `201` for created resources, `204` for no-content (DELETE, PATCH that returns nothing), `422` for Pydantic validation failures (FastAPI emits these automatically).
19. Standardize the error envelope as `{"detail": "..."}` — the FastAPI default. Custom error classes can extend with `code` / `field` for structured client handling, but keep `detail` present for compatibility.
20. Version with a path prefix: `/api/v1/`. Avoids breaking clients when the surface evolves; new versions live under `/api/v2/` alongside the old.

## Settings & Configuration

21. Use Pydantic Settings v2: `from pydantic_settings import BaseSettings`. The legacy `pydantic.BaseSettings` was extracted from `pydantic` core in v2 — it's a separate package now.
22. Load `.env` with `model_config = ConfigDict(env_file=".env", env_file_encoding="utf-8")`. Environment-specific files (`.env.test`, `.env.production`) selectable via `_env_file=` at instantiation.
23. **Validate all settings at startup**, not lazily. If `DATABASE_URL` is missing or malformed, the app should fail to boot — not crash on the first request. Instantiate the Settings object eagerly in `main.py`.
24. Wrap the settings getter in `@lru_cache`:

    ```python
    from functools import lru_cache

    @lru_cache
    def get_settings() -> Settings:
        return Settings()
    ```
    - **Why:** `Settings()` re-reads `.env` and re-validates on every call. Without caching, every dependency that uses `Depends(get_settings)` re-parses your env file on every request. `lru_cache` ensures one parse per process lifetime.

## Database & Performance

25. Configure connection pooling: `pool_size`, `max_overflow`, `pool_pre_ping=True`. Defer to the `sqlalchemy-best-practices` skill for full SQLAlchemy guidance — values like `pool_recycle=3600` and the `expire_on_commit=False` rationale live there.
26. Implement pagination on collection endpoints. Limit/offset is fine for small datasets; cursor-based pagination (encoded `created_at` + `id`) is required for large or rapidly-growing tables — `limit=20, offset>10000` makes the DB scan and discard 10k rows on every request.
27. Use SQLAlchemy 2.0+ patterns (async sessions, `select()`, `Mapped[T]`). The `sqlalchemy-best-practices` skill covers this end-to-end — invoke it for ORM-heavy work rather than duplicating guidance here.
28. Add indexes for columns used in WHERE filters and JOIN conditions. Composite indexes when multiple columns commonly co-filter; column order matters (most-selective first, or matching ORDER BY).

## Testing & Quality

29. Use `httpx.AsyncClient` (with `ASGITransport`) for test clients in async-first apps:

    ```python
    @pytest.fixture
    async def client(app):
        async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as ac:
            yield ac

    async def test_create_user(client):
        resp = await client.post("/api/v1/users", json={...})
        assert resp.status_code == 201
    ```
    `TestClient` works (it spins up a thread to drive the async stack), but `AsyncClient` exercises the same async path the real server uses and integrates cleanly with `pytest-asyncio` fixtures.
30. Test against a **real database** (testcontainers, ephemeral PG container) — not mocked ORM models. Mocked SQLAlchemy queries pass with invalid SQL undetected; real-DB tests catch schema drift, migration bugs, and constraint violations before prod.

## When applying these rules

- **Be opinionated about footguns** (#6 sync I/O in async, #7 CPU work in `BackgroundTasks`, #16 split models, #24 settings caching) — these are silent bugs or performance cliffs, not stylistic preferences.
- **Be flexible about structure** (#1-4 layout, #20 versioning prefix) — match the existing project's convention if it's working. Don't rewrite a flat structure into domain modules unless asked.
- **Read existing code first.** If the project has its own base model, settings module, or test client fixture, conform to it rather than introducing a parallel pattern.
- **Cross-skill awareness:** for SQLAlchemy / async session / ORM work referenced from rules 25 and 27, the `sqlalchemy-best-practices` skill is the deeper authority — defer to it.
