# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project type

This is a **Claude Code plugin marketplace**, not an application codebase. There is no build step, no test suite, no dependency manager — the "code" is markdown `SKILL.md` files that Claude Code lazy-loads when triggered. Don't look for `pyproject.toml` / `package.json` / build scripts; they don't exist here.

The end users of this repo are Claude Code consumers who run `/plugin marketplace add <repo>` and `/plugin install <plugin-name>@<alias>`. The README is the user-facing onboarding doc; this file is for future Claude instances editing the marketplace.

## Structure

The layout follows the canonical Anthropic plugin-marketplace shape (mirrors `anthropics/claude-plugins-official`):

```
.claude-plugin/marketplace.json     # top-level marketplace manifest, lists every plugin
plugins/<plugin-name>/
├── .claude-plugin/plugin.json      # per-plugin manifest (name, description, author)
└── skills/<skill-name>/SKILL.md    # the skill body — markdown with YAML frontmatter
```

Each plugin in this marketplace ships **exactly one skill** (granular one-plugin-per-skill so consumers can install only what they need). The plugin and skill directory names are the same by convention.

## Adding a new skill

1. `mkdir -p plugins/<name>/.claude-plugin plugins/<name>/skills/<name>`
2. Write `plugins/<name>/skills/<name>/SKILL.md` (frontmatter + body, conventions below).
3. Write `plugins/<name>/.claude-plugin/plugin.json` (mirror an existing one — `name`, `description`, `author`).
4. Add an entry to `plugins[]` in `.claude-plugin/marketplace.json` with `name`, `description`, `author`, `category`, `source: "./plugins/<name>"`.
5. Symlink for local dev: `ln -s "$PWD/plugins/<name>/skills/<name>" "$HOME/.claude/skills/<name>"`.
6. Commit. Consumers run `/plugin marketplace update <alias>` to pick up new commits.

For an interactive Q&A workflow + optional eval-loop, install Anthropic's `skill-creator` plugin (`/plugin install skill-creator@claude-plugins-official`) and invoke it for skill drafting / description optimization.

## Skill writing conventions

Read an existing `SKILL.md` (e.g. `plugins/sqlalchemy-best-practices/skills/sqlalchemy-best-practices/SKILL.md`) for the canonical shape. Follow the same pattern — these are non-obvious decisions that apply to every skill in this repo.

### Frontmatter `description` must be "pushy"

Claude tends to **undertrigger** skills. The `description` is the primary triggering mechanism, so it must be rich:

- Open with what the skill covers (the surface area, named concretely).
- Include `Use this skill whenever...` with concrete trigger phrases — function names, file extensions, error messages, library imports, common question phrasings.
- Include `Trigger even when...` for edge-case phrasings (small reviews, single-file glances).
- End with `Do NOT use for...` listing adjacent-but-different domains (e.g. an SQLAlchemy skill explicitly skips Django ORM, Tortoise, raw asyncpg without ORM).

The description can be long. skill-creator's guidance is "feel free to go longer if needed" — pushy + comprehensive beats lean + ambiguous for trigger reliability.

### Depersonalize examples

Skills in this repo are written for portability across projects and potential public publication. Use generic placeholder names — **`MyService`**, **`MyServiceClient`**, **`MyServiceError`**, **`MyServiceErrorDetail`** — not symbols from any specific project's codebase.

- **Keep** production-tested *values* (specific timeouts, pool sizes, retry policy parameters, channel options) — these are domain-independent tuning knobs.
- **Drop** project-specific *names* — service classes, internal package names, company-prefixed identifiers, internal k8s service names.

If you catch yourself baking in a project-specific symbol while editing, swap for a placeholder before committing.

### Why-additions only on footguns

Skills are imperative rules. Add a `**Why:**` line under a rule **only when there's a real failure mode or non-obvious motivation** — silent bugs, performance cliffs, surprising semantics. Don't add Why on stylistic preferences; it bloats the skill into a wall of rationalizations and hides the actual footguns.

### Cross-reference sibling skills

When a rule's territory genuinely belongs to another skill in this marketplace, defer to it explicitly in prose ("For ORM session details, defer to the `sqlalchemy-best-practices` skill"). Cross-references prevent duplication, let each skill stay focused, and signal to the model which authority to consult.

### Closing "When applying these rules" section

Every `SKILL.md` ends with this section. It distinguishes:

- **Footguns** — be opinionated; list specific rule numbers that cause silent bugs / fleet outages.
- **Preferences** — be flexible; match existing project conventions rather than imposing the skill's defaults.
- **Read existing code first** — when a project already has a pattern (a base class, a settings module, a fixture), conform to it.
- **Cross-skill awareness** — name the sibling skills relevant to overlapping domains so the model knows where to defer.

### Defer to Claude Code's built-in safety rules

Claude Code's system prompt contains hard safety rules (no force-push to main, no `--no-verify`, no commit unless asked, prefer specific paths over `git add -A`/`git add .`, never `--amend` after failed pre-commit hook, etc.). **Don't duplicate these in skill bodies.** A skill should add the *style/workflow* layer on top, explicitly noting that safety is handled by the system prompt — not relax or restate it.

## Local dev vs marketplace distribution

Two parallel installation paths coexist for this repo:

- **Local dev (repo owner)**: `~/.claude/skills/<name>` is a symlink into this repo's `plugins/<name>/skills/<name>/`. Edits to `SKILL.md` are live — picked up on the next conversation or `/reload-plugins`.
- **Marketplace distribution (consumers)**: `/plugin marketplace add <repo>` clones the repo into `~/.claude/plugins/marketplaces/<alias>/`. Skills are loaded from that clone, updated via `/plugin marketplace update <alias>`.

Don't mix the two on the same machine — both register the same skill name and conflict. Use symlinks during development; consumers use the marketplace flow.

## Project memory

The auto-memory directory for this project (under `~/.claude/projects/<encoded-path>/memory/`) records non-obvious decisions made during skill development — migration status, depersonalization rationale, decorator-vs-interceptor stance for cross-cutting concerns. Read its `MEMORY.md` first when picking up work in this repo to understand prior context before changing skills.

## Git identity

This repo's `git/config` has `--local` overrides for `user.name` and `user.email` to keep its commit identity separate from any other repo on the same machine. Don't change them without an explicit ask — they're set deliberately.
