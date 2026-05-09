---
name: pytest-best-practices
description: pytest async-first testing best practices for Python backends — covers test layer structure (unit / integration / e2e mirroring `app/`), fixture design with proper scope and yield-based teardown, async testing with `pytest-asyncio` (and `asyncio_mode = "auto"` config), test-data generation with `factory_boy` + `faker`, HTTP mocking with `respx` for `httpx` clients, time mocking with `freezegun`, property-based testing with `hypothesis`, parametrize patterns with `ids=` and `pytest.param`, database test isolation via transaction rollback, the `flush()` pattern for chained DB fixtures under parallel `pytest-xdist` execution, coverage configuration, and CI-friendly defaults. Use this skill whenever the user is writing, reviewing, debugging, or designing pytest tests — including any work involving `@pytest.fixture`, `@pytest.mark.asyncio`, `@pytest.mark.parametrize`, `conftest.py`, async fixtures, factory_boy factories, `respx` mocks, `freezegun`, `hypothesis` strategies, `pytest-xdist` parallelism, `pytest-cov` coverage, or test isolation patterns. Trigger even when the user just shows code under `tests/` or imports from `pytest`, `pytest_asyncio`, `factory`, `faker`, `respx`, `freezegun`, or `hypothesis`. Do NOT use for `unittest` stdlib tests, `nose`, `doctest`, or generic Python questions unrelated to testing.
---

# pytest Best Practices (Async-First, Backend-Focused)

This skill captures production-tested defaults for pytest with async support, factory-driven test data, real-DB integration testing, and CI-friendly configuration. It assumes the test target is a Python backend (FastAPI, SQLAlchemy, async services). UI / Selenium / Playwright e2e patterns are out of scope.

For HTTP-side test client patterns in FastAPI apps, defer to the `fastapi-best-practices` skill (rule 29 covers `httpx.AsyncClient` setup with `ASGITransport`). For DB fixture details specific to async SQLAlchemy 2.0 sessions and `expire_on_commit=False` semantics, cross-reference the `sqlalchemy-best-practices` skill — this one stays focused on pytest mechanics.

## Stack

The rules below assume:
- `pytest` + `pytest-asyncio` for the async event loop
- `factory_boy` + `faker` for test data generation
- `pytest-mock` for the `mocker` fixture (cleaner than raw `unittest.mock`)
- `respx` for `httpx` HTTP mocking
- `freezegun` for time mocking
- `hypothesis` for property-based testing
- `pytest-xdist` for parallel runs
- `pytest-cov` for coverage
- `pytest-benchmark` for performance regressions

## Test Layer Structure

Mirror the app structure under `tests/`:

```
app/services/payment.py  →  tests/unit/services/test_payment.py
app/api/users.py          →  tests/integration/api/test_users.py
```

Three layers, progressively wider scope:
- `tests/unit/` — fast, isolated, mock everything external. Target <1s per file.
- `tests/integration/` — real database (testcontainers / ephemeral PG), real ORM, mocked external HTTP.
- `tests/e2e/` — full stack including external services (or recorded fixtures via `respx`).

Run them in CI in order: unit first (fail fast), then integration, then e2e. Use `-m unit` markers for gating.

## Fixtures & Isolation

1. Pick fixture scope deliberately: `function` (default, fresh per test), `module` (shared across file), `session` (one per test run).
   - **Why:** wrong scope is a classic footgun. `session`-scoped fixtures with mutable state leak between tests; `function`-scoped fixtures for expensive resources (DB engines, HTTP clients) make the suite slow. Default to `function`; promote to `module`/`session` only for genuinely immutable / setup-heavy resources.
2. Always clean up — yield-based fixtures with explicit teardown, or DB transaction rollback (see #5). Never rely on test order to cleanup.
3. Use `pytest-mock`'s `mocker` fixture over raw `unittest.mock.patch`. Auto-cleanup at test end, less boilerplate, plays well with async.
4. Mock outbound HTTP with `respx`, not generic mocks:

   ```python
   async def test_fetch_user(respx_mock):
       respx_mock.get("https://api.example.com/users/42").mock(
           return_value=Response(200, json={"id": 42, "name": "Alice"})
       )
       result = await fetch_user(42)
       assert result.name == "Alice"
   ```

5. Database tests: wrap each test in a transaction that rolls back at teardown. Faster than `TRUNCATE`-per-test, gives full isolation by construction.

## Async Testing

6. Mark async tests with `@pytest.mark.asyncio`. Configure once globally so you don't decorate every test:

   ```toml
   # pyproject.toml
   [tool.pytest.ini_options]
   asyncio_mode = "auto"
   ```

   With `auto` mode, every `async def test_...` is treated as `pytest.mark.asyncio` automatically.
7. Async fixtures use the same machinery — `@pytest_asyncio.fixture` (or just `@pytest.fixture` under `asyncio_mode = "auto"`). Use `async with` inside the fixture for resource lifecycle.

## Database Testing Patterns

8. **Never hardcode entity IDs in tests.** Let the DB generate them.
   - **Why:** static IDs collide under `pytest-xdist` parallel execution; tests passing solo flake under `-n auto`. Even sequential tests break when test order changes.
9. Use the **flush pattern** to get the DB-assigned ID without committing the transaction:

   ```python
   @pytest_asyncio.fixture
   async def user(db_session):
       user = User(email="test@example.com", name="Test User")
       db_session.add(user)
       await db_session.flush()  # populates user.id without commit
       yield user
       # transaction rollback at session teardown handles cleanup

   @pytest_asyncio.fixture
   async def user_profile(db_session, user):
       profile = UserProfile(user_id=user.id, bio="Test bio")
       db_session.add(profile)
       await db_session.flush()
       yield profile
   ```

   - **Why:** `flush()` sends INSERT without committing, so the auto-generated PK is available to dependent fixtures while the entire fixture tree still rolls back at the end of the test.
10. Chain fixtures explicitly via parameters — parent fixture yields the entity, child fixture takes it as a dependency. pytest resolves the graph; you don't manage order manually.

## Parametrize Patterns

11. Use `@pytest.mark.parametrize` for table-driven tests instead of N near-duplicate test functions:

    ```python
    @pytest.mark.parametrize("input,expected", [
        ("", ValidationError),
        ("invalid", ValidationError),
        ("valid@email.com", None),
    ])
    def test_email_validation(input, expected):
        if expected:
            with pytest.raises(expected):
                validate_email(input)
        else:
            validate_email(input)  # should not raise
    ```

12. For complex parameter sets, name them via `ids=` or use `pytest.param(..., id="...")` for readable failure output.
13. `pytest.param(..., marks=pytest.mark.xfail(reason="..."))` documents known-broken cases without removing them — they re-pass loudly when fixed.

## Mocking Strategy

14. **Mock at boundaries, not internals.** Boundaries = network, file I/O, time, randomness, external services. Internals = your own functions, classes, repositories.
    - **Why:** mocking internals couples tests to implementation details. Refactor a private helper → all tests break. Mock at the seam where your code meets the outside world; trust your own code to call your own code correctly.
15. Time-dependent code: use `freezegun`:

    ```python
    @freeze_time("2026-01-15 12:00:00")
    def test_password_reset_token_expires_in_24h():
        token = generate_reset_token()
        # ...
    ```

    - **Why:** `datetime.now()` makes tests flaky at midnight or DST transitions. Freeze the clock for deterministic time math. Caveat: `freezegun` doesn't intercept `time.monotonic()` — use it for wall-clock, not benchmark timing.

## Advanced Patterns

16. **Property-based testing** with `hypothesis` for invariant-heavy logic (parsers, serializers, math, state machines):

    ```python
    @given(st.integers(min_value=0), st.integers(min_value=0))
    def test_addition_commutative(a, b):
        assert add(a, b) == add(b, a)
    ```

17. **Snapshot testing** (`syrupy`, `pytest-snapshot`) for stable-output transformations — diff against committed snapshots, regenerate explicitly when intentional changes happen.
18. **Performance regressions**: `pytest-benchmark` runs a function repeatedly and asserts mean/median against a stored baseline. Useful for hot paths; running in normal CI is overhead.

## Coverage & CI

19. Coverage targets:
    - Library / framework code: ≥90% line + branch
    - Application / glue code: ≥80%
    - Critical flows (auth, money, audit log, tenant isolation): 100% of paths
    Use `--cov-branch` so `if/else` paths both count.
20. Run on every commit: `pytest --cov=app --cov-report=term-missing --cov-branch`. Fail CI on coverage drop with `--cov-fail-under=80`.
21. Parallelize with `pytest-xdist`: `pytest -n auto` runs across CPU cores. Requires test isolation (no shared mutable state, no static IDs) — see #1, #5, #8.

## Essential Practices

22. Test both success and failure paths. A suite where every test exercises the happy path is a false sense of security.
23. **One logical assertion per test** when feasible. Multiple `assert`s are fine when verifying the same behavior from different angles ("created user has correct email AND is active"); split when they verify different behaviors.
24. **Descriptive test names — no docstrings.** The function name should answer "what scenario is this testing?":
    - ❌ `test_user`
    - ✅ `test_calculate_discount_with_zero_percentage_returns_original_price`
25. Keep tests deterministic. No bare `random.random()` (use `hypothesis` or seeded RNG), no real network, no `datetime.now()` (use `freezegun`).
26. Order: fast tests first. Unit (ms) → integration (sec) → e2e (min). Configure with `-m unit` markers and run incrementally.
27. **Test data builders over raw fixtures** for complex objects. `factory_boy` factories accept overrides, scale better than dicts of mock data, and integrate with `faker` for realistic values.

## Documentation in Tests

28. **Test functions: no docstrings.** The descriptive name (#24) is the documentation.
29. **Complex test logic: inline comments**, not docstrings. They sit next to the code they explain.
30. **Test classes**: brief docstring only if the grouping isn't obvious from the class name.
31. **Fixtures with non-trivial setup/teardown**: docstring explaining the lifecycle. Trivial fixtures don't need them.

## When applying these rules

- **Be opinionated about footguns** (#1 fixture scope, #8 static IDs in parallel, #9 flush pattern, #14 mock at boundaries) — these cause flaky or coupled tests, not preferences.
- **Be flexible about layer ratios** — projects vary in how much makes sense at unit vs integration. Match what the team has, don't impose a textbook pyramid.
- **Read existing test infrastructure first.** If the project has a `conftest.py` with `db_session` / `client` fixtures, conform to them rather than introducing parallel patterns. Add to the convention; don't fragment it.
- **Cross-skill awareness:** for FastAPI test client patterns, defer to `fastapi-best-practices` (rule 29 — `httpx.AsyncClient` with `ASGITransport`). For SQLAlchemy session / ORM-specific test fixtures, defer to `sqlalchemy-best-practices`.
