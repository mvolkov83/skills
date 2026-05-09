---
name: sqlalchemy-best-practices
description: SQLAlchemy 2.0 async-first best practices — covers engine and pool config, model design with `Mapped[T]` type annotations, relationship loading strategies (`selectin` / `select` / `write_only`), query optimization, async session and transaction patterns, and FastAPI dependency injection. Use this skill whenever the user is writing, reviewing, debugging, or designing SQLAlchemy code — including any work involving `AsyncSession`, `async_sessionmaker`, `Mapped`, `mapped_column`, `select()`, `selectinload`, `joinedload`, relationships, ORM models, Alembic migrations using `Mapped[T]` syntax, FastAPI endpoints with database dependencies, or performance issues like N+1 queries, lazy-loading errors in async contexts, or session-lifecycle bugs. Trigger even on small reviews or when the user just shows code importing from `sqlalchemy` or `sqlalchemy.ext.asyncio` — the patterns matter on every change. Do NOT use for Django ORM, Tortoise ORM, Peewee, or raw `asyncpg` / `psycopg` without an ORM.
---

# SQLAlchemy Best Practices (Async-First)

This skill captures production-tested defaults for SQLAlchemy 2.0 with async drivers. It assumes 2.0+ syntax (`Mapped[T]`, `mapped_column()`, `select()`) and an async stack (`asyncpg`, `aiomysql`, `aiosqlite`). When you spot SQLAlchemy 1.x patterns (`Column()`, `Query` API, sync sessions) or unsafe async usage, suggest the 2.0 equivalent and explain why.

The numbered rules below mirror the user's canonical reference. Imperative form. **Why:** lines explain rationale where the rule has a footgun or non-obvious motivation — apply judgment when the rule meets an edge case rather than blindly following.

## Core Setup & Configuration

1. Use SQLAlchemy 2.0+ with async support: `sqlalchemy[asyncio]` + `asyncpg` / `aiomysql`.
2. Configure async engine: `create_async_engine()` with `echo=False` in production.
3. Connection pool: `pool_size=20, max_overflow=10, pool_pre_ping=True, pool_recycle=3600`.
   - **Why:** `pool_pre_ping` catches stale connections (DB restarts, network blips) before query time. `pool_recycle` mitigates idle-timeout disconnects from PG/MySQL/proxies.
4. Use `async_sessionmaker` for the session factory. **Never share an `AsyncSession` between concurrent tasks.**
   - **Why:** `AsyncSession` is not concurrency-safe. Sharing across `asyncio.gather()` branches leads to `IllegalStateChangeError`, interleaved transactions, or silent data corruption. Each concurrent task must take its own session from the factory.
5. Enable query statistics in dev: `echo=True`, `echo_pool="debug"`. Off in prod.

## Model Design & Structure

6. Inherit from `DeclarativeBase` with type annotations for columns.
7. Use `mapped_column()` with explicit types: `mapped_column(String(100), nullable=False)`.
8. Primary keys: `BigInteger` for auto-increment IDs by default; UUIDs for distributed systems.
   - **Why:** 4-byte `Integer` overflows at ~2.1B rows — cheap to future-proof now, painful to migrate later.
9. Indexes: composite indexes for common WHERE/JOIN columns. Profile actual queries before adding.
10. Constraints belong at the database level — `UniqueConstraint`, `CheckConstraint`, `ForeignKeyConstraint`.
    - **Why:** application-level checks lose under race conditions (two requests pass the check, both insert). The DB is the single source of truth for invariants.

## Relationships & Loading

11. Default to `lazy="selectin"` for small related sets, `lazy="select"` for large.
    - **Why:** in async contexts, `lazy="select"` (the SQLAlchemy default) emits implicit IO when the attribute is accessed *outside* an awaitable context — raises `MissingGreenlet`. `selectin` pre-fetches via a second query, no implicit IO. For large collections you typically want explicit query-time loading instead (see #13).
12. Use `selectinload()` at query time for eager loading. **Avoid `joinedload()` for collections (one-to-many / many-to-many).**
    - **Why:** `joinedload()` on a collection produces row duplication (Cartesian fan-out) — N×M rows for N parents and M children each. SQLAlchemy de-duplicates in Python but you transferred the duplicate bytes over the wire. `selectinload()` issues a second query with `WHERE id IN (...)`, deterministic and bounded.

    ```python
    # Avoid for collections:
    stmt = select(User).options(joinedload(User.posts))

    # Prefer:
    stmt = select(User).options(selectinload(User.posts))

    # joinedload IS correct for to-one (no fan-out):
    stmt = select(Post).options(joinedload(Post.author))
    ```

13. Implement `lazy="write_only"` for very large collections — use `WriteOnlyCollection`.
    - **Why:** prevents accidental `user.posts` materialization that loads 100k rows into memory. Forces explicit query (`select(Post).where(Post.user_id == user.id)`).
14. Use `back_populates` (explicit) over `backref` (implicit) for bidirectional relationships.
15. Cascade carefully: `cascade="all, delete-orphan"` only for true compositions (child cannot exist without parent).
    - **Why:** delete-orphan triggers when an item is removed from the collection — not just when the parent is deleted. Easy to wipe data unintentionally if relationship is associative rather than composite.

## Query Optimization

16. Use `select()` with explicit columns to avoid loading unnecessary data.
17. Use `select(exists().where(...))` for existence checks instead of `select(func.count(...))`.
    - **Why:** `count(*)` scans the whole matching set; `exists()` short-circuits on first hit.
18. Batch with `in_()` operator: `select(Model).where(Model.id.in_(ids))` rather than N round-trips.
19. Use `bulk_insert_mappings()` for raw inserts; `bulk_save_objects()` only when you understand the trade-offs.
    - **Why:** both bypass ORM events, relationship handling, and identity map. Speed gain is real; data integrity is your responsibility.
20. Raw SQL for complex queries: `await session.execute(text("SELECT ..."), {"param": value})`. Always parameterized — never f-string interpolation.

## Session & Transaction Patterns

21. Always use a context manager: `async with async_session() as session:`. Guarantees close.
22. Explicit transactions: `async with session.begin():` for auto-commit/rollback semantics.
23. **Use `expire_on_commit=False` when configuring the sessionmaker if the application reads ORM attributes after commit.**
    - **Why:** with the default `expire_on_commit=True`, every committed object has its attributes expired and the next access triggers a lazy reload — which in async fails with `MissingGreenlet` outside an awaitable context. This is the single most common async-SQLAlchemy footgun.

    ```python
    async_session = async_sessionmaker(
        engine,
        expire_on_commit=False,  # critical for async + post-commit attribute access
        class_=AsyncSession,
    )
    ```

24. Optimistic locking via `version_id_col` for concurrent updates of the same row.
    - **Why:** prevents lost-update without taking a row-level lock. SQLAlchemy bumps the version on UPDATE; mismatched version raises `StaleDataError`.
25. Set isolation level per transaction when needed: `connection.execution_options(isolation_level="SERIALIZABLE")`. Default `READ COMMITTED` is fine for most reads.

## Async Patterns & Performance

26. Concurrent independent queries: `await asyncio.gather(*[session.execute(q) for q in queries])` — but **each branch needs its own session** (see #4). For genuine parallelism use multiple sessions, one per task.
27. Stream large results: `result = await session.stream(stmt)`; iterate with `async for row in result.yield_per(100)`.
28. Monitor pool health in observability: `pool.size()`, `pool.checked_in()`, `pool.overflow()`.
29. Use `AsyncConnection` directly for bulk raw operations bypassing ORM overhead (no identity map, no flush).
30. Implement retry with exponential backoff for transient errors (deadlock, serialization failure, connection drop). Don't retry constraint violations.

## Dependency Injection Pattern (FastAPI)

31. Create a reusable `SessionDependency` class with configurable commit behavior — single source of truth for session lifecycle.
32. **Separate dependencies for reads and writes**: `get_db` (read-only, no commit) and `get_db_with_commit` (writes, commits at end if no exception).
    - **Why:** read endpoints should not implicitly commit (no-op writes still take a flush + transaction round-trip); write endpoints want commit-on-success / rollback-on-exception without each handler reimplementing it. Mixing them encourages copy-paste of `try/except/commit/rollback` blocks.
33. Implement the callable protocol: `async def __call__(self) -> AsyncIterator[AsyncSession]`. FastAPI handles the generator lifecycle.
34. Guarantee cleanup: rollback on exception, always close the session in `finally`. The async generator's `try/finally` is the right place.
35. Usage in handlers: `async def endpoint(db: AsyncSession = Depends(get_db_with_commit))`.

```python
# Skeleton — adapt to your project's settings/factory:
class SessionDependency:
    def __init__(self, *, commit: bool):
        self._commit = commit

    async def __call__(self) -> AsyncIterator[AsyncSession]:
        async with async_session() as session:
            try:
                yield session
                if self._commit:
                    await session.commit()
            except Exception:
                await session.rollback()
                raise

get_db = SessionDependency(commit=False)
get_db_with_commit = SessionDependency(commit=True)
```

## When applying these rules

- **Be opinionated about footguns** (#4, #11, #12, #23) — these are silent failures in async, not stylistic preferences.
- **Be flexible about defaults** (#3 pool sizes, #8 BigInteger, #25 isolation level) — these are starting points; adjust to the project's load profile.
- **Read existing code first.** If the project already has a `SessionDependency` or sessionmaker config, conform to it rather than introducing a parallel pattern.
- **Flag SQLAlchemy 1.x style proactively.** If you see `Column()`, `session.query(Model)`, or sync sessions in an async project, surface the 2.0 equivalent — but don't refactor unrelated code in the same change unless asked.
