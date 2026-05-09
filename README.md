# mvolkov-skills

A [Claude Code](https://claude.com/claude-code) plugin marketplace bundling best-practices skills for an async-first Python backend stack — SQLAlchemy 2.0, FastAPI, pytest, gRPC (`grpc.aio`), and git workflow.

## What's inside

| Plugin | What it covers |
|---|---|
| [`sqlalchemy-best-practices`](plugins/sqlalchemy-best-practices/skills/sqlalchemy-best-practices/SKILL.md) | SQLAlchemy 2.0 async — engine/pool, `Mapped[T]` models, relationship loading, async sessions, FastAPI DI |
| [`fastapi-best-practices`](plugins/fastapi-best-practices/skills/fastapi-best-practices/SKILL.md) | FastAPI production — domain-organized structure, async correctness, DI, Pydantic v2 split models, Settings, async testing |
| [`pytest-best-practices`](plugins/pytest-best-practices/skills/pytest-best-practices/SKILL.md) | pytest async-first — fixtures, `pytest-asyncio`, factory_boy, respx, freezegun, hypothesis, real-DB integration |
| [`grpc-python-best-practices`](plugins/grpc-python-best-practices/skills/grpc-python-best-practices/SKILL.md) | Python gRPC `grpc.aio` in Kubernetes — server bootstrap, channel factory, decorator pattern, LB via `dns:///` + `round_robin`, error model, k8s deployment |
| [`git-workflow-best-practices`](plugins/git-workflow-best-practices/skills/git-workflow-best-practices/SKILL.md) | Git Flow branches, Conventional Commits, atomic commits, pre-commit hygiene, PR sizing, squash on merge |

Each skill is depersonalized — generic placeholder names (`MyService`, `MyServiceClient`) instead of project-specific symbols. Production-tested values (timeouts, pool sizes, retry policies) preserved.

## Install

In Claude Code:

```
/plugin marketplace add mvolkov83/skills
/plugin install <plugin-name>@skills
/reload-plugins
```

The marketplace alias `skills` is the basename of the repo URL — Claude Code reports the actual alias after `marketplace add` (verify if `install` fails with "marketplace not found").

Examples:

```
/plugin install sqlalchemy-best-practices@skills
/plugin install fastapi-best-practices@skills
/plugin install pytest-best-practices@skills
/plugin install grpc-python-best-practices@skills
/plugin install git-workflow-best-practices@skills
```

Install all five or pick the ones you need.

## Update

```
/plugin marketplace update skills
/reload-plugins
```

## How they work

Each skill lazy-loads when its trigger description matches what you're working on — no context cost when irrelevant. The frontmatter `description` is "pushy" by design (skill-creator best practice for Claude's tendency to undertrigger), so the skill consults itself on edge-case phrasings too.

Skills are independent — installing one doesn't pull the others. Some cross-reference each other in their bodies (e.g., `pytest-best-practices` references `fastapi-best-practices` rule 29 for `httpx.AsyncClient` setup) but those are advisory, not required.

## Contributing

Improvements via PR. Each skill is a single `SKILL.md` with frontmatter + markdown body — easy to read, easy to amend. PRs that genericize project-specific examples (the depersonalization is deliberate) are especially welcome.

## License

MIT — see [LICENSE](LICENSE) if present, or the repo's root metadata.
