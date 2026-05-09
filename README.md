# mvolkov-skills

> A [Claude Code](https://claude.com/claude-code) plugin marketplace bundling production-tested best-practices skills for an async-first Python backend stack — **SQLAlchemy 2.0, FastAPI, pytest, gRPC (`grpc.aio`), and git workflow**.

If you're using Claude Code on Python projects with this stack, these skills make Claude consistent with conventions the team has already converged on, without bloating your context. Each skill lazy-loads only when relevant — no cost on conversations where it doesn't apply.

---

## What's inside

5 standalone skills. Install only what you need:

| Plugin | Triggers on | Adds guidance for |
|---|---|---|
| **[`sqlalchemy-best-practices`](plugins/sqlalchemy-best-practices/skills/sqlalchemy-best-practices/SKILL.md)** | imports of `sqlalchemy` / `sqlalchemy.ext.asyncio`, `AsyncSession`, `Mapped[T]`, `select()`, `selectinload`, ORM models, `MissingGreenlet` errors | SQLAlchemy 2.0 async — engine/pool defaults, `Mapped[T]` model design, relationship loading strategies, async session lifecycle, FastAPI DI |
| **[`fastapi-best-practices`](plugins/fastapi-best-practices/skills/fastapi-best-practices/SKILL.md)** | imports of `fastapi` / `pydantic_settings`, `APIRouter`, `Depends`, `BackgroundTasks`, `BaseSettings`, Pydantic v2 schemas | FastAPI production — domain-organized structure, async correctness, DI patterns with per-request caching, split request/response models, Settings v2 with `lru_cache`, async testing |
| **[`pytest-best-practices`](plugins/pytest-best-practices/skills/pytest-best-practices/SKILL.md)** | code under `tests/`, `@pytest.fixture`, `@pytest.mark.asyncio`, `parametrize`, imports of `pytest_asyncio` / `factory` / `faker` / `respx` / `freezegun` / `hypothesis` | pytest async-first — fixture scope discipline, `pytest-asyncio` with `asyncio_mode=auto`, factory_boy + faker test data, respx HTTP mocks, real-DB integration via testcontainers, `flush()` pattern, `pytest-xdist` parallelism |
| **[`grpc-python-best-practices`](plugins/grpc-python-best-practices/skills/grpc-python-best-practices/SKILL.md)** | `grpc.aio.*`, `*_pb2.py` / `*_pb2_grpc.py` files, `.proto` files, `ServerInterceptor`, `UnaryUnaryClientInterceptor` | gRPC `grpc.aio` Kubernetes-native — server bootstrap with health-drain, shared channel factory, decorator pattern (`@grpc_logger` + `@grpc_error_handler`), `max_connection_age_*` for HPA rebalancing, client-side LB via `dns:///` + `round_robin`, ERROR_MAP with rich `google.rpc.Status` |
| **[`git-workflow-best-practices`](plugins/git-workflow-best-practices/skills/git-workflow-best-practices/SKILL.md)** | `git commit`, `git push`, `gh pr create`, branch creation, PR review prep, commit message writing | Git Flow branches, Conventional Commits, atomic commits with imperative mood, pre-commit hygiene, `pull --rebase`, force-push scope, ~400 LOC PR target, squash on merge |

Each skill is **depersonalized** — generic placeholder names (`MyService`, `MyServiceClient`, `MyServiceError`) instead of project-specific symbols. Patterns are anchored in real production code but written to apply across any project that follows the same stack.

---

## Quick start

In any Claude Code session:

```
/plugin marketplace add mvolkov83/skills
/plugin install sqlalchemy-best-practices@skills
/plugin install fastapi-best-practices@skills
/plugin install pytest-best-practices@skills
/plugin install grpc-python-best-practices@skills
/plugin install git-workflow-best-practices@skills
/reload-plugins
```

That's it. Run a SQLAlchemy/FastAPI/pytest/gRPC question and the relevant skill auto-triggers.

> **Heads up on the alias**: the install alias (`@skills` above) is what Claude Code reports after `/plugin marketplace add` — usually the basename of the repo URL, occasionally truncated. If `/plugin install ...@skills` fails with "marketplace not found", check the exact alias the system printed and retry.

---

## What is a "skill"?

A [Claude Code skill](https://code.claude.com/docs/en/skills) is a markdown file (`SKILL.md`) with structured guidance that Claude **lazy-loads** when its description matches what you're working on. Unlike `CLAUDE.md` instructions (which load every conversation), skills only consume context when the topic is actually relevant.

Each plugin in this marketplace ships exactly one skill. The skill body is plain markdown — no code execution, no surprises. Read any `SKILL.md` to see exactly what guidance Claude gets when the skill triggers.

---

## How they trigger

Skills fire on substantive design / review / debugging work where conventions matter — not on simple one-off operations. Examples:

| You write... | Skill that fires |
|---|---|
| "I'm getting `MissingGreenlet` when accessing `user.posts` after commit" | `sqlalchemy-best-practices` |
| "Set up async SQLAlchemy with FastAPI — engine, sessionmaker, separate read/write deps" | `sqlalchemy-best-practices` + `fastapi-best-practices` |
| "Why does my fixture not roll back between tests?" | `pytest-best-practices` |
| "We need to load-balance gRPC pods after HPA scale-up" | `grpc-python-best-practices` |
| "What's the right commit message for this refactor?" | `git-workflow-best-practices` |
| "Read this file and summarize" | (none — too simple, Claude handles directly) |
| "Generate a Django REST view" | (none — wrong stack, all skills explicitly skip non-FastAPI / non-SQLAlchemy frameworks) |

If you want to verify a skill triggered, ask Claude directly: *"Did `<skill-name>` trigger for this?"* — it'll tell you which skills are loaded and whether they consulted them.

---

## Per-plugin details

### `sqlalchemy-best-practices`

35 rules across 7 sections (Core Setup, Model Design, Relationships & Loading, Query Optimization, Session & Transaction Patterns, Async Patterns, FastAPI DI). Why-explanations on the four real async footguns:

- **Never share `AsyncSession` across `asyncio.gather()` branches** — it's not concurrency-safe; sharing causes `IllegalStateChangeError` or silent corruption.
- **Default `lazy="select"` triggers `MissingGreenlet` in async contexts** — use `lazy="selectin"` or explicit `selectinload()` at query time.
- **`expire_on_commit=False` for post-commit attribute access** — the default `True` triggers lazy reloads after commit which fail in async outside an awaitable.
- **`selectinload()` over `joinedload()` for collections** — `joinedload` produces Cartesian fan-out (N×M rows); `selectinload` issues a bounded second query with `IN(...)`.

### `fastapi-best-practices`

30 rules across 7 sections (Project Structure, Async Patterns, Dependency Injection, Pydantic v2, API Design, Settings & Configuration, Database & Performance, Testing). Notably **corrects** the common misconception about `BackgroundTasks` — they run in the same event loop after the response is sent, so CPU-bound work there still serializes all requests. Use Celery/RQ/arq or `loop.run_in_executor()` with a `ProcessPoolExecutor` for CPU-bound. Cross-references `sqlalchemy-best-practices` for ORM/pool detail.

### `pytest-best-practices`

31 rules across 11 sections — covering test layer structure (unit/integration/e2e), fixture scope discipline (the "session-scoped mutable state leaks between tests" footgun), the `flush()` pattern for chained DB fixtures under parallel `pytest-xdist`, mock-at-boundaries strategy, parametrize with `ids=`, property-based testing with `hypothesis`, time mocking with `freezegun`. Cross-references `fastapi-best-practices` (rule 29 — `httpx.AsyncClient` with `ASGITransport`) and `sqlalchemy-best-practices` (DB fixture semantics).

### `grpc-python-best-practices`

74 rules across 14 sections — the most extensive skill, since gRPC has more surface. Highlights:

- **Server bootstrap with health-drain** — flip `grpc.health.v1.Health` to `NOT_SERVING` *before* `server.stop()` so L7 LBs and `Watch`-subscribed clients see the drain signal immediately.
- **The `max_connection_age_*` triad** — without it, client-side LB doesn't actually rebalance after k8s HPA scale-up, because long-lived HTTP/2 connections don't reconnect on their own.
- **The "why ClusterIP breaks gRPC LB" narrative** — full explanation of HTTP/2 multiplexing × L4 sticky behavior, plus the headless Service + `dns:///` + `round_robin` triad as the standard "no-mesh" answer.
- **The decorator pattern** (`@grpc_logger` + `@grpc_error_handler`) — proto-aware cross-cutting concerns with intentional ordering. `ServerInterceptor` is presented as the alternative for wire-level needs.
- **Full error model** — server-side `ERROR_MAP: dict[type[Exception], (grpc_code, business_code, msg)]` packed into `google.rpc.Status` via `rpc_status.to_status()`; client-side translation hierarchy (`*Unavailable / *InvalidArgument / *Conflict / *Unknown`) so orchestrators never see `grpc.RpcError`.
- **UNAVAILABLE-only retry principle** — transport bounce is the only safe auto-retry; business errors must reach the caller.

### `git-workflow-best-practices`

24 rules across 5 sections. **Explicitly defers SAFETY rules** (force-push protection, hook bypass prevention, no `git add .` without per-file review, no commit unless asked, no `--amend` after failed pre-commit hook) to Claude Code's built-in system prompt — those always apply regardless of trigger. The skill adds the **STYLE/WORKFLOW** layer on top: Conventional Commits format with type list, Git Flow branch prefixes (`feature/` / `bugfix/` / `hotfix/` / `release/`), atomic commits with imperative mood, pre-commit hygiene, `pull --rebase` to keep linear history, max ~400 LOC per PR (research-backed review-fidelity threshold), squash on merge for clean canonical history.

---

## Stack assumptions

The patterns are tuned for an async-first modern Python backend:

- **Python 3.12+**
- **SQLAlchemy 2.0+** with `asyncio` extras (`asyncpg` driver typical)
- **FastAPI** with **Pydantic v2** + Pydantic Settings v2
- **pytest** + `pytest-asyncio` + `pytest-xdist` + `pytest-cov`
- **gRPC**: `grpcio` + `grpcio-tools` + `grpcio-health-checking` + `grpcio-status`
- **Kubernetes** deployment (gRPC LB section assumes k8s + headless Services; non-k8s use cases still get value from the rest)

Sync `grpcio`, Pydantic v1 (`Config` inner class, `@validator`), SQLAlchemy 1.x (`Column()`, `session.query()`) are explicitly **legacy**. When skills see those patterns in code, they suggest the modern equivalent and explain why.

---

## Update

```
/plugin marketplace update skills
/reload-plugins
```

Pulls the latest commits and reloads. No re-install needed.

## Uninstall

A specific plugin:

```
/plugin uninstall <plugin-name>@skills
```

Or remove the marketplace entirely (uninstalls all plugins from it):

```
/plugin marketplace remove skills
```

---

## Contributing

Each skill is a single `SKILL.md` file with YAML frontmatter + markdown body. To improve one:

1. Fork the repo.
2. Edit `plugins/<plugin-name>/skills/<plugin-name>/SKILL.md`.
3. Submit a PR.

PRs especially welcome:

- **Genericize project-specific examples** — depersonalization is deliberate. If you spot a leaked project-specific symbol (like `WalletService`, `wallet_pb2_grpc`, `apps/occ/`), please replace with a placeholder.
- **Why-additions** — rules without rationale are weak. If you know *why* a rule exists (failure mode, performance reason, historical incident), add a `Why:` line under it.
- **Cross-references** — when one skill's pattern interacts with another's, make the cross-reference explicit so the model knows which authority to defer to.
- **Captured team conventions** — if the team has converged on a pattern not yet in a skill, encode it. The goal is for these skills to reflect what works in production, not the author's personal taste.

---

## How this is built

The skills were created using Anthropic's official [`skill-creator`](https://github.com/anthropics/skills/tree/main/skills/skill-creator) plugin, then iteratively validated against real production codebases. Each `SKILL.md` follows the canonical structure:

- **YAML frontmatter** with `name` and a "pushy" `description` (the description is the primary triggering mechanism — it's intentionally rich with positive trigger phrases and explicit negative cases to combat Claude's tendency to undertrigger).
- **Imperative-form rules** with brief `Why:` explanations on footguns.
- **Inline code examples** for the trickiest patterns.
- **Cross-references** to sibling skills where relevant.
- **A closing "When applying these rules" section** distinguishing footguns (be opinionated) from preferences (be flexible).

The repo's structure mirrors Anthropic's official `claude-plugins-official` marketplace conventions — top-level `.claude-plugin/marketplace.json`, per-plugin `<plugin-name>/.claude-plugin/plugin.json`, and skills under `<plugin-name>/skills/<skill-name>/SKILL.md`.
