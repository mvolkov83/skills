# mvolkov-skills

> A [Claude Code](https://claude.com/claude-code) plugin marketplace bundling production-tested best-practices skills for an async-first Python backend stack — **SQLAlchemy 2.0, FastAPI, pytest, gRPC (`grpc.aio`), git workflow, OpenTelemetry observability, and money / payments / ledger engineering** — plus two domain skills for interactive-gaming-platform work: **GLI-19 platform certification readiness** and **GLI Gaming Security Framework (GLI-GSF) security engineering**.

If you're using Claude Code on Python projects with this stack, these skills make Claude consistent with conventions the team has already converged on, without bloating your context. Each skill lazy-loads only when relevant — no cost on conversations where it doesn't apply.

---

## What's inside

9 standalone skills. Install only what you need:

| Plugin | Triggers on | Adds guidance for |
|---|---|---|
| **[`sqlalchemy-best-practices`](plugins/sqlalchemy-best-practices/skills/sqlalchemy-best-practices/SKILL.md)** | imports of `sqlalchemy` / `sqlalchemy.ext.asyncio`, `AsyncSession`, `Mapped[T]`, `select()`, `selectinload`, ORM models, `MissingGreenlet` errors | SQLAlchemy 2.0 async — engine/pool defaults, `Mapped[T]` model design, relationship loading strategies, async session lifecycle, FastAPI DI |
| **[`fastapi-best-practices`](plugins/fastapi-best-practices/skills/fastapi-best-practices/SKILL.md)** | imports of `fastapi` / `pydantic_settings`, `APIRouter`, `Depends`, `BackgroundTasks`, `BaseSettings`, Pydantic v2 schemas | FastAPI production — domain-organized structure, async correctness, DI patterns with per-request caching, split request/response models, Settings v2 with `lru_cache`, async testing |
| **[`pytest-best-practices`](plugins/pytest-best-practices/skills/pytest-best-practices/SKILL.md)** | code under `tests/`, `@pytest.fixture`, `@pytest.mark.asyncio`, `parametrize`, imports of `pytest_asyncio` / `factory` / `faker` / `respx` / `freezegun` / `hypothesis` | pytest async-first — fixture scope discipline, `pytest-asyncio` with `asyncio_mode=auto`, factory_boy + faker test data, respx HTTP mocks, real-DB integration via testcontainers, `flush()` pattern, `pytest-xdist` parallelism |
| **[`grpc-python-best-practices`](plugins/grpc-python-best-practices/skills/grpc-python-best-practices/SKILL.md)** | `grpc.aio.*`, `*_pb2.py` / `*_pb2_grpc.py` files, `.proto` files, `ServerInterceptor`, `UnaryUnaryClientInterceptor` | gRPC `grpc.aio` Kubernetes-native — server bootstrap with health-drain, shared channel factory, decorator pattern (`@grpc_logger` + `@grpc_error_handler`), `max_connection_age_*` for HPA rebalancing, client-side LB via `dns:///` + `round_robin`, ERROR_MAP with rich `google.rpc.Status` |
| **[`git-workflow-best-practices`](plugins/git-workflow-best-practices/skills/git-workflow-best-practices/SKILL.md)** | `git commit`, `git push`, `gh pr create`, branch creation, PR review prep, commit message writing | Git Flow branches, Conventional Commits, atomic commits with imperative mood, pre-commit hygiene, `pull --rebase`, force-push scope, ~400 LOC PR target, squash on merge |
| **[`observability-best-practices`](plugins/observability-best-practices/skills/observability-best-practices/SKILL.md)** | imports of `opentelemetry.*`, `setup_telemetry()` calls, `tracer.start_as_current_span()`, `LoggingInstrumentor`, span attribute setting, structlog with `trace_id` binding, OTLP exporter config, Loki / Tempo / Grafana / Sentry integration | Python OpenTelemetry — SDK bootstrap with off-switch + idempotency guard, auto-instrumentation (gRPC / SQLAlchemy / FastAPI / Logging), `LoggingInstrumentor` + structlog correlation, two-tier log field taxonomy, `<service>.<key>` span attributes, sensitive-field redaction, metrics cardinality control, tail-based sampling, Sentry-with-OTel |
| **[`money-and-payments-best-practices`](plugins/money-and-payments-best-practices/skills/money-and-payments-best-practices/SKILL.md)** | imports of `decimal.Decimal`, money libraries (`py-money` / `dinero` / `stockholm` / `moneyed`), code defining `Transaction` / `Transfer` / `Ledger` / `Journal` / `Account` / `Balance` ORM models, payment / charge / refund / chargeback handlers, idempotency_key handling, PSP webhook code, `parent_transaction_id` references | Engineering best practices for money / payments / ledger systems — Decimal or integer minor units (never float), the two-layer idempotency model (`idempotency_key` + chain CAS, never conflated), double-entry ledger (Accounts + Transfers, append-only, balance computed not stored), two-phase transfers via HOLD, atomic chains, OCC state machines, stateless proxy for PSP integration with three-layer webhook dedup, reversibility via separate transaction with `parent_transaction_id`, DB-enforced invariants |
| **[`gli-19-platform-engineering`](plugins/gli-19-platform-engineering/skills/gli-19-platform-engineering/SKILL.md)** | online casino / interactive gaming work — player accounts and wallets, game sessions and rounds, RGS or aggregation layers, bonus and jackpot engines, back-office adjustment and void endpoints, tables named like `rounds` / `transactions` / `player_accounts` / `bonuses`, phrases like "RTP", "self-exclusion", "game recall", "significant event log", "GLI" | GLI-19 v3.0 certification readiness — server-authoritative outcomes, authoritative system clock, the records a gaming system must maintain (play record, per-theme aggregates, player-account record, significant-event log), append-only financial history with integrity hashing and actor attribution, supervised alteration of accounting data, restricted-credits-first wagering order, game-cycle exclusivity and interrupted-game completion, account lifecycle and MFA boundaries, most-restrictive limit precedence, geolocation, disable controls, regulator reporting surfaces |
| **[`gli-gsf-security`](plugins/gli-gsf-security/skills/gli-gsf-security/SKILL.md)** | securing / hardening / auditing a gaming platform or its infra — access control and RBAC, authn and session management, secrets and key rotation, encryption at rest and in transit, network segmentation and firewall rules, DNSSEC, remote access, SIEM, immutable backups, disaster recovery, secure SDLC, provider integration; phrases like "gaming security", "GLI-GSF", "least privilege", "default-deny", "tamper-evident log", one firewall rule or one service account | GLI Gaming Security Framework (GLI-GSF-1) — the deep security layer beneath GLI-19's Appendix B, owning the infra/network security it deferred: logical access with separation of duties, ephemeral session authorization, key agility, data-at-rest encryption, production-DB network isolation, hardening, default-deny firewalls with no bypass path, tamper-evident logging, immutable off-site backups, release segregation of duties, and provider integration that cannot route into production. Rules carry GIG1/GIG2/GIG3 assurance tiers |

Each skill is **depersonalized** — generic placeholder names (`MyService`, `MyServiceClient`, `MyServiceError`) instead of project-specific symbols. Patterns are anchored in real production code but written to apply across any project that follows the same stack.

---

## Quick start

In any Claude Code session:

```
/plugin marketplace add mvolkov83/skills
/plugin install sqlalchemy-best-practices@mvolkov-skills
/plugin install fastapi-best-practices@mvolkov-skills
/plugin install pytest-best-practices@mvolkov-skills
/plugin install grpc-python-best-practices@mvolkov-skills
/plugin install git-workflow-best-practices@mvolkov-skills
/plugin install observability-best-practices@mvolkov-skills
/plugin install money-and-payments-best-practices@mvolkov-skills
/plugin install gli-19-platform-engineering@mvolkov-skills
/plugin install gli-gsf-security@mvolkov-skills
/reload-plugins
```

That's it. Run a SQLAlchemy / FastAPI / pytest / gRPC / OTel / payments question and the relevant skill auto-triggers.

> **Heads up on the alias**: the install alias (`@mvolkov-skills`) is the `name` field from `.claude-plugin/marketplace.json` — **not** the basename of the repo URL. Claude Code reports the actual alias after `/plugin marketplace add` ("Successfully added marketplace: `<alias>`"); use exactly that. If you see "marketplace not found" on install, double-check the alias from that line.

---

## What is a "skill"?

A [Claude Code skill](https://code.claude.com/docs/en/skills) is a markdown file (`SKILL.md`) with structured guidance that Claude **lazy-loads** when its description matches what you're working on. Unlike `CLAUDE.md` instructions (which load every conversation), skills only consume context when the topic is actually relevant.

Each plugin in this marketplace ships exactly one skill. The skill body is plain markdown — no code execution, no surprises. Read any `SKILL.md` to see exactly what guidance Claude gets when the skill triggers.

One plugin also ships **slash commands**. Skills trigger themselves and answer "how should this be built"; commands are invoked deliberately, over a scope you name, and answer "what is wrong with what exists":

| Command | Scope | Answers |
|---|---|---|
| `/gli-19-review-diff` | a PR, a ref range, or the working branch diff | which requirements *this change* violates or undermines |
| `/gli-19-review-surface` | a service or module path | which requirements the service *does not implement* |
| `/gsf-security-review-diff` | a PR, a ref range, or the working branch diff (incl. infra) | which security controls *this change* violates or weakens |
| `/gsf-security-review-surface` | a service or infrastructure path | which security controls the service/infra *does not implement* |

The split is deliberate: a diff review cannot find absence. It cannot see that a per-theme aggregate, a regulator report, an immutable backup or a network segment doesn't exist, because that isn't in the changed lines. The surface commands exist for exactly those findings, and each opens by establishing the ownership boundary with you before it judges anything — a wrong boundary makes every verdict downstream of it wrong. (The security surface command additionally credits controls *inherited* from the cloud/managed platform, but only against the config that proves the control is on.)

### Wiring the security skill into an existing security-review gate

Installed skills auto-trigger from their description in normal conversation, so a security review you ask for **in words** ("do a security review of this branch", "threat-model the wallet service") pulls in `gli-gsf-security` on a gaming-platform repo. A built-in **slash command** like `/security-review` is different — it runs its own methodology and does not consult installed marketplace skills. To make an existing gate also load this skill, add one line to the gaming repo's `CLAUDE.md` (which loads on every conversation):

```markdown
When performing any security review of this repository (including via `/security-review`),
first load the `gli-gsf-security` skill and apply its rules; it is the security authority
for this interactive-gaming platform.
```

Or just run the dedicated `/gsf-security-review-diff` / `/gsf-security-review-surface` gate, which loads the skill by construction.

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
| "Why is `LoggingInstrumentor` breaking my structlog format?" | `observability-best-practices` |
| "How do I set span attributes so I can search by transaction_id in Tempo?" | `observability-best-practices` |
| "Should I use `Decimal` or store amounts as integer cents?" | `money-and-payments-best-practices` |
| "Two requests with the same `idempotency_key` arrived — what's the right behaviour?" | `money-and-payments-best-practices` |
| "How do I model refunds — mutate the original transaction or create a new one?" | `money-and-payments-best-practices` |
| "Adding an admin endpoint to adjust a player's balance" | `gli-19-platform-engineering` + `money-and-payments-best-practices` |
| "Should bonus credits or real money be consumed first when both are on the balance?" | `gli-19-platform-engineering` |
| "What has to be in the game round record so we can reconstruct a disputed round?" | `gli-19-platform-engineering` |
| "Where should the session-authorization token live, and how long?" | `gli-gsf-security` |
| "Is putting the DB in the same subnet as the web tier a problem for certification?" | `gli-gsf-security` |
| "Review this Terraform security group / firewall rule for a gaming platform" | `gli-gsf-security` |
| "How do we integrate a third-party game provider without failing a security audit?" | `gli-gsf-security` |
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

### `observability-best-practices`

22 rules across 13 sections covering OpenTelemetry SDK bootstrap, log correlation, span attributes, redaction, and metrics. Highlights:

- **Off-switch contract** — `setup_telemetry()` returns early if `OTEL_EXPORTER_OTLP_ENDPOINT` is unset; tests stay deterministic, no zombie exporter threads.
- **Idempotency guard** — single-shot `_initialized` sentinel prevents double-handler-stacking on uvicorn `--reload` (without it, every log record ships 2×, 3×, ... to OTLP).
- **The `LoggingInstrumentor` + structlog correlation pattern** — three footguns that all need solving together: `set_logging_format=False` (preserves stdout format), custom `_inject_trace_context` log hook (bridges SDK gap), `_CleanLoggingHandler` (strips structlog non-serializable attrs).
- **Two-tier log field taxonomy** — Tier-1 structured `key=value` fields for Loki indexing; Tier-2 raw payload after `|` separator for grep when investigating.
- **`<service>.<business_key>` span attribute prefix** — cross-service Tempo search by business IDs, no manual ID copy.
- **Sensitive-field redaction at the serialization boundary** — copy-and-replace before logging proto/JSON; business code stays unredacted (audit trail truthful, logs PII-clean).
- **Strict metric cardinality control** — `transaction_id` / `user_id` / `idempotency_key` are forbidden as labels (one series per request explodes Prometheus). High-cardinality dimensions belong in logs and traces, not metrics.

Cross-references `grpc-python-best-practices` (decorator chain), `fastapi-best-practices` (FastAPIInstrumentor), `sqlalchemy-best-practices` (SQLAlchemyInstrumentor).

### `money-and-payments-best-practices`

47 rules across 11 sections — engineering best practices for money / payments / ledger systems. Anchored in two canonical public references: [Stripe's idempotency engineering](https://stripe.com/blog/idempotency) and [TigerBeetle's debit/credit model](https://docs.tigerbeetle.com/concepts/debit-credit/). Highlights:

- **Money is `Decimal` or integer minor units, never `float`** — `0.1 + 0.2 == 0.30000000000000004` is a one-line argument; downstream calculations diverge across systems that should reconcile to zero.
- **The two-layer idempotency model** — client-driven `idempotency_key` (request dedup, silent return on replay, 422 on payload mismatch) vs chain-CAS via `UNIQUE(aggregate_id, parent_id)` (concurrent state-transition serialisation, fail-loud on conflict). Different problems; **never conflated**.
- **Idempotency keys are client-generated, never server-side** — server-generated keys can't survive network timeouts; the canonical Stripe pattern.
- **Double-entry: Accounts + Transfers, append-only, balance computed from entries** — DB-enforced invariants (`CHECK (balance >= 0 OR account_type IN ('LIABILITY', 'EQUITY'))`), not application-enforced.
- **Two-phase via `HOLD` for outbound flows** — Reserve at intent commit (closes the fraud window), Commit XOR Release on terminal transition. Singleton HOLD per `(account, currency)`.
- **Atomic chains for composite operations** — fee-with-transfer, multi-leg settlement as one DB transaction; no two-phase commit across services.
- **Transaction state machines with chain-CAS OCC** — append-only state log, `parent_id` linkage, atomic `INSERT ... SELECT` tip-and-CAS, `READ COMMITTED` (no `SERIALIZABLE`, no `SELECT FOR UPDATE`). Hash-chain over the log for primary audit.
- **PSP integration via stateless proxy/facade** — credentials in the application service, proxy stateless. Sync request + async webhook + `QueryStatus` fallback. Three-layer webhook dedup (signature → DB UNIQUE → handler idempotency).
- **Reversals are separate transactions with `parent_transaction_id` link** — direction inversion, atomic chain CAS on the parent's state log, mutual exclusion on concurrent recall attempts.

Cross-references `sqlalchemy-best-practices` (ledger / journal table design), `grpc-python-best-practices` (sync RPC + async webhook architecture), `observability-best-practices` (hash-chain as primary audit, structured logs as defense-in-depth), `pytest-best-practices` (property-based testing for money invariants, real-DB ledger tests).

### `gli-19-platform-engineering`

81 rules across 12 sections — the write-time engineering layer of a GLI-19 v3.0 certification. Each rule maps to one or more requirement ids; `references/requirements.md` carries the paraphrased requirement with the testable criteria a laboratory assesses. Highlights:

- **No game logic on the client; the server generates every outcome** — the load-bearing requirement. A client that can compute an outcome can forge one, and it dictates protocol shape: outcome fields must never travel client→server.
- **Restricted incentive credits are wagered before unrestricted funds** — the natural implementation instinct is real-money-first, which is the exact inverse. Make consumption order an explicit tested policy, not an emergent property of list concatenation.
- **Every mutation of accounting data carries an actor** — alteration id, element, value before, value after, timestamp, and *who*. The "who" is the field routinely missing, and a void attributable only to a player-facing session token is both a finding and a live security hole.
- **Integrity hashing on financial rows, chained and verified at reconciliation** — trivial at schema-design time, permanently awkward once the table is populated (the backfill's trustworthiness is exactly what's in question).
- **Per-theme aggregates are a maintained record, not a future query** — the rounds table holds the substrate, teams assume they'll compute it later, and its absence also blocks the theoretical-vs-actual RTP report.
- **Interrupted-game completion ≠ crash recovery** — idempotent resume of an in-flight write is infrastructure; returning the player to the pre-interruption state and letting them finish is a product feature. The first is routinely reported as if it were the second.
- **Most-restrictive limit wins** — precedence implemented as "last writer wins" or "most specific scope wins" can silently loosen a self-exclusion. Encode it as an explicit `min()` over active constraints.

Scope-honest by construction: it covers the ~96 GLI-19 requirements whose evidence is code, infra or config, and explicitly declines the operator's procedural and governance obligations, the submission package, and roughly a fifth of the technical-security appendix that lives in IaC and runbooks (DNS hardening, firewall boundaries, remote access, patching, asset registers). It is engineering guidance with no certification weight — never a substitute for the standard or the laboratory's judgment.

Ships two commands, `/gli-19-review-diff` and `/gli-19-review-surface` — see [How they trigger](#how-they-trigger). Both refuse to emit a certification verdict or a readiness percentage: the denominator depends on scoping rulings and jurisdiction, and a number from a code reader would be read as a compliance metric it isn't. `gli-19-review-surface` additionally takes verdicts from primary artifacts only — code, schema, migrations, config, tests — and never from a README, design doc or prior audit claiming a mechanism exists. Finding the claim without the mechanism is most of its value.

### `gli-gsf-security`

72 rules across 15 sections — the security-engineering layer for an interactive gaming platform under the **GLI Gaming Security Framework (GLI-GSF-1)**, the framework progressively superseding the security portions of GLI-11 / GLI-19 / GLI-27 / GLI-33. Each rule maps to one or more GIS control ids and carries the control's **assurance tier** (GIG1 baseline / GIG2 / GIG3 enhanced). It is the sibling of `gli-19-platform-engineering`: that skill covers application-level gaming correctness and *deferred* the infrastructure and network security; this one **owns exactly that deferred territory** and is the deep version of the access-control, data-at-rest, communications and crypto controls the platform skill only summarized. Highlights:

- **Separation of duties, twice** — the account that administers users must not be the account that uses privileges (rule 3); the developer who wrote a change must not be the sole party that ships it (rule 61). One compromised super-role otherwise defeats every audit-log-based control.
- **Production databases isolated from patron-facing servers** — the control that turns an app-layer compromise of a public service into a contained incident instead of direct DB access. Flat networks are how an injection becomes a breach.
- **Provider integration cannot route into production** — the classic third-party breach path. Terminate provider traffic in a DMZ; a platform that IP-routes between a vendor and production has adopted the vendor's security posture as its own.
- **Ephemeral, per-request authorization** — authorization data cached on the component outlives its grant and travels with a compromised node; fetch-at-request-time is what makes a revocation take effect now.
- **Immutable backups** — a backup the production credentials can delete or rewrite is a second copy with the same blast radius, not a recovery point against ransomware. Object-lock / WORM is the difference.
- **Tamper-evident security logs** — a log an attacker can edit is not evidence; ship it off the generating host to append-only storage.
- **Key agility** — distinct keys per purpose so rotation is routine; one key everywhere makes rotation an all-or-nothing event that never happens.

Same scope honesty as its sibling: it covers the GLI-GSF-1 controls whose evidence is code, infra or config (155 of 163 cited, reported by the coverage check), carries no certification weight, and does not cover the operator's GISMS governance programme. Cross-references `gli-19-platform-engineering` (application/gaming semantics and the financial-integrity model), `money-and-payments-best-practices` (payment and ledger specifics), `observability-best-practices` (instrumentation, log correlation, redaction), and `sqlalchemy-best-practices` (ORM-level data access).

Ships two commands, `/gsf-security-review-diff` and `/gsf-security-review-surface` — see [How they trigger](#how-they-trigger). The diff command gates by security domain and reads infrastructure diffs (Terraform / Helm / security groups / IAM), not just application code; the surface command establishes what is owned vs *inherited* from the cloud/managed platform before it judges, credits an inherited control only against the config that proves it is enabled, and takes verdicts from primary artifacts — never from a policy document asserting a control exists. Neither emits a certification verdict, an assurance-tier attainment, or a security score.

Cross-references `money-and-payments-best-practices` (money representation, idempotency, double-entry structure — the gaming rules are an overlay on it), `observability-best-practices` (log centralization and tamper protection), `sqlalchemy-best-practices` (append-only schema design).

---

## Stack assumptions

The patterns are tuned for an async-first modern Python backend:

- **Python 3.12+**
- **SQLAlchemy 2.0+** with `asyncio` extras (`asyncpg` driver typical)
- **FastAPI** with **Pydantic v2** + Pydantic Settings v2
- **pytest** + `pytest-asyncio` + `pytest-xdist` + `pytest-cov`
- **gRPC**: `grpcio` + `grpcio-tools` + `grpcio-health-checking` + `grpcio-status`
- **Kubernetes** deployment (gRPC LB section assumes k8s + headless Services; non-k8s use cases still get value from the rest)
- **OpenTelemetry**: `opentelemetry-api/sdk` + OTLP HTTP exporters + `opentelemetry-instrumentation-{grpc,sqlalchemy,fastapi,logging}`; structlog for structured logging; Loki + Tempo + Mimir or Grafana Cloud as the typical backend
- **Money handling** (when applicable): `Decimal` from stdlib **or** integer minor units (Stripe pattern); a money library (`py-money`, `dinero`, `stockholm`, `moneyed`) for non-trivial arithmetic; PostgreSQL `UNIQUE` and partial `UNIQUE` constraints as the database-level guards for idempotency and chain-CAS

`gli-19-platform-engineering` and `gli-gsf-security` are the exceptions — they are **domain** skills, not stack skills. Their rules are about what an interactive gaming platform must do and how its security must be built, not which libraries it uses, so they apply regardless of language or framework.

Sync `grpcio`, Pydantic v1 (`Config` inner class, `@validator`), SQLAlchemy 1.x (`Column()`, `session.query()`) are explicitly **legacy**. When skills see those patterns in code, they suggest the modern equivalent and explain why.

---

## Update

```
/plugin marketplace update mvolkov-skills
/reload-plugins
```

Pulls the latest commits and reloads. No re-install needed.

## Uninstall

A specific plugin:

```
/plugin uninstall <plugin-name>@mvolkov-skills
```

Or remove the marketplace entirely (uninstalls all plugins from it):

```
/plugin marketplace remove mvolkov-skills
```

---

## Contributing

Each skill is a single `SKILL.md` file with YAML frontmatter + markdown body; commands, where a plugin ships them, are one markdown file each under `plugins/<plugin-name>/commands/`. To improve one:

1. Fork the repo.
2. Edit `plugins/<plugin-name>/skills/<plugin-name>/SKILL.md` (or the relevant `commands/*.md`).
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

The two GLI skills (`gli-19-platform-engineering` and `gli-gsf-security`) are built differently. Each `references/requirements.md` is **generated** from a machine-readable requirement catalog extracted from the standard, filtered to the requirements whose evidence is code / infra / config and stripped of any project-specific service topology and cloud-vendor names. The rules in `SKILL.md` are authored on top of that filtered set, and a coverage check verifies every rule cites a real requirement id and reports which in-scope requirements no rule covers — so the skills can't silently drift from the catalogs they derive from. The same generator produces both from a shared profile mechanism.

The repo's structure mirrors Anthropic's official `claude-plugins-official` marketplace conventions — top-level `.claude-plugin/marketplace.json`, per-plugin `<plugin-name>/.claude-plugin/plugin.json`, and skills under `<plugin-name>/skills/<skill-name>/SKILL.md`.
